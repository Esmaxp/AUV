# Global Architecture, Security, and Scalability Report for Gamestone Algorithm and Microservices

## 1. Executive Summary and Architectural Vision

The transition from monolithic architectures to distributed microservices improves scalability, fault tolerance, and flexibility but introduces significant challenges regarding data consistency, network latency, and cybersecurity. The logical errors, performance bottlenecks, and security vulnerabilities identified in the current Gamestone algorithm codebase necessitate a fundamental redesign based on event-driven and zero-trust principles. This transformation should align with industry standards defined by the Cloud Native Computing Foundation (CNCF), the Open Worldwide Application Security Project (OWASP), and the National Institute of Standards and Technology (NIST).

This comprehensive research report provides universally applicable architectural guidelines to answer stakeholder questions. It analyzes synchronous and asynchronous communication, distributed transaction patterns, cloud-native caching, data masking, and DevSecOps pipelines. The goal is to evolve the Gamestone algorithm into a highly observable, globally scalable, and secure enterprise architecture.

## 2. Gamestone Algorithm Logic Errors & Improvement Strategies

The logic errors identified in the Gamestone algorithm stem from structural weaknesses that must be addressed at the architectural level rather than with superficial patches.

- **Missing Input Validation**: In distributed architectures, external inputs must be considered untrusted. Failing to validate data payloads can lead to runtime exceptions and security breaches. Mitigation requires strict contract enforcement at the API Gateway or service level using JSON Schemas or gRPC Protocol Buffers.
- **Performance Bottlenecks**: Heavy loops or unoptimized recursive functions cause exponential delays during large computations. To resolve this, computationally expensive tasks should be offloaded to asynchronous background workers, and frequently accessed outputs must be cached.
- **Concurrency & Race Conditions**: Lack of proper transaction management or lock mechanisms leads to data corruption when multiple threads access shared resources simultaneously. Implementing optimistic concurrency control and versioning is highly recommended to improve throughput without the heavy performance penalties of pessimistic locks.
- **Memory Leaks & Error Handling**: Failing to close database connections or file streams can lead to memory exhaustion. Systems must utilize automated resource cleanup (e.g., connection pooling) and structured logging with correlation IDs to replace silent failures with traceable, observable error reporting.

## 3. Stakeholder Engagement & Architectural Requirements

Gathering requirements from stakeholders is critical for shaping the microservice boundaries and technology stack.

- **Functional & Non-Functional Requirements**: Stakeholders must clearly define the algorithm's expected input/output formats, target transaction volumes, and latency tolerances. High concurrency systems with strict latency requirements (<1s) demand different databases and communication protocols compared to batch-processing workloads.
- **Data Schema & Privacy**: Defining database normalization rules and adhering to data privacy frameworks like GDPR or KVKK is legally and technically mandatory.
- **Deployment & Testing**: Infrastructure choices (e.g., Kubernetes, Docker), continuous integration (CI/CD) pipelines, and End-to-End (E2E) testing strategies must be agreed upon to ensure reliable production rollouts.

## 4. Distributed Communication: Synchronous vs. Asynchronous Models

The choice of inter-service communication profoundly affects system latency, resilience, and coupling.

- **Synchronous Communication (REST & gRPC)**: In this model, the client waits for the server's response. REST (HTTP/1.1) is text-based (JSON), highly compatible, and ideal for external/public APIs. However, it suffers from parsing overhead and connection management bottlenecks. gRPC, built on HTTP/2 and Protocol Buffers, transmits binary data and supports multiplexing. Benchmarks show gRPC provides 7-10x higher throughput and significantly lower latency (5-20ms) compared to REST, making it the premier choice for internal microservice communication.
- **Asynchronous Communication (Message Queues)**: Technologies like RabbitMQ or Apache Kafka use an event-driven, "fire-and-forget" model. This completely decouples services, ensuring high fault tolerance. If a consuming service crashes, messages are safely retained in the queue until the service recovers.

Attempting to chain numerous synchronous REST calls across microservices leads to the dreaded "Distributed Monolith" anti-pattern, where a single service failure cascades through the entire system. Gamestone's heavy computational tasks must leverage asynchronous queues, while immediate user-facing validations can utilize gRPC or REST.

| Feature | REST API (HTTP/1.1) | gRPC (HTTP/2) | Message Queues (AMQP/Kafka) |
|---|---|---|---|
| Paradigm | Synchronous (Blocking) | Synchronous / Streaming | Asynchronous (Event-Driven) |
| Data Format | JSON, XML (Text) | Protocol Buffers (Binary) | JSON, Avro, Binary |
| Latency | Medium-High (50-200ms) | Very Low (5-20ms) | Variable (Queue dependent) |
| Coupling | High (Temporal) | High (Temporal) | Low (Loose Coupling) |
| Best For | Public APIs, Web Clients | High-performance internal APIs | Background tasks, High volume |

