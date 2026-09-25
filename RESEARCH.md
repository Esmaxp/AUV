# AUV Engineering Design Report

**Global Architecture, Hardware, and Software Design Specifications**
*Autonomous Underwater Vehicle (AUV) — Perception & Autonomy Subsystems*

## Table of Contents

1. [Global Architecture, Hardware, and Software Design Report: Imaging Cameras Subsystem](#report-1-imaging-cameras-subsystem)
2. [Global Architecture, Hardware, and Software Design Report: Imaging Sonar Subsystem](#report-2-imaging-sonar-subsystem)
3. [Global Architecture, Hardware, and Software Design Report: Lighting Subsystem](#report-3-lighting-subsystem)
4. [Global Architecture, Hardware, and Software Design Report: AI Processing Subsystem](#report-4-ai-processing-subsystem)

---

# Report 1: Imaging Cameras Subsystem

## 1. Executive Summary

This engineering report establishes the definitive architectural, hardware, and software design specifications for the Imaging Cameras subsystem of an Autonomous Underwater Vehicle (AUV). The objective of this document is to provide an exhaustive technical framework for visual data acquisition, hardware allocation, and algorithmic processing in highly degraded underwater environments. The design decisions outlined herein bridge the critical gap between optical marine physics, computational constraints on embedded edge hardware, and the mathematical robustness of deep learning architectures.

This report systematically addresses three primary engineering domains. First, it evaluates the geometric, mathematical, and computational viability of **stereo vision** against **monocular-acoustic sensor fusion**, definitively demonstrating that a tightly coupled monocular visual-inertial-acoustic pipeline utilizing **factor graph optimization** is vastly superior for AUV target positioning in turbid media. Second, it resolves the architectural conflict between shared and dedicated camera payloads by analyzing optical port physics (flat versus dome interfaces), field of view constraints, and zero-copy memory bottlenecks on edge processors. Finally, it provides a rigorous mathematical deconstruction of the **Jaffe-McGlamery** optical model, detailing exactly how environmental degradation — specifically forward scattering and backscatter — destroys feature extraction gradients and destabilizes the bounding box regression loss functions of deep learning models.

The subsequent sections serve as a comprehensive blueprint for the physical integration, optical calibration, and software pipeline architecture required to achieve enterprise-grade autonomous navigation and target seeking.

## 2. Geometric and Computational Analysis: RGB vs. Stereo Vision Paradigms

The fundamental requirement of an AUV imaging system is to provide accurate, real-time spatial representations of the environment to support both autonomous navigation (Simultaneous Localization and Mapping, or **SLAM**) and target interaction. A critical architectural decision is whether to utilize stereo vision to achieve metric depth or to rely on an alternative sensor fusion paradigm incorporating monocular vision, acoustics, and inertial data.

### 2.1 Epipolar Geometry and Baseline Constraints in Hydrodynamic Housings

Stereo vision relies on epipolar geometry to estimate the depth of a scene. Given a stereo camera rig with a physical baseline distance *b* between two optical centers and a focal length *f*, the depth *Z* of a triangulated environmental point is inversely proportional to its pixel disparity *d*, expressed mathematically as:

> ```
> Z = (f * b) / d
> ```

While mathematically elegant in terrestrial applications, the physical and environmental constraints of an AUV severely degrade the efficacy of this formulation. The primary geometric limitation is the **baseline constraint**. To maintain hydrodynamic efficiency, minimize drag, and preserve structural integrity under immense hydrostatic pressure, AUV camera housings are typically restricted to a narrow baseline, often measuring between 10 to 20 centimeters.

The error in depth estimation, delta Z, scales quadratically with the distance to the target, defined as:

> ```
> delta_Z = (Z^2 / (f * b)) * delta_d
> ```

With a highly constrained baseline *b*, the depth error delta Z grows exponentially as the target distance *Z* increases. At operational ranges exceeding 3 to 5 meters, the depth resolution of a narrow-baseline stereo camera becomes too coarse and noisy for accurate autonomous targeting, obstacle avoidance, or precise manipulator interaction.

### 2.2 Turbidity, Marine Snow, and the Degradation of Disparity Correlation

Furthermore, stereo matching algorithms — whether traditional block-matching approaches like **Semi-Global Matching (SGM)** or modern convolutional stereo networks — rely heavily on photometric consistency and the presence of high-frequency textures to establish accurate pixel correspondences between the left and right images.

Turbid underwater environments act as severe spatial low-pass filters, destroying the fine textures and sharp gradients required for disparity correlation. The presence of suspended particulate matter, commonly known as **marine snow**, exacerbates this issue significantly. Particles passing close to the camera lenses create false-positive high-disparity matches, resulting in catastrophic depth map noise that corrupts downstream path planning and obstacle avoidance algorithms. When visual features are sparse or completely obscured by scattering, stereo matching fails to produce a dense or reliable point cloud, rendering the depth data useless for navigation.

### 2.3 Computational Load, Thermal Envelopes, and Edge Processor GFLOPs Constraints

Beyond severe geometric and environmental limitations, stereo vision imposes a massive computational burden on the AUV's embedded hardware.

Generating dense disparity maps requires computing matching costs across multiple disparity hypotheses for every pixel in the frame. For a standard 1080p resolution feed operating with 128 disparity search levels, the computational cost easily exceeds tens of **Giga-Floating Point Operations per Second (GFLOPs)**.

Edge computing platforms, such as the **NVIDIA Jetson Orin** series, operate under strict power and thermal envelopes. Allocating massive GPU and Tensor Core resources to stereo block matching or neural depth estimation starves the deep learning inference threads of required processing cycles. If the AUV is simultaneously running complex object detection networks (e.g., **YOLOv8**) and visual odometry algorithms, the addition of dense stereo disparity computation induces severe thermal throttling and unacceptable pipeline latency. This computational bottleneck often drops the system's overall operational frequency below the 20 to 30 Hz threshold required for stable, closed-loop vehicle control, jeopardizing the mission.

### 2.4 Tightly Coupled Monocular Visual-Inertial-Acoustic Sensor Fusion via Factor Graphs

Given the physical, environmental, and computational limitations of stereo vision, alternative sensor fusion paradigms provide a vastly more efficient, accurate, and robust architecture. The recommended engineering paradigm utilizes a single monocular RGB camera tightly fused with an acoustic **Doppler Velocity Log (DVL)**, an **Inertial Measurement Unit (IMU)**, and a high-resolution pressure (depth) sensor.

In this architecture, the metric scale ambiguity inherently present in monocular visual odometry is resolved not by a secondary camera baseline, but by highly accurate acoustic and inertial measurements. A DVL utilizes the Doppler shift of acoustic beams reflected off the seafloor to provide drift-free metric velocity vectors (v_dvl) in the vehicle's body frame. Because acoustic waves operate at frequencies that are entirely unaffected by the optical turbidity of the water, the DVL provides continuous, reliable metric scaling even when visual feature tracking fails completely due to darkness or sediment clouds.

The optimal integration of these heterogeneous sensors is achieved through **Factor Graph Optimization (FGO)**, utilizing advanced libraries such as **GTSAM** (Georgia Tech Smoothing and Mapping). Unlike traditional **Extended Kalman Filters (EKFs)** which marginalize past states and accumulate linearization errors, a factor graph maintains a sliding window of historical poses (variable nodes) connected by measurement constraints (factor nodes).

The total cost function *J* to be minimized in the optimization backend is the sum of the Mahalanobis distances of the residual errors from all sensor modalities:

> ```
> J = sum(||r_vision||^2_Sigma_v) + sum(||r_imu||^2_Sigma_imu)
>   + sum(||r_dvl||^2_Sigma_dvl) + sum(||r_depth||^2_Sigma_depth)
> ```

where r_imu represents the IMU pre-integration residuals between keyframes. Pre-integration mathematically combines high-frequency angular velocities and linear accelerations into a single relative motion constraint (delta R, delta v, delta p), avoiding the computational cost of integrating the system state at the IMU's high sampling rate. The DVL factor directly constrains the velocity variables in the graph, providing an absolute scale reference that perfectly complements the rich, albeit scale-ambiguous, relative pose constraints generated by the monocular camera's tracked features. The pressure sensor factor constrains the Z-axis (depth) drift, which is notoriously difficult to observe using vision alone.

This monocular-acoustic sensor fusion paradigm entirely negates the need for computationally expensive and geometrically constrained stereo matching. It drastically reduces the vision pipeline's GFLOPs footprint, relies on acoustics for scale (which penetrate turbid water effortlessly), and utilizes the IMU for high-frequency state propagation, establishing a highly robust, low-compute navigation state estimator perfectly suited for the edge constraints of an AUV.

## 3. Payload Allocation: Shared vs. Dedicated Camera Architecture

With the superiority of monocular visual-inertial-acoustic inputs established, the global architecture must define the physical allocation of the camera payloads. The AUV software stack is broadly divided into the **Navigation Group** (Visual Odometry, SLAM, and station-keeping) and the **Autonomy Group** (AI target seeking, semantic classification, and object tracking). A critical architectural conflict arises when attempting to share a single monocular camera payload across both of these highly specialized groups.

### 3.1 Conflicting Field of View and Mounting Angle Requirements

The Navigation and Autonomy groups possess fundamentally conflicting requirements regarding physical mounting angles, optical characteristics, and data throughput processing.

Visual Odometry (VO) algorithms, such as **VINS-Fusion** or **ORB-SLAM3**, rely on continuously tracking static environmental features (e.g., rocks, sand ripples, coral textures) to accurately estimate ego-motion and construct a map. To maximize the duration a feature remains in the frame and to ensure a high density of trackable points, the camera must be pointed toward the most texture-rich and stationary surface available, which is typically the seafloor.

Therefore, the navigation camera requires a downward-facing (nadir) or a steep 45-degree pitch mounting angle. Furthermore, VO algorithms often operate most efficiently on high-framerate, grayscale images to minimize tracking latency and maximize the mathematical robustness of optical flow algorithms or feature descriptors (like **ORB** or **FAST**).

Conversely, AI target seeking models (e.g., **YOLOv8** variants) require a forward-facing (horizontal) perspective to detect objects, navigational gates, or obstacles in the vehicle's forward flight path at maximum possible range. The AI pipeline strictly requires high-resolution RGB imagery, as chromatic data is vital for semantic classification — distinguishing an orange buoy from a gray obstacle, or identifying specific color-coded mission targets.

Attempting to share a single camera forces an unacceptable physical and functional compromise. Mounting a shared camera at a 45-degree downward angle reduces the AI's forward detection range to a few meters, while simultaneously reducing the VO's benthic observation time, degrading the performance of both mission-critical systems simultaneously.

### 3.2 Optical Port Physics: Refraction, Flat Ports, and Snell's Law Constraints

The architectural conflict extends deeply into the physics of the optical interfaces required by each subsystem. Underwater cameras must be enclosed in pressure housings to survive the environment, observing the water through either a **flat port** or a spherical **dome port**.

A flat port acts as a planar interface between the optical density of the external water (where the refractive index n_water is approximately 1.333) and the internal air of the housing (where n_air is approximately 1.0). According to **Snell's Law** (n_1 * sin(theta_1) = n_2 * sin(theta_2)), light rays entering the housing from the water refract away from the normal axis. This refraction mathematically magnifies the image, increasing the effective focal length of the camera lens by approximately 33 percent:

> ```
> f_uw = 1.333 * f_air
> ```

This magnification artificially narrows the camera's Field of View (FOV) by roughly 25 to 30 percent. Furthermore, flat ports introduce severe radial pincushion distortion and lateral chromatic aberration, particularly at the edges of the frame where the angle of incidence is greatest. While SLAM algorithms can mathematically model and partially correct this distortion using extended camera intrinsic matrices (e.g., the **Brown-Conrady model**), the physical loss of FOV is detrimental to early target detection. However, flat ports are significantly easier to manufacture, seal, and calibrate against extreme hydrostatic pressure, making them a highly reliable choice when a wide FOV is not strictly mandatory.

### 3.3 Dome Port Optics: Virtual Images, Entrance Pupil Alignment, and Material Stress

To eliminate refractive magnification and preserve the lens's true FOV, a spherical dome port is required. A dome port consists of concentric spherical surfaces made of glass, fused silica, or optical acrylic (**PMMA**). If the entrance pupil of the camera lens is perfectly aligned with the geometric center of curvature of the dome, all incoming principal light rays pass perpendicular to the interface (theta_1 = 0). Consequently, there is zero refraction at the boundaries, preserving the lens's true focal length, eliminating lateral chromatic aberration, and maintaining the original in-air FOV.

However, dome ports act as negative (diverging) lenses in water. They do not project the actual scene to the camera; instead, they create a curved "virtual image" located at a close distance in front of the port. This distance is calculated approximately as:

> ```
> L_virtual = 3.03 * r
> ```

where *r* is the radius of the dome port. This optical phenomenon requires the internal camera lens to have extreme macro-focusing capabilities, as the lens is effectively focusing on the virtual image located mere inches away, rather than the actual object at a distance.

Furthermore, achieving precise nodal alignment — matching the exact optical entrance pupil of the lens to the dome's geometric center — is highly sensitive to mechanical tolerances. Under deep-water operation, hydrostatic pressure causes micro-deformations in the port material. PMMA acrylic (n = 1.49) is prone to buckling and compression under extreme depth, which can shift the center of curvature and ruin the nodal alignment, reintroducing severe aberrations. Fused silica (n = 1.46) offers superior compressive strength and optical clarity for deep-sea operations, though at a significantly higher manufacturing cost.

The Autonomy group strictly requires a dome port. Preserving a wide FOV is non-negotiable for finding targets in vast volumetric spaces, and eliminating chromatic aberration is essential for maintaining the integrity of RGB gradients fed into the AI. Conversely, the Navigation group can operate highly effectively with a downward-facing flat port, as the magnification is acceptable for benthic feature tracking, and the flat port's mechanical simplicity ensures reliability. Thus, the differing optical physics unequivocally mandate dedicated payloads.

### 3.4 Edge-Processing Memory Architecture: NvBufSurface, Zero-Copy Pipelines, and Thread Contention

Even if a physical and optical compromise were attempted, the memory architecture of modern edge AI platforms dictates a strictly separated approach. The NVIDIA Jetson Orin architecture utilizes a unified memory subsystem where the ARM CPU and the Ampere GPU share physical RAM. However, passing high-resolution video streams between V4L2 capture APIs, SLAM tracking threads, and deep learning inference engines introduces severe memory bandwidth bottlenecks if handled improperly.

To mitigate memory bandwidth saturation, NVIDIA provides the **NvBufSurface** API, which allocates specialized **DMA (Direct Memory Access)** buffers. This enables **"zero-copy"** pipelines where the raw NV12 or RGB camera frames reside entirely in GPU memory and are passed by reference pointer between hardware components (e.g., the ISP, the hardware VIC encoder, and **TensorRT**) without ever executing a costly CPU-to-GPU memory copy.

If a single high-resolution camera is shared, the NvBufSurface pointer must be distributed across multiple distinct software processes via Inter-Process Communication (**IPC**). The GStreamer pipeline feeding the SLAM algorithm requires strict sequential processing with minimal latency to maintain feature tracking stability and accurate preintegration timing. Conversely, the TensorRT pipeline executing YOLOv8 thrives on batching and can tolerate slightly higher latency, but demands massive instantaneous memory bandwidth and GPU compute priority.

Multiplexing a single zero-copy buffer to both of these demanding processes causes severe thread contention, hardware synchronization locks, and variable pipeline latency. The system will stall as the SLAM thread waits for the AI thread to release the buffer lock, leading to dropped frames and inevitable navigation failure.

By deploying strictly dedicated camera architectures — a monochrome down-facing flat-port camera mapped exclusively to the Visual Odometry process, and an RGB forward-facing dome-port camera mapped exclusively to the TensorRT process — the system achieves complete hardware parallelization. The memory buses are decoupled, avoiding thread collision and ensuring deterministic execution times for both flight-critical navigation and tactical autonomy.

## 4. Environmental Degradation and Its Mathematical Impact on Artificial Intelligence

The most significant challenge in underwater computer vision is not computational, but rather the physical degradation of the optical signal itself. Unlike terrestrial AI models which assume uniform atmospheric transparency, underwater models must account for the exponential decay, absorption, and scattering of light in a dense fluid medium.

### 4.1 The Jaffe-McGlamery Optical Image Formation Model in Marine Environments

The propagation of light in a marine environment is mathematically defined by the industry-standard **Jaffe-McGlamery** image formation model. The total irradiance I(x) received by a camera pixel at spatial coordinate *x* is the linear superposition of three distinct optical components:

> ```
> I(x) = D(x) + F(x) + B(x)
> ```

where D(x) represents direct transmission, F(x) represents forward scattering, and B(x) represents backscatter. Each component affects the image, and consequently the AI feature extraction process, in vastly different ways.

**Direct Transmission D(x):**

This component represents the unscattered photons reflected directly from the target object to the camera sensor, containing the actual useful information about the scene. It is governed by the Beer-Lambert law of exponential decay:

> ```
> D(x) = J(x) * exp(-beta_D * z(x))
> ```

where J(x) is the true latent radiance of the object (what the camera would see in a vacuum), z(x) is the distance from the camera to the target, and beta_D is the wideband direct attenuation coefficient.

### 4.2 Jerlov Water Types and Wavelength-Dependent Attenuation

Crucially, the attenuation coefficient beta_D is highly dependent on the wavelength of light (lambda) and the specific inherent optical properties (**IOPs**) of the water body. These properties are categorized globally by the **Jerlov water types**, which range from clear open-ocean oceanic waters (Types I, IA, IB, II, III) to highly turbid coastal waters (Types 1C through 9C).

The attenuation of light is not uniform across the visible spectrum. Red wavelengths (approximately 600 nm) possess significantly higher attenuation coefficients than blue and green wavelengths (approximately 475 to 525 nm) in almost all Jerlov water types.

| Jerlov Water Type | Kd (475 nm) — Blue | Kd (525 nm) — Green | Kd (600 nm) — Red |
|---|---|---|---|
| Type I (Clear Ocean) | 0.023 | 0.051 | 0.242 |
| Type II | 0.044 | 0.070 | 0.261 |
| Type III | 0.101 | 0.118 | 0.328 |

As demonstrated by the attenuation coefficients, red light is absorbed exponentially faster than blue or green light. This causes severe non-linear color distortion, shifting the entire image to a monochromatic cyan or green hue at moderate distances. For an AI model trained on terrestrial datasets, this massive chromatic shift destroys the color-based feature embeddings, rendering semantic classification highly inaccurate.

### 4.3 Forward Scattering, Backscatter, and the Henyey-Greenstein Phase Function

**Forward Scattering F(x):**

This component consists of photons that are reflected by the target but are deflected at small angles by particulate matter before reaching the lens. This causes the photons to land on adjacent pixels rather than their true geometric projection, effectively convolving the direct signal with a low-pass spatial filter. The angular distribution of this forward scattering is typically modeled via the **Henyey-Greenstein volume scattering phase function**, controlled by an asymmetry parameter *g* (the mean cosine of the scattering angle). In highly forward-scattering water (g approaching 1), this phenomenon causes severe blurring of sharp edges and high-frequency textures.

**Backscattering B(x):**

Backscatter is the most destructive element in the optical model. It consists of ambient light or artificial strobe light that reflects off suspended particles in the water volume directly into the camera lens, without ever reaching the target. It is modeled as an additive hazy veil:

> ```
> B(x) = B_inf * (1 - exp(-beta_B * z(x)))
> ```

where B_inf is the veiling light magnitude at infinite distance, and beta_B is the backscatter attenuation coefficient. Backscatter raises the floor of the pixel intensity values across the entire image, drastically reducing the dynamic range and mathematically crushing contrast.

### 4.4 Impact on CSPDarkNet Feature Extraction and Spatial Gradients

The physical phenomena described by the Jaffe-McGlamery model have catastrophic mathematical implications for convolutional neural networks (CNNs) used in object detection, such as the **CSPDarkNet** backbone utilized by the YOLO algorithm.

CNNs rely fundamentally on discrete spatial gradients to extract features. Early convolutional layers act as edge detectors, computing approximations of the spatial derivatives (dI/dx and dI/dy).

When forward scattering F(x) blurs the edges of a target, it acts as a spatial low-pass filter, spreading the transition of pixel intensities over a wider pixel area. This mathematically reduces the magnitude of the spatial derivative. Simultaneously, the additive backscatter veil B(x) crushes the global contrast. If a target object with a latent pixel intensity of 200 is placed against a background of intensity 50 (a high-contrast delta of 150), severe backscatter might raise the background to 140 and attenuate the object to 160 (a low-contrast delta of just 20).

When these squashed, low-magnitude gradients are passed through the CSPDarkNet convolutional filters, the resulting activation maps are numerically weak. During backpropagation, these weak activations contribute to the **vanishing gradient problem**. The network struggles to differentiate the object's weak edges from the ambient noise of the backscatter veil, resulting in highly diluted feature embeddings at the neck of the network.

### 4.5 Destabilization of Complete Intersection over Union (CIoU) Loss

The degradation of feature maps directly destabilizes the loss functions used in the YOLO detection head, specifically the **Complete Intersection over Union (CIoU)** loss and the **Distribution Focal Loss (DFL)**.

The CIoU loss function is designed to optimize bounding box regression by penalizing the distance between the center points of the predicted and ground truth boxes, while also penalizing differences in aspect ratio.

> ```
> L_CIoU = 1 - IoU + (rho^2 / c^2) + alpha * v
> ```

where rho^2 is the Euclidean distance between the center points of the predicted and ground truth boxes, c is the diagonal length of the smallest enclosing box covering both, and v measures the consistency of the aspect ratio.

Because forward scattering heavily blurs the object boundaries, the network's spatial awareness fluctuates wildly from frame to frame. This causes the predicted bounding box coordinates to jitter. The rapid oscillation of the bounding box centers and aspect ratios injects high variance into the rho^2 and v terms of the CIoU formula, preventing the regression loss from smoothly converging during training, and causing bounding boxes to flicker erratically during real-time inference.

### 4.6 Entropy in Distribution Focal Loss (DFL) and Bounding Box Regression

Modern architectures like YOLOv8 replace traditional Dirac delta bounding box regression (predicting a single, hard coordinate) with **Distribution Focal Loss (DFL)**. DFL models the location of a bounding box edge as a continuous probability distribution across a set of discrete spatial bins. The predicted edge position *y* is the expected value of the probabilities P(n) across the bins:

> ```
> y = sum(P(n) * n)
> ```

In a clear, terrestrial image with sharp edges, the network outputs a sharp, low-variance probability distribution (e.g., 95 percent probability on bin 7, near 0 percent on adjacent bins), approximating a Dirac delta function.

However, in turbid underwater conditions, the Jaffe-McGlamery degradation blurs the physical boundary of the object across dozens of pixels. The neural network perceives this blur and mathematically expresses its uncertainty by outputting a flattened, high-variance probability distribution across the spatial bins (e.g., 15 percent probability spread evenly across bins 5 through 10).

The DFL function computes the cross-entropy between this predicted distribution and the ground truth. When the predicted distribution is highly entropic (flat) due to turbidity, the DFL loss spikes. The model attempts to optimize this loss, but the underlying physical data simply lacks the high-frequency information necessary to form a sharp distribution. This results in mathematically unstable box edges that expand and contract randomly as the backscatter noise shifts.

### 4.7 Dilution of Binary Cross-Entropy and Confidence Threshold Decay

Finally, the dilution of the feature gradients causes a fatal failure in the classification branch of the detection head. YOLO utilizes **Binary Cross-Entropy (BCE)** loss to generate an "objectness" and class probability score between 0.0 and 1.0.

Because the feature vectors extracted from the CSPDarkNet backbone are severely diluted by attenuation and backscatter, the resulting pre-activation logits are numerically weak. When passed through the Sigmoid activation function, these weak logits yield low probability scores, frequently hovering between 0.2 and 0.4. Because most AUV deployment pipelines enforce a strict confidence threshold (e.g., rejecting any detection below 0.5 to prevent false positives), the AI system will silently drop valid targets. The physical degradation mathematically forces the network's confidence below the operational threshold, rendering the autonomy group blind despite the target being ostensibly present in the frame.

## 5. Engineering Directives and Architectural Conclusions

Based on the exhaustive geometric, physical, and mathematical analysis provided, the architecture of the AUV Imaging Cameras subsystem must adhere strictly to the following engineering directives:

1. **Deprecate Stereo Vision**: The baseline limitations and severe GFLOPs computational burden of stereo matching render it unsuitable for turbid water edge computing. The system must transition to a monocular-acoustic sensor fusion paradigm, tightly coupling a downward-facing camera, an acoustic Doppler Velocity Log (DVL), an IMU, and a pressure sensor within a Factor Graph Optimization (GTSAM) framework to guarantee robust, drift-free metric scaling.
2. **Implement Dedicated Camera Payloads**: The optical and computational conflicts between navigation and autonomy cannot be bridged by a shared camera. The architecture must deploy dedicated optical payloads: a downward-facing camera behind a flat port strictly for Visual Odometry, and a forward-facing camera behind a precision-aligned fused silica dome port strictly for AI target seeking.
3. **Isolate Zero-Copy Memory Pipelines**: All camera streams must utilize NvBufSurface DMA buffers on the Jetson Orin platform. Dedicated camera feeds will ensure that the low-latency GStreamer SLAM pipelines and the high-throughput TensorRT AI pipelines do not contend for identical memory pointers, eliminating IPC synchronization locks and bandwidth throttling.
4. **Counteract Jaffe-McGlamery Degradation**: To prevent the collapse of CIoU and DFL loss functions due to forward scattering and backscatter, the YOLO architecture must be trained on datasets artificially degraded using Jerlov water-type attenuation coefficients. Furthermore, the inference deployment must implement dynamic confidence thresholding to accommodate the inherently lower Sigmoid activations caused by backscatter-induced vanishing gradients.

Execution of this architecture will yield an enterprise-grade imaging subsystem capable of sustained autonomous operation in highly degraded and unstructured subsea environments.

---

# Report 2: Imaging Sonar Subsystem

## Executive Summary

This document serves as an exhaustive, enterprise-grade engineering report defining the architecture, physical principles, and software design paradigms for the **Forward-Looking Sonar** imaging subsystem in autonomous underwater vehicles. Operating in degraded visual environments necessitates a highly deterministic transition between optical and acoustic sensors. This report rigorously analyzes the physical thresholds dictating this transition, the acoustic and geometric mechanisms enabling precise target differentiation, and the computational frameworks required for continuous, real-time sensor fusion and state estimation. The architectural guidelines presented herein are designed to ensure seamless perception and navigation capabilities in complex, visually denied marine environments.

## Activation Thresholds and Acoustic Physics

The operational transition from optical cameras to Forward-Looking Sonar is not an arbitrary software toggle, but rather a deterministically computed threshold governed by the fundamental physical limitations of electromagnetic wave propagation in aqueous media compared to the relative efficiency of mechanical longitudinal acoustic waves. Establishing the precise activation thresholds requires a rigorous mathematical understanding of both modalities, as well as an analysis of the environmental factors that dictate signal degradation.

### The Physics of Optical Attenuation and the Jaffe-McGlamery Model

Optical sensors operating in the visible spectrum suffer from severe attenuation in underwater environments. The degradation of the optical signal is mathematically formalized by the **Jaffe-McGlamery** underwater image formation model, which remains the foundational paradigm for understanding why and when optical cameras fail. The total irradiance received by a camera sensor at a specific pixel coordinate is defined as the linear combination of three distinct optical components. The equation is represented as:

> ```
> I(x) = D(x) + F(x) + B(x)
> ```

where I(x) is the degraded image recorded by the sensor, D(x) is the direct transmission of light from the target object to the camera lens, F(x) is the small-angle forward scattering, and B(x) is the backscatter.

The direct transmission component represents the actual signal that carries the geometric and textural information of the scene. It decays exponentially as a function of distance according to the Beer-Lambert law of exponential decay. The formula is expressed as:

> ```
> D(x) = J(x) * exp(-beta_c_D * z(x))
> ```

where J(x) is the latent true scene radiance (the image that would be captured in a vacuum), z(x) is the distance from the camera to the scene, and beta_c_D is the wideband direct attenuation coefficient. This attenuation coefficient is an Inherent Optical Property of the specific water body, derived from the sum of the wavelength-dependent absorption coefficient a(lambda) and the scattering coefficient b(lambda). Absorption converts electromagnetic energy into heat, while scattering redirects photons away from the optical axis of the camera lens.

Forward scattering, represented by F(x), involves photons that are scattered at small angles but still manage to reach the camera sensor. This phenomenon does not destroy the image entirely but introduces a severe low-pass filtering effect, manifesting as spatial blur and a devastating loss of high-frequency detail. Advanced underwater imaging models, such as the Henyey-Greenstein phase function, characterize this forward scattering by utilizing an asymmetry parameter to account for the highly directional nature of particulate scattering in water.

The most catastrophic element of the Jaffe-McGlamery model for autonomous navigation is the backscatter component, B(x). Backscatter represents the characteristic hazy veil commonly observed in underwater imagery, caused by ambient or artificial light reflecting off suspended particulate matter directly back into the camera lens before it ever reaches the target object. The backscatter is modeled as:

> ```
> B(x) = B_infinity * (1 - exp(-beta_c_B * z(x)))
> ```

where B_infinity is the value of the veiling light at an infinite distance, and beta_c_B is the wideband backscatter attenuation coefficient. In highly turbid environments, as the distance z(x) increases, the direct transmission D(x) approaches zero while the backscatter B(x) approaches B_infinity. Consequently, the ratio of backscatter to direct signal approaches infinity, plunging the Signal-to-Noise Ratio below the threshold of recoverability, regardless of the dynamic range of the camera sensor or the sophistication of the image processing algorithms.

### The Physics of Acoustic Wave Propagation

In stark contrast to electromagnetic waves, acoustic waves are longitudinal mechanical waves characterized by localized pressure variations propagating through an elastic medium. Because water is an extremely efficient medium for acoustic propagation due to its high density and relative incompressibility, acoustic attenuation is dramatically lower than optical attenuation. While optical attenuation is typically measured in decibels per meter, acoustic attenuation is measured in decibels per kilometer.

The transmission loss (TL) of an acoustic signal in water is a combination of geometric spreading (spherical or cylindrical) and chemical absorption. The fundamental equation is:

> ```
> TL = 20 * log10(R) + alpha * R
> ```

where R is the range in meters and alpha is the frequency-dependent acoustic absorption coefficient in decibels per meter. At the standard operating frequencies of high-resolution Forward-Looking Sonar systems (typically ranging from 500 kHz to 1.2 MHz to achieve the millimeter-level spatial resolution required for autonomous perception), the absorption coefficient alpha is relatively high compared to low-frequency tactical sonars. However, this attenuation remains orders of magnitude lower than optical attenuation. This fundamental physical disparity allows the sonar subsystem to maintain operational efficacy at ranges of 10 to 100 meters, even in entirely opaque, highly turbid water where optical visual range is reduced to zero.

### Establishing Deterministic Activation Thresholds

The precise threshold at which the Forward-Looking Sonar must assume primary navigational and object-detection authority over the optical camera system cannot be static. It must be a dynamically computed heuristic based on real-time environmental metrics, specifically turbidity, ambient illuminance, and calculated visual range.

Turbidity, measured in **Nephelometric Turbidity Units (NTU)**, quantifies the concentration of suspended particulates in the water column, which directly drives the scattering coefficient b(lambda). Oceanographer Nils G. Jerlov established a global classification system for seawater, dividing it into clear open-ocean classes (Jerlov Types I through III) and turbid coastal classes (Jerlov Types 1C through 9C). In Jerlov Water Types I and II, optical cameras can function effectively up to 15 to 20 meters, given sufficient ambient light. However, in coastal, estuarine, or benthic operational environments corresponding to Jerlov Types 5C through 9C, turbidity frequently exceeds 5 to 10 NTU. Extensive engineering analysis dictates that when onboard image entropy algorithms detect a spatial contrast drop corresponding to approximately 5 NTU, the backscatter component B(x) completely dominates the direct transmission D(x). At this precise threshold, optical feature extraction algorithms (such as FAST, ORB, or SuperPoint) experience a catastrophic drop in inlier matching ratios, and the acoustic subsystem must be prioritized to maintain state estimation integrity.

Ambient illuminance, measured in Lux, serves as the second critical activation threshold. At operational depths exceeding 20 meters, or during nocturnal deployments, ambient downwelling irradiance approaches zero, heavily governed by the diffuse attenuation coefficient for downwelling irradiance. While autonomous vehicles utilize artificial strobe illumination, these light sources exacerbate the backscatter phenomenon due to the geometric proximity of the light source to the optical axis of the camera. The radiation intensity distribution of artificial spotlights often creates a blinding veil of near-field backscatter in turbid conditions. When the onboard photometer registers ambient light falling below 10 Lux, and the strobe-induced backscatter reduces the image Signal-to-Interference Ratio below 3 dB, optical features become mathematically untrackable. The vehicle must automatically transition to acoustic perception.

Finally, the predicted visual range serves as the ultimate kinetic safety threshold. Using the **Duntley-Preisendorfer visibility theory**, visual range is defined as the distance at which the apparent contrast of an object falls to two percent, the limit of standard human and algorithmic detectability. The vehicle's autonomy stack calculates stopping distance based on its current velocity, hydrodynamic drag, and maximum reverse thruster output. The Forward-Looking Sonar must take over target tracking when the dynamically predicted visual range drops below the vehicle's safe stopping distance, typically calculated as 3.0 meters for a vehicle traveling at 1.5 meters per second. At ranges below this kinetic inertia threshold, relying solely on optical sensing guarantees collision in the event of an abrupt obstacle detection.

## Object Versus Background Differentiation

The ability of a Forward-Looking Sonar to distinguish artificial objects, such as navigational gates, docking stations, or submerged infrastructure, from natural background clutter like the seabed or geologic formations, relies on a complex interplay of acoustic physics dictated by the **Active Sonar Equation**. The primary mechanisms of differentiation are acoustic return intensity, material acoustic impedance, object geometry, and the deterministic formation of acoustic shadows.

### Acoustic Return Intensity and Target Strength

An imaging sonar detects objects by measuring the intensity of the returning acoustic echo. The standard metric used to quantify the acoustic reflectivity of an object is **Target Strength (TS)**. Target Strength represents the logarithmic ratio of the reflected acoustic intensity (Ir) measured at a reference distance of one meter from the acoustic center of the target, to the incident acoustic intensity (Ii) striking the target. The mathematical formulation is:

> ```
> TS = 10 * log10(Ir / Ii)
> ```

For a perfectly reflecting, rigid sphere of radius *a*, the target strength simplifies to the geometric cross-section formula:

> ```
> TS = 10 * log10(a^2 / 4)
> ```

However, artificial objects encountered by autonomous vehicles possess highly complex geometries that act as highly directional acoustic reflectors. The active sonar equation dictates the Echo Level (EL) received by the transducer array, formulated as:

> ```
> EL = SL - 2 * TL + TS
> ```

where SL is the Source Level of the acoustic pulse and TL is the one-way Transmission Loss due to spreading and absorption. To successfully differentiate a target object from the surrounding seabed, the Target Strength of the object must yield an Echo Level that is significantly higher than the Reverberation Level (RL) of the environment, achieving a positive Signal-to-Noise Ratio at the receiver array.

### Material Acoustic Impedance and Reflection Coefficients

The theoretical maximum magnitude of the acoustic return is fundamentally governed by the specific **acoustic impedance (Z)** of the target material relative to the surrounding aqueous medium. Acoustic impedance is defined as the product of the material's physical density (rho) and the longitudinal speed of sound within that material (c), expressed as:

> ```
> Z = rho * c
> ```

When an incident acoustic wave encounters a boundary between two distinct media, such as water and a submerged steel structure, the intensity reflection coefficient (Rc) is determined by the impedance mismatch between the two materials. The formula is expressed as:

> ```
> Rc = ((Z_target - Z_water) / (Z_target + Z_water))^2
> ```

Water exhibits an acoustic impedance of approximately 1.5 million Rayls. Structural steel, commonly used in offshore infrastructure and docking gates, possesses an immense acoustic impedance of approximately 47 million Rayls. Conversely, natural seabeds and rock formations exhibit impedances ranging from 3 million to 6 million Rayls.

Because the acoustic impedance of steel is monumentally higher than that of water or the natural seabed, the reflection coefficient Rc approaches 0.88, meaning that 88 percent of the incident acoustic energy is reflected back into the water column. This massive impedance mismatch results in a distinct, high-intensity acoustic highlight in the raw sonar imagery. The perception algorithms leverage this physical phenomenon by applying adaptive thresholding to the acoustic intensity maps, isolating engineered objects from acoustically softer, naturally occurring geologic backgrounds like sand, mud, or limestone.

### Object Geometry, Specularity, and Corner Reflectors

While material acoustic impedance governs the maximum possible magnitude of the reflection, the physical geometry of the object dictates the spatial direction of that reflection. Natural geologic formations typically possess rough, irregular surfaces relative to the acoustic wavelength. These irregular surfaces cause diffuse scattering, also known as **Lambertian reflection**, distributing the acoustic energy broadly across multiple directions.

Conversely, manufactured objects such as cylindrical poles, flat metal plates, and structural beams act as highly specular reflectors. If an acoustic beam strikes a flat steel plate at a non-orthogonal angle, the specular nature of the surface will deflect the majority of the acoustic energy away from the sonar receiver array, rendering the object acoustically invisible. This phenomenon is known as **specular loss**. However, manufactured structures also feature sharp right angles, intersecting planes, and hard corners. These specific geometries act as **acoustic corner reflectors**, trapping the acoustic wave and directing it perfectly back to the source, entirely independent of the incident angle. The sonar perception software is specifically trained to identify these bright, concentrated, angular glints to distinguish engineered structures from naturally occurring rocks, which rarely form perfect corner reflectors.

### The Deterministic Role of Acoustic Shadows

Despite the importance of target strength and acoustic impedance, the most deterministic feature for object differentiation and classification in Forward-Looking Sonar imagery is not the acoustic highlight, but the **acoustic shadow**.

An acoustic shadow is formed when an opaque target completely obstructs the propagation of the acoustic wave, preventing the acoustic energy from ensonifying the seabed located directly behind the target relative to the transducer. Crucially, unlike the acoustic highlight, which is highly dependent on material impedance and complex surface reflectivity, the acoustic shadow is a purely geometric and deterministic phenomenon. It is completely independent of the object's surface roughness, material composition, or acoustic reflectivity.

The morphological properties of the shadow, specifically its length (Ls), provide highly deterministic mathematical data regarding the physical height of the object (Ht) relative to the seabed. Based on the vehicle's current altitude above the seabed (A) and the measured slant range to the target (R), the absolute height of the object can be calculated using principles of similar triangles. The relationship is defined by the equation:

> ```
> Ls / Ht = (R + Ls) / A
> ```

Rearranging this formula yields the target height:

> ```
> Ht = (Ls * A) / (R + Ls)
> ```

In highly cluttered benthic environments featuring rocky seabeds, acoustic highlights from natural rocks can frequently mimic the highlights generated by artificial targets. However, rocks typically possess a low, rounded profile, generating short, irregular, and diffuse shadows. An engineered structure, such as a suspended docking gate, a pipeline, or a tall obstacle, will cast a distinct, elongated, and geometrically sharp shadow. Advanced convolutional neural networks applied to sonar object detection rely heavily on shadow morphology; by segmenting the shadow from the highlight and analyzing its geometric boundaries, the vehicle can mathematically verify the three-dimensional structure of an object using only a two-dimensional sonar array, thereby drastically reducing false positive classification rates.

## Sensor Fusion: Simultaneous versus Alternating Paradigms

The architectural integration of Forward-Looking Sonar and optical camera data within the vehicle's navigation and perception stack presents profound engineering challenges regarding computational overhead, data bandwidth limitations, and state estimation consistency. A fundamental architectural decision must be codified: should the optical and acoustic systems operate simultaneously in a continuous fusion matrix, or should they strictly alternate based on environmental visibility conditions?

### State Estimation Continuity and Factor Graph Optimization

For robust autonomous navigation in GPS-denied underwater environments, vehicles rely on continuous state estimation, traditionally executed via an Extended Kalman Filter, or more recently, via optimization-based Factor Graphs utilizing frameworks such as **GTSAM**. The multidimensional state vector continuously tracks the vehicle's precise position, orientation (roll, pitch, yaw), linear velocity, angular velocity, and the inherent biases of the Inertial Measurement Unit.

To maintain a continuous, non-diverging state estimate, simultaneous coupling of all available sensor modalities is mathematically and operationally superior to strict alternation. Strict alternation, defined as terminating the optical data stream and initializing the acoustic data stream upon crossing a turbidity threshold, introduces severe discrete discontinuities into the state estimation matrix. When abruptly switching modalities, the sudden change in observation models, feature tracking mechanisms, and measurement noise characteristics causes massive transient spikes in the error covariance matrix of the Extended Kalman Filter. This discontinuity frequently leads to momentary unbounded drift, scale ambiguity, or catastrophic total navigation failure.

Therefore, both optical and acoustic sensors must execute simultaneously within a dynamically weighted, tightly-coupled factor graph framework. In this architecture, visual features extracted via algorithms like ORB or SuperPoint, and acoustic spatial features extracted from the sonar, are jointly optimized alongside pre-integrated Inertial Measurement Unit factors and Doppler Velocity Log constraints. The optimal state vector is obtained by minimizing the sum of the squared Mahalanobis distances of the residual errors from all sensor factors.

The mathematical formulation for this joint optimization is expressed as minimizing the cost function:

> ```
> J = Sum(|r_imu|^2 / Sigma_imu) + Sum(|r_cam|^2 / Sigma_cam)
>   + Sum(|r_fls|^2 / Sigma_fls) + Sum(|r_dvl|^2 / Sigma_dvl)
> ```

where r represents the residual error of each respective sensor modality and Sigma represents the associated covariance matrix.

In this simultaneous architecture, the dynamic environmental conditions, such as turbidity and optical visual range, act directly upon the covariance matrices. As the vehicle penetrates turbid water, the reprojection error of the visual features increases exponentially. The factor graph algorithm detects this rising error and dynamically scales the visual covariance matrix (Sigma_cam) toward infinity. Mathematically, dividing the visual residual by an infinite covariance effectively nullifies the camera's influence on the overall state estimate without ever abruptly shutting off the data stream. This probabilistic weighting allows the Forward-Looking Sonar and the Doppler Velocity Log factors to smoothly and continuously dominate the nonlinear optimization process, ensuring mathematical continuity and preventing trajectory divergence.

### Computational Overhead, Bandwidth Limitations, and Zero-Copy Architectures

While simultaneous operation is mathematically optimal for state estimation, it places an immense burden on the vehicle's embedded compute architecture. Modern Forward-Looking Sonar systems generate high-frequency raw acoustic ping data requiring intense beamforming calculations, while high-resolution optical cameras generate immense pixel throughput. Streaming high-definition video at 30 frames per second alongside raw sonar beam data easily saturates internal USB 3.0, PCIe, or Gigabit Ethernet backplanes. Furthermore, standard memory pipelines that copy this massive data payload from the central processing unit to the graphics processing unit for deep learning inference introduce severe memory bottlenecking, latency spikes, and thermal throttling.

To mitigate this computational gridlock, enterprise-grade vehicles utilizing system-on-chip platforms like the NVIDIA Jetson Orin must implement strict **zero-copy** memory pipelines. Instead of routing camera and sonar data through standard user-space memory, data must be captured directly into Direct Memory Access buffers, specifically leveraging the **NvBufSurface** application programming interface and NVMM (NVIDIA Memory Management) architectures. By allocating pinned memory that is physically shared between the CPU and the integrated GPU, the software architecture completely circumvents costly host-to-device memory copy operations.

The zero-copy pipeline operates in a highly deterministic sequence. First, the Video4Linux2 framework captures optical frames directly into an NVMM buffer without CPU intervention. Simultaneously, the sonar software development kit writes the raw, beamformed acoustic data into an adjacent shared CUDA memory block. The GPU then executes tensor operations, such as feature extraction and object detection bounding box generation, directly on these pinned buffers in real-time. Finally, only the highly compressed output vectors, representing the coordinates and classes of detected objects, are passed back to the CPU-bound factor graph for state estimation.

To further optimize computational load and thermal management, the system should employ an asymmetric simultaneous framerate protocol. When the optical visibility heuristic indicates severe turbidity, the visual camera's acquisition frame rate is actively throttled from 30 Hz down to 1 Hz. The optical sensor remains active solely to evaluate environmental conditions, checking for clean water layers, and supplying very low-frequency structural constraints to the factor graph. Concurrently, the Forward-Looking Sonar operates at its maximum acoustic update rate, typically 10 to 15 Hz, to provide high-density spatial awareness. This asymmetric approach perfectly balances the mathematical necessity of continuous factor graph estimation with the strict physical constraints of embedded thermal and power management.

## Software Design: Probabilistic Object Detection and Regression Algorithms

Historically, detecting obstacles and targets in Forward-Looking Sonar imagery relied heavily on classical, hand-crafted computer vision techniques, such as Markov Random Fields for shadow segmentation or threshold-based blob detection. However, modern acoustic perception architectures demand the utilization of adapted deep convolutional neural networks, specifically customized variants of the **YOLOv8** architecture, to achieve high mean Average Precision in heavily cluttered benthic environments.

### Handling Acoustic Uncertainty with Distribution Focal Loss

Optical images are generally characterized by sharp, distinct pixel gradients at object boundaries. Conversely, Forward-Looking Sonar imagery is inherently characterized by acoustic speckle noise, multipath interference reverberation, and beamwidth-induced spatial smearing. Applying standard bounding box regression algorithms to an acoustic image is frequently ineffective because the precise pixel boundary of a target is inherently fuzzy and mathematically uncertain.

To solve this acoustic localization problem, the sonar perception stack must abandon standard regression techniques and instead utilize **Distribution Focal Loss** in conjunction with Complete Intersection over Union for bounding box generation. Standard detection models regress the bounding box coordinates as rigid Dirac delta distributions, forcing the network to predict a single, absolute fixed point for an edge. Distribution Focal Loss, however, treats the boundary of the acoustic object as a continuous probability distribution, acknowledging the inherent spatial uncertainty of the sonar return.

Instead of outputting a single coordinate, the neural network predicts a discrete probability distribution over a specific range of possible edge locations. The Distribution Focal Loss algorithm specifically encourages the neural network to focus the highest probability mass on the spatial bins immediately adjacent to the ground truth label. The loss function is mathematically expressed as:

> ```
> DFL(S_i, S_i+1) = -((y_i+1 - y) * log(S_i) + (y - y_i) * log(S_i+1))
> ```

where S_i and S_i+1 are the predicted probabilities of the two closest discrete spatial bins y_i and y_i+1, and y is the continuous ground truth coordinate.

By modeling the bounding box coordinates as flexible probability distributions rather than rigid direct regression targets, Distribution Focal Loss flawlessly handles the indistinct, smeared edges of acoustic highlights and the fuzzy gradients of acoustic shadows. This probabilistic approach allows the YOLOv8 model to accurately capture and quantify the spatial uncertainty of underwater objects, dramatically improving localization precision. Consequently, the vehicle can confidently navigate through complex environments, such as narrow docking gates, even when acoustic reverberation severely degrades the visual sharpness of the target structure.

### Multi-Scale Feature Enhancement for Acoustic Imagery

Because Forward-Looking Sonar data inherently lacks the high-frequency textural detail and RGB color data present in optical images, the neural network relies entirely on macro-level geometric shapes and low-frequency contrast gradients, specifically the spatial relationship between acoustic highlights and corresponding acoustic shadows. To prevent these sparse, highly critical features from vanishing during the sequential downsampling processes in deep convolutional layers, the network architecture must implement rigorous **Multi-Scale Feature Enhancement**.

The YOLOv8 backbone network, specifically the CSPDarknet structure, performs successive downsampling operations that compress shallow, high-resolution feature maps into deep, low-resolution semantic maps. In acoustic imagery, the shallow feature maps contain the critical detail required to sharply delineate an acoustic shadow from the surrounding reverberant seabed. By redesigning the Feature Pyramid Network to explicitly inject shallow branch data directly into the deep prediction heads, the model retains the high-resolution spatial gradients required to distinctly separate closely grouped obstacles. This integration of Multi-Scale Feature Enhancement ensures that the network possesses both the deep semantic understanding required to classify an object and the shallow spatial resolution required to accurately localize its acoustic shadow.

| Feature Type | Optical Camera (Clear Water) | Forward-Looking Sonar (Turbid Water) | Primary Classification Mechanism |
|---|---|---|---|
| Signal Attenuation | High (dB/m) | Low (dB/km) | Direct Transmission vs. Acoustic Return |
| Object Texture | High-frequency details | Low-frequency speckle | RGB gradients vs. Acoustic Impedance |
| Object Geometry | Visual Edge Detection | Specular vs. Diffuse Reflection | Photometric vs. Geometric Shadowing |
| Edge Certainty | High (Dirac Delta applicable) | Low (Probability Distribution required) | Standard Regression vs. Distribution Focal Loss |
| State Estimation | Low Covariance (Primary) | High Covariance (Primary) | Visual Odometry vs. Acoustic Feature Tracking |

## Conclusions and System Architecture Directives

The engineering design and systems integration of an Imaging Sonar subsystem into a highly autonomous underwater vehicle requires a masterful synthesis of acoustic physics, probabilistic mathematics, and highly optimized software engineering. Based on the exhaustive analysis provided in this report, the global architecture must strictly adhere to the following operational directives to ensure mission success.

First, the transition between optical and acoustic modalities must be entirely deterministic, driven by mathematical thresholds rather than heuristic approximations. The autonomy stack must autonomously calculate the Jaffe-McGlamery backscatter limit in real-time, defaulting to acoustic primacy when the dynamically calculated visual range drops below the vehicle's safe stopping distance, or when environmental turbidity exceeds 5 NTU.

Second, the neural network perception algorithms must be structurally weighted to prioritize the identification of acoustic shadows over acoustic highlights. Acoustic shadows are purely geometric phenomena that are totally immune to the highly variable acoustic impedance of different target materials. Relying on shadow morphology ensures highly deterministic obstacle avoidance and target recognition, eliminating the false positives generated by highly reflective, naturally occurring rock formations.

Third, the optical and acoustic subsystems must operate simultaneously at all times. To prevent catastrophic state estimation divergence, sensor data must be fused within a continuous factor graph architecture where the covariance matrix of each specific sensor is dynamically scaled based on environmental optical clarity and acoustic signal-to-noise ratios. Strict switching between sensors is architecturally forbidden.

Finally, to overcome the severe data bandwidth limitations inherent to embedded systems, all sensor processing pipelines must utilize direct GPU memory access protocols, specifically **NvBufSurface** on Jetson architectures, establishing a **zero-copy** data flow. Furthermore, object detection executed on the acoustic data must utilize probabilistic bounding box models, specifically **Distribution Focal Loss**, to mathematically account for the inherent edge uncertainty, beam smearing, and speckle noise native to underwater acoustic propagation. By rigidly adhering to these architectural paradigms, the vehicle will achieve a level of situational awareness capable of sustaining autonomous operations in the most hostile and visually degraded marine environments.

---

# Report 3: Lighting Subsystem

## Executive Summary

This engineering report establishes the architectural and physical design paradigms for the lighting subsystem of an autonomous underwater vehicle. Operating in highly turbid aquatic environments dictates a lighting architecture governed by optical physics, specifically the **Jaffe-McGlamery** image formation model. This document resolves three critical engineering challenges: the determination of optimal color temperature and intensity based on wavelength-dependent attenuation, the hardware synchronization of strobe lighting to maximize the Signal-to-Noise Ratio while minimizing power consumption, and the spatial geometry of luminaire mounting to eliminate near-field backscatter.

## Optimal Intensity, Color Temperature, and Wavelength-Dependent Attenuation

The selection of optimal light intensity and color temperature is dictated by the inherent optical properties of the specific water body, mathematically formalized within the direct transmission component of the Jaffe-McGlamery model. Direct transmission decays exponentially according to the Beer-Lambert law, expressed as:

> ```
> D(x) = J(x) · exp(−β_D · z(x))
> ```

where J(x) is the latent scene radiance, z(x) is the propagation distance, and β_D is the wideband direct attenuation coefficient.

The attenuation coefficient β_D is not uniform but is highly wavelength-dependent, being the sum of the absorption coefficient a(λ) and the scattering coefficient b(λ). In clear oceanic waters (**Jerlov Types I and II**), blue wavelengths near 475 nm exhibit the lowest attenuation. However, in highly turbid coastal and benthic environments (Jerlov Types 1C through 9C), suspended particulate matter and dissolved organic compounds heavily absorb blue light. Consequently, the optimal transmission window shifts toward the green spectrum, specifically around 525 nm. Red wavelengths (600 nm and above) are absorbed almost immediately due to the molecular resonance of water, making warm color temperatures practically useless for long-range illumination.

Therefore, to achieve maximum penetration and lowest backscatter in turbid water, the lighting subsystem must utilize luminaires with a high color temperature, specifically between **5000 K and 6500 K**, effectively omitting the highly attenuated red spectrum. In environments of extreme turbidity, utilizing monochromatic green LED arrays tuned to 525 nm is strongly recommended, as this perfectly matches the minimum attenuation coefficient of coastal water, maximizing the direct signal D(x) while ensuring that the limited vehicle power budget is not wasted on immediately absorbed red wavelengths or heavily scattered blue wavelengths. The intensity must be dynamically adjustable to prevent saturation of the camera sensor, scaling with the inverse square of the target distance and the specific Jerlov attenuation curve.

## Hardware Synchronization and the Mathematics of Strobe Lighting

A critical architectural flaw in standard underwater vehicles is the reliance on continuous forward illumination. Continuous lighting not only drains the vehicle's battery but also continuously illuminates the entire water column, severely degrading the image through constant backscatter generation. The engineering solution is strict hardware synchronization utilizing **strobe lighting** (flash synchronization) tightly coupled with the camera's exposure window.

The backscatter component B(x) in the Jaffe-McGlamery model is an additive hazy veil that integrates over time. By utilizing a high-intensity, microsecond-duration strobe synchronized exactly with a global shutter camera, the system maximizes the instantaneous Signal-to-Interference Ratio. Mathematically, the energy E consumed by the light is the integral of power over time, simplified as:

> ```
> E = P · t
> ```

A continuous 100 W LED operating at 30 frames per second illuminates the scene for 33.3 ms per frame, consuming significant power while generating a relatively low instantaneous irradiance. Conversely, a strobe discharging 1000 W for a duration of 1 ms delivers massive instantaneous irradiance to the target while consuming radically less overall energy per frame.

Hardware synchronization must be implemented via a dedicated trigger circuit. The embedded compute module commands the camera to open its global shutter. Once the sensor is fully active, a hardware pulse triggers the strobe driver. The strobe fires its microsecond pulse, flooding the scene with high-intensity light that overpowers the ambient downwelling irradiance E(d,λ). The shutter closes immediately after the pulse. This paradigm freezes the hydrodynamic motion of the vehicle, eliminating motion blur in the direct transmission signal D(x)_blurred, while drastically reducing the temporal integration of ambient backscatter noise and conserving critical battery reserves for the propulsion and navigation subsystems.

## Spatial Mounting in High Particle Density and the Snowstorm Effect

The physical geometry of the lighting payload relative to the optical axis of the camera is the most deterministic factor in backscatter mitigation. In water with high particle density, continuous forward illumination mounted parallel and adjacent to the camera lens induces a catastrophic failure known as the **snowstorm effect**.

The snowstorm effect is mathematically driven by the backscatter integral:

> ```
> B(x) = B_∞ · (1 − exp(−β_B · z(x)))
> ```

combined with the inverse square law of light propagation. When the luminaire is mounted close to the lens (a narrow baseline), the highest intensity of the light cone intersects the camera's field of view immediately in front of the lens. The suspended particles in this near-field region reflect the high-intensity light directly back into the sensor. Because these particles are extremely close to the receiver, the attenuation is near zero, and their reflected signal completely overpowers the exponentially decayed direct signal D(x) returning from the actual target.

To maximize the Signal-to-Interference Ratio, the architecture must mandate off-axis lighting geometry utilizing a wide baseline. The strobes must be mounted on extended lateral arms, pushing them as far away from the camera's optical axis as the vehicle's hydrodynamic envelope allows. Furthermore, the strobes must be physically angled outward, away from the camera's center line.

The objective of this geometry is to ensure that the inner edge of the light cone does not intersect the camera's field of view in the near-field volume. The intersection of illumination and observation must only occur at the predetermined focal distance of the target object. By maintaining an unilluminated dark zone immediately in front of the camera lens, near-field particles are not ensonified, dropping the backscatter integration in the critical near-field volume to effectively zero. This wide-baseline, off-axis mounting geometry, combined with microsecond strobe synchronization and wavelength-optimized high-Kelvin emitters, constitutes the optimal lighting architecture for high-density particulate environments.

---

# Report 4: AI Processing Subsystem

## Executive Summary

This engineering report establishes the architectural, mathematical, and hardware isolation paradigms for the artificial intelligence processing subsystem of an autonomous underwater vehicle. The integration of complex convolutional neural networks for real-time target detection requires immense computational throughput. This document resolves three critical engineering challenges: defining the strict mathematical hardware requirements to sustain real-time frame rates for obstacle avoidance, evaluating the mathematical and thermal efficacy of **INT8** quantization via **TensorRT**, and establishing the absolute necessity of physical hardware isolation between high-level autonomy processors and low-level deterministic flight controllers.

## FPS Targets and Real-Time Computational Requirements

To run real-time YOLO object detection algorithms successfully for dynamic obstacle avoidance, the autonomous underwater vehicle must sustain an absolute minimum target of **30 Frames Per Second (FPS)**. Operating below this threshold introduces unacceptable latency between visual capture and actuator response, jeopardizing the closed-loop control required to navigate safely around sudden hydrodynamic obstacles. The processing power required to sustain this is a direct function of the model's complexity and the hardware's theoretical throughput. A standard YOLOv8s model requires approximately 28 Giga-Floating Point Operations (GFLOPs) per inference. To sustain 30 FPS, the hardware must reliably deliver a minimum computational throughput calculated as 30 frames multiplied by 28 GFLOPs, equating to 840 GFLOPs, or 0.84 Tera-Floating Point Operations Per Second (TFLOPS) of continuous utilization. The **NVIDIA Jetson Orin Nano 8GB** architecture is ideally suited for this requirement, as its Ampere GPU provides 1024 CUDA cores and 32 Tensor Cores, delivering a peak INT8 sparse computing power of 40 TOPS to 67 TOPS, and an estimated FP16 dense computing power of up to 17.5 TFLOPS. Even when factoring in real-world utilization bottlenecks such as memory bandwidth limitations and kernel launch overheads, which typically restrict utilization to between 5 and 20 percent of theoretical peak, the Orin Nano possesses abundant processing headroom to exceed the 0.84 TFLOPS requirement.

## INT8 Quantization and TensorRT Engine Mathematics

The transition from **FP32** (32-bit floating point) to **INT8** (8-bit integer) quantization induces a slight degradation in the mean Average Precision (mAP) of the neural network, typically dropping from approximately 37.3 percent to 36.2 percent for models like YOLOv8n. This marginal loss in mAP is an entirely acceptable engineering trade-off for subsea edge deployment, as dynamic obstacle avoidance relies on robust bounding box center coordinates and class verification rather than pixel-perfect semantic edge delineation. Exporting a PyTorch YOLO model to a **TensorRT** engine with INT8 calibration drastically increases FPS while lowering thermal output through specific mathematical and architectural optimizations. INT8 quantization maps the expansive FP32 value range to a highly constrained integer space of [-128, 127]. This 4x reduction in precision dramatically slashes memory bandwidth requirements, which is the primary bottleneck in unified memory architectures. Furthermore, the TensorRT compiler performs aggressive graph optimization, fusing the Convolution, Batch Normalization, and ReLU layers into a single executable kernel. By packing these dense kernels and executing them directly on the Orin Nano's dedicated Tensor Cores, the GPU compute latency for a YOLOv8n model drops from roughly 20.88 milliseconds in native PyTorch to just 3.49 milliseconds in INT8 TensorRT. This efficiency translates directly to the thermal and power domains, reducing the GPU power draw from 11.4 or 14.3 Watts down to 12.5 Watts, thereby preserving critical battery reserves and mitigating the risk of thermal throttling inside the sealed pressure hull.

## Hardware Isolation: Autonomy versus Navigation

The architectural relationship between the autonomy processor running the artificial intelligence pipelines and the navigation flight controller must be defined by strict physical hardware isolation. The mission computer, such as the Jetson Orin Nano, handles highly variable, non-deterministic workloads including computer vision, YOLO tensor processing, and sensor fusion. Under maximum load states, such as the MAXN power mode, the Jetson compute module generates immense heat and demands variable power draws that can induce system instability or kernel panics. Conversely, the flight controller executes deterministic, real-time functions, including gyroscopic stabilization, actuator control, and critical failsafe logic. If these two fundamentally different computing domains share the same physical board or processing environment, an AI-induced out-of-memory error or thermal shutdown would inherently crash the flight control loops, resulting in catastrophic loss of the vehicle. Therefore, the architecture mandates a dedicated, low-power microcontroller, such as an **STM32** processing unit, to serve exclusively as the flight controller. This isolation decouples flight safety from the rapidly evolving and computationally intensive AI workloads. In the event that the autonomy board exceeds its thermal envelope or crashes due to processing overloads, the isolated STM32 microcontroller remains fully operational, relying on its independent firmware to maintain vehicle stability, override mission parameters, and execute the critical safety protocol of driving the thrusters to surface the vehicle securely.
