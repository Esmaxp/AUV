# Autonomous Underwater Vehicle (AUV) Perception, Decision & Navigation Architecture

**Comprehensive Technical Research Document & System Specification (research.md)**

---

## 1. Image Enhancement and Object Detection

### 1.1 Underwater Optical Preprocessing & Restoration Benchmark

Underwater optical imaging suffers from severe environmental degradation governed by absorption and scattering:

- **Wavelength-Dependent Light Absorption**: Red wavelengths (~650–700 nm) are absorbed within 3–5 meters of water depth, leaving images dominated by blue-green spectrums.
- **Scattering (Forward and Backward)**: Forward scattering blurs object boundaries, whereas backward scattering acts as high-intensity additive noise, drastically reducing foreground-to-background contrast.

**Preprocessing Algorithms Evaluated:**

**Color Correction & Chromatic Adaptation (Gray World / Max-RGB / Retinex)**
- Mechanism: Recompensates the attenuated red spectrum by estimating global or local illumination maps.
- Latency: Extremely low (<2 ms on embedded GPUs via CUDA/OpenCV).
- Impact on YOLOv8: Critical for identifying color-coded objects (e.g., distinguishing green vs. red navigation buoys).

**Contrast Limited Adaptive Histogram Equalization (CLAHE)**
- Mechanism: Operates on localized image tiles (typically on the luminance channel $L$ in CIELAB or $V$ in HSV) while clipping histogram peaks to prevent noise amplification in uniform water areas.
- Empirical Validation: S. Tani et al. validated CLAHE within their underwater stereo visual odometry pipeline, proving that it preserves repeatable features (SURF/ORB) across complex seafloors and turbid waters. It consistently boosts YOLO detection mAP by +4% to +8% without destabilizing bounding box regression.

**Dehazing (Dark Channel Prior - DCP / UDCP / DehazeNet)**
- Mechanism: Inverts atmospheric scattering models adapted to water transmission.
- Bottleneck: Prohibitive computational cost (>30–45 ms on embedded platforms like Jetson Orin/Nano). It frequently introduces halo artifacts and false gradient edges around structural posts (such as underwater gate boundaries), degrading bounding box localization.

**Architecture Decision:** Real-time edge preprocessing pipeline:
`Refractive Undistortion → CIELAB-space CLAHE (clip limit 2.0, tile grid 8x8) → Red-Channel Stretching`

Deep generative dehazing (e.g., Water-Net) is bypassed to preserve edge FPS throughput.

---

### 1.2 YOLOv8: Pretrained Initialization vs. Training from Scratch & Dataset Landscape

The central research dilemma—should YOLOv8 be trained from scratch or initialized with pretrained weights?—is resolved through comparative literature and empirical benchmarks.

**Empirical Comparison on Underwater Litter & Debris Benchmark (UW_Garbage_Debris_Dataset)**

The benchmark evaluated YOLOv8 variants trained on 5,130 real underwater images (15 classes, 70% train, 20% val, 10% test split):

| Model | Precision | Recall | mAP@0.5 | Training Time | Notes |
|---|---|---|---|---|---|
| YOLOv8n | 100% | 77% | 79% | 1.466 h | Noticeable overfitting on small-sample classes due to limited model capacity |
| YOLOv8s | 97.7% | 90% | 83.0% | 2.288 h | Selected by authors as optimal trade-off between compute budget and recall |
| YOLOv8m | 99.1% | 89% | 83.0% | 4.535 h | — |
| YOLOv8l | 97.2% | 88% | 82.5% | 6.755 h | — |
| YOLOv8x | 99.0% | 91% | 82.0% | 10.873 h | — |

**Pretrained vs. Scratch Conclusion**

In the AUV-DETR (HUST 2025) benchmark, networks initialized with COCO-pretrained weights converged within fewer epochs and achieved 91.85% mAP0.5:0.95, whereas models trained from scratch without large-scale priors failed to overcome underwater feature degradation.

**Architectural Decision:** Fine-tune YOLOv8s initialized with COCO pretrained weights (`yolov8s.pt`).

**Dataset Engineering Strategy for Competition Classes (buoy, door/gate, marker)**

A comprehensive survey of representative underwater datasets reveals a significant taxonomy gap:

