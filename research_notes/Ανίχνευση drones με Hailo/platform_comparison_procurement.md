# Edge AI Platforms for Drone Detection: Hailo vs NVIDIA Jetson vs Rockchip, with a Defense-Procurement Lens (as of Sept 2026)

Research note. Web fetches to many primary sources (diu.mil, congress.gov, govinfo.gov, uscode.house.gov, law.cornell.edu, fcc.gov, NVIDIA forums, Wikipedia, morganlewis.com) were blocked by the sandbox egress proxy. Several legal and export points therefore rest on search-result snippets and secondary summaries, not on statute text I read myself. These are marked **[snippet]** and should be checked against the primary text before anyone relies on them.

---

## 1. Hardware specs: TOPS, power, price, RAM, video, cameras, software, model flexibility

### Takeaway
On paper, Jetson Orin Nano Super / Orin NX have the most compute, flexibility (FP16/FP32, CUDA, TensorRT) and I/O, at 7–40 W and about $250–$700+. Hailo-8/8L/10H are INT8/INT4-only PCIe/M.2 accelerators at about 2.5 W typical. They need a host SoC for cameras, encoding and decoding. Rockchip RV1106 (0.5 TOPS, about $13–$20 boards) and RK3588 (6 TOPS) are cheap, integrated SoCs with ISP and codecs, but they come from China. NVIDIA's headline TOPS are **sparse** INT8 figures. Hailo's and Rockchip's are dense, so the headline numbers overstate Jetson by about 2x in a like-for-like comparison.

### Comparison table (vendor claims unless noted)

| Platform | AI TOPS (vendor) | Typical / max power | Price | RAM | Video enc/dec | Camera I/F | Precision / flexibility | Origin (design / fab) |
|---|---|---|---|---|---|---|---|---|
| **Hailo-8** (M.2) | 26 TOPS INT8 | 2.5 W typical | M.2 standalone price not found. Raspberry Pi AI HAT+ 26T variant exists | None on chip (uses host RAM via PCIe) | None (accelerator only) | None (host provides) | INT8 (dataflow compiler). Unsupported ops fall back to host | Israel / TSMC 16nm |
| **Hailo-8L** (M.2) | 13 TOPS INT8 | ~1.5–2.5 W (not verified separately) | $70 Raspberry Pi AI Kit (M.2 HAT+ plus Hailo-8L) | None on chip | None | None | INT8 | Israel / TSMC (node for 8L not confirmed) |
| **Hailo-10H** (M.2 2242/2280, USB) | 40 TOPS INT4 / ~20 TOPS INT8 | <2.5 W typical, up to 8.25 W | Not found in sources | 4 or 8 GB LPDDR4/4X on module (dedicated DDR) | None | None | INT4/INT8. Aimed at LLM/VLM. -40 to 85 °C | Israel / fab not confirmed |
| **Jetson Orin Nano 8GB (Super mode)** | 67 TOPS (sparse INT8). Original 40 sparse / 20 dense | 7–25 W | Dev kit $249 (was $499). Module 8GB $299 @1k units | 8 GB LPDDR5, 102 GB/s | Orin Nano has **no NVENC** (software encode). HW decode present (encode caveat not verified this session, see Gaps) | Dev kit: 2x MIPI CSI (up to 4-lane) | FP32/FP16/INT8, CUDA, TensorRT, DeepStream. Broadest op coverage | US (NVIDIA) / TSMC |
| **Jetson Orin NX 16GB (Super)** | 157 TOPS (Super, sparse). 100 TOPS original | 10–40 W (Super) | Module price not verified here. Third-party dev kits sold | 16 GB LPDDR5 128-bit, 102.4 GB/s | HW encode + decode | Up to 8 CSI lanes, up to 6 active streams (module) | Same as above. 1024-core Ampere, 32 Tensor Cores, 8x A78AE | US / TSMC |
| **Rockchip RV1106 (Luckfox Pico)** | 0.5 TOPS (INT4/INT8/INT16) | ~1 W class (not verified) | Luckfox Pico Pro/Max from $12.71 (AliExpress), Pico Ultra from $18 | 64–256 MB in-package DDR (Pico Pro 128 MB) | H.264/H.265 encode. ISP 4MP@30 | MIPI CSI 2-lane (board). SoC: 2x MIPI/LVDS + DVP | INT8/INT16 via RKNN. Small models only | China (Rockchip, Fuzhou) / fab not confirmed |
| **Rockchip RK3588** | 6 TOPS (INT4/INT8/INT16/FP16), 3 NPU cores | ~5–10 W board (not verified) | SoMs/SBCs ~$100–$200 (not verified). Module examples exist | Up to 16–32 GB LPDDR4X/5 | 8K60 H.265/VP9 decode, 8K30 H.264, 4K60 AV1. 8K encode | Up to 4x 4-lane MIPI CSI | RKNN toolkit. FP16 supported on NPU | China / 8nm (Samsung per common reporting; not verified) |