## 5. Distributed Data Consistency and Transaction Management

In a microservices architecture, the "Database per Service" pattern breaks traditional ACID transactional guarantees. Maintaining consistency across distributed boundaries requires specialized patterns.

- **Two-Phase Commit (2PC)**: 2PC relies on a central coordinator and enforces strict consistency through a "prepare" and "commit" phase. However, 2PC is considered an anti-pattern for highly scalable microservices because it locks database rows across all participating services until the transaction finishes. This blocks other operations and severely degrades throughput.
- **The Saga Pattern**: The industry standard for distributed transactions is the Saga pattern. It abandons strict atomicity in favor of "eventual consistency." A Saga breaks a large distributed transaction into a sequence of local transactions. If one step fails, the system executes "Compensating Transactions" to undo the previously committed steps. Sagas eliminate global locks, thereby providing the massive scalability and high availability required for systems like Gamestone.

## 6. Cloud-Native Databases and Caching Strategies

Selecting the right database depends on the read/write profile and the schema rigidity. Relational databases (e.g., PostgreSQL) excel at ACID transactions, while NoSQL solutions (e.g., MongoDB, ScyllaDB) are preferred for flexible schemas and horizontal write scaling.

Regardless of the database, a distributed in-memory cache (like Redis) is essential to reduce latency.

- **Cache-Aside (Lazy Loading)**: The application checks the cache; on a miss, it queries the database and updates the cache. This is the most common and resource-efficient pattern for read-heavy workloads.
- **Write-Through**: The application updates the cache and the database simultaneously. It ensures strong consistency and eliminates stale reads but adds latency to write operations.
- **Write-Behind (Write-Back)**: Data is written immediately to the cache and asynchronously flushed to the database. This offers extreme write speeds but carries a risk of data loss if the cache crashes before syncing.
- **Cache Invalidation**: To prevent a "Thundering Herd" (where many keys expire at the exact same moment, crushing the database with a sudden spike in queries), TTL (Time to Live) values must always include a randomized "Jitter" to stagger expirations.

## 7. Event-Driven Autoscaling with KEDA

Standard Kubernetes Horizontal Pod Autoscalers (HPA) rely on CPU and Memory metrics. However, for asynchronous message queues, CPU usage does not reflect the actual backlog of work.

KEDA (Kubernetes-based Event Driven Autoscaling), a CNCF graduated project, solves this by scaling workloads based on external events, such as Kafka lag, RabbitMQ queue depth, or AWS SQS visible messages. Crucially, KEDA supports "Scale-to-Zero," spinning down pods completely when queues are empty, drastically reducing cloud compute costs.

## 8. Microservices Security: Zero Trust and Service Mesh

In modern architectures, perimeter security is insufficient. The NIST SP 800-204 series mandates a "Zero Trust" architecture, where no service is inherently trusted, regardless of its network location.

This is practically implemented using a Service Mesh (e.g., Istio, Linkerd) and the "Sidecar Proxy" pattern. A lightweight proxy (like Envoy) runs alongside every application container, intercepting all inbound and outbound traffic.

- **Mutual TLS (mTLS)**: The sidecar enforces mTLS, authenticating both the client and the server cryptographically and encrypting all internal service-to-service communication.
- **Decoupled Authorization**: Security logic is removed from the application code. The mesh acts as a Policy Enforcement Point (PEP) for Attribute-Based Access Control (ABAC), validating JWTs and ensuring least privilege dynamically.

## 9. API Security and OWASP Top 10 (2023)

APIs are the primary attack vector for distributed systems. The OWASP API Security Top 10 outlines the most critical threats:

- **API1:2023 - Broken Object Level Authorization (BOLA)**: The most common vulnerability, where an authenticated user manipulates an object ID (e.g., `/api/users/100` to `101`) to access unauthorized data. Every request must validate the user's specific ownership of the requested resource.
- **API4:2023 - Unrestricted Resource Consumption**: Automated bots can overwhelm APIs, causing Denial of Service or massive cloud billing spikes. This is mitigated through robust rate-limiting algorithms, such as the Token Bucket, applied at the API Gateway level.
- **API7:2023 - Server-Side Request Forgery (SSRF)**: Exploited when an API blindly fetches data from a user-supplied URL, potentially exposing internal cloud infrastructure.

To secure APIs, organizations must implement defense-in-depth: API Gateways for rate-limiting, JWT validation for identity, and OAuth2 scopes for specific permissions.

## 10. Data Privacy and ISO 27001 (Annex A 8.11)

Compliance with GDPR, KVKK, and ISO/IEC 27001:2022 (Annex A 8.11) requires strict handling of Personally Identifiable Information (PII).

