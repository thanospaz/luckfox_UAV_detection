# Μεταφέρετε τον ανιχνευτή drones στο Hailo

**Σύντομη απάντηση.** Δεν υπάρχει ένα έτοιμο open-source project που να δίνει μαζί εκπαιδευμένο μοντέλο, dataset, tracking και ενσωμάτωση σε edge hardware για ανίχνευση drones. Χτίζετε από τρία κομμάτια:

- **Datasets:** DUT Anti-UAV για οπτικό φάσμα, Anti-UAV300/410/600 για θερμικό, και το σύνολο Halmstad (CC0) για ψεύτικους στόχους όπως πουλιά, αεροπλάνα και ελικόπτερα.
- **Τεχνικές:** τρεις δοκιμασμένες μέθοδοι. Tiling, δηλαδή τεμαχισμός του κάδρου. Κεφαλή P2 για πολύ μικρούς στόχους. Και tracking με επιβεβαίωση ίχνους.
- **Εργαλεία Hailo:** το `hailo-apps` της Hailo έχει ήδη έτοιμα tracker και tiling.

Για ανίχνευση με ένα μικρό μοντέλο, το **Hailo-8 (26 TOPS, ~2,5–3,2 W)** είναι η καλύτερη επιλογή απόδοσης ανά watt από όσα έχετε στο εργαστήριο. Το Hailo-8L επαρκεί για 30 fps χωρίς tiling. Το Hailo-10H αξίζει μόνο αν χρειάζεστε και LLM/VLM στο σκάφος. Το Jetson Orin κερδίζει σε ευελιξία (FP16, CUDA, πολλά μοντέλα μαζί) και χάνει σε κατανάλωση. Τα Rockchip (RV1106/RK3588) είναι φθηνά και ολοκληρωμένα, αλλά η κινεζική τους προέλευση τα κάνει πρακτικά απορριπτέα σε αμυντικές προμήθειες ΗΠΑ και πολύ πιθανώς σε όσους πελάτες του NATO ακολουθούν τη λογική των ΗΠΑ.

Η μεταφορά του `luckfox_UAV_detection` είναι κυρίως επανεγγραφή και όχι μετάφραση κώδικα:

- **Ξαναγράφονται:** λήψη εικόνας, προεπεξεργασία, inference, encoding και RTSP.
- **Μεταφέρονται αυτούσια:** η καταγραφή σε flash και η λογική των κατωφλίων.
- **Ξαναφτιάχνεται:** το MAVLink, με τυποποιημένα μηνύματα.

Πριν ξεκινήσετε, διορθώστε δύο επιβεβαιωμένα bugs στο repo:

1. Αναντιστοιχία στο `#if` που προστατεύει τον κώδικα του UART.
2. Το custom MAVLink πακέτο δεν υπολογίζει το CRC_EXTRA, οπότε κάθε τυποποιημένος parser θα το απορρίψει.

**Ο εξοπλισμός σας (επιβεβαιωμένος): Raspberry Pi 5 + Raspberry Pi AI HAT+ 26 TOPS, δηλαδή Hailo-8.** Το πλάνο της §6 είναι γραμμένο γι' αυτή τη διάταξη και προϋποθέτει τα εξής:

- **Γραμμή λογισμικού Hailo-8:** Model Zoo v2.x, DFC 3.x, HailoRT 4.x, πακέτο `hailo-all`, μεταγλώττιση με `--hw-arch hailo8`.
- **PCIe:** Gen3 x1, που το HAT+ ενεργοποιεί αυτόματα.
- **Encoding:** software, γιατί το Pi 5 δεν έχει hardware encoder H.264.
- **Σειριακή προς Pixhawk:** `/dev/ttyAMA0`.

**Η μεταγλώττιση του μοντέλου δεν γίνεται στο Pi.** Γίνεται σε ξεχωριστό x86 Linux PC, ιδανικά με NVIDIA GPU. Για Hailo-8L, Hailo-10H ή host x86 υπάρχουν μόνο σύντομες πλάγιες σημειώσεις.

---

## 0. Πώς να διαβάσετε τις πηγές: τρία επίπεδα αξιοπιστίας

**Η βασική ιδέα.** Κάθε νούμερο στην αναφορά έχει «βαθμό διαβάθμισης πληροφορίας», όπως στις αναφορές πληροφοριών (A1, B2 κ.λπ.). Η ίδια τιμή FPS σημαίνει άλλο πράγμα αν τη δίνει ο κατασκευαστής, άλλο αν τη μέτρησε ανεξάρτητος τρίτος και άλλο αν βρέθηκε μόνο σε απόσπασμα μηχανής αναζήτησης.

Η αναφορά χρησιμοποιεί τις εξής ετικέτες:

- **[vendor]**: Hailo, NVIDIA ή Rockchip. Συνήθως μετρημένο υπό ιδανικές συνθήκες. Για παράδειγμα, όλα τα FPS του Hailo Model Zoo μετρήθηκαν σε x86 με PCIe Gen3 x4 και μετρούν μόνο το chip, χωρίς προ- και μετα-επεξεργασία.
- **[ανεξάρτητο]**: μέτρηση τρίτου με γνωστή μεθοδολογία.
- **[hobby]**: μεμονωμένο ή φοιτητικό repo. Χρήσιμο, αλλά χωρίς peer review.
- **[snippet]**: η σελίδα δεν ανοίχτηκε απευθείας, επειδή το δίκτυο της έρευνας μπλόκαρε hailo.ai, community.hailo.ai, raspberrypi.com, jeffgeerling.com, diu.mil, congress.gov και άλλα. Ό,τι έχει αυτή την ετικέτα πρέπει να ελεγχθεί πριν χρησιμοποιηθεί σε απόφαση.

Τρεις ακόμη παρατηρήσεις για τις πηγές:

- **Νομικά.** Τα νομικά σημεία για NDAA και FCC στηρίζονται σε δευτερογενείς περιλήψεις δικηγορικών γραφείων και όχι στο κείμενο του νόμου. Είναι **ερμηνεία, όχι νομική γνώμη**.
- **Σύγκριση πλατφορμών.** Δεν βρέθηκε καμία ενιαία, ανεξάρτητη μελέτη που να συγκρίνει Hailo, Jetson και Rockchip με την ίδια μεθοδολογία. Κάθε σύγκριση πλατφορμών παρακάτω συνθέτει αριθμούς από διαφορετικές πηγές.
- **Κώδικας repo.** Όσα αφορούν τον κώδικα του repo διαβάστηκαν απευθείας από τα αρχεία και είναι τα πιο αξιόπιστα στοιχεία της αναφοράς.

---

## 1. Τα πιο χρήσιμα ανοιχτά repos είναι benchmarks, όχι έτοιμα προϊόντα

**Η βασική ιδέα.** Τα πολλά «YOLO drone detection» repos στο GitHub είναι κυρίως tutorials. Τα πραγματικά πολύτιμα είναι τρία είδη:

- repos με δεδομένα και μετρικές αξιολόγησης, που λειτουργούν σαν πεδίο βολής για να μετρήσετε το σύστημά σας·
- λίγα ισχυρά ερευνητικά baselines·
- ελάχιστα, μικρά repos που τρέχουν πραγματικά σε edge.

Κανένα δεν καλύπτει όλη την αλυσίδα από τον αισθητήρα μέχρι το MAVLink.

### Ποια repos αξίζουν τον χρόνο σας

Ο πίνακας κρατά μόνο όσα έχουν πρακτική αξία για εσάς. Τα αστέρια και η τελευταία ενημέρωση (push) μετρήθηκαν στις 24/9/2026.