| Dataset | Size | Focus |
|---|---|---|
| Brackish (Pedersen et al.) | 14,518 frames, 25,613 annotations | 6 biological classes (fish, crab, jellyfish, shrimp, starfish) |
| MUED (Jian et al.) | 8,600 images, 430 salient object categories | Illumination, turbidity, posture changes |
| RUIE (Liu et al.) | 3 subsets incl. UHTS (300 images) | Color cast and task-driven enhancement |
| UWD (Fan et al.) | 10,000 images | 4 seafood classes (sea cucumber, octopus, scallop, starfish) |
| TrashCan (Hong et al.) | 7,212 annotated images | Underwater garbage, marine robots, divers |
| UDD / AUDD (Liu et al.) | 2,227 / 18,661 images | Sea floor farm organisms |
| DUO (Liu et al.) | 7,782 images (URPC merge) | Underwater harvesting targets |

**The Domain Gap:** Public benchmarks do not provide competition-specific navigation primitives (buoy, gate/door, marker).

**Mitigation Strategy:**
1. Acquire pool competition subsets from RoboSub / RoboNation Roboflow Benchmarks (direct bounding box annotations for gate posts and colored buoys).
2. Implement physics-based synthetic data rendering using the Stonefish AUV Simulator (Cieslak 2019; López-Barajas et al. 2026) to generate variable lighting, turbid backgrounds, and gate orientations.
3. Conduct local in-situ pool/tank filming with actual competition scale mockups, annotating in normalized YOLOv8 format (`class_id x_center y_center width height`).

---

### 1.3 Confidence Gating & Dynamic False Positive Suppression

**Dual-Threshold Detection Logic:**
- **Initialization Threshold** ($\tau_{\text{init}} = 0.65$): A new target tracklet is registered only if detection confidence exceeds 0.65.
- **Tracking Maintenance Threshold** ($\tau_{\text{track}} = 0.35$): Once registered, tracklets accept lower-confidence detections (down to 0.35) to bridge temporary turbidity drops or light flashes.
- **Non-Maximum Suppression (NMS)**: IoU threshold fixed at 0.45.
- **Temporal Consistency Window**: A bounding box must persist across at least 3 consecutive frames within a bounded Euclidean distance centroid ($\Delta d \le 40\text{ pixels}$) to filter out particulate matter, bubbles, or surface reflections.

---

## 2. Object Tracking and Spatial Localization

### 2.1 Multi-Object Tracking: ByteTrack vs. SORT

**Failure Modes of Standard Trackers:** SORT (and DeepSORT) rely on a single confidence threshold and immediately drop detections when water turbidity lowers the score. This triggers frequent track termination and identity switches (ID switches).

**The ByteTrack Mechanism:** Retains both high- and low-confidence boxes, performing two-stage Hungarian association:
1. **First association**: Matches high-score detections with existing tracklets using IoU distance.
2. **Second association**: Matches unmatched tracklets against remaining low-score detections, recovering occluded or blurred targets.

**Benchmark Verification (AUV-DETR & TD-ByteTrack, HUST 2025)**

When evaluated on real subsea video sequences against BoT-SORT and StrongSORT, the ByteTrack framework achieved:
- MOTA (Multi-Object Tracking Accuracy): 86.10%
- HOTA (Higher Order Tracking Accuracy): 81.95%
- IDF1 (Identification F1-Score): 82.68%
- ID Switches: Suppressed to only 4 identity switches across entire evaluation sequences.
- Execution Speed: 34 FPS operating strictly on a standard CPU.

**Decision:** Implement ByteTrack for subsea target association. It provides high resilience against sediment occlusions without the computational burden of deep Re-ID neural networks.

---

### 2.2 Refractive Geometry & Underwater Camera Calibration

