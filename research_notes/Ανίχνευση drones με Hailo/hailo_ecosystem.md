# Hailo AI Accelerator Ecosystem for Real-Time YOLO Detection on Drones/Robots (as of Sept 2026)

Method note: hailo.ai, raspberrypi.com, community.hailo.ai, jeffgeerling.com, seeedstudio wiki and theregister.com were blocked by the research sandbox's egress proxy. Facts from those domains come only from search-engine snippets and are labelled **[snippet]**. Facts labelled **[verified]** were read directly from the primary source, mostly GitHub raw files: the hailo-ai repos, the raspberrypi/documentation repo, and third-party READMEs. Vendor-reported numbers are marked **(vendor)**. Numbers measured by third parties are marked **(independent)**.

## 1. Specs per chip/module (TOPS, power, form factor, memory, price, temperature)

### Takeaway
Hailo-8 has 26 TOPS INT8 at about 2.5 W typical. Hailo-8L has 13 TOPS. Hailo-10H has 40 TOPS INT4 (about 20 TOPS INT8) and its own 4–8 GB LPDDR4(X), which it needs for GenAI. Hailo-15 is a camera SoC with 7–20 TOPS, not a PCIe accelerator. For Raspberry Pi the options are the AI HAT+ (13 TOPS for about $70, 26 TOPS for about $110) and the AI HAT+ 2 (Hailo-10H, 8 GB, about $130). The M.2 modules come in industrial-temperature variants (−40 to +85 °C).

