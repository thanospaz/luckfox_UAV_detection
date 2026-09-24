# Open-source landscape for vision-based drone/UAV detection and tracking (EO + IR), edge-oriented — state as of Sept 2026

Method note: GitHub star counts and `pushed_at` (last push, which is a proxy for the last commit) come from the GitHub search API, queried on 2026-09-24. The `updated_at` field changes whenever someone stars a repo, so it is NOT used as "last activity". arXiv, MDPI, NCBI/PMC, ScienceDirect and WordPress were blocked by the egress proxy. Paper numbers below therefore come from search-engine snippets of those pages, or from GitHub READMEs, and are marked that way. Reliability tags: **[PR]** peer-reviewed venue, **[preprint]** arXiv only, **[vendor]**, **[hobby]** individual or student repo, **[agg]** aggregator snippet.

## 1. Most active and complete open-source repos (stars, last push, license, contents, edge targets)

### Takeaway
No single repo gives you trained weights, a dataset, tracking and edge deployment all together. The most useful building blocks today are:
- benchmark and dataset repos: ZhaoJ9014/Anti-UAV, Anti-UAV410, DUT-Anti-UAV and the Halmstad multi-sensor dataset;
- one strong recent detector+tracker baseline: YOLOv12-BoT-SORT-ReID, a CVPR 2025 Anti-UAV challenge entry, AGPL;
- SAHI for tiled inference (MIT, very active);
- a handful of small but recent edge repos: RPi5+Hailo-8L and RK3588 NPU.

Most "YOLO drone detection" repos with more stars are hobby or tutorial grade.