- **Data Masking / Redaction**: Irreversibly obscuring data for display (e.g., `j***@email.com`) or structured logging to ensure PII does not leak into monitoring tools.
- **Pseudonymization**: Replacing direct identifiers with reversible tokens. Under GDPR, this is still classified as personal data, and the mapping keys must be heavily secured.
- **Anonymization**: A permanent, irreversible transformation where the data can never be linked back to an individual. Only truly anonymized data falls outside the scope of GDPR/KVKK compliance.

## 11. DevSecOps and Continuous Security

Security cannot be an afterthought in cloud-native systems. NIST SP 800-204C requires shifting security "left" by integrating automated tools into the CI/CD pipeline.

- **SAST (Static Application Security Testing)**: Tools like SonarQube or Semgrep scan the source code during Pull Requests to find hardcoded secrets and logic flaws (like SQL injection) before deployment.
- **SCA (Software Composition Analysis)**: Tools like Trivy or Snyk scan third-party dependencies and container images for known vulnerabilities (CVEs).
- **DAST (Dynamic Application Security Testing)**: Tools like OWASP ZAP attack the running application in a staging environment to detect runtime misconfigurations and authentication bypasses.

## 12. Observability and Microservice Resilience

In distributed systems, a single user request spans multiple services.

- **Observability**: Using the OpenTelemetry standard, an API Gateway injects a Correlation ID (Trace ID) into HTTP/gRPC headers, allowing developers to trace the request end-to-end and pinpoint latency across microservices.
- **Resilience Patterns**: System failures must be contained. The Circuit Breaker pattern cuts off traffic to a failing downstream service to prevent cascading failures. Retries manage transient network errors but must include Exponential Backoff and Jitter to avoid overwhelming the recovering service. Bulkheads isolate resources so a failure in one module does not exhaust system-wide threads.

## 13. Strategic Conclusion

Successfully modernizing the Gamestone algorithm requires replacing outdated, monolithic habits with cloud-native engineering disciplines. Internal communication should leverage high-performance gRPC, while heavy compute workflows must be decoupled using asynchronous message queues governed by KEDA for dynamic, cost-efficient scaling. Data consistency should be managed via Saga patterns, and security must be enforced holistically using a Service Mesh, mTLS, and CI/CD-integrated DevSecOps tooling. Adhering to these standards ensures the Gamestone system will be highly resilient, performant, and secure against the modern API threat landscape.
# Global Architecture, Hardware, and Software Design Report for a Budget-Constrained Autonomous Underwater Vehicle (AUV)

## 1. Executive Summary and Architectural Vision

The transition from monolithic, tightly coupled architectures to distributed, event-driven systems—as demonstrated in the modernization of the Gamestone algorithm—provides a foundational blueprint for designing robust embedded robotics. In the context of an Autonomous Underwater Vehicle (AUV) engineered for highly competitive, resource-constrained environments such as RoboSub or TEKNOFEST, the principles of microservices, fault tolerance, and scalable asynchronous communication must be transposed from cloud infrastructure directly to the vehicle's embedded systems.

Traditional AUV software often succumbs to the "Distributed Monolith" anti-pattern, where a single computational bottleneck (e.g., blocking image processing loops or synchronous sensor polling) cascades into catastrophic failure across navigation and control layers. The logical errors, concurrency issues, and performance bottlenecks identified in legacy monolithic frameworks necessitate a fundamental redesign based on decoupled nodes, localized state estimation, and deterministic data distribution.

This exhaustive research report establishes a highly observable, globally scalable, and universally applicable architectural framework for a student-budget AUV. By applying cloud-native philosophies to hardware selection, middleware frameworks, and algorithmic design, the proposed system ensures resilience in the harsh, communication-denied underwater environment. The analysis evaluates state-of-the-art computational hardware, flight controllers, deep learning vision pipelines (YOLOv8 via TensorRT), sensor fusion methodologies (Extended Kalman Filters), and hydrodynamic propulsion systems. The objective is to identify the optimal price-to-performance components that satisfy strict budgetary constraints (often sub-$2,000) while fulfilling the deterministic, real-time requirements of autonomous subsea operations.

## 2. Distributed Software Architecture: Transposing Microservices to Robotics

The modernization of the Gamestone algorithm highlighted the dangers of synchronous blocking operations and memory leaks in monolithic systems. In an AUV, a blocking computer vision function waiting for a camera frame can stall the primary Proportional-Integral-Derivative (PID) control loop, leading to immediate loss of vehicle stability. To resolve this, the software architecture must migrate to a distributed middleware framework that completely isolates the perception, navigation, and control layers.

### 2.1 The Robot Operating System (ROS 2) as an Event-Driven Backbone