### Cited Findings
- Hailo-10H: 40 TOPS INT4 (~20 TOPS INT8), <2.5 W typical and up to 8.25 W, PCIe Gen3 x4, M.2 2242/2280, 4/8 GB LPDDR4/4X, -40 to 85 °C — [Hailo-10H M.2 datasheet](https://hailo.ai/hailo-files/hailo-10h-m-2-key-m-et-datasheet-en/); [AAEON Hailo-10H M.2](https://www.aaeon.com/en/product/detail/ai-modules-hailo-10h-m-2-2280); [CNX Software](https://www.cnx-software.com/2024/04/04/hailo-10-m-2-key-m-module-brings-generative-ai-to-the-edge-with-up-to-40-tops-of-performance/)
- Hailo-8 M.2: 26 TOPS, 2.5 W typical — [Hailo-8 M.2 product page](https://hailo.ai/products/ai-accelerators/hailo-8-m2-ai-acceleration-module/)
- Hailo-8L M.2: 13 TOPS, Key B+M and A+E — [Hailo-8L M.2 page](https://hailo.ai/products/ai-accelerators/hailo-8l-m-2-ai-acceleration-module-for-ai-light-applications/)
- Raspberry Pi AI Kit (M.2 HAT+ plus 13 TOPS Hailo-8L) costs $70. The AI HAT+ comes in 13 and 26 TOPS variants (Hailo-8L / Hailo-8) — [Raspberry Pi news](https://www.raspberrypi.com/news/raspberry-pi-ai-kit-available-now-at-70/); [CNX Software](https://www.cnx-software.com/2024/06/04/70-raspberry-pi-ai-kit-combines-official-m-2-hat-with-hailo-8l-ai-accelerator/)
- Orin Nano Super Dev Kit: 67 TOPS, $249 (the previous kit was 40 TOPS at $499), 6-core A78AE @1.7 GHz, 8 GB, 102 GB/s, module configurable from 7 to 25 W, 2x MIPI CSI connectors — [NVIDIA Technical Blog](https://developer.nvidia.com/blog/nvidia-jetson-orin-nano-developer-kit-gets-a-super-boost/); [Tom's Hardware](https://www.tomshardware.com/tech-industry/artificial-intelligence/nvidia-launches-new-usd249-ai-development-board-that-does-67-tops)
- Orin Nano 8GB module costs $299 at 1k units. The original rating was "40 Sparse or 20 Dense TOPs" — [Wikipedia: Nvidia Jetson](https://en.wikipedia.org/wiki/Nvidia_Jetson) [snippet; page fetch blocked]
- Orin NX 16GB: 1024-core Ampere, 32 Tensor Cores, 8-core A78AE, 16 GB LPDDR5 at 102.4 GB/s. Super mode gives up to 157 TOPS at 10–40 W. CSI has 8 pairs / 20 Gbps aggregate with up to 6 active streams — [NVIDIA Orin NX data sheet](https://developer.nvidia.com/downloads/jetson-orin-nx-series-data-sheet); [Waveshare Orin NX](https://www.waveshare.com/jetson-orin-nx.htm)
- RV1106: Cortex-A7 up to 1.2 GHz, RISC-V co-processor, 0.5 TOPS NPU (INT4/INT8/INT16), 4MP@30 ISP, H.264/H.265 encoder, 2x MIPI CSI/LVDS plus DVP. Luckfox Pico Pro/Max start at $12.71 and Pico Ultra at $18 — [CNX Software](https://www.cnx-software.com/2024/02/29/luckfox-pico-pro-pico-max-rockchip-rv1106-boards-100m-ethernet-5mp-camera/); [Luckfox Pico Pro](https://www.luckfox.com/EN-Luckfox-Pico-Pro); [Electronics-Lab Pico Ultra](https://www.electronics-lab.com/luckfox-pico-ultra/)
- RK3588: 8 nm, 4x A76 + 4x A55, 6 TOPS NPU (INT4/INT8/INT16/FP16), 8K60 H.265/VP9 decode, 8K30 H.264, 4K60 AV1, up to 4x 4-lane MIPI-CSI — [Rockchip RK3588 datasheet (FriendlyELEC mirror)](https://wiki.friendlyelec.com/wiki/images/e/ee/Rockchip_RK3588_Datasheet_V1.6-20231016.pdf); [Rockchips.net](https://rockchips.net/product/rk3588/)
- RK3588 SoMs exist in a Jetson Nano-compatible 260-pin form factor (ArmSoM AI Module7), which could allow swapping onto Jetson carrier boards — [Crowd Supply](https://www.crowdsupply.com/armsom/rk3588-ai-module7)
- Hailo Model Zoo supports Hailo-8/8L on the v2.x branch and Hailo-10H/15L on master. Toolchains are split per device family — [Hailo Model Zoo GitHub](https://github.com/hailo-ai/hailo_model_zoo)

### Inferences
- **TOPS are not comparable across vendors.** NVIDIA's 67/157 TOPS are sparse figures, so dense INT8 is about 33/78. Hailo's 26 TOPS is dense and its 10H 40 TOPS is INT4. Rockchip's figures are INT8 (RK3588's includes 3 cores). Measured FPS (Section 2) is the better comparison.
- Hailo parts are **accelerators, not SoCs**. BOM, power and origin analysis must include the host (Raspberry Pi CM/Pi 5 = UK design, Broadcom SoC; or an x86/Arm industrial host). With a Rockchip host, the system would carry a Chinese SoC even though the accelerator is Israeli.
- Jetson is the only option here with native FP16/FP32 execution and CUDA. It is the most flexible for custom ops, trackers, and multi-model pipelines such as detection plus tracking plus re-ID or RF fusion.

### Gaps
- Standalone Hailo-8 / Hailo-10H M.2 prices were not found. Hailo sells through distributors and quotes.
- Orin NX 16GB module 1k-unit price was not verified this session.
- Orin Nano's lack of a hardware video encoder (NVENC) is widely reported, but I could not verify it on the NVIDIA datasheet this session because the fetch was blocked. Confirm before relying on it for a downlink-encode design.
- The RK3588 foundry (commonly reported as Samsung 8nm), the RV1106 foundry, and real-world power numbers for Rockchip boards were not verified.

---

## 2. Independent YOLO FPS comparisons

### Takeaway
No single independent, same-methodology study covering all of Hailo-8/8L/10H, Orin Nano/NX, RV1106 and RK3588 was found. Available numbers are scattered and use different methods, which are not comparable: NPU-only vs end-to-end, batch size, resolution, PCIe vs USB. Vendor model-zoo numbers can be 2–3x higher than community-reproduced ones.

### Cited Findings
- **Idein aicast benchmark (independent-ish; Idein sells the Hailo-8-based "ai cast")**, YOLOX-S: Orin Nano TensorRT FP16 97.8 FPS, INT8 126.3 FPS. Hailo-8 INT8 250.5 FPS. mAP50-95 was 35.2 for Orin FP16, 31.0 for Orin INT8 and 34.3 for Hailo-8 INT8. Power under benchmark was Orin Nano 12.7 W vs ai cast (CM4 + Hailo-8) 7.8 W. Setup was JetPack 5.1.2 / TensorRT 8.5.2 (pre-Super) and HailoRT 4.10 — [Idein aicast_jetson_benchmark](https://github.com/Idein/aicast_jetson_benchmark). Caveat: the author has a commercial interest in Hailo. FPS comes from trtexec/hailortcli, i.e. inference only.
- **RK3588 (community, GitHub issue on airockchip/rknn_model_zoo)**: YOLOv8n INT8 on all 3 NPU cores gives 199 FPS @640, 330 @480 and 649 @320. YOLO11s gives 85 FPS @640. INT8 cost 0.5–1.5 mAP (YOLOv8n @640: 35.9 INT8 vs 37.3 FP32). The pipeline used MPP decode + RGA resize with zero-copy — [rknn_model_zoo issue #454](https://github.com/airockchip/rknn_model_zoo/issues/454). Caveat: this is one user on a Vicharak Axon board at max clocks. Throughput is aggregated across 3 cores, and single-stream latency will be worse.
- **Hailo-8L YOLOv8n**: Model Zoo v2.13 claims 182 FPS at batch 1 and 464 FPS at batch 8. A community user measured only 61.9 FPS (b=1) and 121.4 FPS (b=8) — [Hailo Community thread](https://community.hailo.ai/t/yolov8n-real-performance-didnt-expected-from-hailo-model-zoo-documentation/18904). Host interface matters: the Pi 5 has a PCIe Gen2/3 x1 link.
- **Hailo-10H YOLOv8m** over USB 3.1: 39.67 FPS measured vs 49.9 FPS Model Zoo reference — [Hailo Community](https://community.hailo.ai/t/yolov8m-fps-lower-than-model-zoo-on-ugen300-hailo-10h/19320)
- A Raspberry Pi 5/CM4 + Hailo-8L YOLOv8s benchmark thread exists — [Raspberry Pi Forums](https://forums.raspberrypi.com/viewtopic.php?t=373867) (figures not extracted)
- YOLOv8 DeepStream benchmarks exist for Orin Nano 4GB/8GB, NX and TX2 — [Medium, Maro JEON](https://medium.com/@MaroJEON/yolov8-jetson-deepstream-benchmark-test-orin-nano-4gb-8gb-nx-tx2-f3993f9c8d2f) (figures not extracted)

### Inferences
- For a single nano-class detector at 640 px, all of Hailo-8, Orin Nano and RK3588 plausibly exceed 60–100 FPS. The bottleneck for drone detection is usually **resolution/tiling** (small targets need high-res input or SAHI-style tiles), not raw FPS at 640. Dense-TOPS-per-watt favours Hailo. Flexibility for tiling pipelines favours Jetson.
- RV1106 (0.5 TOPS) is roughly 12x below RK3588 in NPU compute. Expect it to run YOLOv5n/v8n at low resolution (e.g. 320) at modest rates, adequate only for near-range or cueing tasks. No independent RV1106 YOLOv8n FPS figure was found in this session.

### Gaps
- No independent RV1106 YOLOv8n/YOLOv5n FPS measurement was found.
- No independent Orin Nano **Super**-mode YOLOv8n FPS with methodology was captured. Most published numbers predate JetPack 6.2 Super mode.
- No independent Hailo-10H vs Orin Nano Super head-to-head was found.
- No academic paper comparing all these accelerators was located this session.

---

## 3. Country of origin, and what US drone laws say about components

### Takeaway
Rockchip (China) components are squarely a problem under US DoD UAS procurement law and the Dec-2025 FCC Covered List action. Hailo (Israel, TSMC-fabbed) and NVIDIA (US, TSMC-fabbed) are not from "covered foreign countries". The US statutes enumerate specific critical components: flight controllers, radios, data transmission devices, cameras, gimbals, ground control systems, operating software, and network/data storage. A standalone AI compute module is **not explicitly named** in the snippets available. Whether an onboard AI computer counts as a "flight controller", as "operating software", or falls outside the list is a matter of interpretation that I could not resolve from primary text.

### Cited Findings
**Law text / regulation (as reported; primary text fetch blocked):**
- Sec. 848 of the FY2020 NDAA (now codified at 10 U.S.C. 4872) prohibits DoD from operating or procuring UAS manufactured in a covered foreign country or by an entity domiciled there. It also bars UAS that use flight controllers, radios, data transmission devices, cameras or gimbals made in a covered foreign country, and UAS that use a ground control system or operating software developed there — [DIU "FY20 NDAA Sec 848 Component Definition Guidance"](https://www.diu.mil/blue-uas-policy) [snippet]; [10 U.S.C. 4872 (govinfo PDF, 2024 ed.)](https://www.govinfo.gov/content/pkg/USCODE-2024-title10/pdf/USCODE-2024-title10-subtitleA-partV-subpartI-chap385-subchapIII-sec4872.pdf) [snippet]
- 10 U.S.C. 4872 also covers UAS that use network connectivity or data storage located in or administered by an entity domiciled in a covered foreign country. It names DJI as a "covered unmanned aircraft system company". From 1 Oct 2024, DoD may not contract with entities that operate equipment from such a company in performing a DoD contract — [10 U.S.C. 4872 govinfo](https://www.govinfo.gov/link/uscode/10/4872) [snippet]
- "Covered foreign country" was originally China. The FY2023 NDAA (Sec. 817) added Russia, Iran and North Korea, extended the restriction to certain counter-UAS equipment, and reached contractors and subcontractors — [Advexure summary](https://advexure.com/pages/ndaa-compliant-blue-uas-drones); [PilieroMazza FY2023 NDAA summary](https://www.pilieromazza.com/congress-passes-fy2023-ndaa-and-implements-significant-changes-to-federal-procurement-policy/) [secondary interpretations; statute text not read]
- **American Security Drone Act of 2023** (FY2024 NDAA Title XVIII, Subtitle B): "covered unmanned aircraft system" takes the meaning of "unmanned aircraft system" in 49 U.S.C. 44801. It includes associated elements related to collection and transmission of sensitive information (communication links and components that control the aircraft). The Federal Acquisition Security Council maintains a list of associated elements — [H.R.6143 text](https://www.congress.gov/bill/118th-congress/house-bill/6143/text) [snippet]. The FAR implementation is FAR 52.240-1 (Nov 2024) — [Federal Register](https://www.federalregister.gov/documents/2024/11/12/2024-26061/federal-acquisition-regulation-prohibition-on-unmanned-aircraft-systems-from-covered-foreign); [FAR 52.240-1](https://www.acquisition.gov/far/52.240-1)
- **FCC Covered List (22 Dec 2025)**: the FCC added all foreign-produced UAS and "UAS critical components". Critical components are defined as "including but not limited to" data transmission devices, communications systems, flight controllers, ground control stations and UAS controllers (the list continues; full text not retrieved). Exemptions run until 1 Jan 2027 for items on DCMA's Blue UAS Cleared List and for "domestic end products" under the Buy American standard. Previously authorized equipment and DoW/DHS waivers are also exempt — [Pillsbury](https://www.pillsburylaw.com/en/news-and-insights/fcc-categorical-prohibition-foreign-produced-uas-critical-components.html); [Holland & Knight](https://www.hklaw.com/en/insights/publications/2025/12/fcc-adds-all-foreign-made-drones); [Wiley](https://www.wiley.law/alert-In-Unexpected-First-of-Its-Kind-Action-FCC-Adds-All-Foreign-Produced-Uncrewed-Aircraft-Systems-and-UAS-Critical-Components-to-Covered-List); [FCC DA 26-22 (Jan 2026 exemptions)](https://docs.fcc.gov/public/attachments/DA-26-22A1.pdf) [snippets]
  - Note: this FCC action covers **all foreign-produced** (not only Chinese) UAS critical components for FCC equipment authorization purposes. An Israeli-made or Taiwanese-assembled component could be affected if it falls within "critical components". Whether an AI accelerator module is a "critical component" is unclear from snippets.

**Origin of each chip:**
- Hailo: Israeli company founded in 2017 by former IDF intelligence-unit members. Hailo-8 is TSMC 16nm, 26 TOPS INT8 at 2.5 W — [Jared Watkins research note on Hailo](https://www.jaredwatkins.com/research/datacenters/edge-ai-accelerators/hailo/); [Wikipedia: Hailo Technologies](https://en.wikipedia.org/wiki/Hailo_Technologies) [snippet]
- NVIDIA: US company. Orin is fabbed at TSMC (widely reported; not verified from a primary source this session).
- Rockchip: Chinese company (Fuzhou). RK3588 is 8nm — [Rockchips.net](https://rockchips.net/product/rk3588/)

**Blue UAS / Framework:**
- Blue UAS list management moved from DIU to DCMA (Unmanned Systems–Experimental Command, Palmdale) on 3 Dec 2025. DCMA emphasises component-level vetting — [DIU announcement](https://www.diu.mil/latest/dius-blue-uas-list-to-transition-to-dcma); [Inside Unmanned Systems interview](https://insideunmannedsystems.com/the-blue-uas-shift-interview-with-dcma-commander/)
- Framework compute: ModalAI VOXL 2 (Qualcomm QRB5165, 8 GB LPDDR5, 15 TOPS, 16 g) and VOXL 2 Mini (11 g, 42x42 mm) are Blue UAS Framework autopilot/companion computers — [ModalAI Blue UAS Framework](https://www.modalai.com/pages/blue-uas-framework); [VOXL 2](https://www.modalai.com/products/voxl-2)
- A secondary mention places Jetson TX2 in a Framework context. I could not confirm that any Jetson Orin carrier/module is a Framework-listed component. The BlackAtlas component-list page was blocked — [ModalAI "Top 5 companion computers"](https://www.modalai.com/blogs/blog/top-5-companion-computers-for-uavs) [unclear]
- Israeli supplier precedent: Elsight's Halo (a connectivity module, **not Hailo**) made the Blue UAS list in 2026, described as a signal for allied (non-US) suppliers — [DroneLife, May 2026](https://dronelife.com/2026/05/01/elsight-halo-blue-uas-allied-suppliers/)
- No evidence found of any **Hailo**-based component on the Blue UAS Framework list.

### Inferences
- **Rockchip (RV1106/RK3588)**: if the Rockchip SoC is also the camera SoC (as on Luckfox, where the ISP and encoder sit on the RV1106) or handles data transmission, it very likely hits the "cameras" / "data transmission devices" language of 10 U.S.C. 4872 for DoD use. Even as a pure compute node, it is a China-origin part that DCMA vetting and most US/allied defense customers will reject. **Interpretation, not legal conclusion.** It remains usable for R&D, prototyping and ground-truth work, and for non-DoD, non-federal customers where not prohibited.
- **Hailo and NVIDIA** are not covered-foreign-country origin, so they are not directly barred by Sec. 848/817 or ASDA. The practical path to acceptance is being part of a Blue-listed or Framework system. On Hailo, the path is to integrate into a vetted host. The FCC "all foreign-produced" action may add friction for non-US parts (including Hailo) if they are deemed "critical components" requiring FCC authorization, since a passive accelerator without RF may not need FCC equipment authorization. This is uncertain and should be checked with counsel.
- For ground-station counter-UAS sensors, the FY2023 NDAA counter-UAS extension (as summarised) makes China-origin compute in a C-UAS sensor node a probable procurement blocker for DoD.

### Gaps
- Verbatim text of 10 U.S.C. 4872 / Sec. 817 was not read (all sources blocked). Whether "flight controller" or "operating software" has been interpreted to include companion/AI computers is not established.
- The full FCC "UAS critical components" list (cameras? gimbals? onboard computers?) was not retrieved.
- The current DCMA Blue UAS Framework list (2026) was not retrieved. I could not confirm which Jetson-based boards are listed.

---

## 4. Export controls (EAR/ITAR, Israel, 2024–2026)

### Takeaway
NVIDIA publicly classifies Jetson products as ECCN **5A992.c** (mass-market encryption, not "advanced computing" 3A090/4A090). Under that classification they generally ship license-free to most destinations except embargoed ones and restricted end-uses or end-users. No Hailo ECCN was found in public sources. Hailo is Israeli, so Israeli export rules (Defense Export Control Law / Wassenaar-based dual-use regime) apply to its exports, plus EAR re-export rules if US content exceeds de minimis. No 2024–2026 US rule specifically restricting Jetson Orin edge modules was found.

### Cited Findings
- The Jetson FAQ lists ECCN 5A992.c for Jetson products, and classification queries go to NVClassification@nvidia.com — [NVIDIA Jetson FAQ](https://developer.nvidia.com/embedded/faq); [NVIDIA forum: Jetson ECCN codes](https://forums.developer.nvidia.com/t/jetson-eccn-codes/216341/3) [snippet]; [NVIDIA ECCN/HTS lookup](https://nvidia.custhelp.com/app/answers/detail/a_id/2863/~/how-can-i-obtain-the-eccn-or-hts-number-for-nvidia-products)
- The Dec 2024 BIS rules strengthened controls on advanced computing items and semiconductor manufacturing items. They target high-performance datacenter-class compute (3A090/4A090 thresholds) — [Holland & Knight, Dec 2024](https://www.hklaw.com/en/insights/publications/2024/12/us-strengthens-export-controls-on-advanced-computing-items)
- No Hailo ECCN was found in any accessible source (search returned only generic ECCN pages) — [BIS ECCN overview](https://www.bis.doc.gov/index.php/licensing/commerce-control-list-classification/export-control-classification-number-eccn)

### Inferences
- A 5A992.c item integrated into a military UAV does not become ITAR by itself. However, a system "specially designed" for military use may be ITAR/USML Category VIII/XI or ECCN 9A610/9A012 at the system level. **Classification applies per item and system; I cannot conclude it here.**
- End-use/end-user controls still apply: EAR Part 744 military end-use/user rules for China, Russia and others, and Entity List screening. Selling Jetson-based drone detectors to covered destinations needs screening even at 5A992.c.

### Gaps
- Hailo's ECCN / Israeli dual-use classification was not found. Ask Hailo directly.
- I could not verify whether any 2025–2026 BIS rule (e.g. the rescinded AI Diffusion rule or later replacements) touched Jetson Orin/Thor. The NVIDIA forum thread on Orin Nano export control was blocked.

---

## 5. EU/NATO guidance on Chinese components

### Takeaway
The EU has no bloc-wide legal ban comparable to US Sec. 848 on Chinese drone components. Restrictions are national, e.g. Lithuania and the Netherlands for military procurement. EU initiatives in 2025–2026 focus on capability and production (Readiness Roadmap 2030, EDDI/"drone wall", Feb-2026 Action Plan on Drone and Counter-Drone Security) rather than component-origin rules. Chinese export restrictions and sanctions on EU drone supply chains are pushing European buyers toward "non-red" (e.g. Taiwanese) supply in practice.

### Cited Findings
- Lithuania and the Netherlands have banned military procurement of Chinese drones (Lithuania also across all public-sector agencies). Lithuania is the only EU state with a policy of decoupling defense supply chains from China. For the EU as a whole, decoupling is not a priority — [RUSI, Nov 2025, "Drones: Decoupling Supply Chains from China"](https://static.rusi.org/rp-drone-supply-chains-china-nov-2025_0.pdf) [snippet]
- The European Commission adopted the Readiness Roadmap 2030 (Oct 2025), launched EDDI (ex-"Drone Wall"), and published an Action Plan on Drone and Counter-Drone Security (Feb 2026) — [Capstone DC](https://capstonedc.com/insights/national-security-insights/new-threats-drive-european-drone-and-counter-drone-demand/) [secondary]
- China sanctioned elements of the EU military/drone supply chain in 2026 — [The Diplomat, Apr 2026](https://thediplomat.com/2026/04/chinas-sanctions-hit-europes-emerging-drone-doctrine/)
- Taiwan is promoting "non-red" drone supply chains, including cooperation with Lithuania — [DSET](https://dset.tw/en/research/the-invisible-drone-wall-taiwans-quiet-support-for-a-china-free-european-drone-supply-chain/); [Focus Taiwan, May 2026](https://focustaiwan.tw/sci-tech/202605190019)

### Inferences
- For EU/NATO defense customers, Rockchip-based designs carry procurement and reputational risk and possible supply risk from Chinese export controls, even where not legally barred. NVIDIA and Hailo are generally acceptable origins. Hailo, being Israeli, may face political/procurement sensitivities in some European states. **This is speculation; no source found.**

### Gaps
- No NATO-level (e.g. NSPA/STANAG) guidance on component origin was found.
- National rules for Greece (the user's likely context), Germany, France and the UK were not researched in this pass.

---

## 6. SWaP-C fit: when each platform makes sense

### Takeaway
- **RV1106 / Luckfox Pico** (~$13–20, 0.5 TOPS, all-in-one camera SoC): best for R&D prototypes, cheap attritable demonstrators, or low-res cueing sensors. It is **not suitable for US DoD/NDAA-sensitive deliverables** (China-origin camera SoC).
- **RK3588** (6 TOPS, rich codecs, 4x CSI): strong price/performance for multi-camera ground nodes in commercial or non-restricted markets. Same origin problem for defense.
- **Hailo-8L/8/10H + non-Chinese host** (~2.5 W accelerator, dense INT8): best TOPS/W for small or medium UAVs and battery ground nodes where the model set is fixed and INT8-quantizable. The price for that efficiency is that you must build or choose a compliant host (camera, ISP, encode), and toolchain lock-in (Hailo Dataflow Compiler, device-family branches).
- **Jetson Orin Nano Super / Orin NX** (7–40 W, CUDA/TensorRT, FP16): best for larger ISR UAVs and ground stations needing multi-model pipelines, tiling for small targets, tracking and sensor fusion, and fast model iteration. It has the most mature ecosystem (DeepStream, Isaac, ROS 2) and a US origin, but the highest power and cost.

### Cited Findings
- Measured power in one benchmark: Orin Nano about 12.7 W vs CM4 + Hailo-8 about 7.8 W for YOLOX-S. Hailo-8 had about 2x the FPS in that test — [Idein aicast benchmark](https://github.com/Idein/aicast_jetson_benchmark)
- The Orin Nano power envelope is 7–25 W and Orin NX Super's is 10–40 W — [NVIDIA Technical Blog](https://developer.nvidia.com/blog/nvidia-jetson-orin-nano-developer-kit-gets-a-super-boost/); [Waveshare Orin NX](https://www.waveshare.com/jetson-orin-nx.htm)
- Blue UAS Framework compute exists today in the 11–16 g class (VOXL 2 / Mini, 15 TOPS). This is the incumbent for US-DoD small UAS — [ModalAI VOXL 2 Mini](https://www.modalai.com/products/voxl-2-mini)
- Industrial temperature (-40 to 85 °C) Hailo-10H modules exist — [Hailo-10H datasheet](https://hailo.ai/hailo-files/hailo-10h-m-2-key-m-et-datasheet-en/)

### Inferences
- A defense-oriented migration path from a Luckfox RV1106 prototype would be: keep the same YOLO model family, retrain and quantize for Hailo (with a US/EU host) for low-SWaP airborne use, or for Jetson Orin for ground stations. The alternative is to target VOXL 2 (Blue Framework) if a US DoD customer is the goal.
- For small FPV/attritable use, unit cost and grams dominate. Jetson modules (~$299 at 1k, plus carrier and heatsink) are hard to justify, and a Hailo-8L (~$70 kit-level) on a compliant low-cost host is more plausible. For ISR UAVs over 5 kg or fixed ground nodes, Jetson's flexibility outweighs its power draw.

### Gaps
- Weight and thermal data for each module was not collected.
- No sourced unit cost at volume for Hailo modules. No verified cost of a compliant (non-Chinese) Hailo host board.