| Repo | Τι δίνει | Άδεια | Κατάσταση | Αξιοπιστία |
|---|---|---|---|---|
| [ZhaoJ9014/Anti-UAV](https://github.com/ZhaoJ9014/Anti-UAV) | Επίσημο repo των Anti-UAV300/410/600, baselines, toolkit αξιολόγησης | MIT (κώδικας) | 855★, τελευταίο push 5/2025 (μέτρια στάσιμο) | peer-reviewed benchmark |
| [wish44165/YOLOv12-BoT-SORT-ReID](https://github.com/wish44165/YOLOv12-BoT-SORT-ReID) | Ανίχνευση και tracking πολλών UAV σε θερμικό, 3η θέση στο CVPR 2025 Anti-UAV | **AGPL-3.0** | 211★, ενεργό | [CVPRW 2025](https://openaccess.thecvf.com/content/CVPR2025W/Anti-UAV/html/Chen_Strong_Baseline_Multi-UAV_Tracking_via_YOLOv12_with_BoT-SORT-ReID_CVPRW_2025_paper.html) |
| [obss/sahi](https://github.com/obss/sahi) | Tiled inference για μικρούς στόχους, ανεξάρτητο από το μοντέλο | MIT | 5.515★, πολύ ενεργό | peer-reviewed (ICIP 2022) |
| [pierrosimonestd-cpu/hailo-uav-tracker](https://github.com/pierrosimonestd-cpu/hailo-uav-tracker) | **Pi 5 + Hailo-8L**, YOLOv8n σε DUT Anti-UAV, ByteTrack + Kalman, pan/tilt με ESP32 | MIT | Μικρό, με CI | hobby, αλλά με καθαρές μετρήσεις |
| [alebal123bal/khadas_yolov8n_multithread](https://github.com/alebal123bal/khadas_yolov8n_multithread) | YOLOv8n για UAV σε NPU RK3588S, pipeline ISP+RGA+NPU | Apache-2.0 | 91★, ενεργό | hobby, αυτοαναφερόμενο «46 FPS» |
| [Irisky123/YOLOMG](https://github.com/Irisky123/YOLOMG) | Συνδυασμός κίνησης (motion map) με εμφάνιση για ανίχνευση drone από drone | GPL-3.0 | 81★ | preprint |
| [lusher00/hailo-tracker](https://github.com/lusher00/hailo-tracker) | Pi 5 + Hailo-8L, ανίχνευση COCO με track IDs, web UI | — | Όχι ειδικά για drones, χωρίς MAVLink | hobby |

Το hailo-uav-tracker είναι το πιο συγγενικό με το δικό σας project, γιατί αποτελεί την ίδια ιδέα σε Hailo. Έχει και ένα διδακτικό εύρημα: ο βρόχος pan/tilt είναι «περιορισμένος από την καθυστέρηση, όχι από τους κινητήρες». Το σφάλμα σκόπευσης ανεβαίνει από **0,68° RMS στα 10 ms σε 2,77° στα 300 ms** καθυστέρηση ([hailo-uav-tracker](https://github.com/pierrosimonestd-cpu/hailo-uav-tracker)). Για όποιον έχει δουλέψει με σύστημα ελέγχου πυρός, αυτό είναι γνώριμο: μετράει ο χρόνος από τον αισθητήρα στην ενέργεια, όχι η ταχύτητα του servo.

Η σημείωση αξιοπιστίας εδώ είναι σημαντική. Το ίδιο το README προειδοποιεί ότι οι αριθμοί του επιταχυντή δεν μετρήθηκαν όλοι πάνω στον δικό του ανιχνευτή. Επίσης, δεν διανέμει βάρη μοντέλου ή HEF.

**Δύο κενά που επηρεάζουν άμεσα τα σχέδιά σας:**

1. Δεν βρέθηκε **κανένα επαληθευμένο δημόσιο repo που να συνδέει Hailo με ArduPilot/PX4 μέσω MAVLink**. Ένα project με PX4 + ROS 2 + Hailo-8L εμφανίζεται σε σελίδες topics του GitHub, αλλά το URL του δεν επιβεβαιώθηκε ([GitHub topic](https://github.com/topics/companion-computer)). Το στρώμα MAVLink θα το γράψετε εσείς.
2. Δεν βρέθηκε ώριμο open-source project ανίχνευσης drones ειδικά για Jetson/DeepStream.

### Τα datasets καλύπτουν διαφορετικές «αποστολές»

**Η βασική ιδέα.** Κάθε dataset είναι σαν διαφορετικό σενάριο άσκησης:

- Άλλο σας εκπαιδεύει σε θερμικό.
- Άλλο σε οπτικό με καθαρό ουρανό.
- Άλλο σε διάκριση drone από πουλί, που είναι το δυσκολότερο.

Ένα μοντέλο που «πέτυχε 98%» σε εύκολο σενάριο δεν λέει τίποτα για το δύσκολο.

| Dataset | Φάσμα | Μέγεθος | Γιατί σας ενδιαφέρει | Πρόσβαση/Άδεια |
|---|---|---|---|---|
| **DUT Anti-UAV** | Οπτικό | 10.000 εικόνες (5.200/2.600/2.200), 20 βίντεο tracking, 35 τύποι UAV | Καθαρό σετ ανίχνευσης με σύννεφα, κτίρια, νύχτα και σούρουπο ([arXiv 2205.10851](https://arxiv.org/abs/2205.10851)) | Apache-2.0 repo, στάσιμο από 2023 |
| **Anti-UAV410** | Θερμικό | 410 βίντεο, >438K πλαίσια | Το benchmark αναφοράς για IR tracking. Κορυφή AUC ~63–64 ([TPAMI](https://dl.acm.org/doi/10.1109/TPAMI.2023.3335338)) | Google Drive/Baidu. Άδεια δεδομένων ασαφής |
| **Anti-UAV300 / 600** | RGB+IR / IR | Full HD βίντεο | Ζεύγη RGB+IR (300), μεγαλύτερος όγκος θερμικού (600) ([GitHub](https://github.com/ZhaoJ9014/Anti-UAV)) | ModelScope για το 600 |
| **Halmstad (Svanström)** | IR + οπτικό + ήχος | 650 βίντεο, 203.328 πλαίσια, κλάσεις drone/πουλί/αεροπλάνο/ελικόπτερο | **Το μόνο πλήρως ανοιχτό (CC0) σετ με ψεύτικους στόχους** και κατηγορίες απόστασης κατά DRI ([GitHub](https://github.com/DroneDetectionThesis/Drone-detection-dataset)) | CC0. Ετικέτες σε .mat, υπάρχει Python decoder |
| **Drone-vs-Bird (WOSDETC)** | Οπτικό | 77 βίντεο (104.760 εικόνες) στη μελέτη του 2021 | Η δυσκολότερη δοκιμή: mAP 0,667 έναντι 0,978 στο MAV-VID ([ICCVW 2021](https://openaccess.thecvf.com/content/ICCV2021W/AntiUAV/papers/Isaac-Medina_Unmanned_Aerial_Vehicle_Visual_Detection_and_Tracking_Using_Deep_Neural_ICCVW_2021_paper.pdf)) | Περιορισμένη πρόσβαση, με συμφωνία |
| **MAV-VID** | Οπτικό | ~40K εικόνες | Μεγάλοι στόχοι (μέσος όρος 136×77 px), άρα εύκολο | Μη αντιπροσωπευτικό για μεγάλες αποστάσεις |
| **Det-Fly** | Οπτικό, αέρος-αέρος | ~13,3K εικόνες 4K | Στόχοι έως 1×1 px, λήψη από άλλο drone ([LRDDv3](https://arxiv.org/html/2605.25942v1)) | [snippet] |

**Προτεινόμενο μείγμα εκπαίδευσης για μονοκλασικό ανιχνευτή σε Hailo:**

1. DUT Anti-UAV και το οπτικό μέρος του Halmstad, ως βάση για οπτικό φάσμα.
2. Anti-UAV410/600 και το IR μέρος του Halmstad, αν θα γίνει θερμική εκδοχή.
3. Πουλιά, αεροπλάνα και ελικόπτερα από το Halmstad (και από το Drone-vs-Bird αν αποκτήσετε πρόσβαση) ως **hard negatives**, δηλαδή εικόνες που το μοντέλο πρέπει να μάθει να *μην* σημαδεύει.
4. **Δικά σας πλάνα από την πραγματική κάμερα**, γιατί η διαφορά ανάμεσα σε datasets (domain shift) είναι τεκμηριωμένα σοβαρό πρόβλημα ([arXiv 2403.16669](https://arxiv.org/pdf/2403.16669)).

---

## 2. Ένα drone στα 5 pixels είναι «κάτι», όχι «drone»

**Η βασική ιδέα.** Τα κριτήρια Johnson/DRI (Detection, Recognition, Identification) που ξέρετε από τα θερμικά σκοπευτικά ισχύουν και εδώ. Το σύνολο Halmstad τα εφαρμόζει ρητά:

- **≥15 px πλάτος:** αναγνώριση ταυτότητας (identification).
- **5–15 px:** διάκριση «drone ή κάτι άλλο» (recognition).
- **<5 px:** μόνο «υπάρχει κάτι» (detection) ([arXiv 2111.01888](https://arxiv.org/pdf/2111.01888)).

Κάτω από τα ~5 px, κανένας ανιχνευτής ενός κάδρου δεν μπορεί φυσικά να ξεχωρίσει drone από πουλί με βάση την εμφάνιση. Η λύση είναι να δείτε **περισσότερα pixels** ή **περισσότερο χρόνο**.

### Τέσσερις μοχλοί που δουλεύουν

**1. Περισσότερα pixels με tiling (SAHI).** Το tiling κόβει το μεγάλο κάδρο σε κομμάτια στο μέγεθος εισόδου του μοντέλου, όπως ένας παρατηρητής σαρώνει τομείς με κιάλια αντί να κοιτά όλο τον ορίζοντα με γυμνό μάτι. Στα αεροφωτογραφικά benchmarks VisDrone/xView:

- Μόνο το tiling στο inference ανέβασε την AP κατά **5–7 μονάδες**.
- Μαζί με fine-tuning πάνω σε κομμάτια, το κέρδος έφτασε **12,7–14,5 μονάδες** ([SAHI, arXiv 2202.06934](https://arxiv.org/abs/2202.06934)).

Το κόστος είναι γραμμικό: 6 tiles σημαίνουν 6 inferences ανά κάδρο.

Το Hailo έχει ενσωματωμένη εκδοχή, το `hailo-tiling` στο hailo-apps, και το προεπιλεγμένο του demo είναι ακριβώς αεροφωτογραφικό (YOLOv8n σε βίντεο VisDrone) ([hailo-apps tiling](https://github.com/hailo-ai/hailo-apps/blob/main/hailo_apps/python/pipeline_apps/tiling/README.md)). Ο κανόνας επικάλυψης είναι `επικάλυψη ≥ μικρότερος στόχος / είσοδος μοντέλου`, δηλαδή 0,05 για στόχους 32 px σε είσοδο 640.

**2. Κεφαλή P2.** Τα YOLO «βλέπουν» σε τρεις κλίμακες (P3–P5). Μια επιπλέον κεφαλή P2 (stride 4) διατηρεί λεπτομέρεια για στόχους ~4–16 px. Μια μελέτη αναφέρει +1,7 μονάδες mAP50 ([YOLO-UTD](https://pmc.ncbi.nlm.nih.gov/articles/PMC13306799/)). Όμως αφορά οδική κυκλοφορία από UAV, όχι anti-drone, και το νούμερο προέρχεται από snippet. **Συνέπεια για το Hailo:** μια τροποποιημένη κεφαλή φεύγει από τα έτοιμα configs του Model Zoo και αυξάνει το ρίσκο στη μεταγλώττιση (βλ. §5).

**3. Κίνηση και εμφάνιση μαζί.** Με σταθερή κάμερα, η αφαίρεση φόντου (background subtraction) προτείνει περιοχές και ένα CNN τις ταξινομεί ([PMC9527012](https://pmc.ncbi.nlm.nih.gov/articles/PMC9527012/)). Το YOLOMG ενώνει χάρτη κίνησης με εμφάνιση ([GitHub](https://github.com/Irisky123/YOLOMG)). Σε edge σύστημα αυτό μοιράζεται φυσικά: το CPU κάνει την ανίχνευση κίνησης και το NPU ταξινομεί μόνο τις υποψήφιες περιοχές.

**4. Tracking ως φίλτρο ψευδών συναγερμών.** Σκεφτείτε το σαν επιβεβαίωση ίχνους σε ραντάρ (M-of-N): ένα «χτύπημα» δεν είναι στόχος, τρία συνεχόμενα είναι. Όλοι οι νικητές του 2025 συνδυάζουν ανίχνευση σε πολλές κλίμακες με χρονική συνέπεια:

- Ο νικητής του Drone-vs-Bird 2025 χρησιμοποίησε επεξεργασία συνέπειας από κάδρο σε κάδρο ([Fraunhofer](https://publica.fraunhofer.de/entities/publication/267aa1ec-a62d-4afa-99ad-cc369f57642b)).
- Το baseline του CVPR 2025 είναι YOLOv12 + BoT-SORT.

Στο Hailo, ο `hailotracker` (Kalman + IoU) είναι ήδη μέρος του pipeline ανίχνευσης του hailo-apps ([detection_pipeline.py](https://github.com/hailo-ai/hailo-apps/blob/main/hailo_apps/python/pipeline_apps/detection/detection_pipeline.py)).

### Δύο παγίδες που μετράνε στο δικό σας project

**Το INT8 πλήττει περισσότερο τους μικρούς στόχους.** Στο DUT Anti-UAV, το YOLOv8n έχασε AP από 0,556 σε 0,518 με την κβάντωση. Το **AP_S (μικροί στόχοι) όμως έπεσε από 0,418 σε 0,358**, ενώ το AP50 μόλις από 0,894 σε 0,875 ([hailo-uav-tracker](https://github.com/pierrosimonestd-cpu/hailo-uav-tracker), [hobby], μέτρηση σε CPU ONNX και όχι σε Hailo). Το 52% των στόχων στο σετ είναι κάτω από 32×32 px.

Η Hailo δίνει για το COCO απώλεια μόλις 0,6–1,5 mAP ([Model Zoo Hailo-8](https://github.com/hailo-ai/hailo_model_zoo/blob/master/docs/public_models/HAILO8/HAILO8_object_detection.rst), [vendor]). Το COCO όμως έχει κυρίως μεγάλους στόχους. **Μην εμπιστευτείτε το vendor νούμερο για τον δικό σας χρήστη.**

**Τα πουλιά είναι ο κύριος εχθρός.** Στο ίδιο project, με κατώφλι 0,35, το 23,4% των φωτογραφιών πουλιών έδωσε ψευδή ανίχνευση. Όταν προστέθηκαν 600 εικόνες πουλιών ως φόντο στην εκπαίδευση, το ποσοστό **έπεσε στο 7,1%**. Είναι η φθηνότερη βελτίωση που υπάρχει.

Ισχυρισμοί όπως «μείωση ψευδών συναγερμών 40% με HEDD» προέρχονται από snippets με ασαφή απόδοση ([PMC12788262](https://pmc.ncbi.nlm.nih.gov/articles/PMC12788262/)). **Μην τους χρησιμοποιήσετε σε προτάσεις ή pitch.**

**Τι δεν υπάρχει στη βιβλιογραφία:** καμία δημοσιευμένη καμπύλη πιθανότητας ανίχνευσης ανά μέγεθος σε pixels (Pd στα 4/8/16 px) και κανένα μετρημένο ποσοστό ψευδών συναγερμών ανά ώρα σε πραγματική ανάπτυξη ανοιχτού συστήματος. Αν τα μετρήσετε εσείς, θα έχετε διαφοροποιητικό στοιχείο.

---

## 3. Το Hailo είναι επιταχυντής, όχι υπολογιστής, και το λογισμικό του έχει χωριστεί στα δύο

**Η βασική ιδέα.** Το Hailo μοιάζει με εξειδικευμένο οπλικό σύστημα χωρίς δικό του σκοπευτικό και τροφοδοσία. Κάνει ένα πράγμα εξαιρετικά αποδοτικά: πολλαπλασιασμούς πινάκων σε INT8, δηλαδή σε ακέραιους 8 bit αντί για δεκαδικούς. Κάμερα, ISP (επεξεργασία εικόνας), κωδικοποίηση βίντεο και λογική εφαρμογής τα παρέχει ένας **host**: Raspberry Pi 5, x86 PC ή άλλη ARM πλακέτα. Συνδέεται μέσω PCIe (κάρτα M.2 ή HAT).

Αντίθετα, το RV1106 στη Luckfox κάνει τα πάντα σε ένα chip, όπως ένα «all-in-one» σύστημα.

### Οι τρεις εκδοχές με μια ματιά

| | Hailo-8 | Hailo-8L | Hailo-10H |
|---|---|---|---|
| Υπολογιστική ισχύς | 26 TOPS INT8 | 13 TOPS INT8 | 40 TOPS INT4 (~20 INT8) |
| Κατανάλωση | 2,5 W τυπική [vendor]. **3,19 W** μετρημένα σε streaming στο παράδειγμα της ίδιας της Hailo ([BENCHMARKS.rst](https://github.com/hailo-ai/hailo_model_zoo/blob/master/docs/BENCHMARKS.rst)) | Δεν επαληθεύτηκε χωριστά | <2,5 W τυπική, **έως 8,25 W** κατά datasheet ([Hailo-10H datasheet](https://hailo.ai/hailo-files/hailo-10h-m-2-key-m-et-datasheet-en/), [snippet]). Ο Geerling αναφέρει «μέγιστο 3 W» [snippet]: **οι πηγές διαφωνούν** |
| Μνήμη | Όχι, χρησιμοποιεί του host | Όχι | 4 ή 8 GB LPDDR4 στο module |
| Μορφή | M.2 2230 A+E (PCIe x2) / 2280 B+M (x4), AI HAT+ 26T | M.2 2242, AI HAT+ 13T, AI Kit (εκτός παραγωγής) | M.2 2242/2280, AI HAT+ 2 |
| Τιμή σε Pi | AI HAT+ 26T ~$110 [snippet] | AI HAT+ 13T ~$70 [snippet] | AI HAT+ 2 ~$130 [snippet], €208 σε EU retailer |
| Θερμοκρασία | Βιομηχανική εκδοχή −40…+85 °C | Εκδοχή «EXT TMP» στο AI Kit | −40…+85 °C |

Πηγές: [Raspberry Pi docs AI HAT+](https://github.com/raspberrypi/documentation/blob/master/documentation/asciidoc/accessories/ai-hat-plus/about.adoc), [Hailo-8 M.2](https://hailo.ai/products/ai-accelerators/hailo-8-m2-ai-acceleration-module/) [snippet], [CNX Software Hailo-10](https://www.cnx-software.com/2024/04/04/hailo-10-m-2-key-m-module-brings-generative-ai-to-the-edge-with-up-to-40-tops-of-performance/).

Υπάρχει και το **Hailo-15**, που είναι SoC για κάμερες (7–20 TOPS με δικό του ISP και encoder), όχι επιταχυντής ([CNX Software](https://www.cnx-software.com/2023/03/12/hailo-15-ai-vision-processor-delivers-up-to-20-tops-for-smart-cameras/)). Είναι ο φυσικός διάδοχος της φιλοσοφίας «όλα σε ένα chip» του RV1106. Προϋποθέτει όμως δική σας πλακέτα ή SOM, οπότε δεν σας αφορά για τον εξοπλισμό που ήδη έχετε.

### Τα vendor FPS είναι το ταβάνι, όχι αυτό που θα δείτε

Ο παρακάτω πίνακας είναι [vendor], με host x86 i5-9400, PCIe Gen3 x4, είσοδο 640×640 και μέτρηση μόνο του chip. Το πρώτο νούμερο είναι FPS με batch 1 και το δεύτερο με batch 8 ([Hailo-8](https://github.com/hailo-ai/hailo_model_zoo/blob/master/docs/public_models/HAILO8/HAILO8_object_detection.rst), [Hailo-8L](https://github.com/hailo-ai/hailo_model_zoo/blob/master/docs/public_models/HAILO8L/HAILO8L_object_detection.rst), [Hailo-10H](https://github.com/hailo-ai/hailo_model_zoo/blob/master/docs/public_models/HAILO10H/HAILO10H_object_detection.rst)):

| Μοντέλο | mAP (float → Hailo) | Hailo-8 | Hailo-8L | Hailo-10H |
|---|---|---|---|---|
| yolov8n | 37,0 → 36,4 | **1036 / 1036** | 202 / 438 | 375 / 370 |
| yolov8s | 44,6 → 43,9 | 491 / 491 | 110 / 208 | 166 / 252 |
| yolov5s | 35,3 → 34,0 | 543 / 543 | 124 / 243 | 250 / 296 |
| yolov11n | 39,0 → 37,5 | 185 / 541 | 157 / 371 | 302 / 332 |
| yolo26n | 40,0 | 155 / 427 | 111 / 255 | 233 / 294 |
| yolov5m6 (1280) | 50,7 | 33,4 / 50,1 | 21,8 / 29,7 | 40,3 / 50,4 |

**Πώς διαβάζετε τον πίνακα.** Όταν το batch 1 και το batch 8 δίνουν το ίδιο νούμερο, το μοντέλο χωράει ολόκληρο στο chip («single context»). Όταν διαφέρουν πολύ, το chip εναλλάσσει τμήματα του μοντέλου και σε real-time βίντεο (batch 1) χάνετε ταχύτητα.

Στο Hailo-8, τα YOLOv8n/s και YOLOv5s χωράνε. Τα YOLO11 και YOLO26 δεν χωράνε. Άρα **για drone βίντεο σε Hailo-8, το YOLOv8n είναι καλύτερη επιλογή από το YOLO11n**, παρά το ελαφρώς χαμηλότερο mAP. Αυτό είναι συμπέρασμα από τα vendor δεδομένα.

Το Hailo-10H είναι **πιο αργό από το Hailo-8** σε YOLOv8n/s με batch 1 και ταχύτερο σε YOLO11/26. Σε σχέση με το 8L είναι ~1,5–3× ταχύτερο σε όλα. Ο Geerling καταλήγει ότι για ανίχνευση αντικειμένων «το φθηνότερο μοντέλο των 13 ή 26 TOPS αρκεί» ([Jeff Geerling](https://www.jeffgeerling.com/blog/2026/raspberry-pi-ai-hat-2/), [snippet]).

**Το YOLOv5n που χρησιμοποιεί σήμερα το repo δεν υπάρχει στο Model Zoo.** Υπάρχουν μόνο YOLOv5s/m, yolov5n_seg και `yolov5xs_wo_spp` στα 512.

### Στο Raspberry Pi 5, το PCIe x1 είναι ο στενός διάδρομος ανεφοδιασμού

Το Pi 5 δίνει **μία μόνο λωρίδα PCIe**. Από προεπιλογή είναι Gen2, και τα AI HAT+ την ανεβάζουν αυτόματα σε Gen3, η οποία όμως «δεν είναι πιστοποιημένη» από τη Raspberry Pi ([Pi PCIe docs](https://github.com/raspberrypi/documentation/blob/master/documentation/asciidoc/computers/raspberry-pi/pcie.adoc)). Επιπλέον, το NMS, δηλαδή η απαλοιφή διπλών πλαισίων, τρέχει στο CPU του host (`engine=cpu` στα configs του zoo).

Οι μετρήσεις σε πραγματικό Pi 5 είναι πολύ πιο κάτω από τα vendor νούμερα:

| Διάταξη | Μετρημένο | Vendor | Πηγή / αξιοπιστία |
|---|---|---|---|
| Pi 5 + Hailo-8L, yolov8n INT8 | **72,7 FPS** (13,8 ms end-to-end, p99 14,2 ms) | 202 | [hailo-uav-tracker](https://github.com/pierrosimonestd-cpu/hailo-uav-tracker) [hobby] |
| Pi 5 + Hailo-8L, yolov8s INT8 | 49,5 FPS | 110 | ίδια |
| Pi 5 + Hailo-8L, yolov8n | 61,9 (b=1) / 121,4 (b=8) | 182 (zoo v2.13) | [Hailo Community](https://community.hailo.ai/t/yolov8n-real-performance-didnt-expected-from-hailo-model-zoo-documentation/18904) [snippet] |
| Pi 5 + Hailo-8L, yolov8s, Gen3 vs Gen2 | ~80 FPS, **Gen3 ≈ 2× Gen2** | — | [Seeed wiki](https://wiki.seeedstudio.com/benchmark_on_rpi5_and_cm4_running_yolov8s_with_rpi_ai_kit/) [snippet] |
| Pi 5 + Hailo-8, yolov8n | ~431 FPS | 1036 | [Hailo Community](https://community.hailo.ai/t/official-fps-benchmark-on-hailo-8-using-raspberry-pi-5/18873), **μόνο από περίληψη αναζήτησης, μη επαληθευμένο** |
| Hailo-10H μέσω USB, yolov8m | 39,67 FPS | 49,9 | [Hailo Community](https://community.hailo.ai/t/yolov8m-fps-lower-than-model-zoo-on-ugen300-hailo-10h/19320) [snippet] |
| Pi 5 CPU μόνο, yolov8n | 6,5 FPS fp32 / 13,3 int8 | — | [hailo-uav-tracker](https://github.com/pierrosimonestd-cpu/hailo-uav-tracker) |

**Εμπειρικός κανόνας:** σε Pi 5 υπολογίστε **35–50% του vendor νούμερου** για μικρά μοντέλα. Είναι παρέκταση από λίγα σημεία, όχι νόμος. Ακόμη και με αυτή την έκπτωση, το Hailo-8L είναι άνετα πάνω από τα 30 fps του σημερινού RV1106 (αυτοαναφερόμενα στο README και όχι επαληθευμένα).

Με tiling ο προϋπολογισμός αλλάζει. Ένα κάδρο 1920×1080 σε πλέγμα 3×2 σημαίνει 6 inferences, άρα **~11 FPS σε Pi 5 + Hailo-8L**. Ο υπολογισμός είναι δικός μας και δεν έχει μετρηθεί. Στο Hailo-8 το ίδιο πλέγμα χωράει άνετα στα 30 fps, εφόσον το PCIe x1 και το CPU του Pi προλαβαίνουν το cropping και το NMS.

### Η διάσπαση του toolchain είναι το σημαντικότερο πρακτικό εύρημα

**Η βασική ιδέα.** Η Hailo έχει πλέον **δύο ασύμβατες γραμμές λογισμικού**, όπως δύο οπλικά συστήματα που δεν μοιράζονται πυρομαχικά. Ένα μοντέλο μεταγλωττισμένο (HEF) για τη μία γραμμή **δεν τρέχει** στην άλλη.

| | Hailo-8 / 8L | Hailo-10H (και 15) |
|---|---|---|
| Model Zoo | branch **v2.x** (π.χ. v2.17, v2.19.1) | branch **master** |
| Dataflow Compiler (DFC) | **3.x** (π.χ. 3.33.0) | **5.x** (5.3+/5.4) |
| HailoRT (runtime) | **4.x**, branch `hailo8` | **5.x**, branch `master` |
| Πακέτο Pi | `hailo-all` | `hailo-h10-all` (**δεν συνυπάρχουν**) |

Πηγές: [hailo_model_zoo README](https://github.com/hailo-ai/hailo_model_zoo), [hailort](https://github.com/hailo-ai/hailort), [Pi AI getting started](https://github.com/raspberrypi/documentation/blob/master/documentation/asciidoc/computers/ai/getting-started.adoc).

**Τι είναι ανοιχτό και τι κλειστό:**

- **Ανοιχτά:** HailoRT (MIT), `hailonet` για GStreamer (LGPL), TAPPAS (LGPL), hailo-apps (MIT), Model Zoo (MIT).
- **Κλειστό:** ο **DFC**. Κατεβαίνει μόνο από το Hailo Developer Zone με λογαριασμό, τρέχει μόνο σε **x86_64 Ubuntu** (ή WSL2) και χρειάζεται **NVIDIA GPU**. Χωρίς GPU πέφτει σε «optimization level 0 … not recommended for production» ([retraining_example](https://github.com/hailo-ai/hailo-apps/blob/main/doc/developer_guide/retraining_example.md)).

Για τον ορισμό του προϊόντος σας αυτό σημαίνει ότι **το Pi 5 δεν μπορεί να μεταγλωττίσει μοντέλα**. Χρειάζεστε ξεχωριστό σταθμό εργασίας x86 με GPU, και το CI pipeline δεν μπορεί να κατεβάσει τον DFC ανώνυμα. Οι όροι της άδειας χρήσης (EULA) του DFC δεν διαβάστηκαν, γιατί το hailo.ai ήταν μπλοκαρισμένο.

---

## 4. Hailo, Jetson και Rockchip: ο καθένας κερδίζει σε άλλο πεδίο

**Η βασική ιδέα.** Η σύγκριση μοιάζει με επιλογή πλατφόρμας για αποστολή. Δεν υπάρχει «καλύτερο» γενικά, υπάρχει καλύτερο για συγκεκριμένο προφίλ SWaP-C (Size, Weight, Power, Cost) και συγκεκριμένο πελάτη.

Πρώτα μια παγίδα: **τα TOPS δεν συγκρίνονται μεταξύ κατασκευαστών**. Η NVIDIA δίνει TOPS για «sparse» INT8, δηλαδή υποθέτει ότι οι μισοί αριθμοί του μοντέλου είναι μηδενικοί. Τα 67 TOPS του Orin Nano Super είναι περίπου 33 «dense». Τα 26 του Hailo-8 είναι dense. Τα 40 του Hailo-10H είναι INT4. Μετράνε μόνο τα μετρημένα FPS.

### Σύγκριση σε πέντε άξονες

| Άξονας | Hailo-8/8L/10H + host | Jetson Orin Nano Super / NX | Rockchip RV1106 / RK3588 |
|---|---|---|---|
| **Απόδοση** | Hailo-8: 250,5 FPS σε YOLOX-S έναντι 126,3 του Orin Nano INT8 ([Idein](https://github.com/Idein/aicast_jetson_benchmark)) | Ευρύτερη κάλυψη πράξεων (ops), FP16. Ανεξάρτητα νούμερα για Super mode δεν βρέθηκαν | RK3588: 199 FPS yolov8n@640 σε 3 πυρήνες ([rknn issue #454](https://github.com/airockchip/rknn_model_zoo/issues/454), ένας χρήστης). RV1106: 0,5 TOPS, μόνο μικρά μοντέλα σε χαμηλή ανάλυση |
| **Κατανάλωση** | Chip 2,5–3,2 W. Σύστημα CM4+Hailo-8 **7,8 W** υπό φορτίο (Idein) | Orin Nano 7–25 W, **12,7 W** μετρημένα (Idein). NX 10–40 W ([NVIDIA](https://developer.nvidia.com/blog/nvidia-jetson-orin-nano-developer-kit-gets-a-super-boost/)) | RV1106 κατηγορίας ~1 W, RK3588 ~5–10 W (**μη επαληθευμένα**) |
| **Κόστος** | $70–130 σε μορφή HAT, συν τον host. Τιμή M.2 module μόνο κατόπιν προσφοράς | Dev kit $249, module 8GB $299 ανά 1k τεμάχια [snippet] | Luckfox Pico από **$12,71**, RK3588 SBC ~$100–200 (μη επαληθευμένο) |
| **Ευελιξία** | Μόνο INT8/INT4, κλειστός compiler, toolchain ανά γενιά. Δεν έχει ISP ή encoder | **Η μεγαλύτερη**: CUDA, TensorRT, DeepStream, ROS 2, πολλά μοντέλα ταυτόχρονα. Το Orin Nano **δεν έχει NVENC** (το ευρέως αναφερόμενο δεν επαληθεύτηκε στο datasheet) | Ολοκληρωμένο SoC με ISP, H.264/H.265 και έως 4 κάμερες MIPI (RK3588). RKNN toolkit |
| **Προέλευση** | Ισραήλ, κατασκευή στην TSMC. Ο host καθορίζει την τελική εικόνα | ΗΠΑ, κατασκευή στην TSMC | **Κίνα** (Fuzhou) |

Για το benchmark της Idein υπάρχει μια σημαντική επιφύλαξη αξιοπιστίας. Η Idein πουλάει προϊόν βασισμένο στο Hailo-8, οπότε έχει εμπορικό συμφέρον. Το benchmark έγινε σε JetPack 5.1.2, πριν από το Super mode, και μετρά μόνο το inference. Το mAP ήταν επίσης **υψηλότερο στο Orin με FP16 (35,2) από το Hailo INT8 (34,3)**. Το benchmark δείχνει κατεύθυνση, δεν δίνει οριστική κατάταξη.

**Συμπέρασμα απόδοσης.** Για έναν ανιχνευτή κλάσης nano στα 640 px, Hailo-8, Orin Nano και RK3588 περνούν όλα τα 60–100 FPS. Ο πραγματικός περιορισμός στην ανίχνευση drones είναι η **ανάλυση και το tiling**, όχι το ωμό FPS. Εκεί το Hailo κερδίζει σε FPS ανά watt και το Jetson σε ευελιξία του pipeline.

### Αμυντικές προμήθειες: ο Rockchip αποκλείεται, Hailo και NVIDIA χρειάζονται «όχημα»

**Η βασική ιδέα.** Στις ΗΠΑ η προέλευση κρίνεται **ανά εξάρτημα**, όχι μόνο για το τελικό προϊόν. Ένα κινεζικό chip στη θέση της «κάμερας» ή της «μετάδοσης δεδομένων» μπορεί να αποκλείσει ολόκληρο το σύστημα.

Το νομικό πλαίσιο, όπως περιγράφεται σε δευτερογενείς πηγές (το κείμενο των νόμων δεν διαβάστηκε):

- **Sec. 848 FY2020 NDAA / 10 U.S.C. 4872.** Το DoD δεν προμηθεύεται UAS που χρησιμοποιούν **flight controllers, ράδια, συσκευές μετάδοσης δεδομένων, κάμερες ή gimbals** από «covered foreign country», ούτε ground control ή λογισμικό λειτουργίας που αναπτύχθηκε εκεί ([DIU](https://www.diu.mil/blue-uas-policy), [govinfo](https://www.govinfo.gov/content/pkg/USCODE-2024-title10/pdf/USCODE-2024-title10-subtitleA-partV-subpartI-chap385-subchapIII-sec4872.pdf), [snippet]).
- **Sec. 817 FY2023 NDAA.** Πρόσθεσε Ρωσία, Ιράν και Β. Κορέα, και **επέκτεινε τον περιορισμό σε εξοπλισμό counter-UAS** και σε υπεργολάβους ([PilieroMazza](https://www.pilieromazza.com/congress-passes-fy2023-ndaa-and-implements-significant-changes-to-federal-procurement-policy/), δευτερογενές).
- **American Security Drone Act 2023** και η υλοποίησή του στο FAR 52.240-1 (Νοέμβριος 2024) ([Federal Register](https://www.federalregister.gov/documents/2024/11/12/2024-26061/federal-acquisition-regulation-prohibition-on-unmanned-aircraft-systems-from-covered-foreign)).
- **FCC Covered List (22/12/2025).** Προστέθηκαν **όλα** τα ξένης παραγωγής UAS και «κρίσιμα εξαρτήματα», όχι μόνο τα κινεζικά. Εξαιρούνται έως 1/1/2027 όσα είναι στη Blue UAS Cleared List ή είναι «domestic end products» ([Pillsbury](https://www.pillsburylaw.com/en/news-and-insights/fcc-categorical-prohibition-foreign-produced-uas-critical-components.html), [Holland & Knight](https://www.hklaw.com/en/insights/publications/2025/12/fcc-adds-all-foreign-made-drones), [snippet]). Αυτό **θεωρητικά αγγίζει και το ισραηλινό Hailo**, αν ένας επιταχυντής θεωρηθεί «κρίσιμο εξάρτημα». Ένας παθητικός επιταχυντής χωρίς RF μάλλον δεν χρειάζεται έγκριση FCC, αλλά αυτό είναι αβέβαιο και θέλει νομικό σύμβουλο.
- **Blue UAS.** Η διαχείριση της λίστας πέρασε από τη DIU στην **DCMA** στις 3/12/2025, με έμφαση στον έλεγχο ανά εξάρτημα ([DIU](https://www.diu.mil/latest/dius-blue-uas-list-to-transition-to-dcma)). Ο καθιερωμένος υπολογιστής στο Blue UAS Framework είναι το **ModalAI VOXL 2** (Qualcomm, 15 TOPS, 16 g) ([ModalAI](https://www.modalai.com/pages/blue-uas-framework)). **Δεν βρέθηκε κανένα εξάρτημα με Hailo στη λίστα.** Υπάρχει όμως προηγούμενο ισραηλινού προμηθευτή: η Elsight μπήκε το 2026 με το Halo, ένα module συνδεσιμότητας που δεν έχει σχέση με το Hailo ([DroneLife](https://dronelife.com/2026/05/01/elsight-halo-blue-uas-allied-suppliers/)).

**Η ερμηνεία μας (όχι νομική γνώμη):**

- **Rockchip.** Στη Luckfox το RV1106 **είναι** ταυτόχρονα ISP της κάμερας και encoder του βίντεο. Αυτό πιθανότατα εμπίπτει στις κατηγορίες «κάμερες» και «μετάδοση δεδομένων». Για αμερικανικό DoD, και ιδίως για counter-UAS μετά τη Sec. 817, είναι πρακτικά αποκλεισμένο. Παραμένει χρήσιμο για Ε&Α, πρωτότυπα και μη ομοσπονδιακούς πελάτες.
- **Hailo και NVIDIA.** Δεν είναι από «covered country». Ο δρόμος αποδοχής περνά από ενσωμάτωση σε ελεγμένο σύστημα ή host.
- **Κρυφή παγίδα.** Hailo πάνω σε host Rockchip (π.χ. RK3588) **ξαναφέρνει το κινεζικό πρόβλημα**. Η προέλευση του host μετράει όσο και του επιταχυντή.

**Εξαγωγικοί έλεγχοι.**

- Η NVIDIA δηλώνει τα Jetson ως **ECCN 5A992.c** (mass-market κρυπτογραφία), όχι ως «advanced computing» ([NVIDIA Jetson FAQ](https://developer.nvidia.com/embedded/faq)). Ισχύουν πάντα οι έλεγχοι τελικής χρήσης και τελικού χρήστη.
- **Για το Hailo δεν βρέθηκε δημόσιο ECCN.** Επειδή είναι ισραηλινό, εφαρμόζεται το ισραηλινό καθεστώς ελέγχου εξαγωγών. Ζητήστε ταξινόμηση απευθείας από τη Hailo.
- Ένα εξάρτημα 5A992.c δεν κάνει το σύστημα ITAR. Ένα σύστημα όμως «ειδικά σχεδιασμένο» για στρατιωτική χρήση μπορεί να ταξινομηθεί αλλιώς στο σύνολό του.

**ΕΕ / NATO.**

- Η ΕΕ **δεν έχει** απαγόρευση σε επίπεδο Ένωσης αντίστοιχη της Sec. 848. Λιθουανία και Ολλανδία απαγορεύουν κινεζικά drones σε στρατιωτικές προμήθειες. Η Λιθουανία είναι η μόνη με πολιτική αποσύνδεσης της αμυντικής εφοδιαστικής αλυσίδας από την Κίνα ([RUSI, Νοέμβριος 2025](https://static.rusi.org/rp-drone-supply-chains-china-nov-2025_0.pdf), [snippet]).
- Οι πρωτοβουλίες της ΕΕ (Readiness Roadmap 2030, EDDI, Action Plan Φεβρουαρίου 2026) εστιάζουν σε ικανότητες και παραγωγή, όχι σε κανόνες προέλευσης εξαρτημάτων ([Capstone DC](https://capstonedc.com/insights/national-security-insights/new-threats-drive-european-drone-and-counter-drone-demand/)).
- Στην πράξη, οι κινεζικές κυρώσεις του 2026 σε ευρωπαϊκές αλυσίδες drones ([The Diplomat](https://thediplomat.com/2026/04/chinas-sanctions-hit-europes-emerging-drone-doctrine/)) σπρώχνουν την αγορά προς «non-red» προμηθευτές.
- **Κενό:** δεν βρέθηκε οδηγία NATO/NSPA για την προέλευση εξαρτημάτων, και **οι ελληνικοί κανόνες δεν ερευνήθηκαν**.

### Πότε ταιριάζει το καθένα

Για μικρά αναλώσιμα (attritable) και FPV, το κόστος ανά μονάδα και τα γραμμάρια κυριαρχούν. Εκεί το Hailo-8L/8 σε φθηνό, μη κινεζικό host έχει νόημα, ενώ ένα Jetson ($299 + carrier + ψύξη) δύσκολα δικαιολογείται.

Για ISR UAV άνω των ~5 kg ή σταθερούς επίγειους κόμβους counter-UAS με σύντηξη αισθητήρων (EO+IR+RF+ακουστικό), η ευελιξία του Jetson υπερτερεί της κατανάλωσης.

Αν ο στόχος είναι αμερικανικό DoD, ο πιο σύντομος δρόμος είναι μια πλατφόρμα ήδη στο Framework (VOXL 2). Το Hailo θα χρειαστεί δική του διαδρομή έγκρισης.

---

## 5. Δύο bugs στο σημερινό repo πρέπει να διορθωθούν πρώτα

**Η βασική ιδέα.** Πριν μεταφέρετε ένα σύστημα, κάνετε το υπάρχον αξιόπιστο σημείο αναφοράς, όπως θα βαθμονομούσατε τον αισθητήρα αναφοράς πριν συγκρίνετε νέο. Και τα δύο bugs επιβεβαιώθηκαν με ανάγνωση του κώδικα.

### Bug 1: Αναντιστοιχία στο `#if` του UART

Στο `luckfox_pico_rtsp_yolov5_UAV/src/main.cc`:

- Ορίζεται μόνο το `#define PRINT_ON_UART false` (γραμμή 48).
- Η αρχικοποίηση του UART, που **δηλώνει** το `int serial_fd`, είναι μέσα σε `#if PRINT_UART` (γραμμή 159). Το `PRINT_UART` δεν ορίζεται πουθενά, οπότε το block δεν μεταγλωττίζεται ποτέ.
- Οι χρήσεις του `serial_fd` (γραμμές 269–271 για αποστολή, 353–354 για `uart_close`) είναι μέσα σε `#if PRINT_ON_UART`.

**Αποτέλεσμα:** αν κάποιος γυρίσει το `PRINT_ON_UART` σε `true` για να ενεργοποιήσει τη σειριακή έξοδο, το `serial_fd` είναι αδήλωτο και **η μεταγλώττιση αποτυγχάνει**. Η σειριακή/MAVLink έξοδος δηλαδή δεν έχει δουλέψει ποτέ σε αυτή τη μορφή του κώδικα.

**Διόρθωση:** αλλάξτε τη γραμμή 159 σε `#if PRINT_ON_UART`.

### Bug 2: Το MAVLink πακέτο δεν έχει CRC_EXTRA

Στο `src/mavlink_comm.cc` το πακέτο φτιάχνεται με το χέρι:

- msgid 9000.
- Checksum CRC-16/MCRF4XX πάνω σε header και payload: `crc_calculate(buffer + 1, 9 + payload_len)` (γραμμή 87).

**Λείπει το byte CRC_EXTRA.** Η προδιαγραφή MAVLink v2 ορίζει ότι το checksum «περιλαμβάνει το CRC_EXTRA». Είναι ένα byte που παράγεται από τον ορισμό του μηνύματος, ώστε πομπός και δέκτης να επιβεβαιώνουν ότι μιλούν για το ίδιο μήνυμα ([MAVLink serialization](https://github.com/mavlink/mavlink-devguide/blob/master/en/guide/serialization.md)). Λειτουργεί σαν κοινό codebook: χωρίς αυτό, ο δέκτης δεν αναγνωρίζει το σήμα.

**Αποτέλεσμα:** pymavlink, MAVSDK, ArduPilot και mavlink-router **θα απορρίψουν** το πακέτο, ακόμη κι αν ορίσετε dialect XML για το msg 9000.

**Διόρθωση:** μην γράφετε MAVLink με το χέρι. Χρησιμοποιήστε headers που παράγει το `mavgen` ή το pymavlink, και κατά προτίμηση τυποποιημένα μηνύματα (βλ. §6, βήμα 7).

Υπάρχει και ένα τρίτο, μικρότερο θέμα. Οι συντεταγμένες x,y στέλνονται κανονικοποιημένες στο −1..1, **όχι ως γωνίες**. Για να στρέψει ένα autopilot ή gimbal προς τον στόχο χρειάζεται `angle_x = atan((cx_px − cx0)/fx)` με τα intrinsics της κάμερας.

---

## 6. Πρακτικό πλάνο: Pi 5 + AI HAT+ 26 TOPS (Hailo-8)

**Η βασική ιδέα.** Το σημερινό app είναι ένας βρόχος C++ δεμένος στο hardware της Rockchip:

`κάμερα (RKMPI VI 720×480) → RGA letterbox 512×512 → RKNN YOLOv5 → RGA σχεδίαση → VENC H.264 → rtsp_demo :554/live/0`

Δίπλα τρέχουν το (σπασμένο) MAVLink σε UART3 και η καταγραφή σε `/userdata`. Στο Pi 5 + Hailo-8 αλλάζει ολόκληρο το «όχημα». Μεταφέρονται το «φορτίο» (η λογική) και τα «πρότυπα» (κατώφλια, μορφή αρχείων).

Η εργασία μοιράζεται σε **δύο μηχανήματα**, όπως ένα εργαστήριο οπλουργείου και ένα πεδίο βολής:

- **x86 Linux PC με NVIDIA GPU:** εκπαιδεύει και μεταγλωττίζει το μοντέλο (ONNX → HEF). Εκεί τρέχει ο κλειστός DFC 3.x, που **δεν τρέχει σε ARM**, άρα ούτε στο Pi.
- **Pi 5 + AI HAT+ 26T:** εκτελεί μόνο το έτοιμο HEF μέσω HailoRT 4.x.

### Εξοπλισμός: τι έχετε και τι χρειάζεται να αγοράσετε

Οι επιλογές της λίστας είναι μηχανικές συστάσεις. Όπου υπάρχει πηγή, αναφέρεται.

| Είδος | Κατάσταση | Σημείωση |
|---|---|---|
| Raspberry Pi 5 | **Έχετε** | Προτιμήστε 8 GB, για ταυτόχρονο x264, GStreamer και Python |
| AI HAT+ 26 TOPS (Hailo-8) | **Έχετε** | Ενεργοποιεί αυτόματα PCIe Gen3 ([Pi docs](https://github.com/raspberrypi/documentation/blob/master/documentation/asciidoc/computers/ai/getting-started.adoc)) |
| **Τροφοδοτικό 27 W USB-C** (επίσημο Pi 5, 5 V/5 A) | Αγορά, αν δεν το έχετε | Το Pi 5 με HAT, κάμερα και UART υπό φορτίο δεν πρέπει να τρέχει με φορτιστή κινητού. Στο αεροσκάφος χρειάζεται BEC 5 V/5 A |
| **Active Cooler** για Pi 5 | Αγορά | Συνεχές φορτίο από x264 και NMS στο CPU, και το Hailo-8 τραβά ~2,5–3,2 W. Για το AI HAT+ 2 η Pi συστήνει ρητά πρόσθετη ψύξη για αποφυγή throttling ([Pi docs](https://github.com/raspberrypi/documentation/blob/master/documentation/asciidoc/accessories/ai-hat-plus/about.adoc)). Ελέγξτε ότι χωράει κάτω από το HAT με τα spacers |
| **Κάμερα** | Αγορά | Η SC3336 της Luckfox **δεν** κουμπώνει στο Pi. Για γρήγορα drones: **Raspberry Pi Global Shutter Camera**, χωρίς rolling-shutter παραμόρφωση, με φακό C/CS της επιλογής σας για στενό FOV και μεγαλύτερη εμβέλεια. Για γενική χρήση: **Camera Module 3 (IMX708)**, με σταθερό σύντομο χρόνο έκθεσης. Κανένα από τα δύο δεν έχει benchmark σε drones. Χρειάζεστε επίσης καλώδιο CSI 22-pin για Pi 5 |
| **Flight controller με ArduPilot** (Pixhawk-class, με θύρα TELEM2) | Αγορά ή από το εργαστήριο | Για τις δοκιμές MAVLink. Ελέγξτε ότι το FC δέχεται MAVLink 2 στο TELEM2. Για NDAA-ευαίσθητο προϊόν, επιλέξτε FC μη κινεζικής προέλευσης |
| Καλώδια JST-GH → Dupont (TELEM2 → GPIO) | Αγορά | Μόνο TX, RX και GND, στα 3,3 V |
| Κάρτα microSD υψηλής αντοχής (endurance) ή USB SSD | Αγορά | Για τον δακτύλιο των logs. Το PCIe το πιάνει το HAT, άρα δεν υπάρχει NVMe χωρίς switch |
| **x86 Linux PC** με NVIDIA GPU (Pascal ή νεότερη), 16–32 GB RAM, ~50 GB δίσκος | Έχετε ή βρείτε | Ubuntu 22.04/24.04. Χωρίς GPU ο DFC πέφτει σε optimization level 0, «not recommended for production» ([retraining_example](https://github.com/hailo-ai/hailo-apps/blob/main/doc/developer_guide/retraining_example.md)). Εκτός από Ubuntu, λειτουργεί και σε WSL2 |
| Λογαριασμός Hailo Developer Zone | Δωρεάν εγγραφή | Για το wheel του DFC και του Model Zoo |

### Χάρτης μεταφοράς

| Σήμερα (RV1106) | Στόχος σε Pi 5 + Hailo-8 | Τύπος εργασίας |
|---|---|---|
| RKMPI VI (κάμερα SC3336) | Picamera2 μέσω hailo-apps (`--input rpi`) | Επανεγγραφή |
| RGA NV12→RGB letterbox | GStreamer `videoscale/videoconvert`, ή HEF με είσοδο NV12 | Διαγραφή / επανεγγραφή |
| RKNN YOLOv5n + C post-process | HEF (hailo8) + `hailonet` + `hailofilter`, με το NMS μέσα στο HEF | **Επανεκπαίδευση και μεταγλώττιση**. Το C post-process φεύγει |
| RGA draw_box | `hailooverlay` | Διαγραφή |
| VENC + rtsp_demo | `x264enc` (software) → mediamtx | Επανεγγραφή |
| `mavlink_comm.cc` σε UART3 | pymavlink σε `/dev/ttyAMA0`, με τυποποιημένα μηνύματα | Επανεγγραφή (μικρή) |
| `flash_storage` / `uav_detection_log` | **Αυτούσια** (καθαρό POSIX C), αλλαγή μόνο του φακέλου | 1:1 |
| `read_detections`, ανάλυση PicoClaw | Αμετάβλητα αν κρατηθεί το format | 1:1 |

### Βήματα

**Βήμα 0: Επιβεβαιώστε το chip.** Στο Pi, μετά την εγκατάσταση του βήματος 4, τρέξτε `hailortcli fw-control identify`. Πρέπει να αναφέρει Hailo-8 και όχι 8L ([Pi docs](https://github.com/raspberrypi/documentation/blob/master/documentation/asciidoc/computers/ai/getting-started.adoc)). Ένα HEF για hailo8l ή hailo10h δεν θα φορτώσει.

**Βήμα 1: Διορθώστε το repo.** Διορθώστε τα δύο bugs της §5, ώστε η έκδοση Luckfox να γίνει έγκυρο σημείο αναφοράς για σύγκριση.

**Βήμα 2: Βρείτε ή ξαναφτιάξτε το μοντέλο (στο x86 PC).**

- Το repo περιέχει μόνο `.rknn` (~2 MB), χωρίς `.pt` ή `.onnx`.
- **Δεν υπάρχει υποστηριζόμενη μετατροπή RKNN → ONNX**: τα βάρη είναι ήδη κβαντισμένα ειδικά για το hardware της Rockchip.
- Αν το αρχικό `best.pt` υπάρχει κάπου (στον συγγραφέα ή σε workspace του rknn_model_zoo), εξάγετε ONNX από αυτό. Αλλιώς, **η επανεκπαίδευση είναι το βασικό σενάριο**.
- Μοντέλο: **YOLOv8n**, μία κλάση, στα 640, με το μείγμα δεδομένων της §1 και δικά σας πλάνα από την κάμερα του Pi.
  - Γιατί όχι YOLOv5n: δεν υπάρχει στο zoo.
  - Γιατί όχι YOLO11n: στο Hailo-8 δεν χωράει σε single context (185 έναντι 541 FPS σε batch 1 και 8).
  - Αν θέλετε μεγαλύτερη εμβέλεια, δοκιμάστε **YOLOv8s**, που επίσης χωράει σε single context (491/491 FPS).
- Αφήστε την κεφαλή του Ultralytics αμετάβλητη.
- Η Hailo προσφέρει docker επανεκπαίδευσης YOLOv8 με το fork της ([Model Zoo YOLOv8 retraining](https://github.com/hailo-ai/hailo_model_zoo/blob/master/training/yolov8/README.rst)). Τρέξτε το έξω από το docker του SW Suite.
- Εξαγωγή με `yolo export model=best.pt imgsz=640 format=onnx opset=11`.

**Βήμα 3: Μεταγλωττίστε για Hailo-8 (στο x86 PC).**

- Εγκαταστήστε τη **γραμμή Hailo-8**: Model Zoo **v2.x** + DFC **3.x**. Για παράδειγμα, v2.17 ↔ DFC 3.33.0 ↔ HailoRT 4.23.0, σε Python 3.10–3.12 με CUDA 12.5.1 και cuDNN 9.10 ([v2.17 GETTING_STARTED](https://github.com/hailo-ai/hailo_model_zoo/blob/v2.17/docs/GETTING_STARTED.rst)).
- **Μην** χρησιμοποιήσετε το branch `master`: είναι μόνο για Hailo-10/15 ([hailo_model_zoo README](https://github.com/hailo-ai/hailo_model_zoo)).
- Εντολή: `hailomz compile --ckpt best.onnx --yaml yolov8n.yaml --classes 1 --calib-path <εικόνες> --hw-arch hailo8 [--performance]`.
- Calibration: **≥1024 εικόνες από την κάμερα του Pi και τον πραγματικό ουρανό**, με σύννεφα, ήλιο, δέντρα, σούρουπο, πουλιά και μικρούς στόχους ([OPTIMIZATION.rst](https://github.com/hailo-ai/hailo_model_zoo/blob/master/docs/OPTIMIZATION.rst)). Γενικές εικόνες τύπου COCO χαλάνε την ανάκληση μικρών στόχων.
- Επαλήθευση **πριν** πάτε στη συσκευή, με το `hailomz eval --target emulator`. Μετρήστε ιδιαίτερα το AP_S.
- Η μεταγλώττιση «μπορεί να πάρει αρκετές ώρες».
- Γνωστά σφάλματα: λάθος end-nodes μετά από custom export (`NMSConfigPostprocessException`) και «Model Script Not Found» ([Hailo Community #5033](https://community.hailo.ai/t/hailo-sdk-client-tools-core-postprocess-nms-postprocess-nmsconfigpostprocessexception-the-layer-yolov8n-conv41-doesnt-have-one-output-layer/5033), [snippet]).
- **Παγίδα κλάσεων:** η Hailo **προσθέτει κλάση φόντου στο index 0**, οπότε το «UAV» γίνεται class id **1**. Περάστε το `--labels-json` και βάλτε `class_id=1` στον tracker ([retraining_example](https://github.com/hailo-ai/hailo-apps/blob/main/doc/developer_guide/retraining_example.md)).
- **Κλειδώστε τις εκδόσεις.** Γράψτε ποια έκδοση DFC έβγαλε το HEF. Η HailoRT στο Pi πρέπει να ταιριάζει.

**Βήμα 4: Στήστε το Pi 5.**

- Raspberry Pi OS 64-bit (Trixie), ενημερωμένο.
- `sudo apt install dkms hailo-all`. Το πακέτο περιέχει driver, firmware, HailoRT 4.x και TAPPAS core. **Όχι** `hailo-h10-all`: τα δύο πακέτα δεν συνυπάρχουν.
- Το PCIe Gen3 ενεργοποιείται αυτόματα από το AI HAT+. Επιβεβαιώστε με `lspci -vv` ότι η ταχύτητα είναι 8 GT/s. Η Pi προειδοποιεί ότι το Gen3 «δεν είναι πιστοποιημένο» και «μπορεί να είναι ασταθές» ([Pi PCIe docs](https://github.com/raspberrypi/documentation/blob/master/documentation/asciidoc/computers/raspberry-pi/pcie.adoc)). Αν δείτε σφάλματα, το Gen2 υπάρχει ως εφεδρεία, με περίπου μισά FPS κατά Seeed [snippet].
- Αν η HailoRT του apt δεν ταιριάζει με τον DFC του βήματος 3, κλειδώστε τις εκδόσεις όπως δείχνουν τα docs της Pi (π.χ. `hailort=4.19.0-3 hailo-tappas-core=3.30.0-1 hailo-dkms=4.19.0-1`) ή ξαναμεταγλωττίστε με ταιριαστό DFC.
- Εγκαταστήστε το `hailo-apps` ([hailo-apps](https://github.com/hailo-ai/hailo-apps)). Το `hailo-rpi5-examples` έχει χαρακτηριστεί «outdated».

**Βήμα 5: Πρώτο «φως».**

1. `hailortcli benchmark uav.hef`, για FPS του chip και ισχύ.
2. `hailo-detect --hef-path uav.hef --labels-json uav.json --input rpi`.
3. Σημειώστε **τα δικά σας** FPS, την καθυστέρηση, τη θερμοκρασία του CPU και αν υπάρχει throttling (`vcgencmd get_throttled`).

Αναμενόμενη τάξη μεγέθους: το vendor νούμερο για YOLOv8n σε Hailo-8 είναι 1036 FPS σε x86 με x4 lanes. Στο Pi 5 με x1 lane θα είναι πολύ χαμηλότερο. Η μόνη ένδειξη που βρήκαμε είναι ~431 FPS από snippet, μη επαληθευμένη. Όποια κι αν είναι η ακριβής τιμή, ο περιορισμός θα είναι η κάμερα (30–60 fps), ο x264 και η Python, όχι το Hailo-8.

**Βήμα 6: Pipeline με tracker και RTSP.** Αντιγράψτε την εφαρμογή detection του hailo-apps και χτίστε:

`Picamera2 (1280×720@30) → tee ─┬─ hailonet(HEF hailo8) → hailofilter → hailotracker(class_id=1) → callback (MAVLink + log) → hailooverlay → videoconvert → x264enc tune=zerolatency speed-preset=ultrafast key-int-max=30 → rtspclientsink → mediamtx (rtsp://<pi>:8554/live)`

- **Στο Pi 5 δεν υπάρχει hardware encoder H.264** ([Pi docs rpicam_vid](https://github.com/raspberrypi/documentation/blob/develop/documentation/asciidoc/computers/camera/rpicam_vid.adoc), [Pi Forums](https://forums.raspberrypi.com/viewtopic.php?t=376952)). Το software encoding βγάζει άνετα 1080p30, αλλά με **μεγαλύτερη καθυστέρηση** από το VENC του RV1106. Το hailo-apps έχει ήδη helper για `x264enc` ([gstreamer_helper_pipelines.py](https://github.com/hailo-ai/hailo-apps/blob/main/hailo_apps/python/core/gstreamer/gstreamer_helper_pipelines.py)).
- Κρατήστε το stream στα 720×480 ή 1280×720 και 2–4 Mbps, ώστε ο x264 να τρώει ~1 πυρήνα. Η ανάλυση ανίχνευσης είναι ανεξάρτητη από την ανάλυση του stream.
- **Μην αφήσετε το mediamtx να «κατέχει» την κάμερα** (`source: rpiCamera`). Τα κάδρα τα χρειάζεται και το Hailo, οπότε στέλνετε το σχολιασμένο stream στο mediamtx ([mediamtx docs](https://github.com/bluenviron/mediamtx/blob/main/docs/3-publish/14-raspberry-pi-cameras.md)).
- Ελέγξτε ότι υπάρχει το `rtspclientsink`, που βρίσκεται στο πακέτο `gstreamer1.0-rtsp`.
- Glass-to-glass καθυστέρηση για Pi 5 + Hailo + x264 RTSP **δεν έχει δημοσιευτεί**. Μετρήστε την, π.χ. με ένα χρονόμετρο οθόνης μπροστά στην κάμερα.

**Βήμα 7: MAVLink προς Pixhawk/ArduPilot.**

- **Καλωδίωση:** GPIO14/TXD (pin 8) στο RX του TELEM2, GPIO15/RXD (pin 10) στο TX του TELEM2, GND με GND. Και τα δύο στα 3,3 V. **Μην** συνδέσετε το 5 V του FC στο Pi όταν το Pi τροφοδοτείται χωριστά.
- **Στο Pi:**
  - Βάλτε `dtparam=uart0=on` στο `config.txt`.
  - Απενεργοποιήστε το serial console (raspi-config ή `cmdline.txt`).
  - Χρησιμοποιήστε **`/dev/ttyAMA0`**. Στο Pi 5 το `/dev/serial0` δείχνει στο 3-pin debug header (`ttyAMA10`), **όχι** στα GPIO14/15 ([Pi interfaces](https://github.com/raspberrypi/documentation/blob/develop/documentation/asciidoc/computers/configuration/interfaces.adoc)). Οι οδηγίες του ArduPilot wiki για `/dev/serial0` γράφτηκαν για παλαιότερα Pi.
- **Στο FC:** `SERIAL2_PROTOCOL=2` και `SERIAL2_BAUD=921` ([ArduPilot wiki](https://github.com/ArduPilot/ardupilot_wiki/blob/master/dev/source/docs/raspberry-pi-via-mavlink.rst)). Αν χρειάζεστε και GCS μέσω Wi-Fi, βάλτε mavlink-router στη μέση.
- **Κώδικας:** `mavutil.mavlink_connection('/dev/ttyAMA0', baud=921600, source_system=1, source_component=191)`. HEARTBEAT στο 1 Hz.
- **Τι στέλνετε:**
  - **Για κάθε επιβεβαιωμένο ίχνος:** `CAMERA_TRACKING_IMAGE_STATUS` (id 275) με `tracking_mode=RECTANGLE` και κανονικοποιημένα `rec_top/bottom`. Μεταφέρει ακριβώς το bbox του παλιού msg 9000, αλλά σε τυποποιημένη μορφή ([common.xml](https://github.com/mavlink/mavlink/blob/master/message_definitions/v1.0/common.xml)).
  - **Για καταγραφή στο log του FC:** `NAMED_VALUE_FLOAT` (`uav_conf`, `uav_ax`, `uav_ay`). Το ArduPilot τα γράφει ως **NVAL**, συγχρονισμένα με το flight log ([GCS_Common.cpp](https://github.com/ArduPilot/ardupilot/blob/master/libraries/GCS_MAVLink/GCS_Common.cpp)).
  - Οι γωνίες υπολογίζονται από τα intrinsics της κάμερας.
- **Αντίδραση του οχήματος** (yaw ή gimbal προς τον στόχο): script Lua με `mavlink:register_rx_msgid` ([AP_Scripting docs](https://github.com/ArduPilot/ardupilot/blob/master/libraries/AP_Scripting/docs/docs.lua)). **Δεν επαληθεύτηκε** ότι το ArduPilot χρησιμοποιεί το `CAMERA_TRACKING_IMAGE_STATUS` από companion για έλεγχο του οχήματος: στον κώδικα φτάνει μόνο στο AP_Camera.
- **Μην χρησιμοποιήσετε `LANDING_TARGET`.** Οδηγεί precision landing και σημασιολογικά είναι λάθος για «εντόπισα άλλο drone».
- **Επαλήθευση:** MAVLink Inspector στο Mission Planner ή στο QGC, και εγγραφές NVAL στο `.bin` log του FC.
- **Κανόνας αποστολής:** στέλνετε και καταγράφετε **μόνο επιβεβαιωμένα ίχνη** (π.χ. ≥3 συνεχόμενα χτυπήματα) και χρησιμοποιείτε το track ID ως `target_num`.

**Βήμα 8: Καταγραφή.**

- Μεταγλωττίστε το `flash_storage.cc` και το `uav_detection_log.cc` αυτούσια στο Pi και αλλάξτε μόνο τον φάκελο (π.χ. `/var/lib/uav_detections`). Εναλλακτικά, ξαναγράψτε τα σε Python με `struct.pack` και την ίδια εγγραφή των 40 bytes, ώστε το `read_detections` και το PicoClaw να δουλεύουν χωρίς αλλαγή.
- Κρατήστε το batching (32 εγγραφές) και τον δακτύλιο 8 αρχείων: η κάρτα SD φθείρεται όπως το flash του RV1106. Χρησιμοποιήστε `noatime` και `fsync` μόνο στην περιστροφή αρχείων.
- Προσθέστε `track_id`, γωνίες, και θέση/στάση από το MAVLink (`GLOBAL_POSITION_INT`, `ATTITUDE`), με αύξηση του `FLASH_STORAGE_VERSION`.

**Βήμα 9: Μικροί στόχοι με tiling.** Εδώ αποδίδει η επιλογή του Hailo-8 έναντι του 8L. Δοκιμάστε `hailo-tiling --hef uav.hef --tiles-x 2 --tiles-y 2 --input rpi` και μετά 3×2. Ορίστε επικάλυψη 0,05 για στόχους ~32 px ([hailo-apps tiling](https://github.com/hailo-ai/hailo-apps/blob/main/hailo_apps/python/pipeline_apps/tiling/README.md)).

Το Hailo-8 έχει περιθώριο για 4–6 tiles στα 30 fps με YOLOv8n. Ο πρακτικός περιορισμός θα είναι η μεταφορά μέσω PCIe x1 και το cropping/NMS στο CPU του Pi. Μετρήστε και επιλέξτε το πλέγμα με βάση τον προϋπολογισμό FPS.

### Πλάγιες σημειώσεις για άλλες διατάξεις

- **Hailo-8L (AI HAT+ 13T / AI Kit):** ίδια γραμμή λογισμικού (v2.x / DFC 3.x / `hailo-all`), αλλά `--hw-arch hailo8l`. Περίπου 70 FPS YOLOv8n σε Pi 5 ([hailo-uav-tracker](https://github.com/pierrosimonestd-cpu/hailo-uav-tracker)), άρα με tiling 3×2 πέφτετε στα ~11 fps.
- **Hailo-10H (AI HAT+ 2):** **διαφορετική** γραμμή: Model Zoo master, DFC 5.x, HailoRT 5.x, `hailo-h10-all`, `--hw-arch hailo10h`. Αξίζει μόνο για LLM/VLM στη συσκευή.
- **Host x86 με Hailo M.2:** PCIe x2/x4, οπότε πλησιάζετε τα vendor FPS, και hardware encoding (`vaapih264enc` / `nvh264enc`). Σειριακή μέσω αντάπτορα USB-UART 3,3 V. Ταιριάζει σε επίγειο κόμβο, όχι σε αεροσκάφος.

**Άδειες χρήσης.** Το Ultralytics (YOLOv8/11) είναι **AGPL-3.0**, όπως και το YOLOv12-BoT-SORT-ReID. Για κλειστό αμυντικό προϊόν, ελέγξτε νομικά τις υποχρεώσεις ή σχεδιάστε εναλλακτική. Το hailo-uav-tracker (MIT) και το khadas RK3588 (Apache-2.0) είναι καθαρότερα ως κώδικας. Τα datasets έχουν δικές τους άδειες: το Halmstad είναι CC0, η άδεια των δεδομένων Anti-UAV είναι ασαφής.

**Σημείωση προμηθειών για τη διάταξή σας.** Το Pi 5 είναι βρετανικού σχεδιασμού με SoC της Broadcom, και το Hailo-8 ισραηλινό. Κανένα δεν προέρχεται από «covered country». Για προϊόν που θα διεκδικήσει προμήθειες ΗΠΑ ή NATO, ελέγξτε επίσης την προέλευση της κάμερας και του FC, που είναι ρητά κατονομαζόμενα εξαρτήματα στη Sec. 848.

---

## Τι να αγνοήσετε ή να αντιμετωπίσετε με καχυποψία

- **Vendor FPS ως αναμενόμενη απόδοση σε Pi 5.** Είναι 2–3× υψηλότερα από τις πραγματικές μετρήσεις.
- **Το «~431 FPS Hailo-8 σε Pi 5».** Προέρχεται μόνο από περίληψη μηχανής αναζήτησης.
- **Το benchmark της Idein ως οριστική σύγκριση Hailo και Jetson.** Ο συγγραφέας έχει εμπορικό συμφέρον, το JetPack είναι παλιό και μετριέται μόνο το inference.
- **Νούμερα mAP ~0,98 στο MAV-VID.** Οι στόχοι είναι μεγάλοι, οπότε δεν λένε τίποτα για μεγάλες αποστάσεις.
- **«Μείωση ψευδών συναγερμών 20–40%».** Οι ισχυρισμοί αυτοί προέρχονται από snippets με ασαφή απόδοση.
- **Hobby repos με YOLOv11x/YOLOv8x.** Είναι πολύ βαριά για NPU και δεν συνοδεύονται από edge μετρήσεις. Άσχετα με τον σκοπό σας.
- **Ερευνητικοί trackers τύπου transformer** (FocusTrack, MemLoTrack, UAUTrack). Είναι για GPU, δεν θα τρέξουν real-time σε μικρό NPU χωρίς βαριά απόσταξη (distillation).
- **Στάσιμα repos:** ucas-vg/Anti-UAV (2022), DUT-Anti-UAV (2023), TIB-Net (αρχειοθετημένο), thermal_signature_drone_detection (2021), drone-net (2018).
- **Μη οπτικοί ανιχνευτές** (ακουστικοί: batear, VolAnti· RF: RFUAV). Είναι χρήσιμοι για μελλοντική σύντηξη αισθητήρων, όχι για αυτή τη μεταφορά.
- **Κάθε νομικό συμπέρασμα αυτής της αναφοράς.** Είναι ερμηνεία πάνω σε δευτερογενείς πηγές. Επιβεβαιώστε με νομικό σύμβουλο πριν από προσφορά.

---

## Συμπέρασμα

Η μεταφορά από Luckfox σε Hailo αλλάζει φύση το project, όχι μόνο ταχύτητα. Στο RV1106 ο περιορισμός ήταν η υπολογιστική ισχύς του NPU. Στο Hailo ο περιορισμός μετακινείται σε τρία σημεία. Πρώτο, ο **host**: PCIe x1, software encoder, NMS στο CPU. Δεύτερο, η **αλυσίδα μεταγλώττισης**: κλειστός DFC σε x86 με GPU και δύο ασύμβατες γενιές λογισμικού. Τρίτο, η **ποιότητα των δεδομένων calibration**, αφού το INT8 τιμωρεί δυσανάλογα τους μικρούς στόχους. Η περίσσεια FPS του Hailo-8 δεν αξίζει ως «περισσότερα fps». Αξίζει γιατί μπορείτε να την «ξοδέψετε» σε tiling και tracking, τα δύο εργαλεία που ανεβάζουν πραγματικά την εμβέλεια και ρίχνουν τους ψευδείς συναγερμούς από πουλιά.

Στρατηγικά, το ίδιο βήμα λύνει και το πρόβλημα προμηθειών. Φεύγοντας από το RV1106, που είναι ταυτόχρονα κάμερα και encoder κινεζικής προέλευσης, το σύστημα γίνεται αποδεκτό σε πελάτες ΗΠΑ και NATO, **υπό την προϋπόθεση ότι και ο host δεν είναι κινεζικός**. Υπάρχουν δύο κενά αγοράς που μπορείτε να καλύψετε με δικές σας μετρήσεις. Κανένα ανοιχτό σύστημα δεν δημοσιεύει πιθανότητα ανίχνευσης ανά μέγεθος σε pixels ή ψευδείς συναγερμούς ανά ώρα, και κανένα δεν ενώνει Hailo με MAVLink. Τέτοια δεδομένα από δοκιμές πεδίου είναι αυτό που ένας αξιολογητής προμηθειών ζητά και σπάνια βρίσκει.