Attempting to chain numerous synchronous serial commands across a robot's internal components mirrors the fragile REST API chains seen in poorly designed microservices. The architectural solution is the integration of the Robot Operating System 2 (ROS 2), specifically the Humble Hawksbill distribution. ROS 2 operates on the Data Distribution Service (DDS) standard, which serves as the robotic equivalent of asynchronous message queues like Apache Kafka or RabbitMQ.

DDS utilizes an event-driven, "fire-and-forget" publish-subscribe model. This thoroughly decouples the system's nodes. If a computationally heavy vision node crashes due to memory exhaustion, the DDS middleware ensures that the independent navigation node continues to receive critical telemetry (e.g., depth and attitude) from the hardware interface, safely maintaining the vehicle's posture until the vision node automatically restarts. Furthermore, ROS 2 implements Quality of Service (QoS) profiles, allowing developers to define network reliability, durability, and latency deadlines at the protocol level—features strictly required for real-time robotic control.

### 2.2 Hardware Abstraction: MAVLink and micro-ROS

The physical computing architecture is divided into two distinct domains: the high-level Companion Computer (the "cerebral cortex" handling complex algorithms and AI) and the low-level Flight Controller (the "brainstem" handling real-time sensor fusion and motor mixing).

Communication across this hardware boundary must mirror the strict contract enforcement seen in API Gateways. The industry standard is the MAVLink protocol, running asynchronously over a UART serial connection. The companion computer executes high-computation tasks and translates the output into MAVLink RC override or `SET_POSITION_TARGET_LOCAL_NED` messages.

For custom peripheral microcontrollers (e.g., an Arduino or ESP32 managing a specialized torpedo launcher or gripper), the architecture leverages micro-ROS. This framework extends the DDS transport layer (via Micro XRCE-DDS) directly to resource-constrained microcontrollers, eliminating the need for custom serial parsers and establishing a seamless, end-to-end event-driven architecture across the entire vehicle.

## 3. Core Computational Hardware and Flight Controllers

Selecting the correct processing hardware requires balancing the massive computational demands of modern computer vision against the strict financial limitations of student projects.

### 3.1 Companion Computer: Price-to-Performance Analysis

The companion computer is responsible for executing the ROS 2 graph, processing video streams, running autonomous decision-making algorithms, and logging telemetry for observability.

**The Baseline Choice: Raspberry Pi 5**

The Raspberry Pi 5 represents the optimal entry point for budget-constrained AUVs. Priced under $100, it provides a quad-core ARM processor capable of running standard computer vision pipelines (such as OpenCV-based color masking) and basic autonomy scripts. Its primary advantages are extreme affordability, low power consumption (under 10W load), an ultra-lightweight footprint, and vast community support. However, it lacks a dedicated Neural Processing Unit (NPU) or Tensor Cores. Consequently, while it excels at traditional algorithmic tasks, it becomes a severe performance bottleneck when attempting real-time deep learning inference, resulting in unacceptable latency spikes.

**The Enterprise-Grade Choice: NVIDIA Jetson Orin Nano**

For competitive teams requiring complex, AI-driven autonomy, the NVIDIA Jetson Orin Nano is the premier edge-computing module. Delivering up to 40 TOPS (Tera Operations Per Second) of AI performance, it features an NVIDIA Ampere architecture GPU with 1024 CUDA cores and 32 Tensor Cores. This architecture allows the Orin Nano to execute complex neural networks (such as YOLOv8) with extreme efficiency, matching server-level throughput in a compact embedded form factor. While significantly more expensive (ranging from $300 to $500 depending on RAM and the specific carrier board), it is the mandatory choice if the AUV must perform real-time, high-framerate object detection for dynamic obstacle avoidance or complex target interaction.

### 3.2 Flight Controller: Precision on a Budget

The flight controller manages the high-frequency PID loops, sensor fusion, and direct hardware actuation. It must support the ArduSub firmware, a specialized branch of the open-source ArduPilot ecosystem designed exclusively for underwater ROVs and AUVs. ArduSub natively manages multi-axis motor mixing matrices, depth hold, and attitude stabilization, effectively offloading these critical, real-time tasks from the companion computer.

Historically, the Pixhawk series (e.g., Pixhawk 6C or CubePilot Orange) has been the industry standard for commercial drones. However, their high cost ($200 to $400+) places a severe burden on student budgets. The optimal price-to-performance alternative identified through this research is the Matek H743-Wing V3 or Matek H743-Slim V4.

At approximately $70 to $100, the Matek H743 series utilizes the highly powerful STM32H743 MCU running at 480MHz with 2MB of Flash memory. Crucially, it includes dual ICM42688P Inertial Measurement Units (IMUs), a high-precision DPS310/368 barometer, built-in OSD, and native ArduPilot (ChibiOS) support. This board provides enterprise-grade sensor redundancy, vibration isolation, and computing power at a fraction of the cost of a Pixhawk.