### Cited Findings
- **Raspberry Pi AI HAT+**: Hailo-8L (13 TOPS, INT8) or Hailo-8 (26 TOPS, INT8). It uses the Pi 5's own memory and has no LLM/VLM support. Use cases listed: object detection, camera post-processing, robotics. Board is about 66 × 56.5 mm and the NPU about 17 × 17 mm. **[verified]** — [raspberrypi/documentation ai-hat-plus/about.adoc](https://github.com/raspberrypi/documentation/blob/master/documentation/asciidoc/accessories/ai-hat-plus/about.adoc)
- **Raspberry Pi AI HAT+ 2**: Hailo-10H (40 TOPS, INT4) with its own 8 GB onboard memory, running "LLMs and VLMs up to ~6 billion parameters". Same 66 × 56.5 mm footprint. It ships with an extra heatsink, which Raspberry Pi recommends "especially if you're running benchmarks or intensive AI workloads" to avoid throttling. **[verified]** — [same doc](https://github.com/raspberrypi/documentation/blob/master/documentation/asciidoc/accessories/ai-hat-plus/about.adoc)
- **AI Kit** = M.2 HAT+ plus a Hailo-8L module in **M.2 2242** form factor (13 TOPS). It is "no longer in production" and "functionally equivalent" to the 13 TOPS AI HAT+. **[verified]** — [ai-kit/about.adoc](https://github.com/raspberrypi/documentation/blob/master/documentation/asciidoc/accessories/ai-kit/about.adoc)
- On a Pi, `hailortcli fw-control identify` on an AI Kit reports "HAILO-8L AI ACC M.2 B+M KEY MODULE EXT TMP", i.e. a B+M key, extended-temperature module. **[verified]** — [computers/ai/getting-started.adoc](https://github.com/raspberrypi/documentation/blob/master/documentation/asciidoc/computers/ai/getting-started.adoc)
- Prices: AI HAT+ 13 TOPS $70, 26 TOPS $110 **[snippet]** — [Raspberry Pi news: AI HAT+](https://www.raspberrypi.com/news/raspberry-pi-ai-hat/). AI HAT+ 2 $130 **[snippet]** — [Jeff Geerling](https://www.jeffgeerling.com/blog/2026/raspberry-pi-ai-hat-2/), [RPi news AI HAT+ 2](https://www.raspberrypi.com/news/introducing-the-raspberry-pi-ai-hat-plus-2-generative-ai-on-raspberry-pi-5/). The EU retailer price for the AI HAT+ 2 was €208 **[snippet]** — [welectron](https://www.welectron.com/Official-Raspberry-Pi-AI-HAT-2).
- **Hailo-8 M.2 module** (vendor): 26 TOPS, "2.5W typical power consumption". PCIe Gen3 x2 on the A+E key (2230) and x4 on the M-key/B+M (2280) module. Industrial variant rated −40 to +85 °C ambient. Linux and Windows. Datasheet part numbers: HM218B1C2KAE (A+E 2230) and HM218B1C2LAE (B+M 2280). **[snippet]** — [Hailo-8 M.2 product page](https://hailo.ai/products/ai-accelerators/hailo-8-m2-ai-acceleration-module/), [A+E datasheet](https://hailo.ai/hailo-files/hailo-8-m-2-key-a-e-et-datasheet-en/), [B+M datasheet](https://hailo.ai/hailo-files/hailo-8-m-2-key-b-m-et-datasheet-en/)
- In `hailortcli benchmark` output, Hailo's own documentation example shows "Power in streaming mode (average) = 3.19 W (max 3.20 W)" for resnet_v1_50 on a Hailo device. This shows that sustained load can exceed the 2.5 W "typical" figure. **[verified]** — [hailo_model_zoo docs/BENCHMARKS.rst](https://github.com/hailo-ai/hailo_model_zoo/blob/master/docs/BENCHMARKS.rst)
- **Hailo-10H M.2** (vendor): up to 40 TOPS INT4 and 20 TOPS INT8, 4 GB or 8 GB LPDDR4/4X, PCIe Gen3 x4, M.2 Key M (2242/2280), −40 to 85 °C. **[snippet]** — [Hailo-10H M.2 page](https://hailo.ai/products/ai-accelerators/hailo-10h-m-2-ai-acceleration-module/), [AAEON Hailo-10H M.2 2280](https://www.aaeon.com/en/product/detail/ai-modules-hailo-10h-m-2-2280), [CNX Software](https://www.cnx-software.com/2024/04/04/hailo-10-m-2-key-m-module-brings-generative-ai-to-the-edge-with-up-to-40-tops-of-performance/)
- Hailo-10H power: "2.5W typical", with "full-load draw … around 3-4W" per a third-party review of lower reliability **[snippet]** — [Botmonster](https://botmonster.com/hardware/hailo-10-ai-accelerator-edge-inference-review/). Geerling's AI HAT+ 2 article states the chip "runs at a maximum of 3W" **[snippet]** — [Jeff Geerling](https://www.jeffgeerling.com/blog/2026/raspberry-pi-ai-hat-2/).
- **Hailo-15** family (vision-processor SoC for cameras, not an add-on accelerator): Hailo-15L 7 TOPS, 15M 11 TOPS, 15H 20 TOPS. Quad-core Cortex-A53, 12 MP ISP, H.264/H.265 encode, up to 4K streams. SolidRun sells a Hailo-15 SOM. **[snippet]** — [CNX Software](https://www.cnx-software.com/2023/03/12/hailo-15-ai-vision-processor-delivers-up-to-20-tops-for-smart-cameras/), [CNX SolidRun SOM](https://www.cnx-software.com/2024/04/01/solidrun-hailo-15-som-20-tops-ai-vision-processor/), [Hackster](https://www.hackster.io/news/hailo-s-latest-hailo-15-chips-bring-up-to-20-tops-of-compute-to-bear-on-edge-ai-image-workloads-9324a23ff3cb)
- The Model Zoo publishes separate model tables for Hailo-15H and Hailo-15L. **[verified]** — [hailo_model_zoo README](https://github.com/hailo-ai/hailo_model_zoo)

### Inferences
- For a drone payload, the Hailo-8 (26 TOPS) at about 2.5–3.2 W is the best choice for pure detection throughput. Section 5 shows that its Model Zoo FPS on YOLOv8n/s is well above the Hailo-10H's. The Hailo-10H only pays off if you also need on-board LLM/VLM, or for some newer models (YOLO11/YOLO26) at batch size 1.
- Hailo-15 is relevant only if you design a custom smart-camera board. The Model Zoo publishes Hailo-15 numbers, but it is not a drop-in for a Pi or Luckfox companion computer.

### Gaps
- Official Hailo-8L standalone power figure and Hailo-8/8L M.2 retail prices: I could not read hailo.ai directly and the retailer snippets gave no reliable price.
- Hailo-10H measured power under YOLO load: I found no primary measurement.

## 2. Host platforms and PCIe limitations

### Takeaway
Hailo officially targets x86_64 Ubuntu, Raspberry Pi 5, Windows (hailo-apps) and aarch64 via HailoRT built from source or deb packages. The community has run it on RK3588 and Jetson Orin Nano. The Pi 5 exposes only PCIe x1. It defaults to Gen2 (5 GT/s), which the AI HATs auto-raise to Gen3 (8 GT/s, not certified by Raspberry Pi). All published Model Zoo FPS figures were measured on x86 at PCIe Gen3 x4, so Pi numbers are lower, especially for models with large outputs or at batch size 1.

### Cited Findings
- The Pi 5 FPC connector "breaks out a PCIe Gen 2.0 ×1 interface". "Raspberry Pi 5 isn't certified for Gen 3.0 speeds. PCIe Gen 3.0 connections might be unstable." **[verified]** — [raspberrypi/documentation pcie.adoc](https://github.com/raspberrypi/documentation/blob/master/documentation/asciidoc/computers/raspberry-pi/pcie.adoc)
- The AI Kit needs Gen3 enabled manually (`dtparam=pciex1_gen=3` or raspi-config). For the AI HAT+ and AI HAT+ 2 "the setting is automatically applied". Requires 64-bit Raspberry Pi OS (Trixie). **[verified]** — [computers/ai/getting-started.adoc](https://github.com/raspberrypi/documentation/blob/master/documentation/asciidoc/computers/ai/getting-started.adoc)
- Model Zoo FPS test conditions: "System host: Intel Core i5-9400 … PCIe Gen 3 x 4 lanes, room temperature". **[verified]** — [HAILO8 object detection table](https://github.com/hailo-ai/hailo_model_zoo/blob/master/docs/public_models/HAILO8/HAILO8_object_detection.rst)
- Seeed benchmark (YOLOv8s, INT8, 640×640, batch 1): "The frame rate of Pi5 under PCIe gen3 is twice as high as under PCIe gen2". A snippet quotes about 80 FPS for YOLOv8s on Hailo-8L at Gen3. The test also ran on CM4 (Gen2 x1). **(independent)** **[snippet]** — [Seeed wiki](https://wiki.seeedstudio.com/benchmark_on_rpi5_and_cm4_running_yolov8s_with_rpi_ai_kit/). The repo [Seeed-Projects/Benchmarking-YOLOv8-on-Raspberry-PI-reComputer-r1000-and-AIkit-Hailo-8L](https://github.com/Seeed-Projects/Benchmarking-YOLOv8-on-Raspberry-PI-reComputer-r1000-and-AIkit-Hailo-8L) exists **[verified]**, but its results are published only as images, and it also compares against a Jetson Orin NX 16GB running TensorRT.
- hailo-apps supported platforms: Raspberry Pi 5, Ubuntu x86_64, Windows. Accelerators: Hailo-8, Hailo-8L, Hailo-10H. **[verified]** — [hailo-ai/hailo-apps README](https://github.com/hailo-ai/hailo-apps)
- HailoRT "supports Linux and Windows, and it can be compiled from sources to be integrated with various x86 and ARM processors". **[verified]** — [hailo-ai/hailort](https://github.com/hailo-ai/hailort)
- TAPPAS manual install supports Ubuntu x86 22.04/24.04 and Ubuntu aarch64 20.04, plus Yocto BSPs and Raspberry Pi OS. **[verified]** — [hailo-ai/tappas README](https://github.com/hailo-ai/tappas)
- RK3588: a vendor repo documents compiling the HailoRT PCIe driver in the RK3588 SDK ([industrialtablet/Compiling-HailoRT-PCI-Driver-in-RK3588-SDK](https://github.com/industrialtablet/Compiling-HailoRT-PCI-Driver-in-RK3588-SDK)). ArmSoM sells a "Sige7 + Hailo-8" combination ([ArmSoM](https://www.armsom.org/post/sige7-hailo-8)). **[snippet]**
- Jetson Orin Nano Super: a user reports the Hailo-8 M.2 installed and working with JetPack 6.2 / Ubuntu 22.04 **[snippet]** — [Hailo Community thread](https://community.hailo.ai/t/installed-hailo-8-m-2-on-a-nvidia-jetson-orin-nano-super/14773). Anton Maltsev wrote a guide to running Hailo on assorted ARM boards **[snippet]** — [Medium](https://medium.com/@zlodeibaal/how-to-run-hailo-on-arm-boards-d2ad599311fa).
- The Pi AI Kit's own Hailo-8L runs at Gen3 x1 on the Pi, while the Hailo-8 M.2 module supports up to x4 on M-key hosts (see §1). **[snippet]**

### Inferences
- At Gen3 x1 (about 1 GB/s raw), a 640×640×3 uint8 frame is about 1.2 MB. That caps the pure-transfer ceiling at roughly 800 fps in and adds output tensors, so small models (YOLOv8n) become host/PCIe/CPU-postprocess bound on a Pi long before the NPU saturates. This fits the gap between the Model Zoo's 202 FPS and the measured 72.7 FPS for YOLOv8n on the Hailo-8L (§5).
- The RK3588 (Luckfox-class or Rock 5) and i.MX8 need HailoRT built for their kernel. Hailo ships Debian aarch64 packages, but kernel-module (hailo-dkms / hailort-drivers) builds against vendor BSP kernels are the usual friction point.

### Gaps
- An official NXP i.MX8 support statement from Hailo (known from hailo.ai partner pages, which I could not access).
- No verified Hailo-10H-on-Jetson or Hailo-10H-on-RK3588 report found.

## 3. Software stack and licensing (DFC, HailoRT, TAPPAS, hailo-apps, Model Zoo, SW Suite)

### Takeaway
The runtime side is open source on GitHub: HailoRT (MIT, with the GStreamer `hailonet` element under LGPL-2.1+), TAPPAS (LGPL-2.1+), hailo-apps (MIT) and the Model Zoo (MIT). The **Dataflow Compiler (DFC) is closed-source** and is a wheel downloaded from the Hailo Developer Zone, which requires a login. The installable runtime .debs/.whl for non-Pi hosts also come from the Developer Zone. On the Pi they come from the Raspberry Pi apt repository (`hailo-all` / `hailo-h10-all`). The stack is now **split by generation**: Hailo-8/8L use DFC 3.x + HailoRT 4.x + Model Zoo v2.x, while Hailo-10H/15 use DFC 5.x + HailoRT 5.x + Model Zoo master.

### Cited Findings
- "The Hailo-8 and Hailo-8L devices are supported on the **Hailo Model Zoo v2.x** branch, in combination with the **Hailo Dataflow Compiler v3.x** branch. The `master` branch is intended for **Hailo-10** and **Hailo-15** devices only." Model Zoo master is at DFC v5.4.0 / HailoRT v5.4.0. **[verified]** — [hailo_model_zoo README](https://github.com/hailo-ai/hailo_model_zoo)
- Model Zoo v2.17 (the Hailo-8 line) requires DFC v3.33.0 and HailoRT 4.23.0, Ubuntu 22.04/24.04 (or WSL2), Python 3.10–3.12, an NVIDIA Pascal/Turing/Ampere GPU, driver 525, CUDA 12.5.1 and cuDNN 9.10. "The Hailo Model Zoo supports Hailo-8 connected via PCIe only." **[verified]** — [hailo_model_zoo v2.17 GETTING_STARTED](https://github.com/hailo-ai/hailo_model_zoo/blob/v2.17/docs/GETTING_STARTED.rst)
- Model Zoo master requires DFC v5.3.0+, GPU driver 555+ and CUDA 12.5.1. **[verified]** — [master GETTING_STARTED](https://github.com/hailo-ai/hailo_model_zoo/blob/master/docs/GETTING_STARTED.rst)
- Model Zoo: MIT licence. For DFC and HailoRT: "In case you are not Hailo customer please contact hailo.ai". Issue reporting on GitHub is disabled, and support goes through community.hailo.ai. **[verified]** — [hailo_model_zoo README](https://github.com/hailo-ai/hailo_model_zoo), [LICENSE](https://github.com/hailo-ai/hailo_model_zoo/blob/master/LICENSE)
- The Model Zoo ships Claude Code skills `/hailo-parse`, `/hailo-optimize` and `/hailo-compile` that wrap the public `hailo_sdk_client` API ("the DFC remains a black box"). **[verified]** — [hailo_model_zoo README](https://github.com/hailo-ai/hailo_model_zoo)
- HailoRT: libhailort, pyhailort and hailortcli are MIT. `hailonet` (GStreamer) is LGPL-2.1-or-later. The PCIe driver is in [hailo-ai/hailort-drivers](https://github.com/hailo-ai/hailort-drivers). The HailoRT `master` branch supports only Hailo-10/15, and Hailo-8/8R/8L use the `hailo8` branch. **[verified]** — [hailo-ai/hailort](https://github.com/hailo-ai/hailort)
- TAPPAS: LGPL-2.1-or-later, GStreamer 1.16/1.18/1.20, compatible with HailoRT 4.24.0 (Hailo-8) and 5.4.0 (Hailo-10H). Hailo recommends installing the "Hailo SW Suite" first (Ubuntu x86 22.04/24.04). Hailo-15 apps live in [hailo-camera-apps](https://github.com/hailo-ai/hailo-camera-apps). **[verified]** — [hailo-ai/tappas](https://github.com/hailo-ai/tappas)
- hailo-apps (MIT, v26.03.0 adds Windows support, YOLO26, and agentic dev) has 30+ apps: GStreamer pipeline apps, standalone Python/C++ HailoRT apps, and GenAI apps for Hailo-10H. It requires the HailoRT PCIe driver, HailoRT, TAPPAS Core and their Python wheels, all "Download from the Hailo Developer Zone" on non-Pi hosts. Commands include `hailo-detect-simple`, `hailo-pose`, `hailo-seg`, `hailo-depth` and `hailo-tiling`. **[verified]** — [hailo-ai/hailo-apps](https://github.com/hailo-ai/hailo-apps)
- `hailo-rpi5-examples` is marked "Outdated Repository … More up-to-date code examples are available here: hailo-apps". Older docs reference `hailo-apps-infra`. **[verified]** — [hailo-ai/hailo-rpi5-examples](https://github.com/hailo-ai/hailo-rpi5-examples)
- Pi install: `sudo apt install dkms hailo-all` (AI Kit / AI HAT+) or `hailo-h10-all` (AI HAT+ 2). "These packages can't co-exist." The packages bundle the kernel driver and firmware, HailoRT and TAPPAS core. `rpicam-apps` has Hailo post-process stages (e.g. `rpicam-hello --post-process-file /usr/share/rpi-camera-assets/hailo_yolov8_inference.json`, plus yolov5/yolov6/yolox, segmentation and pose). Versions must match between HEF compiler, HailoRT and driver. Raspberry Pi documents pinning e.g. `hailort=4.19.0-3 hailo-tappas-core=3.30.0-1 hailo-dkms=4.19.0-1`. **[verified]** — [computers/ai/getting-started.adoc](https://github.com/raspberrypi/documentation/blob/master/documentation/asciidoc/computers/ai/getting-started.adoc)
- DFC host requirements per third-party guides: Ubuntu 64-bit, 16+ GB RAM (32 GB recommended), about 50 GB disk; a GPU is "not required" but some optimization functions need it. **[snippet]** — [Macnica](https://www.macnica.co.jp/en/business/semiconductor/articles/hailo/144988/), [RidgeRun](https://developer.ridgerun.com/wiki/index.php/Hailo/Hailo-8/AI_Software_and_Tools/Hailo_AI_Software_Suite_Installation). The DFC can run under WSL2 **[snippet]** — [Hailo Community guide](https://community.hailo.ai/t/how-to-install-the-hailo-dataflow-compiler-dfc-on-wsl2/2890).

### Inferences
- The Developer Zone requires registration (a free account per common community reports). Only the DFC is truly closed. This is a practical blocker for CI pipelines, because the DFC wheel cannot be fetched anonymously.
- Pick the toolchain branch by chip. A Hailo-8L HEF compiled with DFC 3.x will not run on a Hailo-10H, and the reverse is also true. HEF, HailoRT and driver versions must be pinned together on the drone.

### Gaps
- The exact Developer Zone licence terms (EULA for the DFC) could not be read (hailo.ai blocked).

## 4. Custom YOLO workflow (train → ONNX → parse → optimize/quantize → compile HEF) and pitfalls

### Takeaway
The standard path has five steps:
1. Train with Ultralytics.
2. Export ONNX (opset 11 in Hailo docs).
3. Run `hailomz compile --ckpt best.onnx --calib-path <imgs> --yaml yolov8s.yaml --classes N --hw-arch hailo8|hailo8l|hailo10h`. This runs parse → optimize (INT8 quantization with the calibration set) → compile to HEF.

Use a GPU host. Without one the DFC drops to optimization level 0 (no accuracy-recovery algorithms). Hailo recommends ≥1024 calibration images. YOLO NMS is configured as a post-process with `engine=cpu`, meaning it runs on the host in HailoRT. Retraining dockers exist for YOLOv5 and YOLOv8.

### Cited Findings
- Model Zoo YOLOv8 retraining docker: `docker build … -t yolov8:v0`, run with `--gpus all` (requires nvidia-container-toolkit). Train with `yolo detect train data=… model=yolov8s.pt`. Export with `yolo export model=best.pt imgsz=640 format=onnx opset=11`. Compile with `hailomz compile --ckpt yolov8s.onnx --calib-path /path/to/calib/imgs --yaml path/to/yolov8s.yaml [--start-node-names …] [--end-node-names …] --classes 80`. "The model zoo will take care of adding the input normalization." If the input size was changed, update `preprocessing.input_shape` in yolo.yaml. Run the training outside the SW Suite docker. **[verified]** — [training/yolov8/README](https://github.com/hailo-ai/hailo_model_zoo/blob/master/training/yolov8/README.rst). The retraining code is Hailo's fork [hailo-ai/ultralytics](https://github.com/hailo-ai/ultralytics).
- A YOLOv5 retraining guide exists in the same tree ([training/yolov5](https://github.com/hailo-ai/hailo_model_zoo/blob/master/training/yolov5/README.rst)). An index page covers the other retrainable models ([RETRAIN_ON_CUSTOM_DATASET](https://github.com/hailo-ai/hailo_model_zoo/blob/master/docs/RETRAIN_ON_CUSTOM_DATASET.rst)). **[verified]**
- The hailo-apps retraining example (YOLOv8s on a 2-class barcode dataset) has these steps:
  - Train on Colab or any GPU.
  - Compile with `hailomz compile --ckpt best.onnx --calib-path …/valid --yaml yolov8s.yaml --classes 2 --hw-arch hailo10h --performance`. "Other platforms (Hailo 8, 8L) are also supported."
  - Download the `yolov8s_nms_config.json` post-process file from the YAML's zip.
  - Without a GPU, `CUDA_VISIBLE_DEVICES=""` gives "Reducing optimization level to 0 (accuracy won't be optimized and compression won't be used) … not recommended for production".
  - Compilation "can take several hours".
  - **Pitfall:** "Hailo conversion adds a background class at index 0, shifting all class IDs." Use `--labels-json` at runtime.

  **[verified]** — [hailo-apps doc/developer_guide/retraining_example.md](https://github.com/hailo-ai/hailo-apps/blob/main/doc/developer_guide/retraining_example.md)
- Optimization: "it is recommended to run this step on a GPU machine with dataset size of at least 1024 images". Full-precision optimizations (Equalization, TSE, pruning) run first, then quantization to **4/8/16-bit** weights and activations with IBC and QFT (quantization-aware fine-tuning). Evaluate with `hailomz eval --target emulator --har model.har`. "Hailo Model Zoo provides the following functionality for Model Zoo models only. If you wish to use your custom model, use the Dataflow Compiler directly." In practice, custom retrains of Model Zoo architectures reuse its YAML/ALLS. **[verified]** — [docs/OPTIMIZATION.rst](https://github.com/hailo-ai/hailo_model_zoo/blob/master/docs/OPTIMIZATION.rst)
- The YOLOv8n ALLS model script applies normalization /255, changes the output activations of conv42/53/63 to sigmoid, and calls `nms_postprocess("…yolov8n_nms_config.json", meta_arch=yolov8, engine=cpu)`. It is identical on v2.17 (Hailo-8) and master. The YOLOv5s ALLS uses `nms_postprocess(…, yolov5, engine=cpu)` and `model_optimization_config(calibration, batch_size=4)`. The base YOLOv8 config uses `nms_iou_thresh: 0.7`, `score_threshold: 0.001` and `nms_max_output_per_class: 300`. **[verified]** — [yolov8n.alls (v2.17)](https://github.com/hailo-ai/hailo_model_zoo/blob/v2.17/hailo_model_zoo/cfg/alls/generic/yolov8n.alls), [yolov5s.alls (v2.17)](https://github.com/hailo-ai/hailo_model_zoo/blob/v2.17/hailo_model_zoo/cfg/alls/generic/yolov5s.alls), [cfg/base/yolov8.yaml](https://github.com/hailo-ai/hailo_model_zoo/blob/master/hailo_model_zoo/cfg/base/yolov8.yaml)
- The Model Zoo also lists `*_nms_core` variants (e.g. `yolov6n_0.2.1_nms_core`, `yolov5xs_wo_spp_nms_core`), which run NMS on the chip. They are much slower: yolov6n_0.2.1 runs at 1256 FPS but its nms_core variant at 237 FPS on Hailo-8. **(vendor) [verified]** — [HAILO8 detection table](https://github.com/hailo-ai/hailo_model_zoo/blob/master/docs/public_models/HAILO8/HAILO8_object_detection.rst)
- Other flags: `--input-conversion nv12_to_rgb` fuses colour conversion on-chip, and `--performance` enables a slower, more aggressive compile. **[verified]** — [GETTING_STARTED](https://github.com/hailo-ai/hailo_model_zoo/blob/master/docs/GETTING_STARTED.rst), [retraining_example](https://github.com/hailo-ai/hailo-apps/blob/main/doc/developer_guide/retraining_example.md)
- Common community failures:
  - `NMSConfigPostprocessException: The layer yolov8n/conv41 doesn't have one output layer`, caused by wrong end-node names after custom export.
  - "Model Script Not Found".
  - Custom-class models needing different end nodes.

  **[snippet]** — [Hailo Community #5033](https://community.hailo.ai/t/hailo-sdk-client-tools-core-postprocess-nms-postprocess-nmsconfigpostprocessexception-the-layer-yolov8n-conv41-doesnt-have-one-output-layer/5033), [#17613](https://community.hailo.ai/t/convert-to-hef-for-hailo8l-unable-to-compile-yolov8-onnx-model-with-hailomz-model-script-not-found/17613), [#7299](https://community.hailo.ai/t/compile-custom-onnx-models-on-hailo8/7299), [#2995](https://community.hailo.ai/t/cant-optimize-or-compile-yolov8-with-trained-on-custom-dataset/2995)
- A third-party guide walks through YOLOv11n to Hailo-8 HEF **[snippet]** — [Rose City Robotics guide](https://common.rosecityrobotics.com/YOLO_ObjectDetection/YOLOv11n_to_Hailo8_Guide.html).
- Quantization accuracy drop, measured independently on the drone dataset DUT Anti-UAV (CPU INT8 ONNX, not Hailo): YOLOv8n AP fell from 0.556 to 0.518, and AP_S (small objects) from 0.418 to 0.358. This is a warning that small-object AP suffers most from INT8. **(independent) [verified]** — [hailo-uav-tracker](https://github.com/pierrosimonestd-cpu/hailo-uav-tracker)
- Model Zoo float vs hardware (INT8) mAP on COCO, measured by the vendor: YOLOv8n 37.0 → 36.4, YOLOv8s 44.6 → 43.9, YOLOv11n 39.0 → 37.5, YOLOv5s 35.3 → 34.0. That is about 0.6–1.5 mAP lost. **(vendor) [verified]** — [HAILO8 table](https://github.com/hailo-ai/hailo_model_zoo/blob/master/docs/public_models/HAILO8/HAILO8_object_detection.rst)

### Inferences
- For a drone detector with 1–2 classes, the lowest-risk path is:
  - Stick to an architecture the Model Zoo already has an ALLS/YAML for (yolov8n/s, yolov11n/s, yolov5s).
  - Keep the Ultralytics head unmodified.
  - Export at the intended input size with opset 11.
  - Pass `--classes N`.
  - Calibrate with ≥1024 representative in-domain frames, including sky, birds and small targets.
  - Validate with `hailomz eval --target emulator` before flashing.
- Because NMS runs on the CPU (`engine=cpu`), Pi CPU load grows with the number of candidates. Keep the score threshold sensible at runtime; the Model Zoo's 0.001 is an evaluation setting only.

### Gaps
- Exact DFC optimization-level semantics (levels 0–4, compression levels) and the default calibration size in DFC 3.x/5.x. These are documented only in the DFC User Guide in the Developer Zone, which I could not access.
- INT4 applicability to YOLO on the Hailo-10H: no verified guidance found.

## 5. Published benchmarks (FPS/latency), Model Zoo vs real Pi 5

### Takeaway
Vendor Model Zoo numbers (x86 host, Gen3 x4, `hailortcli` hardware-only) for 640×640 COCO are shown in the table below. On a real Pi 5, one independent measurement of YOLOv8n on the Hailo-8L gives 13.6 ms inference, 13.8 ms end-to-end and **72.7 FPS**, against the Model Zoo's 202 FPS. That is roughly 2.8× lower, with one PCIe lane and host post-processing.

### Cited Findings
Model Zoo object detection tables for Hailo-8, Hailo-8L and Hailo-10H. **(vendor) [verified]** — [HAILO8](https://github.com/hailo-ai/hailo_model_zoo/blob/master/docs/public_models/HAILO8/HAILO8_object_detection.rst), [HAILO8L](https://github.com/hailo-ai/hailo_model_zoo/blob/master/docs/public_models/HAILO8L/HAILO8L_object_detection.rst), [HAILO10H](https://github.com/hailo-ai/hailo_model_zoo/blob/master/docs/public_models/HAILO10H/HAILO10H_object_detection.rst)
- Conditions: i5-9400 host, PCIe Gen3 x4, room temperature. The Hailo-8/8L headers say "Dataflow Compiler v2.19.0" and the Hailo-10H header says "v5.4.0". The Hailo-8 "v2.19.0" likely refers to the Model Zoo version, since HEF links are under `ModelZoo/Compiled/v2.19.0/hailo8/`; the README says Hailo-8 uses DFC 3.x. This is a documentation inconsistency.
- Columns in each chip cell: FPS at batch 1 / FPS at batch 8 · hardware (quantized) mAP. All inputs are 640×640 unless noted.

| Model | Float mAP | GOPs | Hailo-8 | Hailo-8L | Hailo-10H |
|---|---|---|---|---|---|
| yolov5s | 35.3 | 17.44 | 543/543 · 34.0 | 124/243 · 34.0 | 250/296 · 34.2 |
| yolov5m | 42.6 | 52.17 | 156/156 | 60.4/105 | 111/174 |
| yolov5m6_6.1 (1280×1280) | 50.7 | 200.04 | 33.4/50.1 | 21.8/29.7 | 40.3/50.4 |
| yolov6n | 34.3 | 11.12 | 1250/1250 | 356/356 | 515/516 |
| yolov8n | 37.0 | 8.74 | 1036/1036 · 36.4 | 202/438 · 36.4 | 375/370 · 36.4 |
| yolov8s | 44.6 | 28.6 | 491/491 · 43.9 | 110/208 · 43.9 | 166/252 · 44.1 |
| yolov8m | 49.9 | 78.93 | 66.9/149 · 49.2 | 51.0/87.0 · 49.2 | 76.2/132 · 49.2 |
| yolov10n | 38.5 | 6.8 | 194/567 | 150/359 | 303/345 |
| yolov11n | 39.0 | 6.55 | 185/541 · 37.5 | 157/371 · 37.5 | 302/332 · 38.0 |
| yolov11s | 46.3 | 21.6 | 111/303 · 45.1 | 92.0/192 · 45.1 | 142/223 · 45.5 |
| yolov11m | 51.1 | 68.1 | 50.2/102 · 49.9 | 35.3/58.1 · 49.9 | 70.6/124 · 50.2 |
| yolo26n | 40.0 | 5.5 | 155/427 | 111/255 | 233/294 |
| yolo26s | 47.5 | 20.9 | 97.8/263 | 66.6/147 | 125/212 |
| yolov7e6 (1280×1280) | 55.4 | 515 | 11.4/16.0 | 6.44/7.97 | 16.6/20.4 |

- YOLOv5n is **not** in the Model Zoo detection tables (verified by parsing all three tables). The smallest YOLOv5 entry is `yolov5xs_wo_spp` at 512×512: 1139 FPS on Hailo-8, 206/438 on Hailo-8L, 361/513 on Hailo-10H. **[verified]** — same tables
- Independent Pi 5 + Hailo-8L numbers (DUT Anti-UAV project, via HailoRT):
  - yolov8n-int8: inference 13.6 ms, end-to-end 13.8 ms, p99 14.2 ms, **72.7 FPS**.
  - yolov8s-int8: 19.9 ms, 20.2 ms e2e, **49.5 FPS**.
  - For comparison, Pi 5 CPU with ONNX: fp32 153.6 ms (6.5 FPS), int8 75.0 ms (13.3 FPS).

  **(independent) [verified]** — [pierrosimonestd-cpu/hailo-uav-tracker](https://github.com/pierrosimonestd-cpu/hailo-uav-tracker)
- Seeed: Pi 5 + Hailo-8L YOLOv8s 640 at batch 1 reaches about 80 FPS at Gen3, and Gen3 is about 2× Gen2. **(independent) [snippet]** — [Seeed wiki](https://wiki.seeedstudio.com/benchmark_on_rpi5_and_cm4_running_yolov8s_with_rpi_ai_kit/), [RPi forum thread](https://forums.raspberrypi.com/viewtopic.php?t=373867)
- lusher00/hailo-tracker (Pi 5 + Hailo-8L, COCO detection plus tracking, MJPEG web UI) claims "~30 FPS real-time inference" end-to-end, including camera capture and streaming. **(independent, self-reported) [verified]** — [lusher00/hailo-tracker](https://github.com/lusher00/hailo-tracker)
- Geerling (AI HAT+ 2 review): for vision the Hailo-10H is not a big step over the Hailo-8, and if you mainly want object detection "the cheaper 13 or 26 TOPS model is perfectly adequate". **[snippet]** — [Jeff Geerling AI HAT+ 2](https://www.jeffgeerling.com/blog/2026/raspberry-pi-ai-hat-2/), [Geerling Frigate + Hailo](https://www.jeffgeerling.com/blog/2026/frigate-with-hailo-for-object-detection-on-a-raspberry-pi/)
- `hailortcli benchmark model.hef` reports FPS (hw_only and streaming), hardware latency and power. This is the method behind Model Zoo figures and excludes host pre/post-processing. **[verified]** — [BENCHMARKS.rst](https://github.com/hailo-ai/hailo_model_zoo/blob/master/docs/BENCHMARKS.rst)

### Inferences
- A large batch-1 vs batch-8 gap (e.g. Hailo-8 yolov11n at 185 vs 541, yolov8m at 66.9 vs 149) usually means the model does not fit on-chip in a single context and needs context switching. For real-time drone video at batch 1, **YOLOv8n/s and YOLOv5s fit in one context on Hailo-8 (identical bs1/bs8 FPS)**, while YOLO11/YOLO26 do not. For a latency-critical Hailo-8 deployment, YOLOv8 is therefore the better choice over YOLO11 despite YOLO11's slightly higher mAP.
- The Hailo-10H is slower than the Hailo-8 on YOLOv8n/s/YOLOv5s at batch 1, but faster on YOLO11/YOLO26/yolov8m. It is roughly 1.5–3× faster than the Hailo-8L across the board.
- For 1280×1280 input (important for small, distant drones), expect only about 20–40 FPS even for a medium model (yolov5m6: 33 FPS on Hailo-8, 22 FPS on Hailo-8L at bs1, vendor x86). Tiling a 640 model is often better (§7).
- Expect Pi 5 end-to-end throughput of about 35–50% of the Model Zoo bs1 figure for small models on the Hailo-8L. This is extrapolated from a single YOLOv8n data point (72.7 vs 202).

### Gaps
- No verified independent Pi 5 + Hailo-8 (26 TOPS) or Pi 5 + Hailo-10H YOLO FPS measurements with exact numbers (Geerling and Seeed pages blocked).
- No published Hailo latency numbers per model except the independent hailo-uav-tracker ones. The Model Zoo tables list FPS only; per-model latency is in the profiler HTML reports linked from the tables.

## 6. Existing open-source projects combining Hailo with drones / counter-UAS / MAVLink

### Takeaway
There are few verified projects. The best-verified counter-UAS example is **pierrosimonestd-cpu/hailo-uav-tracker**: Pi 5 + Hailo-8L, YOLOv8n trained on DUT Anti-UAV, pan/tilt turret, and bird hard-negative analysis. Its control path uses an ESP32 servo link, not MAVLink. I found **no verified public repo** that integrates Hailo with ArduPilot/PX4 over MAVLink. A PX4 + ROS2 + Hailo-8L payload-drop project appears in GitHub topic snippets, but I could not confirm its URL.

### Cited Findings
- **hailo-uav-tracker** (MIT, Python 3.10+, has CI) **[verified]** — [GitHub](https://github.com/pierrosimonestd-cpu/hailo-uav-tracker)
  - Architecture: "Detection runs on the NPU; the tracker and a latency-compensating controller run on the Pi's CPU; an ESP32 drives the servos and holds a failsafe."
  - DUT Anti-UAV detection (COCO protocol, 2200 test images): YOLOv8n fp32 AP 0.556 / AP50 0.894 / AP_S 0.418. "52% of the objects in this dataset are smaller than 32×32 pixels".
  - Bird false positives: 23.4% at 0.35 confidence on 593 Open Images bird photos. This falls to 7.1% after adding 600 bird background images.
  - Latency vs pointing error: 10 ms → 0.68° RMS, 45 ms → 0.83°, 300 ms → 2.77°. The loop is "latency-limited, not actuator-limited".
- **lusher00/hailo-tracker**: Pi 5 + Hailo-8L, COCO detection with persistent track IDs, dual CSI cameras, web UI, webhooks and SQLite events. Not drone-specific and **no MAVLink** (no "mavlink" string in the README). **[verified]** — [GitHub](https://github.com/lusher00/hailo-tracker)
- A gist, "RPI 5 - Hailo 8L - tracking", exists **[snippet]** — [gist Vincent-Stragier](https://gist.github.com/Vincent-Stragier/dad4af4287d6f57b38be2487b818fac0)
- A PX4 + ROS 2 Jazzy "autonomous payload-drop system powered by Hailo-8L inference with real-time YOLOv8s detection, OFFBOARD control, and full HIL validation (RPi5 ↔ PX4 SITL)" is referenced on GitHub topic pages, but the repo URL could not be retrieved or verified. **[snippet, unverified]** — [GitHub topic: companion-computer](https://github.com/topics/companion-computer)
- Hailo's own tiling demo defaults to a drone/aerial use case: model `hailo_yolov8n_4_classes_vga` on the VisDrone video `tiling_visdrone_720p.mp4`. **[verified]** — [hailo-apps tiling README](https://github.com/hailo-ai/hailo-apps/blob/main/hailo_apps/python/pipeline_apps/tiling/README.md)
- A Hailo Community thread discusses "Improving Small Object Detection in 4K Drone Footage Using Tiling with Hailo AI Processors" **[snippet]** — [community.hailo.ai #15983](https://community.hailo.ai/t/improving-small-object-detection-in-4k-drone-footage-using-tiling-with-hailo-ai-processors/15983)
- Generic Pi 5 ↔ ArduPilot MAVLink companion setup (pymavlink/MAVProxy) is documented by ArduPilot **[snippet]** — [ArduPilot dev docs](https://ardupilot.org/dev/docs/raspberry-pi-via-mavlink.html). Non-Hailo PX4 + ROS 2 + YOLOv8 aerial detection simulation repo: [monemati/PX4-ROS2-Gazebo-YOLOv8](https://github.com/monemati/PX4-ROS2-Gazebo-YOLOv8) **[snippet]**. Non-Hailo ArduPilot + pymavlink + Pi 5 companion project: [Pummet/my-drone-project](https://github.com/Pummet/my-drone-project) **[snippet]**.

### Inferences
- Integrating Hailo with MAVLink is straightforward glue. A hailo-apps detection callback (Python) produces boxes. A tracker then turns them into bearing/angle errors, and pymavlink or MAVSDK sends `SET_POSITION_TARGET_LOCAL_NED`, gimbal `GIMBAL_MANAGER_SET_ATTITUDE` or `LANDING_TARGET` messages. The absence of a mature reference repo means the drone project will have to build this layer itself.
- The hailo-uav-tracker results on bird false positives and small-object AP loss are directly relevant to a counter-UAS detector design.

### Gaps
- No verified Hailo + ArduPilot/PX4 follow-me or precision-landing repo. GitHub search APIs were not available in this sandbox, so more may exist.

## 7. Tiling / SAHI-like support on Hailo

### Takeaway
Hailo provides built-in tiling. It was historically a TAPPAS app and is now the `hailo-tiling` pipeline app in hailo-apps. The app splits each frame into a grid of model-sized tiles with automatically computed overlap, optionally adds multi-scale grids, runs every tile through `hailonet`, and merges results with NMS and removal of objects on tile borders. This is effectively SAHI on-device, and it trades FPS for tile count.

### Cited Findings
- "The tiling pipeline demonstrates splitting each frame into several tiles which are processed independently by the `hailonet` element … especially effective for detecting small objects in high-resolution frames". The default is aerial detection with `hailo_yolov8n_4_classes_vga` and VisDrone footage. **[verified]** — [hailo-apps tiling README](https://github.com/hailo-ai/hailo-apps/blob/main/hailo_apps/python/pipeline_apps/tiling/README.md)
- Usage:
  - `hailo-tiling`
  - `--multi-scale` with `--scale-levels 1|2|3`, which adds 1×1 (+1 tile), 1×1+2×2 (+5) or 1×1+2×2+3×3 (+14) grids
  - `--tiles-x 3 --tiles-y 2 --hef custom.hef`
  - `--input rpi`
  - `--frame-rate` to throttle
  - Example: a 4×3 grid plus scale-level 2 = 17 tiles per frame.

  **[verified]** — same
- Pipeline: "Crop → Inference → Post-process → Aggregate → Remove border objects → Perform NMS". Overlap rule: `min-overlap ≥ smallest_object_size / model_input_size`. The default is 0.10 (64 px objects at 640), and 0.05 suits 32 px objects. "It's very easy to reach very high FPS requirements by using too many tiles." **[verified]** — same
- Earlier Hailo material describes tiling as using "Hailo's high throughput to handle high-resolution images (FHD, 4K) by dividing them into smaller tiles" **[snippet]** — [Edge AI and Vision Alliance demo](https://www.edge-ai-vision.com/2021/11/hailo-demonstration-of-leveraging-high-processor-throughput-to-improve-object-detection-on-the-edge/). DeGirum documents tiling on Hailo in its PySDK **[snippet]** — [DeGirum docs](https://docs.degirum.com/hailo/intermediate-guides/tiling).

### Inferences
- Budget example for a 1920×1080 frame with a 640 model: a 3×2 grid is 6 tiles. With YOLOv8n on a Pi 5 + Hailo-8L at about 70 inferences/s measured, that gives about 11 FPS. The same YOLOv8n on a Hailo-8 (1036 FPS vendor at bs1, more in tile batches) would allow much higher rates if the Pi's PCIe x1 link and CPU cropping/NMS keep up. Tiles can be batched, which helps multi-context models (bs8 FPS applies).
- For distant-drone detection, tiling a 640 model is generally more efficient than compiling a 1280 model (e.g. yolov5m6 at 1280 is 33 FPS on Hailo-8).

### Gaps
- No published tiling FPS benchmarks on the Pi 5 were found.
- It is unclear whether TAPPAS still ships the older C++ `tiling` app separately; it was not checked in the current TAPPAS tree.
