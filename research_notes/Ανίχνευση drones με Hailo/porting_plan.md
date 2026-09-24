# Porting plan: Luckfox Pico (RV1106) UAV detector to a Hailo system (Pi 5 + AI HAT+/AI Kit, or x86 + Hailo M.2)

Research date: 2026-09-24. Facts carry an inline source. "Inferences" are engineering recommendations, not verified facts. Some primary sites (ardupilot.org, mavlink.io, mediamtx.org, community.hailo.ai, wiki.seeedstudio.com) were blocked by the network proxy, so the same content was read from their GitHub source repos (ArduPilot/ardupilot_wiki, ArduPilot/ardupilot, mavlink/mavlink, mavlink/mavlink-devguide, bluenviron/mediamtx, raspberrypi/documentation, hailo-ai/*).

## Q1. What does the current repo pipeline do, and what maps 1:1 vs. must be rewritten?

### Takeaway
The current app is one C++ loop tied to Rockchip hardware: RKMPI VI capture 720x480, then RGA NV12-to-RGB letterbox to 512x512, then RKNN YOLOv5 (1 class, anchor-based, int8 post-process on CPU), then RGA box drawing, then RKMPI VENC H.264, then a bundled `rtsp_demo` server on :554 `/live/0`. It also has an optional hand-built MAVLink v2 packet over UART3 and a batched binary log in `/userdata`. Only the app logic ports as-is: thresholds, the logging module, and the idea of the MAVLink payload. Capture, preprocessing, inference, encoding and RTSP are all Rockchip-specific and must be replaced.

### Cited Findings (local repo, read directly)
- Resolution and model: `DISP_WIDTH 720`, `DISP_HEIGHT 480`, `MODEL_SIZE 512`, `BOX_CONF_THRESHOLD 0.45f` — `luckfox_pico_rtsp_yolov5_UAV/src/main.cc` lines 29–56.
- Models present are only `model/yolov5.rknn` and `model/yolov5_512.rknn` (~2 MB each), plus `anchors_yolov5.txt` and a 1-line label file (`coco_1_label_list.txt`). No `.pt` or `.onnx` is in the repo — `luckfox_pico_rtsp_yolov5_UAV/model/`.
- Post-processing is YOLOv5 anchor-based decoding with an int8 affine-quantised threshold, then CPU NMS (`qnt_f32_to_affine`, `OBJ_CLASS_NUM`, IoU NMS) — `src/postprocess.cc` lines 114–142, 204–291, 381–410.
- RTSP uses the Rockchip `rtsp_demo` library (`create_rtsp_demo(554)`, session `/live/0`, H.264), fed by `RK_MPI_VENC_GetStream` — `src/main.cc` lines 142–147, 297–319.
- **Bug (confirmed):** UART init is inside `#if PRINT_UART` (line 159), but it is used inside `#if PRINT_ON_UART` (lines 269, 353), and only `PRINT_ON_UART` is `#define`d (line 48, `false`). If someone sets `PRINT_ON_UART true`, `serial_fd` is never declared or initialised, so the build fails. Both guards need the same macro — `src/main.cc`.
- **MAVLink non-compliance (confirmed):** the custom packet uses msgid 9000 and computes the checksum over header+payload with CRC-16/MCRF4XX, but it never adds the message's `CRC_EXTRA` byte (`crc_calculate(buffer + 1, 9 + payload_len)`) — `src/mavlink_comm.cc` lines 8–9, 17–26, 86–91. The MAVLink v2 spec says the checksum "Includes CRC_EXTRA byte", which exists so sender and receiver can check they share the same message definition — [MAVLink serialization guide](https://github.com/mavlink/mavlink-devguide/blob/master/en/guide/serialization.md). So standard parsers (pymavlink, MAVSDK, ArduPilot, mavlink-router with dialect checks) will reject or ignore this packet as it stands, even with an XML dialect that defines msg 9000.
- Custom payload fields: `time_usec, x, y (normalized -1..1, centre 0), width, height (0..1), confidence, target_num, class_id` — `include/mavlink_comm.h`.
- Log record is 40 bytes: `timestamp_us, x, y, w, h (int32 px), confidence, frame_w/h, class_id, target_num`. Defaults: batch of 32 records, 1 MiB files, 8-file ring, min confidence 0.5, decimate every 5th frame, directory `/userdata/uav_detections`, file magic `0x55415644` — `include/uav_detection_log.h` lines 37–75, `include/flash_storage.h` lines 53–65.

### Inferences (port map)
| Current (RV1106) | Hailo target | Port type |
|---|---|---|
| RKMPI VI (SC3336 MIPI) | Pi camera via Picamera2/libcamera (hailo-apps `--input rpi`), USB `/dev/videoN`, or `rtsp://` source | Rewrite (use hailo-apps SOURCE_PIPELINE) |
| RGA NV12→RGB letterbox | GStreamer `videoscale/videoconvert` in hailo-apps pipelines, or HEF compiled with NV12/RGBX input (Model Zoo lists NV12/RGBX HEF variants) | Rewrite / delete |
| RKNN YOLOv5 + custom C post-process | HEF + `hailonet` + `hailofilter` (or HailoRT's built-in NMS output) | Retrain + recompile. The C post-process can be dropped. |
| RGA draw_box | `hailooverlay` | Delete |
| RKMPI VENC H.264 + rtsp_demo | `x264enc` (software) → mediamtx or gst-rtsp-server | Rewrite |
| mavlink_comm.cc (hand-rolled) | pymavlink / MAVLink C headers generated from `common.xml`, standard messages | Rewrite (small) |
| flash_storage / uav_detection_log | Port as-is (plain POSIX C) or replace with Python SQLite/binary writer | 1:1 possible |
| read_detections + PicoClaw log analysis | Unchanged if the binary format is kept | 1:1 |

### Gaps
- I did not verify the exact on-board FPS or latency of the current RV1106 build. The README claims 30 FPS and ~25 ms.

## Q2. Model: can the RKNN be converted back, and how to produce a Hailo HEF?

### Takeaway
Don't expect to reverse the `.rknn`. The path is to retrain (or recover the original `.pt`), export ONNX, and compile with Hailo's Dataflow Compiler (DFC) through the Model Zoo `hailomz compile` flow, using target-domain calibration images and `--classes 1`. YOLOv8n or YOLO11n is the natural choice because it is in Hailo's zoo and used by hailo-apps. YOLOv5n itself has no detection config in the zoo (only YOLOv5s/m and yolov5n_seg).

### Cited Findings
- Hailo's official retraining flow: train with Ultralytics, then `model.export(format='onnx', opset=11)`, then `hailomz compile --ckpt best.onnx --calib-path <valid images> --yaml yolov8s.yaml --classes 2 --hw-arch <arch> --performance`. The DFC and Model Zoo wheels come from the Hailo Developer Zone. The guide recommends using the validation set as calibration data — [hailo-apps retraining_example.md](https://github.com/hailo-ai/hailo-apps/blob/main/doc/developer_guide/retraining_example.md).
- Without a GPU, "optimization level [is reduced] to 0 ... not recommended for production" — same source.
- "Hailo conversion adds a background class at index 0, shifting all class IDs". Custom labels are passed with `--labels-json`, and a custom HEF with `--hef-path` — same source.
- Hailo also has a YOLOv5 retraining docker in `hailo_model_zoo/training/yolov5` (their fork), with the same `hailomz compile ... --classes N` step — [Model Zoo YOLOv5 retraining doc](https://github.com/hailo-ai/hailo_model_zoo/blob/master/training/yolov5/README.rst) (contents read from repo docs).
- **Software-line split:** "The Hailo-8 and Hailo-8L devices are supported on the Hailo Model Zoo v2.x branch, in combination with the Hailo Dataflow Compiler v3.x branch. The master branch is intended for Hailo-10 and Hailo-15" — [hailo_model_zoo README](https://github.com/hailo-ai/hailo_model_zoo). The master branch is at DFC/HailoRT v5.4.0 (same README). The latest Hailo-8 branch seen is `update-to-version-v2.19.1`.
- Model Zoo network configs include `yolov8n.yaml`, `yolov11n.yaml`, `yolov5s.yaml`, `yolov5n_seg.yaml`. There is no plain `yolov5n.yaml` — [hailo_model_zoo cfg/networks](https://github.com/hailo-ai/hailo_model_zoo/tree/master/hailo_model_zoo/cfg/networks).
- NMS placement: in the v2.19.1 (Hailo-8) branch, `yolov8n.alls` ends with `nms_postprocess("../../postprocess_config/yolov8n_nms_config.json", meta_arch=yolov8, engine=cpu)`, and `yolov5s.alls` uses `nms_postprocess(..., yolov5, engine=cpu)`. The YAML sets `device_pre_post_layers: nms: true` and `hpp: true` — [yolov8n.alls v2.19.1](https://github.com/hailo-ai/hailo_model_zoo/blob/update-to-version-v2.19.1/hailo_model_zoo/cfg/alls/generic/yolov8n.alls), [yolov5s.alls](https://github.com/hailo-ai/hailo_model_zoo/blob/update-to-version-v2.19.1/hailo_model_zoo/cfg/alls/generic/yolov5s.alls). So box decoding and NMS are packaged in the HEF but run on the host through HailoRT (`engine=cpu`), not on the NN core. That is still a single inference call, with output shaped `classes x 5 x max_boxes` (`output_shape: 80x5x100` for COCO yolov8n).
- The Model Zoo publishes HEF variants with NV12 and RGBX input formats as well as RGB — [HAILO8 object detection table legend](https://github.com/hailo-ai/hailo_model_zoo/blob/master/docs/public_models/HAILO8/HAILO8_object_detection.rst).

### Inferences
- First check whether the original training run (`best.pt`) exists anywhere, for example with the author or in a Luckfox/rknn_model_zoo conversion workspace. If it does, export ONNX from it directly. The RKNN file holds int8 weights that are already quantised and hardware-specific, and there is no supported RKNN→ONNX path. Treat retraining as the default plan.
- Recommended model: YOLOv8n (or YOLO11n) trained at 640 (or 512 to match the current setup) on a UAV dataset plus your own SC3336/Pi-camera frames. Use `--classes 1`, then map Hailo class id 1 to "UAV".
- Use 500–1500 calibration images from the real camera and the real sky/background conditions: blue sky, cloud, sun glare, trees, dusk. Int8 calibration on generic COCO-style images tends to lose small-object recall.
- Compile on an x86 Ubuntu machine with an NVIDIA GPU so you get optimization level >0. Use the v2.x Model Zoo and DFC 3.x for Hailo-8/8L, or the master line (v5.x) for Hailo-10H (AI HAT+ 2).
- Ultralytics YOLOv8/11 is AGPL-3.0 (`license_name: AGPL-3.0` in the zoo YAML). Check this if the project will be distributed.

### Gaps
- I did not verify whether the Hailo-8 DFC can place NMS on the NN core for YOLOv8 (older docs mention `engine=nn_core` for some YOLOv5 configs). Current zoo configs use `engine=cpu`.
- I did not fetch exact DFC version numbers for the Hailo-8 line (the Developer Zone needs a login).

## Q3. Runtime pipeline on Pi 5 + Hailo: capture, inference, encode, RTSP

### Takeaway
Start from `hailo-apps` (the current official repo; `hailo-rpi5-examples` is marked outdated). Use its detection pipeline with a custom HEF and labels, and add a `hailotracker` stage. It is built on GStreamer (`hailonet`, `hailofilter`, `hailooverlay`, `hailotracker`) and supports rpi-camera (Picamera2), USB, file and RTSP inputs. The Pi 5 has **no hardware H.264 encoder**, so RTSP output must use software x264 (`x264enc tune=zerolatency speed-preset=ultrafast`) and be served by mediamtx or gst-rtsp-server. 720p/1080p at 30 fps is feasible on the CPU, but latency is higher than on the RV1106 VENC.

### Cited Findings
- hailo-rpi5-examples README: "This repository is outdated. More up-to-date code examples are available here: https://github.com/hailo-ai/hailo-apps" — [hailo-rpi5-examples](https://github.com/hailo-ai/hailo-rpi5-examples).
- hailo-apps supports Hailo-8, Hailo-8L, Hailo-10H on Raspberry Pi 5, Ubuntu x86_64 and Windows. Its pipeline apps include `detection`, `detection_simple`, `tiling`, `multisource`, `reid_multisource`, and others — [hailo-apps](https://github.com/hailo-ai/hailo-apps) (latest release seen: 26.03.1, commit April 2026, in `changelog.md`).
- Source handling in `gstreamer_helper_pipelines.py`: `/dev/video*` → usb, `rpi` → Picamera2 (appsrc), `libcamera` → `libcamerasrc` ("not suggested"), `rtsp://` → `rtspsrc`, `udp://`, image files, and video files — [gstreamer_helper_pipelines.py](https://github.com/hailo-ai/hailo-apps/blob/main/hailo_apps/python/core/gstreamer/gstreamer_helper_pipelines.py).
- The same helper file already contains an encoder helper using `x264enc tune=zerolatency bitrate=... speed-preset=ultrafast` after `videoconvert ! video/x-raw,format=I420` — same source.
- Raspberry Pi docs: "Raspberry Pi 5 uses software video encoders. These generally output frames with a longer latency than the old hardware encoders"; `--low-latency` helps, and it "will still easily achieve 1080p30" — [raspberrypi/documentation rpicam_vid.adoc](https://github.com/raspberrypi/documentation/blob/develop/documentation/asciidoc/computers/camera/rpicam_vid.adoc).
- Forum/press confirmation: "There is no hardware H264 or H265 encoder on Pi5" — [Raspberry Pi Forums t=376952](https://forums.raspberrypi.com/viewtopic.php?t=376952); [XDA](https://www.xda-developers.com/modern-raspberry-pi-surprisingly-powerful-transcoder/) (secondary).
- mediamtx supports Pi cameras natively on Raspberry Pi OS Trixie/Bookworm. `rpiCameraCodec: auto` picks "hardwareH264 when ... hardware encoder is available, softwareH264 when ... not available" — [mediamtx docs 14-raspberry-pi-cameras.md](https://github.com/bluenviron/mediamtx/blob/main/docs/3-publish/14-raspberry-pi-cameras.md), [mediamtx.yml](https://github.com/bluenviron/mediamtx/blob/main/mediamtx.yml). mediamtx also accepts publishing from GStreamer or FFmpeg over RTSP/RTMP/SRT/WebRTC — [mediamtx README](https://github.com/bluenviron/mediamtx).
- The Pi 5 PCIe connector is "PCIe Gen 2.0 ×1" by default. Gen 3 is enabled with `dtparam=pciex1_gen=3` or raspi-config, with the warning "Raspberry Pi 5 isn't certified for Gen 3.0 speeds" — [raspberrypi/documentation pcie](https://github.com/raspberrypi/documentation/blob/develop/documentation/asciidoc/computers/raspberry-pi/pcie.adoc) (read from repo).
- AI HAT+ comes in 13 TOPS (Hailo-8L) and 26 TOPS (Hailo-8) versions. AI HAT+ 2 is 40 TOPS (Hailo-10H). The AI Kit (Hailo-8L) "is no longer in production" — [Raspberry Pi AI HAT+ docs](https://github.com/raspberrypi/documentation/blob/develop/documentation/asciidoc/accessories/ai-hat-plus/about.adoc) (read from repo).

### Inferences
- Recommended pipeline (Pi 5):
  `Picamera2 (1280x720@30 or 1536x864) → [hailo-apps SOURCE] → tee ─┬─ hailocropper/videoscale → hailonet(HEF) → hailofilter → hailotracker → identity(callback: MAVLink + log) → hailooverlay → videoconvert → x264enc tune=zerolatency speed-preset=ultrafast key-int-max=30 → rtspclientsink location=rtsp://127.0.0.1:8554/live` (mediamtx serves it).
  Do **not** let mediamtx own the camera (`source: rpiCamera`), because Hailo also needs the frames. Publish the annotated stream into mediamtx instead.
- Keep the RTSP stream resolution modest (720x480 or 1280x720, 2–4 Mbps) so x264 uses about 1 core. Detection resolution is independent of the stream resolution.
- Write the callback in Python (the hailo-apps `app_callback` pattern) for MAVLink and logging. At ≤30 Hz with a few boxes, Python overhead is negligible.
- On x86 + Hailo M.2 you can use VA-API/NVENC (`vaapih264enc`, `nvh264enc`) for hardware encoding. The pipeline is otherwise the same.
- For the best latency and global-shutter behaviour on fast drones, consider the Pi Global Shutter camera or Camera Module 3 (IMX708) with a fixed short exposure. This is a recommendation; no benchmark was found.

### Gaps
- I found no published glass-to-glass latency figure for Pi 5 + Hailo + x264 RTSP.
- I did not verify the exact `rtspclientsink` element availability in the default Pi OS GStreamer packages (it is in `gstreamer1.0-rtsp`; check on the device).

## Q4. MAVLink integration with standard messages

### Takeaway
Replace msg 9000 with standard `common.xml` messages generated by an official generator (pymavlink or the MAVLink C headers), so CRC_EXTRA and field ordering are correct. There is **no standard common message for "detected foreign aircraft in camera"** that ArduPilot acts on. Choose by purpose:
- **Logging/telemetry only:** `NAMED_VALUE_FLOAT`/`DEBUG_VECT`. ArduPilot logs `NAMED_VALUE_FLOAT` as NVAL.
- **Autopilot reaction:** `CAMERA_TRACKING_IMAGE_STATUS` (bbox, normalized, as a MAVLink camera component), or custom Lua handling of any message.
- **Avoidance:** `OBSTACLE_DISTANCE` (only if you estimate range/bearing).
- **Not recommended:** `LANDING_TARGET`, which drives precision landing and is semantically wrong for "detect another drone".

Wire the Pi 5 GPIO14/15 (`/dev/ttyAMA0`, 3.3 V) to TELEM2 with `SERIAL2_PROTOCOL=2` and `SERIAL2_BAUD=921`. Run mavlink-router if a GCS/QGC also needs the link.

### Cited Findings
- `LANDING_TARGET` (id 149): `time_usec, target_num, frame, angle_x, angle_y (rad), distance, size_x, size_y`, plus extensions `x,y,z,q,type,position_valid` — [MAVLink common.xml](https://github.com/mavlink/mavlink/blob/master/message_definitions/v1.0/common.xml).
- ArduPilot uses `LANDING_TARGET` for precision landing from a companion computer: set `PLND_TYPE` = 1 (MAVLink `LANDING_TARGET`) — [ArduPilot wiki precision-landing-and-loiter.rst](https://github.com/ArduPilot/ardupilot_wiki/blob/master/copter/source/docs/precision-landing-and-loiter.rst). The landing-target service says "ArduPilot supports messages with these fields if position_valid is 0", and for position form ArduPilot supports only `MAV_FRAME_BODY_FRD` with `distance` filled — [MAVLink landing_target service](https://github.com/mavlink/mavlink-devguide/blob/master/en/services/landing_target.md).
- `CAMERA_TRACKING_IMAGE_STATUS` (id 275): `tracking_status, tracking_mode, target_data, point_x, point_y, radius, rec_top_x, rec_top_y, rec_bottom_x, rec_bottom_y` (normalized 0..1), plus `camera_device_id` — [common.xml](https://github.com/mavlink/mavlink/blob/master/message_definitions/v1.0/common.xml). In ArduPilot master, this message is routed to `AP_Camera::handle_message` only when `AP_CAMERA_MAVLINKCAMV2_ENABLED` — [GCS_Common.cpp](https://github.com/ArduPilot/ardupilot/blob/master/libraries/GCS_MAVLink/GCS_Common.cpp). ArduPilot's AP_Camera handles `MAV_CMD_CAMERA_TRACK_POINT/RECTANGLE/STOP_TRACKING` (GCS→camera direction) — [AP_Camera.cpp](https://github.com/ArduPilot/ardupilot/blob/master/libraries/AP_Camera/AP_Camera.cpp).
- `OBSTACLE_DISTANCE` (id 330): 72×uint16 distances (cm), `increment`/`increment_f`, `angle_offset`, `frame`, `min/max_distance`. ArduPilot handles it in its proximity library when `HAL_PROXIMITY_ENABLED` — [common.xml](https://github.com/mavlink/mavlink/blob/master/message_definitions/v1.0/common.xml), [GCS_Common.cpp](https://github.com/ArduPilot/ardupilot/blob/master/libraries/GCS_MAVLink/GCS_Common.cpp).
- ArduPilot's `handle_named_value` writes received `NAMED_VALUE_FLOAT` into the dataflash log as `NVAL` (`TimeUS,TimeBootMS,Name,Value,SSys,SCom`) — [GCS_Common.cpp](https://github.com/ArduPilot/ardupilot/blob/master/libraries/GCS_MAVLink/GCS_Common.cpp).
- ArduPilot Lua scripting can receive arbitrary MAVLink messages with `mavlink:register_rx_msgid(msg_id)` and `mavlink:receive_chan()`, and send with `mavlink:send_chan()` — [AP_Scripting docs.lua](https://github.com/ArduPilot/ardupilot/blob/master/libraries/AP_Scripting/docs/docs.lua).
- ArduPilot also ingests `ADSB_VEHICLE` (under `HAL_ADSB_ENABLED`) — [GCS_Common.cpp](https://github.com/ArduPilot/ardupilot/blob/master/libraries/GCS_MAVLink/GCS_Common.cpp).
- Companion wiring (ArduPilot dev wiki): connect TELEM2 GND/TX/RX to the Pi. Set `SERIAL2_PROTOCOL = 2` (MAVLink 2) and `SERIAL2_BAUD = 921` (921600). Disable the serial login shell and enable serial hardware in raspi-config. Use mavlink-router (`/etc/mavlink-router/main.conf`, Device and Baud) or MAVProxy; "If the Raspberry PI is heavily loaded, mavproxy.py might not provide a reliable connection" — [ArduPilot wiki raspberry-pi-via-mavlink.rst](https://github.com/ArduPilot/ardupilot_wiki/blob/master/dev/source/docs/raspberry-pi-via-mavlink.rst).
- **Pi 5 UART caveat:** on Pi 5, `/dev/serial0` (primary UART) points to `/dev/ttyAMA10`, the 3-pin debug header, **not** GPIO14/15. GPIO14/15 is UART0 = `/dev/ttyAMA0`. All Pi 5 UARTs are "PL011 (disabled by default)", and if `enable_uart=1` with no debug cable, kernel logging goes to GPIO14/15 — [raspberrypi/documentation interfaces.adoc](https://github.com/raspberrypi/documentation/blob/develop/documentation/asciidoc/computers/configuration/interfaces.adoc). So the ArduPilot wiki's `/dev/serial0` advice (written for older Pis) does not directly apply to Pi 5.

### Inferences (recommended design)
1. Use pymavlink (`mavutil.mavlink_connection('/dev/ttyAMA0', baud=921600, source_system=1, source_component=MAV_COMP_ID_ONBOARD_COMPUTER (191) or MAV_COMP_ID_CAMERA (100))`). Or connect to mavlink-router's local UDP endpoint so the Pi, a GCS over Wi-Fi and the FC share one link. Send `HEARTBEAT` at 1 Hz (type `MAV_TYPE_ONBOARD_CONTROLLER` or `MAV_TYPE_CAMERA`).
2. Per detection (or only the best track): send `CAMERA_TRACKING_IMAGE_STATUS` with `tracking_mode=RECTANGLE` and normalized `rec_top/bottom`. This carries exactly the bbox the old custom message carried, in a standard form that QGC and gimbal/camera stacks understand.
3. For logging and GCS graphs: send `NAMED_VALUE_FLOAT` values `"uav_conf"`, `"uav_ax"`, `"uav_ay"` (ArduPilot writes them to NVAL, time-aligned with the flight log), or `DEBUG_VECT` (x=angle_x, y=angle_y, z=conf).
4. To make an ArduPilot vehicle *react* (yaw toward target, gimbal ROI, or avoidance): write a small Lua script that receives the message above and commands the mount or yaw. Alternatively, convert to `OBSTACLE_DISTANCE` sectors if you have a range estimate.
5. If a custom message is still preferred: define it in an XML dialect (msg id in the free/custom range), generate code with `mavgen`, and add the dialect to the GCS/receiver. Never hand-roll it without CRC_EXTRA.
6. Convert pixel coordinates to angles with the camera intrinsics: `angle_x = atan((cx_px - cx0)/fx)`. The old normalized −1..1 value is not an angle.
7. Wiring: Pi GPIO14 (TXD, pin 8) → FC TELEM RX, GPIO15 (RXD, pin 10) → FC TELEM TX, GND→GND. Both sides are 3.3 V TTL. Don't connect the FC 5 V pin to the Pi when the Pi is powered separately. Add `dtparam=uart0=on` (or `enable_uart=1`) and remove the serial console from `cmdline.txt`.
8. PX4: PX4 accepts `LANDING_TARGET` for precision landing (position form in `MAV_FRAME_LOCAL_NED`, per the MAVLink service doc cited above). For object reporting, use `CAMERA_TRACKING_IMAGE_STATUS` or MAVSDK's camera/tracking plugins. I did not verify PX4 consumption.

### Gaps
- I could not open ardupilot.org or mavlink.io directly (blocked). I used their GitHub sources, which may differ slightly from the rendered sites.
- I did not verify whether ArduPilot uses `CAMERA_TRACKING_IMAGE_STATUS` from a *companion* for vehicle control, as opposed to forwarding and handling it for MAVLink camera v2. The code path above shows it only reaches AP_Camera.
- I did not verify PX4's handling of `CAMERA_TRACKING_IMAGE_STATUS` or `OBSTACLE_DISTANCE`.

## Q5. Logging equivalents and SD-card wear

### Takeaway
`flash_storage.cc` and `uav_detection_log.cc` are plain POSIX C with fixed 40-byte records, batching, and file rotation. They compile unchanged on Pi OS; only change the directory (e.g. `/var/lib/uav_detections` or a USB SSD). Alternatively, reimplement them in Python with `struct.pack` for the same format (so `read_detections` and PicoClaw keep working), or use SQLite in WAL mode with batched transactions.

### Cited Findings
- Current design: batch 32 records in RAM, 1 MiB files, 8-file ring, confidence ≥0.5, every 5th frame — `include/uav_detection_log.h`, `include/flash_storage.h` (local repo).

### Inferences
- Keep the batch-and-rotate design, because SD cards wear the same way the RV1106 SPI-NAND/eMMC does. Add `fsync` only on rotation or shutdown, mount with `noatime`, and consider log2ram/tmpfs with periodic flush, or an NVMe/USB SSD. Note that on a Pi 5 the PCIe x1 port is taken by the AI HAT+, unless you use a PCIe switch HAT.
- Useful new fields: `track_id` (from hailotracker), angle_x/angle_y, and a GPS/attitude snapshot from MAVLink (`GLOBAL_POSITION_INT`, `ATTITUDE`). This needs a new record version (bump `FLASH_STORAGE_VERSION`).
- SQLite is easier to query (PicoClaw could run SQL) but gives up the fixed binary format. Use `PRAGMA journal_mode=WAL` and one transaction per batch.

### Gaps
- I found no primary-source quantitative data on SD-card endurance under this write pattern. The wear advice above is engineering judgement.

## Q6. Tracking and small-object improvements available on Hailo

### Takeaway
hailo-apps offers two ready-made improvements. The first is the `hailotracker` GStreamer element (Kalman + IoU, JDE-style), exposed as `TRACKER_PIPELINE(class_id=...)` and already used in the detection pipeline. It gives stable track IDs, which is better input for MAVLink and logs than per-frame boxes. The second is the `tiling` pipeline app, whose default demo is aerial/drone small-object detection with a VisDrone YOLOv8n and optional multi-scale tiling. There is also a Python ByteTrack (`BYTETracker`) in the standalone (non-GStreamer) object-detection app.

### Cited Findings
- `TRACKER_PIPELINE(class_id, kalman_dist_thr=0.8, iou_thr=0.9, init_iou_thr=0.7, keep_new_frames=2, keep_tracked_frames=15, keep_lost_frames=2, ...)` "Creates a GStreamer pipeline string for the HailoTracker element" — [gstreamer_helper_pipelines.py](https://github.com/hailo-ai/hailo-apps/blob/main/hailo_apps/python/core/gstreamer/gstreamer_helper_pipelines.py). The detection pipeline uses `TRACKER_PIPELINE(class_id=1)` — [detection_pipeline.py](https://github.com/hailo-ai/hailo-apps/blob/main/hailo_apps/python/pipeline_apps/detection/detection_pipeline.py).
- The standalone object-detection app imports `hailo_apps.python.core.tracker.byte_tracker.BYTETracker` ("ByteTrack tracker instance") — [object_detection.py](https://github.com/hailo-ai/hailo-apps/tree/main/hailo_apps/python/standalone_apps/object_detection).
- Tiling app: "splitting each frame into several tiles which are processed independently by the hailonet element ... especially effective for detecting small objects". Its default model is `hailo_yolov8n_4_classes_vga` "optimized for aerial object detection" with a VisDrone video. It supports `--multi-scale`, `--tiles-x/--tiles-y`, `--hef`, `--input rpi`, and the warning "It's very easy to reach very high FPS requirements by using too many tiles" — [tiling README](https://github.com/hailo-ai/hailo-apps/blob/main/hailo_apps/python/pipeline_apps/tiling/README.md).

### Inferences
- A distant drone is a small object. Tiling a 1280x720 or 1920x1080 frame into 2×2 or 3×2 tiles of 640 input multiplies the NPU load by 4–6x. On Hailo-8 (hundreds of FPS for yolov8n) this is affordable at 30 fps. On Hailo-8L over PCIe x1, expect to trade off with the frame rate.
- Tracker usage: send MAVLink and log only for confirmed tracks (for example ≥3 consecutive hits), and use the track ID as `target_num`. This cuts false alarms from birds or clouds far more cheaply than raising the confidence threshold.
- Set `class_id=1` in the tracker because of Hailo's background-class shift.

### Gaps
- I did not benchmark tiling FPS on Pi 5 for a single-class drone HEF.

## Q7. Published performance numbers

### Takeaway
Hailo's official Model Zoo numbers are measured on an x86 host over PCIe Gen3 x4. Examples: YOLOv8n (640) on Hailo-8 ~1036 FPS, on Hailo-8L ~202 FPS at batch 1. The Pi 5 has a single PCIe lane (Gen2 by default, Gen3 optional), so real Pi numbers are lower. Community reports put yolov8n on Pi 5 + Hailo-8 in the hundreds of FPS at batch 1, while full camera pipelines are typically limited to camera rate (30–60 fps). Either accelerator comfortably beats the RV1106's 30 FPS at 512 input; the bottleneck on Pi 5 moves to capture, software encoding, and Python.

### Cited Findings
- Hailo-8 Model Zoo (host i5-9400, "PCIe Gen 3 x 4 lanes"). Columns: float mAP / HW mAP / FPS batch1 / FPS batch8 at 640x640:
  - yolov8n 37.0/36.4/1036/1036
  - yolov11n 39.0/37.5/185/541
  - yolov5s 35.3/34.0/543/543
  - yolov8s 44.6/43.9/491/491
  - Source: [HAILO8_object_detection.rst](https://github.com/hailo-ai/hailo_model_zoo/blob/master/docs/public_models/HAILO8/HAILO8_object_detection.rst)
- Hailo-8L Model Zoo, same conditions:
  - yolov8n 37.0/36.4/202/438
  - yolov11n 39.0/37.5/157/371
  - yolov5s 35.3/34.0/124/243
  - yolov8s 44.6/43.9/110/208
  - Source: [HAILO8L_object_detection.rst](https://github.com/hailo-ai/hailo_model_zoo/blob/master/docs/public_models/HAILO8L/HAILO8L_object_detection.rst)
- Pi 5 PCIe is Gen2 x1 by default, and Gen3 is optional and uncertified — [Raspberry Pi PCIe docs](https://github.com/raspberrypi/documentation/blob/develop/documentation/asciidoc/computers/raspberry-pi/pcie.adoc).
- Community threads report that Pi 5 users cannot reach the Model Zoo FPS with Hailo-8, which they attribute to PCIe x1. The search summary cites ~431 FPS for yolov8n batch 1 on Pi 5 + Hailo-8, and says Gen3 roughly doubles FPS vs Gen2 — [Hailo Community: official FPS benchmark on Hailo-8 using RPi5](https://community.hailo.ai/t/official-fps-benchmark-on-hailo-8-using-raspberry-pi-5/18873), [RPi Forums t=396633](https://forums.raspberrypi.com/viewtopic.php?t=396633), [Seeed wiki benchmark](https://wiki.seeedstudio.com/benchmark_on_rpi5_and_cm4_running_yolov8s_with_rpi_ai_kit/). **Caveat:** I could not open these pages directly (proxy blocked). The 431 FPS figure comes from the search-engine summary only and should be re-verified.

### Inferences
- For a single-class nano model at 640, even the Hailo-8L gives large headroom over 30 fps. Pick Hailo-8 (26 TOPS) if you plan tiling or a larger model (yolov8s) for longer detection range. Pick Hailo-10H (AI HAT+ 2) only if you also want on-device LLM/VLM (it replaces the PicoClaw-over-cloud idea), keeping in mind it uses the separate v5.x toolchain.
- Always enable `dtparam=pciex1_gen=3` and verify stability; run `hailortcli benchmark` / `hailortcli run` on the actual HEF on the Pi.

### Gaps
- There is no official Hailo/Raspberry Pi table of Pi 5-specific FPS per model. Community numbers are unverified here.
- I found no published numbers for yolov5n on Hailo (it is not in the zoo detection table).

---

### Concrete step-by-step porting plan (engineering recommendation, synthesised from the findings above)
1. **Fix the repo first:** unify the `PRINT_UART`/`PRINT_ON_UART` guard, and replace the hand-rolled MAVLink with generated headers (adds CRC_EXTRA). This makes the Luckfox build a valid reference.
2. **Data and model:** collect or label UAV frames (existing training data plus Pi-camera footage). Train YOLOv8n/YOLO11n (1 class, 640), export ONNX opset 11, validate mAP on a held-out drone set, and compare against the RKNN model on the same clips.
3. **Compile:** on an x86 GPU box, install the DFC + Model Zoo matching the chip (v2.x/DFC 3.x for 8/8L, v5.x for 10H). Run `hailomz compile --ckpt best.onnx --yaml yolov8n.yaml --classes 1 --calib-path <1k target images> --hw-arch hailo8|hailo8l`. Check with `hailortcli run` on the device.
4. **Bring-up:** Pi OS (Bookworm/Trixie) + `hailo-all` / hailo-apps `install.sh`. Run `hailo-detect --hef-path uav.hef --labels-json uav.json --input rpi`.
5. **App:** copy the detection pipeline app, then add the tracker, a callback (MAVLink via pymavlink or mavlink-router, and logging), and an RTSP output branch (x264enc → mediamtx).
6. **FC integration:** TELEM2 ↔ GPIO14/15 (`/dev/ttyAMA0`), `SERIAL2_PROTOCOL=2`, `SERIAL2_BAUD=921`. Send HEARTBEAT + CAMERA_TRACKING_IMAGE_STATUS + NAMED_VALUE_FLOAT, verify in the MAVLink Inspector (Mission Planner/QGC) and in NVAL log entries, and optionally add a Lua reaction script.
7. **Small targets:** evaluate the hailo-apps tiling pipeline with the custom HEF, and pick the tile grid by FPS budget.
8. **Logging:** compile the existing C logger unchanged or port it to Python with the same struct. Add track_id and a new version number.