| Component Feature | Pixhawk 6C / CubePilot | Matek H743-Wing V3 / Slim V4 |
|---|---|---|
| Microcontroller (MCU) | STM32H753 (480MHz) | STM32H743 (480MHz) |
| IMU Setup | Triple Redundancy (Isolated) | Dual ICM42688-P |
| ArduSub Compatibility | Yes (Native) | Yes (ChibiOS firmware) |
| I/O Interfaces | Extensive (CAN, I2C, UART) | 7 UARTs, I2C, CAN, 13 PWM |
| Estimated Cost | ~$230 - $400+ | ~$70 - $110 |
| Budget Suitability | Low | High |

**Strategic Sourcing Note:** When procuring hardware in specific international regions (such as Turkey), teams must account for stringent e-commerce import regulations. Recent legislative changes have lowered the tax-free customs limit for personal imports to €30, which applies directly to electronic components and hobbyist robotics. To avoid exorbitant import duties, AUV teams must utilize strategic, split purchasing across multiple orders or source heavily through local distributors.

## 4. Perception Layer: Vision Systems and Optical Restoration

Underwater imaging introduces profound physical and algorithmic challenges not present in terrestrial robotics. Because water acts as a highly dense scattering medium, raw camera feeds suffer from severe color distortion, loss of contrast, and optical blurring. Just as data inputs in a Zero-Trust microservice architecture must be validated and sanitized before processing, underwater optical data must be fundamentally restored before being fed into decision-making algorithms.

### 4.1 The Jaffe-McGlamery Optical Model

The degradation of underwater vision is mathematically described by the foundational Jaffe-McGlamery Image Formation Model (IFM). According to this physical model, the total irradiance received by the camera is the linear superposition of three distinct components:

- **Direct Signal Component (E_d):** The light reflected directly from the target object to the camera lens. This signal decays exponentially as a function of depth and distance due to the attenuation coefficient of the water.
- **Forward-Scattering Component (E_f):** Light that reflects off the object but is deflected by suspended particulate matter at small angles before reaching the lens. This alters the direction of light propagation, resulting in a blurred, low-resolution effect.
- **Back-Scattering Component (E_b):** Ambient background light that reflects off suspended particles directly into the camera lens without ever hitting the target. This creates a "veiling glare" or dense fog effect that severely degrades image contrast.

The total received image at pixel *x* for color channel *c* is formally expressed as:

```
I_c(x) = J_c(x) · t(x) + B_c · (1 − t(x))
```

Where `J_c(x)` is the clear scene radiance (the true image), `t(x)` is the transmission map (representing attenuation based on distance and wavelength), and `B_c` is the global background light.

### 4.2 Camera Hardware Selection

To maximize visual input quality and reduce the mathematical burden on the restoration software, the camera sensor must be highly sensitive to low-light conditions. The analysis strongly recommends modules based on the Sony IMX327 sensor.

The IMX327 is a 2-Megapixel (1080p), 1/2.8-inch STARVIS CMOS sensor featuring a 2.9µm pixel size. Designed specifically for ultra-low-light surveillance, it provides near 0-lux performance, heavily mitigating the rapid attenuation of light at depth. Procuring a bare USB 2.0 IMX327 board camera with a wide-angle lens (~96 degrees) is highly cost-effective—often negotiable between $10 and $30 via OEM suppliers—and interfaces seamlessly with both Raspberry Pi and Jetson platforms.

### 4.3 Algorithmic Vision Restoration Pipeline (Classical)

Because red light (wavelengths > 600 nm) is absorbed rapidly within the first few meters of water, underwater images inherently exhibit a washed-out blue-green hue. To solve this, a highly optimized, deterministic software pipeline must be implemented, leveraging parallel processing via NumPy and OpenCV CUDA operations to achieve real-time performance:

1. **White Balance (Gray World Assumption):** To compensate for the severe attenuation of the red channel, the algorithm forces the global average of all color channels (R, G, B) to be equal. By dividing each channel by the global mean, the lost red spectrum is artificially boosted, normalizing the image for further processing.
2. **CLAHE (Contrast Limited Adaptive Histogram Equalization):** Standard global histogram equalization amplifies back-scatter noise and ruins color fidelity. CLAHE solves this by operating on localized tiles of the image. The image is first converted to the LAB color space, and CLAHE is applied only to the L (Luminance) channel. This restores contrast lost to back-scattering without altering the chromaticity (A and B channels). The application order is critical: White Balance must precede CLAHE; reversing the order amplifies the color shift.
3. **HSV Masking & Morphological Operations:** For computationally lightweight tasks (e.g., following a colored line on the pool floor), the restored image is converted to the HSV (Hue, Saturation, Value) color space to isolate color tone from variable brightness. Binary masking is applied to the Hue channel, followed by morphological Erosion (removing small noise blobs) and Dilation (closing holes in the detected object). Finally, contour analysis extracts the object's centroid via image moments.