### Cited Findings
Verified repo metadata (stars / last push / license):
- **obss/sahi**: 5,515★, pushed 2026-09-24, MIT. Model-agnostic sliced inference that is used for small objects. Very active. — [GitHub search API result](https://github.com/obss/sahi)
- **ZhaoJ9014/Anti-UAV** (official Anti-UAV repo): 855★, pushed 2025-05-07, MIT. Hosts Anti-UAV300 (RGB+IR), Anti-UAV410 (IR) and Anti-UAV600 (IR, via ModelScope). Also includes baseline code, a Jittor port, a pysot evaluation toolkit and a demo notebook, plus links for the CVPR 2020, ICCV 2021, CVPR 2023 and CVPR 2025 challenges. No push in ~16 months, so it is moderately stale. — [GitHub](https://github.com/ZhaoJ9014/Anti-UAV) [PR-backed]
- **DroneDetectionThesis/Drone-detection-dataset** (Halmstad / Svanström): 324★, pushed 2026-04-26, CC0-1.0. Dataset only, labels in MATLAB .mat format. A Python decoder (mcos-decoder) is now available. — [GitHub](https://github.com/DroneDetectionThesis/Drone-detection-dataset)
- **wangdongdut/DUT-Anti-UAV**: 283★, pushed 2023-10-10, Apache-2.0. Dataset plus benchmark results. Stale since 2023. — [GitHub](https://github.com/wangdongdut/DUT-Anti-UAV)
- **ntu-aris/MMAUD** (ICRA 2024, multi-modal anti-UAV dataset): 275★, pushed 2026-07-29, MIT. The default branch is gh-pages, so this is a project site. — [GitHub](https://github.com/ntu-aris/MMAUD) [PR]
- **wish44165/YOLOv12-BoT-SORT-ReID**: 211★, pushed 2026-09-02, AGPL-3.0.
  - A CVPR 2025 4th Anti-UAV Workshop entry, 3rd place per its README badge.
  - Multi-UAV thermal-IR detection plus BoT-SORT-ReID tracking.
  - Active. AGPL is a constraint for closed products.
  - Sources: [GitHub](https://github.com/wish44165/YOLOv12-BoT-SORT-ReID); paper [CVPRW 2025](https://openaccess.thecvf.com/content/CVPR2025W/Anti-UAV/html/Chen_Strong_Baseline_Multi-UAV_Tracking_via_YOLOv12_with_BoT-SORT-ReID_CVPRW_2025_paper.html) [PR]
- **HwangBo94/Anti-UAV410**: 188★, pushed 2025-07-11, no license file. Benchmark and evaluation toolkit (TPAMI 2023). — [GitHub](https://github.com/HwangBo94/Anti-UAV410); [TPAMI](https://dl.acm.org/doi/10.1109/TPAMI.2023.3335338) [PR]
- **ucas-vg/Anti-UAV**: 147★, pushed 2022-03-09, MIT. Mirror or earlier version of the Anti-UAV benchmark. **Stale.** — [GitHub](https://github.com/ucas-vg/Anti-UAV)
- **doguilmak/Drone-Detection-YOLOv11x**: 110★, pushed 2025-08-12, MIT. Also from the same author: Drone-Detection-YOLOv8x (147★) and YOLOv7 (85★). These are notebooks with a custom dataset and weights; the YOLOv11x repo adds a tracking and heatmap demo. YOLOv11x is too heavy for NPUs without retraining. — [GitHub](https://github.com/doguilmak/Drone-Detection-YOLOv11x) [hobby]
- **wd-sir/UAVDETR**: 101★, pushed 2026-08-01, no license. Official code for "UAV-DETR: DETR for Anti-Drone Target Detection" (arXiv 2603.22841). — [GitHub](https://github.com/wd-sir/UAVDETR); [arXiv](https://arxiv.org/pdf/2603.22841) [preprint]
- **alebal123bal/khadas_yolov8n_multithread**: 91★, pushed 2026-06-19, Apache-2.0. **Edge (Rockchip):** YOLOv8n UAV detection on the RK3588S NPU. The README claims "46 FPS", "~140 MB RAM" and an ISP+RGA+NPU pipeline. — [GitHub](https://github.com/alebal123bal/khadas_yolov8n_multithread) [hobby]
- **Irisky123/YOLOMG**: 81★, pushed 2025-05-15, GPL-3.0. Dataset and code for drone-to-drone detection that fuses appearance with pixel-level motion. This is the motion+CNN hybrid pattern. — [GitHub](https://github.com/Irisky123/YOLOMG) [preprint]
- **earth-insights/Dist-Tracker**: 41★, pushed 2025-07-02, no license. Claims the championship of Track 3 (multi-UAV tracking) at the CVPR 2025 4th Anti-UAV Challenge. — [GitHub](https://github.com/earth-insights/Dist-Tracker)
- **moured/TY-RIST** (ICCV 2025, "Tactical YOLO Tricks for Real-time Infrared Small Target Detection"): 28★, pushed 2025-10-01, MIT. — [GitHub](https://github.com/moured/TY-RIST) [PR]
- **cyuquan8/thermal_signature_drone_detection**: 25★, pushed 2021-05-28, Apache-2.0. A YOLOv3 thermal detector with motion encoding. **Abandoned.** — [GitHub](https://github.com/cyuquan8/thermal_signature_drone_detection)

Other repos I confirmed exist through GitHub search. I did not check their last push date.
- **vero1925/FocusTrack**: 41★. Code for an efficient Anti-UAV tracker; ~62.8 AUC on Anti-UAV410 per the MemLoTrack comparison below. — [GitHub](https://github.com/vero1925/FocusTrack)
- **xuefeng-zhu5/EDTC**: 25★. Evidential detection–tracking collaboration plus a benchmark. — [GitHub](https://github.com/xuefeng-zhu5/EDTC)
- **PCwenyue/CST-Anti-UAV**: 30★. A thermal-IR tiny-UAV tracking benchmark toolkit. — [GitHub](https://github.com/PCwenyue/CST-Anti-UAV); [arXiv 2507.23473](https://arxiv.org/pdf/2507.23473)
- **Prabhdeep1999/uav-detection**: 66★. YOLOv5 IR drone detection with the topic tag jetson-tx2. — [GitHub](https://github.com/Prabhdeep1999/uav-detection) [hobby]
- **kyn0v/TIB-Net**: 46★. "Drone Detection Network With Tiny Iterative Backbone". **Archived.** — [GitHub](https://github.com/kyn0v/TIB-Net)
- **yjwong1999/IJCNN2025-DvB**: 15★. WRN-YOLO, which the README says finished "Top 3 in 8th Drone-vs-Bird Detection Challenge". — [GitHub](https://github.com/yjwong1999/IJCNN2025-DvB)
- **KostadinovShalon/UAVDetectionTrackingBenchmark**: code for the Isaac-Medina ICCVW 2021 benchmark. — [GitHub](https://github.com/KostadinovShalon/UAVDetectionTrackingBenchmark)
- **chuanenlin/drone-net**: 165★. YOLOv3/darknet with a small drone image set and weights, from 2018. Obsolete. — [GitHub](https://github.com/chuanenlin/drone-net) [hobby]
- **maliyildirim/drone-vs-bird-realtime-detection**: 4★, created 2026-05. A two-stage detector plus drone-vs-bird classifier with TensorRT FP16. — [GitHub](https://github.com/maliyildirim/drone-vs-bird-realtime-detection) [hobby]
- **pierrosimonestd-cpu/hailo-uav-tracker**. **Edge (Hailo):**
  - Setup: RPi 5 + Hailo-8L (13 TOPS), YOLOv8n, ByteTrack + Kalman, pan/tilt servos, MIT.
  - DUT Anti-UAV test set (COCO protocol): yolov8n fp32 gets AP 0.556, AP50 0.894, AP_S 0.418. INT8 gets AP 0.518, AP50 0.875, AP_S 0.358.
  - Speed: "72.7" FPS on Pi5+Hailo-8L versus 6.5 FPS on the Pi5 CPU.
  - Tracking success AUC is 0.593.
  - Weights and HEF are not shipped; compiling a HEF needs the proprietary Hailo Dataflow Compiler (x86_64).
  - The README itself cautions that the accelerator numbers are not measured on this project's own detector.
  - Source: [GitHub README](https://github.com/pierrosimonestd-cpu/hailo-uav-tracker) [hobby]
- **Daksh7112003/drone-seraphim**: YOLOv8n trained on the "Seraphim Drone Dataset" with a Hailo-10H pipeline. Surfaced via search only; I did not inspect it. — [GitHub](https://github.com/Daksh7112003/drone-seraphim) [hobby]
- Non-vision repos in the same space, noted for context only: acoustic detectors batear-io/batear (412★) and agamrossen/VolAnti (385★), and RF detector kitoweeknd/RFUAV (444★). — [batear](https://github.com/batear-io/batear), [VolAnti](https://github.com/agamrossen/VolAnti), [RFUAV](https://github.com/kitoweeknd/RFUAV)

Edge benchmark context:
- A UAV-detection paper on a standalone RPi5: YOLOv8 with selective knowledge distillation keeps "85.3% mAP" at ~10.1 FPS. — [CEUR-WS Vol-3970](https://ceur-ws.org/Vol-3970/PAPER1.pdf) [PR, workshop; via search snippet]
- RPi5 + Hailo-8: the snippet says YOLOv9-C reaches 54.8 mAP@50-95 at 21.2 FPS and YOLOX-Tiny runs at 195.6 FPS. These are COCO numbers, not drone numbers. — [visionanalysis.org](https://www.visionanalysis.org/articles/best-object-detection-rpi5-hailo8) [hobby/blog]

### Inferences
- The best license-clean starting point for a Hailo or Rockchip product is Ultralytics-free code (Ultralytics is AGPL), such as hailo-uav-tracker (MIT) or the khadas RK3588 repo (Apache-2.0), plus training on DUT Anti-UAV and/or the Halmstad data (CC0). YOLOv12-BoT-SORT-ReID and all Ultralytics-based repos carry AGPL obligations.
- The research-grade trackers (FocusTrack, MemLoTrack, UAUTrack) are transformer single-object trackers built for GPUs. They are unlikely to run in real time on small NPUs without heavy distillation.
- Stale or abandoned: ucas-vg/Anti-UAV (2022), DUT-Anti-UAV (2023), TIB-Net (archived), thermal_signature_drone_detection (2021), drone-net (2018).

### Gaps
- For many repos I have no `pushed_at` value: FocusTrack, EDTC, CST-Anti-UAV, uav-detection, hailo-uav-tracker and drone-seraphim. The direct GitHub REST API was blocked from the sandbox.
- I found no official Hailo Model Zoo or vendor drone-detection model. hailo-ai repos were not verified.
- I found no mature Jetson- or DeepStream-specific open drone-detection repo; only topic tags such as jetson-tx2.

## 2. Public datasets and benchmarks

### Takeaway
- IR tracking is dominated by the Anti-UAV family: 300 is RGB+IR, while 410 and 600 are IR only. Anti-UAV410 SOTA is ~63–64 AUC.
- Drone-vs-Bird (WOSDETC) is the main EO small-target-plus-bird-confusion benchmark. It is on its 8th edition (IJCNN 2025) and access is restricted.
- The only CC0, fully open multi-sensor IR/visible/audio set with bird, airplane and helicopter distractors is the Halmstad (Svanström) dataset.
- Det-Fly and MAV-VID cover air-to-air or mixed-view scenes.
- DUT Anti-UAV is a clean 10k-image visible detection set.

### Cited Findings
- **Anti-UAV300 / 410 / 600** (ZhaoJ9014/Anti-UAV):
  - Modality: 300 is RGB+IR; 410 and 600 are IR only. The repo describes the data as "Full HD video sequences", "densely annotated", with visibility flags.
  - Access: Google Drive or Baidu with passwords for 300 and 410; ModelScope for 600. MIT repo license.
  - Sources: [GitHub](https://github.com/ZhaoJ9014/Anti-UAV); modality confirmation at [anti-uav.github.io](https://anti-uav.github.io/dataset/)
- **Anti-UAV410**:
  - Size: 410 thermal-IR videos with over 438K manually annotated boxes (TPAMI 2023).
  - SOTA: MemLoTrack reports AUC 63.6 and SA 64.0, versus FocusTrack at 62.8 AUC and 63.9 SA. UAUTrack (arXiv 2512.02668) claims TIR single-modality SOTA, +1.1 AUC over the second best.
  - Sources: [TPAMI](https://dl.acm.org/doi/10.1109/TPAMI.2023.3335338); [MemLoTrack, PMC](https://pmc.ncbi.nlm.nih.gov/articles/PMC12694105/) [PR]; [UAUTrack](https://arxiv.org/pdf/2512.02668) [preprint] (all via search snippets)
- **4th Anti-UAV Challenge (CVPR 2025)**:
  - Track 3 is new: multi-UAV tracking with 300 sequences (200 train, 100 test).
  - The data was enlarged with dynamic backgrounds and tiny-scale targets.
  - The Track 1 top AOA was 73.23, then 73.08 and 71.45.
  - Sources: [anti-uav.github.io](https://anti-uav.github.io/); [Codalab Track 3](https://codalab.lisn.upsaclay.fr/competitions/21806)
- **Drone-vs-Bird (WOSDETC)**:
  - Task: raise an alarm and localize only drones in videos where birds are also present.
  - 8th edition at IJCNN 2025, with 16 algorithms submitted. The winner used multi-scale small-target detection, copy-paste augmentation and frame-to-frame consistency post-processing. Summary paper: "The Drone-vs-Bird Detection Grand Challenge at IJCNN 2025".
  - Sources: [Fraunhofer publica](https://publica.fraunhofer.de/entities/publication/267aa1ec-a62d-4afa-99ad-cc369f57642b); [wosdetc2025 site](https://wosdetc2025.wordpress.com/) [PR; via snippet]
  - Earlier-edition summary: Coluccia et al., Sensors 2021. — [MDPI](https://www.mdpi.com/1424-8220/21/8/2824)
  - As used by Isaac-Medina: 77 videos (104,760 images), two classes (drone, bird), static and moving cameras. Best mAP 0.667 (DETR); COCO AP only 0.283 (Faster R-CNN). This is clearly the hardest of the three datasets they tested. — [ICCVW 2021 benchmark](https://openaccess.thecvf.com/content/ICCV2021W/AntiUAV/papers/Isaac-Medina_Unmanned_Aerial_Vehicle_Visual_Detection_and_Tracking_Using_Deep_Neural_ICCVW_2021_paper.pdf) [PR]
- **MAV-VID**:
  - Size: 53 training videos (29,500 images) and 11 validation videos (10,732 images).
  - Mean object size is 136×77 px (0.66% of the image), so objects are large and the set is relatively easy.
  - Faster R-CNN reaches 0.978 mAP. — [Isaac-Medina ICCVW 2021](https://openaccess.thecvf.com/content/ICCV2021W/AntiUAV/papers/Isaac-Medina_Unmanned_Aerial_Vehicle_Visual_Detection_and_Tracking_Using_Deep_Neural_ICCVW_2021_paper.pdf) [PR]
- **Det-Fly**:
  - Size and format: ~13.3k air-to-air images at 3840×2160, taken from a DJI Mavic2 with front, top and bottom views.
  - Target sizes range from 1×1 to 149×95 px.
  - EA-DINO is reported at mAP50 96.6%.
  - Source: [LRDDv3 arXiv 2605.25942](https://arxiv.org/html/2605.25942v1) / [PMC air-to-air](https://pmc.ncbi.nlm.nih.gov/articles/PMC10830470/) [agg via snippet]
- **DUT Anti-UAV**:
  - Visible modality: 10,000 detection images (5,200 train / 2,600 val / 2,200 test) and 20 tracking videos (short- and long-term).
  - Covers 35 UAV types. Backgrounds include sky, dark clouds, jungle, buildings and farmland. Lighting covers day, night, dawn and dusk; weather covers sunny, cloudy and snowy.
  - Sources: [arXiv 2205.10851](https://arxiv.org/abs/2205.10851) (IEEE T-ITS [PR]); [GitHub](https://github.com/wangdongdut/DUT-Anti-UAV)
- **Halmstad multi-sensor dataset** (Svanström, Alonso-Fernandez, Englund):
  - Contents: 650 ten-second videos (365 IR and 285 visible), 90 audio clips and 203,328 annotated frames.
  - Video classes: drone, bird, airplane, helicopter. Audio classes: drone, helicopter, background.
  - Resolution: IR 320×256, visible 640×512.
  - Distance bins follow DRI/Johnson. "Close" runs out to where the target is 15 px wide in IR (identification). "Medium" covers 15 down to 5 px (recognition). "Distant" is below 5 px (detection).
  - License: CC0.
  - Sources: [GitHub](https://github.com/DroneDetectionThesis/Drone-detection-dataset); [Data in Brief / arXiv 2111.01888](https://arxiv.org/pdf/2111.01888) [PR]
- **Other sets surfaced** (not examined in depth):
  - MMAUD (ICRA 2024, multi-modal). — [GitHub](https://github.com/ntu-aris/MMAUD)
  - CST Anti-UAV (thermal tiny-UAV tracking). — [arXiv 2507.23473](https://arxiv.org/pdf/2507.23473)
  - LRDDv3 (high-resolution long-range, with range labels and thermal data, 2026). — [arXiv 2605.25942](https://arxiv.org/html/2605.25942v1)
  - UAVNet-MS (multispectral, 2026). — [arXiv 2605.20963](https://arxiv.org/pdf/2605.20963)
  - RealDroneVision (WACV 2026). — [CVF](https://openaccess.thecvf.com/content/WACV2026/papers/Sivapuram_RealDroneVision_Dataset_and_Architecture_Advancements_for_Small-Object_Drone_Detection_WACV_2026_paper.pdf)
  - SynDroneVision (synthetic). — [arXiv 2411.05633](https://arxiv.org/pdf/2411.05633)

### Inferences
- For a Hailo edge detector, a practical training mix is:
  - DUT Anti-UAV and Halmstad visible for EO;
  - Halmstad IR plus Anti-UAV410/600 for thermal;
  - Drone-vs-Bird, if access is granted, and Halmstad's bird, airplane and helicopter clips as hard negatives.
- MAV-VID's ~0.98 mAP says little about long-range performance because its targets are large.

### Gaps
- For Drone-vs-Bird I could not confirm: the exact current dataset size, the resolution range, the access agreement terms, and the 2025 winning scores. The wordpress and MDPI pages were blocked.
- For Anti-UAV600 I could not confirm the exact video count or resolution (IR in Anti-UAV410 is commonly 640×512, but not confirmed here).
- The license of the Anti-UAV data (as opposed to the code) is not stated explicitly.

## 3. Techniques that work for tiny drones at range

### Takeaway
The proven levers are:
- more pixels on target: higher input resolution, or tiling/SAHI (+5–7 AP inference-only, +13–15 AP with sliced fine-tuning, on aerial benchmarks);
- a P2 (stride-4) head for targets of roughly 4–16 px;
- temporal information: motion cues fused with appearance (YOLOMG, background difference + YOLO), and tracking-by-detection (ByteTrack/BoT-SORT + Kalman) with frame-consistency filtering.

The 2025 Drone-vs-Bird winner and the Anti-UAV multi-UAV entries all combine multi-scale detection with temporal consistency.

### Cited Findings
- **SAHI tiled inference**:
  - Inference-only slicing raises AP by 6.8, 5.1 and 5.3 points for FCOS, VFNet and TOOD on VisDrone/xView.
  - Adding sliced fine-tuning gives cumulative gains of 12.7, 13.4 and 14.5 AP.
  - It integrates with Detectron2, MMDetection and YOLOv5.
  - Source: [SAHI paper arXiv 2202.06934](https://arxiv.org/abs/2202.06934) (ICIP 2022 [PR]); [repo](https://github.com/obss/sahi)
- **P2 head**: a stride-4 head (160×160 feature map at 640 input) keeps detail for ~4×4 px objects. One study reports that adding P2 and removing P5 raised mAP50 by +1.7 points (to 0.308) and mAP50-95 by +0.9 points. Caveat: that figure appears to come from a UAV-view traffic paper, not anti-drone work. — [YOLO-UTD PMC](https://pmc.ncbi.nlm.nih.gov/articles/PMC13306799/); related anti-drone: [Improved YOLO for long range detection of small drones, Sci Rep 2025](https://www.nature.com/articles/s41598-025-95580-z), [YOLO-Drone](https://www.researchgate.net/publication/373544079_YOLO-Drone_An_Optimized_YOLOv8_Network_for_Tiny_UAV_Object_Detection) [agg via snippet]
- **Motion + CNN hybrids**:
  - Background subtraction to propose regions, with a CNN (CaffeNet or VGG) to classify them.
  - A recurrent correlation network extracting motion of tiny objects in 4K video from static cameras.
  - "Background difference + SAG-YOLOv5s" for high-resolution drone detection.
  - TRX CFAR temporal-anomaly detection combined with small spatio-temporal CNNs.
  - Sources: [PMC9527012](https://pmc.ncbi.nlm.nih.gov/articles/PMC9527012/); [PMC12788262](https://pmc.ncbi.nlm.nih.gov/articles/PMC12788262/) [PR, via snippet]
  - YOLOMG fuses a pixel-level motion map with appearance for drone-to-drone detection. — [GitHub](https://github.com/Irisky123/YOLOMG)
- **Tracking for false-alarm rejection**:
  - The 2025 Drone-vs-Bird winner used frame-to-frame consistency post-processing. — [Fraunhofer publica](https://publica.fraunhofer.de/entities/publication/267aa1ec-a62d-4afa-99ad-cc369f57642b)
  - The CVPR 2025 multi-UAV thermal baseline is YOLOv12 + BoT-SORT with ReID. — [CVPRW 2025](https://openaccess.thecvf.com/content/CVPR2025W/Anti-UAV/html/Chen_Strong_Baseline_Multi-UAV_Tracking_via_YOLOv12_with_BoT-SORT-ReID_CVPRW_2025_paper.html)
  - "A Simple Detector with Frame Dynamics is a Strong Tracker" (Anti-UAV 2025). — [arXiv 2505.04917](https://arxiv.org/pdf/2505.04917) [preprint]
  - The edge hailo-uav-tracker uses ByteTrack + Kalman. — [GitHub](https://github.com/pierrosimonestd-cpu/hailo-uav-tracker)
- **IR vs visible**:
  - Svanström's YOLOv2-based thermal-IR detector reached an average F1 of 0.7601 (confidence threshold and IoU 0.5).
  - The system fused IR, visible and acoustic (MFCC+LSTM) sensors on a pan/tilt platform.
  - The IR camera was only 320×256, versus 640×512 visible.
  - Sources: [arXiv 2007.07396 / ICPR 2020](https://arxiv.org/pdf/2007.07396) [PR]; [emergentmind summary](https://www.emergentmind.com/papers/2007.07396) [agg]
- **Pixel size vs detectability**:
  - The DRI/Johnson thresholds used in the Halmstad dataset are ≥15 px wide for identification, 5–15 px for recognition (drone vs other object) and <5 px for detection only. — [arXiv 2111.01888](https://arxiv.org/pdf/2111.01888)
  - Det-Fly targets go down to 1×1 px at 4K. — [LRDDv3](https://arxiv.org/html/2605.25942v1)
- **Quantization cost on small objects**: for YOLOv8n on DUT Anti-UAV, going from fp32 to int8 drops AP_S from 0.418 to 0.358 (−6 points), a larger loss than AP50 (0.894 to 0.875). — [hailo-uav-tracker](https://github.com/pierrosimonestd-cpu/hailo-uav-tracker) [hobby]

### Inferences
- On Hailo-8/8L, SAHI-style tiling multiplies the inference cost by the number of tiles. At ~70 FPS for YOLOv8n-640, a 1280×720 frame split into 2×2 tiles would still run at roughly 15–18 FPS. That estimate is my own arithmetic, not a measurement.
- Motion-gated ROI cropping, meaning a static-camera background difference followed by a CNN classifier on crops, is cheaper and suits CPU+NPU edge splits.
- Small objects lose disproportionately under INT8. This argues for quantization-aware training or mixed precision on small-object heads.

### Gaps
- I found no published curve of detection probability versus drone pixel size (for example Pd at 4, 8 or 16 px) with numbers. Only the DRI thresholds and qualitative claims are available. The Halmstad per-distance-bin F1 breakdown likely exists in Svanström 2021/2022 ([arXiv 2207.01927](https://arxiv.org/pdf/2207.01927)) but was not retrievable.
- I have no quantitative head-to-head of ByteTrack versus BoT-SORT for drone false-alarm rejection.

## 4. Known limitations and failure modes

### Takeaway
The dominant failure mode is confusing birds with drones at small pixel sizes. Others:
- low-contrast or cluttered backgrounds (clouds, buildings, vegetation);
- thermal noise and low contrast in IR;
- tiny targets under 5 px, where a detector can at best say "something is there";
- fast motion and blur;
- domain shift between datasets.

Detection AP on the bird-rich Drone-vs-Bird set remains far lower than on MAV-VID (0.667 vs 0.978 mAP; COCO AP 0.283).

### Cited Findings
- CNNs tend to confuse drones with birds because of similar size and motion, which gives high false positives. Reported mitigations: spatio-temporal attention cut the false-positive rate by ~20%, and a method called HEDD cut bird-induced false alarms by 40% against a baseline. These are secondary claims from a snippet with weak attribution. — [PMC12788262](https://pmc.ncbi.nlm.nih.gov/articles/PMC12788262/) / [T&F 2026](https://www.tandfonline.com/doi/full/10.1080/24751839.2026.2616897) [agg]
- Drone-vs-Bird: "variability of the results underscores the complexity of the task". The dataset mixes static and moving cameras. — [IJCNN 2025 challenge summary](https://publica.fraunhofer.de/entities/publication/267aa1ec-a62d-4afa-99ad-cc369f57642b); [Isaac-Medina 2021](https://openaccess.thecvf.com/content/ICCV2021W/AntiUAV/papers/Isaac-Medina_Unmanned_Aerial_Vehicle_Visual_Detection_and_Tracking_Using_Deep_Neural_ICCVW_2021_paper.pdf)
- Thermal multi-UAV tracking is "inherently challenging due to low contrast, environmental noise, and small target sizes". — [anti-uav.github.io / CVPRW 2025](https://arxiv.org/html/2503.17237)
- Distant drones are usually blurred and occupy very few pixels, so context information is needed. — [PMC9527012](https://pmc.ncbi.nlm.nih.gov/articles/PMC9527012/)
- DUT Anti-UAV deliberately includes dark clouds, buildings and night/dusk conditions as hard cases. — [arXiv 2205.10851](https://arxiv.org/abs/2205.10851)
- Domain shift is serious enough to motivate dedicated benchmarks for domain-adaptive MAV detection. — [arXiv 2403.16669](https://arxiv.org/pdf/2403.16669)
- In Halmstad, airplanes, helicopters and birds are included as distractor classes because they are plausible false alarms. — [GitHub](https://github.com/DroneDetectionThesis/Drone-detection-dataset)

### Inferences
- Bird rejection on the edge is best handled temporally, using track-level trajectory and flight-pattern cues plus persistence thresholds, rather than by a per-frame classifier alone. Below ~5 px, appearance-based discrimination is not physically supported by DRI criteria.
- Insects near the lens and cloud edges appear in the literature mainly as qualitative clutter sources.

### Gaps
- I found no quantified false-alarm rates (for example false alarms per hour) for open systems in real deployments.
- Insects as a failure mode are rarely quantified in the vision literature I could reach.
- Several primary PDFs were unreachable (MDPI Drone-vs-Bird summary, ScienceDirect Halmstad paper, PMC articles, the wosdetc site), so some numbers rely on search snippets.