**Physical Phenomenon (Snell's Law):**

$$\frac{\sin \theta_1}{\sin \theta_2} = \frac{\mu_2}{\mu_1}$$

Light passing through water ($\mu_w \approx 1.334$), the waterproof port ($\mu_g \approx 1.49$), and the internal air chamber ($\mu_a \approx 1.0$) refracts at each interface.

**Why the In-Air Pinhole Model Fails:**
- Pinhole projection assumes all viewing rays intersect at a single projection center. Underwater refraction prevents this intersection, making projection distance-dependent.
- Submersion scales effective focal length: $f_{\text{water}} \approx 1.334 \times f_{\text{air}}$. In-air calibration matrices lead to ~30% metric scale errors and severe radial distortion.

**Port Geometries:**
- **Flat Port**: Prone to distance-dependent non-linear distortion, chromatic aberration, and ~33% field-of-view reduction.
- **Dome Port**: Eliminates refraction only if the camera's entrance pupil is placed precisely at the sphere center. Decentering introduces complex radial asymmetry.

**Calibration Protocol:**
- Perform in-situ calibration using submerged ChArUco targets (avoids corner loss from underwater low-contrast edges).
- Utilize dedicated refractive calibration frameworks like Calibmar (Kiel University / GEOMAR) to estimate viewport interface normals $(n_x, n_y, n_z)$ and distance $r_{\text{flat}}$.

---

### 2.3 Distance Estimation: Stereo Triangulation vs. Monocular PnP

**Operational Breakdown**

**Monocular PnP (Perspective-n-Point)**
- Solves camera pose from known 3D CAD model points matched to 2D image coordinates (requires $N \ge 4$ points).
- Critical Weakness: If the gate or buoy is partially truncated by the camera frame during close-range visual servoing, PnP experiences mathematical singularity and scale divergence.

**Stereo Disparity Triangulation**
- Computes metric depth directly from calibrated baseline disparity:

$$Z = \frac{f \cdot B}{d}$$

- Operates independently of prior object CAD knowledge.

**The Unified 3D-to-2D Solution (University of Pisa & UIB Benchmark)**

Real-world AUV trials on Turbot AUV (equipped with an AVT Manta stereo pair, DVL, IMU, and depth sensor) validated a 3D-to-2D stereo visual odometry architecture:

1. At timestamp $t-1$, triangulate stereo feature correspondences to obtain a metric 3D point cloud:

$$\min\_num\_features \ge 30, \quad \min\_num\_3D\_points \ge 20$$

2. At timestamp $t$, track 2D features in the left camera and solve relative motion using PnP with RANSAC outlier filtering ($\min\_num\_inliers \ge 15$).

Across 424 m and 390 m real seabed monitoring surveys (Missions 1 & 2), this 3D-to-2D hybrid approach achieved trajectory drift rates of only 7% to 13% compared to physical DVL benchmarks.

**Implementation Decision:**
- Use Stereo Triangulation to extract target 3D centroid positions $(X, Y, Z)$ and maintain obstacle avoidance distance.
- Use PnP to compute 6-DOF relative vehicle-to-gate orientation when all gate vertices are visible in the frame.

---

## 3. Multimodal Sensor Fusion

### 3.1 Optical Camera vs. Sonar Priority Resolution

**Complementary Modalities:**
- **Optical Stereo Camera**: High update rate (15–30 Hz), high angular resolution, rich semantic classification (YOLO), but range degrades to $<3\text{ m}$ in turbid waters.
- **Forward-Looking Sonar (FLS) / Profiler**: Long detection range (10–50 m), penetrates turbidity, but lacks color semantics and exhibits poor angular resolution with multipath surface reflections.

**Industrial Validation (Ultralytics & MarineSitu Case Study):** MarineSitu achieved >96% uptime monitoring subsea turbines by pairing optical cameras with imaging sonars. Real-time YOLO detections dynamically gated and synchronized acoustic records, ensuring targets were detected despite murky water conditions.

**Coarse-to-Fine Operational Rules:**
- **Range Gating** ($Distance > 3.0\text{ m}$ or Turbidity Index $> \tau_{\text{murk}}$): Sonar takes priority. The system relies on sonar range hypotheses to steer the vehicle toward the target area.
- **Range Gating** ($Distance \le 3.0\text{ m}$): Camera takes priority. High-rate optical YOLO detections govern 6-DOF relative alignment and visual servoing.

**EKF Dynamic Covariance Adaptation:**

$$R_{\text{measurement}} = \begin{bmatrix} R_{\text{cam}}(\sigma_{\text{conf}}) & 0 \\ 0 & R_{\text{sonar}}(\text{SNR}) \end{bmatrix}$$

$R_{\text{cam}}$ increases exponentially as YOLO confidence drops, smoothly shifting Kalman filter weights to acoustic tracking.

---

### 3.2 Spatial Reference Frames (ROS 2 REP-103 / REP-105 Standards)

All spatial calculations conform to standardized coordinate frame conventions:

- **camera_optical_frame**: Origin at left lens optical center ($Z$ forward along optical axis, $X$ right, $Y$ down).
- **base_link**: Center of mass of the AUV ($X$ forward bow, $Y$ port/left, $Z$ upward heave; REP-103 FLU standard).
- **odom**: World-fixed, continuous, drift-accumulating odometry frame.
- **world / map**: Earth-fixed global navigation frame (NED: North-East-Down).

$$\mathbf{p}_{\text{base}} = \mathbf{R}_{\text{camera}}^{\text{body}} \cdot \mathbf{p}_{\text{optical}} + \mathbf{t}_{\text{camera}}^{\text{body}}$$

Target centroid positions are transformed directly into `base_link` for reactive visual servoing PID loops, while simultaneously broadcast to `odom` via `tf2_ros` for path planning.

---

## 4. Decision Mechanism and Mission Management

### 4.1 Behavior Tree (BT) Architecture

Finite State Machines (FSMs) suffer from state explosion and rigid transitions in unpredictable marine environments. A hierarchical Behavior Tree (implemented in BehaviorTree.CPP and visualized via Groot) provides reactive, modular task execution.

```
[Root: Autonomous Mission Pipeline]
   └── [Sequence: Task Execution]
         ├── <Condition: IsSystemReady (Sensors, Thrusters, Comms)>
         ├── [RetryNode: num_attempts = 3]
         │     └── [Sequence: Navigation & Search]
         │           ├── <Action: GoToHome / Waypoint Transit>
         │           └── [Fallback: Target Acquisition]
         │                 ├── <Condition: ObjectDetected (Perception Callback)>
         │                 └── <Action: SearchPattern (Yaw Sweep +/- 15 deg)>
         ├── [RetryNode: num_attempts = 3]
         │     └── [Fallback: Intervention & Alignment]
         │           ├── [Sequence: Visual Servoing Approach]
         │           │     ├── <Action: PublishTargetPose (Dynamic Offset)>
         │           │     ├── <Action: ActivateTask (Visual Servo PID)>
         │           │     └── [Parallel: Monitor Convergence]
         │           │           ├── <Condition: CheckError (Distance < 0.25 m)>
         │           │           └── <Decorator: Timeout (duration = 10000 ms)>
         │           └── <Action: RecoveryManeuver (Station-Hold & Re-align)>
         ├── <Action: ExecuteTaskAction (Pass Gate / Drop Marker / Touch Buoy)>
         └── <Action: MissionComplete / Standby (Idle)>
```

---

### 4.2 Fallback Behaviors & Fault Management (CIRTESU IEEE Access 2026 Reference)

Experimental validation of underwater mobile manipulation by S. López-Barajas et al. (2026) verified the following recovery mechanisms:

- **Parallel Timeout Monitoring**: Action subtrees (such as GoTo or Approach) execute in parallel with a Timeout node (typically 5,000 to 10,000 ms). If tracking errors fail to settle within tolerance before the timer expires, the node returns `FAILURE`.
- **Recovery / Reset Sequence**: A `FAILURE` status triggers a Fallback branch that halts forward thrust, activates vehicle station-hold, resets actuators to nominal posture, and retries the approach up to 3 times before issuing a terminal mission abort.
- **Sensor Dropouts**: If camera frame acquisition freezes for $>1.0\text{ s}$, the tree switches the active localization provider on the Blackboard from visual tracking to acoustic dead-reckoning.

---

### 4.3 Blackboard Shared Data Schema

| Key | Data Type | Units / Range | Functional Description |
|---|---|---|---|
| `target_detected` | bool | true / false | Immediate perception detection flag |
| `target_class` | string | "gate", "buoy", "marker" | Semantic label from YOLO detector |
| `target_confidence` | float | 0.0 – 1.0 | Filtered detection confidence score |
| `target_pose_base` | geometry_msgs/PoseStamped | Meters, Quaternions | 3D target coordinates relative to base_link |
| `target_range` | float | Meters | Metric Euclidean distance to target |
| `mission_stage` | uint8 | Enum (0 to 5) | Active subtree phase index |
| `system_fault_flags` | uint32 | Bitmask | Error bits: leak, thruster overcurrent, camera drop |

---

## 5. Motion Planning and Control Boundary

### 5.1 Reactive Avoidance vs. Global Planners (A* / RRT*)

**Global Planners (3D A\* / Informed RRT\*)**
- Strengths: Mathematically optimal paths without local minima.
- Weaknesses: Requires a 3D occupancy voxel grid (OctoMap), which is computationally expensive to build underwater due to acoustic multipath noise and odometry drift.

**Local Reactive Methods (Artificial Potential Fields - APF / Dynamic Window Approach - DWA)**
- Strengths: Microsecond computational footprint; provides immediate collision avoidance around transient obstacles (buoys, mooring lines, divers).
- Weaknesses: Vulnerable to local minima traps (e.g., oscillating directly between two gate posts).

**Selected Hybrid Framework:**
- **Global Layer**: Simple Waypoint Guidance (Line-of-Sight / Dubins curves) navigates between task waypoints.
- **Local Layer**: 3D Artificial Potential Field (APF) with attractive goal potential and repulsive obstacle potential governs obstacle avoidance and gate center visual servoing.

---

### 5.2 Autonomy vs. Navigation Functional Interface

**Autonomy Group Output:**
- High-level navigation goals: `geometry_msgs/msg/PoseStamped` sent to the Navigation Action Server.
- Closed-loop visual servoing commands: `geometry_msgs/msg/TwistStamped` published directly to `/cmd_vel` during final target alignment.

**Navigation Group Responsibilities:**
- Low-level thruster allocation matrix inversion.
- Hydrodynamic damping and buoyancy compensation.
- 6-DOF closed-loop PID/LQR/MPC velocity and heading regulation.

---

## 6. Navigation Interface and ROS 2 Architecture

### 6.1 Communication Paradigms (Topic vs. Action/Service)

**High-Rate Asynchronous Topics (Pub/Sub):**
- `/camera/left/image_raw`, `/camera/right/image_raw` (`sensor_msgs/Image`)
- `/camera/point_cloud` (`sensor_msgs/PointCloud2`)
- `/perception/detections_2d` (`vision_msgs/Detection2DArray`)
- `/perception/target_pose` (`geometry_msgs/PoseStamped`)
- `/cmd_vel` (`geometry_msgs/TwistStamped` at 20–30 Hz)

**Synchronous Services & Action Servers:**
- `/navigate_to_pose` (`nav2_msgs/action/NavigateToPose`): Action server interface allowing Autonomy to dispatch waypoints, track distance feedback, and preempt/cancel paths.
- `/controller/activate_task` (`std_srvs/SetBool`): Service interface to toggle low-level task priority controllers dynamically.

---

### 6.2 Rate and Hardware Synchronization

**Decoupled Execution Loops:**
- Perception / YOLOv8 Inference: 15–20 Hz on NVIDIA Jetson TensorRT engine.
- Point Cloud Segmentation & Centroid Fitting: 4–6 Hz (accommodating computational limits observed in CIRTESU tests).
- Low-Level Task-Priority & Thrust Control Loops: 20–30 Hz deterministic execution.

**Temporal Frame Alignment:** Stereo camera frame pairs and IMU messages are bound using `message_filters::ApproximateTimeSynchronizer` with an allowable jitter window of $\Delta t \le 15\text{ ms}$.

---

## References

1. López-Barajas, S., Solis, A., Marín, R., & Sanz, P. J. (2026). *A Novel Underwater Mobile Manipulator for Pipeline Inspection and Material Degradation Testing.* IEEE Access, 14, 66413–66430. https://doi.org/10.1109/ACCESS.2026.3685422
2. Zhai, X., Tang, G., Zhou, X., Deng, Y., Wang, S., Xiao, W., & Dong, J. (2025). *Real-Time Detection and Continuous Tracking for Short-Range Recovery of Autonomous Underwater Vehicles via AUV-DETR and TD-ByteTrack.* Ocean Engineering / arXiv:2501.00000.
3. Tani, S., Ruscio, F., Bresciani, M., Nordfeldt, B. M., Bonin-Font, F., & Costanzi, R. (2023). *Underwater Navigation and Visual Odometry using Stereo Vision and Extended Kalman Filter.* Ocean Engineering, 280, 114687. https://doi.org/10.1016/j.oceaneng.2023.114687
4. Hasib, S. (2024). *Multi-View-Fusion-Object-Detection-for-underwater-robotic-systems.* GitHub Repository: ShahidHasib586/Multi-View-Fusion-Object-Detection-for-underwater-robotic-systems.
5. Ultralytics & MarineSitu. (2024). *Automating Subsea Wildlife and Infrastructure Monitoring with Ultralytics YOLO Models and Multimodal Sensing.* Ultralytics Industrial Case Study.
6. Jian, M., et al. (2024). *Underwater Object Detection and Datasets: A Survey.* Intelligent Marine Technology and Systems, 2(9).
7. Pedersen, M., et al. (2019). *The Brackish Dataset: A Subsea Computer Vision Dataset for Tracking Marine Life.* IEEE VISIGRAPP.