### 4.4 Artificial Intelligence and YOLOv8 Optimization

For complex object detection scenarios (such as identifying distinct torpedo targets or interacting with multi-colored buoys), classical computer vision is brittle and insufficient. The architecture must incorporate YOLOv8 (You Only Look Once, version 8), a state-of-the-art single-stage object detection model.

Because raw PyTorch models introduce severe latency on edge devices, the model must be aggressively optimized using NVIDIA TensorRT, a library that restructures the neural network graph to maximize hardware utilization on Jetson GPUs.

The optimization pipeline involves exporting the YOLOv8 model to the ONNX intermediate representation format, and then using `trtexec` to compile a hardware-specific TensorRT engine. This compilation process includes layer fusion (combining Convolution, BatchNorm, and ReLU layers into a single CUDA kernel to eliminate memory read/write overhead).

- **FP16 Quantization:** By reducing the precision of the neural network weights from 32-bit floats (FP32) to 16-bit floats (FP16), memory bandwidth usage is halved. This activates the specialized Tensor Cores on the Jetson Orin Nano, allowing matrix multiplication to complete in a single cycle. This provides a massive boost in Frames Per Second (FPS) with virtually zero loss in bounding-box accuracy.
- **INT8 Quantization:** For maximum compression, weights are quantized to 8-bit integers. This requires passing a calibration dataset through the network to map the FP32 activation ranges into a compressed [-128, 127] space. While this yields the highest possible FPS (often exceeding 60-70 FPS for YOLOv8n on Orin Nano), it can introduce minor accuracy degradation depending on the visual complexity of the underwater environment.

| Model & Device | Precision | Latency (ms) | Approximate FPS | Hardware Utilization |
|---|---|---|---|---|
| YOLOv8s (PyTorch) - Jetson Orin | FP32 | ~20.88 ms | ~48 FPS | Low (CPU/GPU bound) |
| YOLOv8s (TensorRT) - Jetson Orin | FP32 | ~8.36 ms | ~120 FPS | Medium |
| YOLOv8s (TensorRT) - Jetson Orin | FP16 | ~4.43 ms | ~225 FPS | High (Tensor Cores active) |
| YOLOv8s (TensorRT) - Jetson Orin | INT8 | ~3.49 ms | ~286 FPS | Maximum (calibration req.) |

*Note: Data derived from theoretical maximums on unconstrained Orin architectures; real-world AUV deployment running concurrent ROS 2 nodes will yield lower, yet still highly performant, deterministic framerates.*

## 5. Navigation, State Estimation, and Sensor Fusion

In underwater environments, the rapid absorption of high-frequency radio waves renders standard GNSS (GPS) entirely useless, plunging the vehicle into a strictly denied-communication environment. Consequently, the AUV must rely entirely on internal proprioceptive sensors and acoustic telemetry to maintain spatial awareness.

### 5.1 Critical Sensory Hardware

- **Inertial Measurement Unit (IMU):** Measures 3-axis linear acceleration and 3-axis angular velocity. The Matek H743 features dual ICM42688P IMUs, providing hardware-level redundancy against high-frequency vibration noise caused by thruster cavitation.
- **Depth Sensor:** The TE Connectivity MS5837-30BA is the undisputed industry standard for ROV depth sensing. Operating over an I2C bus, it provides an absolute depth resolution of 2 mm and can withstand pressures up to 30 bar (300 meters depth). It is highly affordable (~$17-$25) and natively supported by the ArduSub firmware.
- **Acoustic Sonar:** A Ping Sonar (or an equivalent single-beam echosounder) provides absolute distance measurements to forward obstacles or the pool floor. Operating in a 30-degree cone geometry up to 50 meters, it is critical for enclosed pipe-tracking missions or when visual data is temporarily blinded by turbidity.

### 5.2 The Extended Kalman Filter (EKF3) and Error-State Variants

Raw IMU data is inherently noisy; integrating acceleration to find velocity, and integrating velocity to find position, leads to rapid, unbounded quadratic drift. To achieve stable 6-Degrees-of-Freedom (6-DoF) state estimation, the architecture utilizes the Extended Kalman Filter (EKF3) embedded within the ArduPilot ecosystem.

The EKF algorithm mathematically fuses high-rate, drifting IMU data with low-rate, absolute measurements (such as depth from the MS5837 and distances from the sonar). Because AUV dynamics involve complex hydrodynamic damping and Coriolis forces, the system is highly non-linear. The standard Kalman Filter cannot handle non-linearities, so the EKF linearizes the dynamic model around the current mean and covariance using Jacobian matrices.

The filter operates iteratively in two continuous phases:

1. **Prediction Phase (Time Update):** The state vector (position, velocity, orientation) is propagated forward in time using the kinematic (or dynamic) model and the integrated IMU data.
2. **Update Phase (Measurement Update):** When a new absolute measurement arrives (e.g., a depth reading), the EKF calculates the "Innovation"—the exact mathematical difference between the state predicted by the IMU and the state measured by the external sensor. Using the dynamically calculated Kalman Gain (which balances the trust between the mathematical model's uncertainty and the sensor's known noise profile), the filter optimally weights the measurement against the prediction, updating the final state estimate while aggressively suppressing outlier noise.

By monitoring EKF Innovations (e.g., IMX, IMY, IMZ for magnetometer variations or IPD for depth), the system gains profound observability into sensor health. Advanced academic implementations, such as the Error-State Kalman Filter (ESKF) or those utilizing Lie algebra, estimate the error in the state rather than the state itself. This avoids singularities ("gimbal lock") associated with 3D Euler angles and improves mean square error (MSE) performance in highly dynamic maneuvers.

### 5.3 Dead Reckoning (Ölü Hesaplama)

If the camera loses sight of the target and acoustic data is unavailable, the AUV state machine triggers Dead Reckoning. The vehicle relies solely on the EKF's integration of IMU and depth data to estimate its continued trajectory. Because drift accumulates rapidly in the absence of absolute horizontal position fixes, this mode is capped by a strict temporal failsafe limit. If the target is not reacquired within this window, the vehicle executes a predictive search pattern or safely surfaces.

## 6. Guidance, Kinematics, and Motor Mixing

Once the vehicle's state is accurately estimated by the EKF, the Guidance and Control layers execute movement commands to achieve mission objectives.

### 6.1 PID Control and Tuning

The Proportional-Integral-Derivative (PID) controller minimizes the error between the desired setpoint (e.g., target depth or visual centroid alignment) and the measured state.

- **Proportional (P):** Drives the vehicle toward the target proportionally to the current magnitude of the error.
- **Integral (I):** Accumulates past errors over time to eliminate steady-state offsets (e.g., compensating for persistent water currents or slightly positive vehicle buoyancy).
- **Derivative (D):** Predicts future error based on the current rate of change, providing artificial damping to prevent overshoot and oscillation.

PID parameters must be empirically tuned in the water. The architecture dictates using the Ziegler-Nichols tuning methodology: the I and D gains are zeroed, and the P gain (Kp) is steadily increased until the vehicle exhibits continuous, stable oscillation. The ultimate gain (Ku) and oscillation period (Tu) are measured, and the final gains are calculated using standardized formulas (e.g., Kp = 0.6·Ku, Ti = 0.5·Tu).

### 6.2 6-DoF Vectored Kinematics and Motor Mixing

To achieve true holonomic movement (independent control over surge, sway, heave, roll, pitch, and yaw without altering vehicle orientation), a Vectored-6DoF frame configuration utilizing eight thrusters is required (four vectored horizontally, four positioned vertically).

Because the high-level ROS 2 logic outputs abstract geometric commands (e.g., "pitch down 10 degrees" or "surge forward at 1 m/s"), a Motor Mixing Matrix is required to map these abstract commands into specific PWM (Pulse Width Modulation) signals for each individual Electronic Speed Controller (ESC).

The mixing matrix calculates the geometric moment arm of each thruster relative to the vehicle's Center of Gravity (CoG). The physical distances of each motor from the pitch and roll axes are measured. The roll and pitch factors in the matrix are determined by calculating the trigonometric ratio of a specific motor's distance from the axis compared to the maximum distance of any thruster on the frame.

In ArduSub, this matrix is defined within the `AP_Motors6DOF.cpp` library. The algorithm ensures that the sum of the moments around any axis equals zero to maintain rotational stability (i.e., forward thrust does not inadvertently cause the vehicle to pitch up). The firmware handles this transparently in real-time, provided the thrusters are connected to the correct hardware output channels and their rotational directions (CW/CCW) are accurately configured via the QGroundControl automatic motor detection tool.

## 7. Propulsion, Kinematics, and Environmental Sealing

### 7.1 Thruster Selection: Balancing Cost and Reliability

The propulsion system is the most mechanically stressed and power-hungry component of the AUV. The industry benchmark for student and research AUVs is the Blue Robotics T200 Thruster. It is a fully flooded, three-phase brushless DC motor utilizing water-lubricated plastic bearings, producing highly reliable thrust for approximately $150 per unit.

Due to strict budget limitations, students often investigate cheaper alternatives sourced from AliExpress, such as generic 2820 thrusters or ApisQueen models (ranging from $40 to $60). While these offer significant upfront financial savings, empirical testing indicates severe performance discrepancies. For example, benchmark testing on generic 2820 thrusters reveals they often produce only 56% of their mathematically stated thrust and suffer from rapid bushing degradation if inadvertently run dry on the bench. Furthermore, varying thrust outputs between mismatched cheap motors make tuning the PID loops nearly impossible, as the vehicle will constantly yaw or pitch unintentionally.

The analysis concludes that while AliExpress thrusters are acceptable for initial bench prototyping, a competitive AUV must allocate the necessary budget for genuine T200 thrusters to ensure consistent hydrodynamic performance and prevent catastrophic mid-competition failure.

### 7.2 Environmental Sealing and Potting Compounds

Protecting custom PCBs, power distribution boards, and soldered wire junctions from the highly corrosive, pressurized underwater environment is paramount. Standard O-ring enclosures are prone to leaks over time; therefore, permanent encapsulation (potting) is heavily utilized for external components. The selection of the potting compound dictates the physical resilience, repairability, and thermal dissipation of the electronics.

- **Epoxy Resins:** Offer the highest mechanical strength (Shore D 70-90) and superior dielectric insulation. However, they cure completely rigid, making them highly brittle and prone to cracking under thermal shock. Crucially, epoxy encapsulation is permanent, rendering underlying electronics utterly unrepairable.
- **Silicones:** Provide extreme flexibility and temperature resistance (-60°C to +200°C). They absorb mechanical vibrations and do not stress delicate surface-mount components during thermal expansion, but they represent the most expensive chemical option.
- **Polyurethane (PU):** The optimal choice for a student budget. Polyurethane offers a highly balanced profile of tough elasticity, good impact resistance, and excellent waterproofing at the lowest cost. Its primary drawback is a severe sensitivity to ambient moisture during the curing process (hygroscopy), requiring careful application in a climate-controlled environment.

For budget AUVs, Polyurethane potting is highly recommended for sealing custom power distribution boards and external sensor junctions, providing maximum protection per dollar spent.

## 8. Failsafe Mechanisms and System Resilience

In accordance with the DevSecOps and Zero-Trust principles outlined in the enterprise architecture, the AUV software cannot assume that any hardware subsystem will operate flawlessly. A robust, centralized Task State Machine must govern the operational lifecycle, enforcing strict fail-safe parameters.

- **Sensor Sanity Checks:** If the EKF covariance matrix diverges rapidly, or if the IMU reports angular velocities exceeding physical possibilities, the state machine must immediately isolate the faulty sensor. The system degrades gracefully to an alternative navigation mode (e.g., relying solely on depth and sonar data) rather than feeding corrupted data into the PID loops.
- **Vision Timeouts:** If the camera feed freezes or returns solid black frames over consecutive iterations, the state machine flags the vision node as compromised. The AUV immediately transitions from visual servoing to Dead Reckoning, governed by a strict timeout threshold to prevent runaway.
- **Depth and Leak Overrides:** Hardware parameters provide the ultimate safety net. If the MS5837-30BA sensor detects a depth exceeding the maximum competition limit (or if an internal chassis leak sensor is triggered), the state machine immediately terminates all lateral task execution. The PID controllers are instantly reconfigured to force a maximum positive heave command, ensuring the vehicle surfaces for recovery and overriding all prior autonomous logic.

## 9. Strategic Conclusion

Designing a highly capable Autonomous Underwater Vehicle on a strict student budget requires intelligent engineering compromises backed by rigorous architectural discipline. This research indicates that transitioning from a monolithic script framework to a ROS 2 DDS-backed microservices architecture is the fundamental prerequisite for stability, allowing the vehicle to isolate failures and maintain real-time determinism.

From a hardware perspective, the system should avoid the exorbitant costs of commercial flight controllers by adopting the ArduSub-compatible Matek H743-Wing V3. Computationally, the Raspberry Pi 5 serves as an excellent low-cost base, though teams aiming for advanced computer vision must invest in the NVIDIA Jetson Orin Nano. Utilizing the Jetson allows the deployment of TensorRT-optimized, FP16-quantized YOLOv8 models, unlocking real-time AI inference at high frame rates.

The optical challenges mathematically dictated by the Jaffe-McGlamery model must be counteracted at the hardware level using the Sony IMX327 low-light sensor, and at the software level using deterministic LAB-space CLAHE algorithms to eliminate back-scatter. Navigation must rely on mathematically fusing the MS5837-30BA depth sensor with dual IMUs via the Extended Kalman Filter (EKF3), ensuring precise 6-DoF spatial awareness even in the absence of visual data. Finally, while Polyurethane potting provides cost-effective waterproofing, propulsion remains the one area where budget cuts are dangerous; utilizing genuine Blue Robotics T200 thrusters over generic alternatives guarantees the hydrodynamic reliability required to complete the mission.

By strictly adhering to these integrated hardware and software strategies, student engineering teams can produce a globally competitive, structurally resilient, and highly scalable autonomous underwater vehicle.
