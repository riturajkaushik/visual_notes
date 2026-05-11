# HiveCore Architecture — Visual Reference

> Quick-revision guide for the NestOS UXV-Main autonomy stack architecture. Each infographic below summarizes a key architectural layer or module.

---

## 1. Global System Architecture

![Global System Architecture](images/1%20Global%20System%20Architecture%20Infographic.png)

The UXV-Main stack is a modular ROS 2 autonomy system running on NVIDIA Jetson Orin inside Docker. It comprises five core modules — **IPM** (camera ingestion), **ORTM** (AI perception), **HLPM** (path planning), **SHMM** (state orchestration), and **DMM** (data management). Data flows left-to-right: IPM feeds scaled images to ORTM and video to DMM; ORTM provides object tracks; HLPM sends setpoints to MAVROS/PX4. SHMM sits at the center, broadcasting system state intent to all modules and collecting health reports. External systems include the Ground Control Station (GCS) for mission plans and MAVROS for flight controller interface, all communicating over DDS middleware.

---

## 2. IPM — Input Processing Module

![IPM — Input Processing Module](images/2%20IPM%20—%20Input%20Processing%20Module%20Infographic.png)

IPM is the sensor ingestion layer responsible for capturing frames from RGB and thermal cameras via V4L2, RTSP, or Jetson GStreamer (NvArgus). Frames undergo hardware-accelerated preprocessing using NVIDIA VPI — undistortion, temporal noise reduction (TNR), and resolution scaling — producing AI-ready scaled images for downstream consumption. The module runs 3 composable nodes (camera_rgb, camera_thermal, ip_state_manager) in a single multi-threaded container enabling zero-copy IPC. The IP State Manager implements an HSM lifecycle, reporting module health to SHMM and gating camera operation based on system state transitions.

---

## 3. ORTM — Perception Pipeline

![ORTM — Perception Pipeline](images/3%20ORTM%20—%20Perception%20Pipeline%20Infographic.png)

ORTM is a 3-stage perception pipeline: **Object Recognition** performs TensorRT/YOLO GPU inference on camera images (supporting RGB-only, thermal-only, or blended modes) to produce 2D bounding box detections. **Multi-Object Tracking** applies the SORT algorithm (Kalman filter prediction + Hungarian assignment via IoU) to maintain persistent track IDs across frames. **Object Localization** projects 2D tracks to 3D geolocated positions using ray-ground intersection through the coordinate chain: pixel → camera → FLU body → gimbal → base → ENU map → WGS84 GNSS. All 4 nodes run as composable components with an ORT State Manager handling lifecycle.

---

## 4. HLPM — Path Planning

![HLPM — Path Planning](images/4%20HLPM%20—%20Path%20Planning%20Infographic.png)

HLPM provides autonomous path planning with two guidance strategies. **Terminal Guidance** generates smooth Bézier-curve approach paths to swarm-provided target points using configurable geometry (elevation angle θ, approach azimuth ψ, radius). **Flightplan Guidance** follows GCS-uploaded mission plans by converting GNSS waypoints to local NED coordinates via GeographicLib, tracking ingress/egress progress with pause/continue/pull support. The **Path Translator** converts generated PoseArray waypoints into real-time MAVROS PositionTarget setpoints at 30 Hz, supporting position and velocity modes with multiple yaw strategies (face_forward, pose_yaw, fixed_yaw).

---

## 5. SHMM — State Orchestration

![SHMM — State Orchestration](images/5%20SHMM%20—%20State%20Orchestration%20Infographic.png)

SHMM is the central state orchestrator running a hierarchical state machine (HSM) that governs the entire system lifecycle: Initialize → Disable → Standby → Premission → Mission (Waypoint / Terminal Guidance sub-states) → Postmission, with Emergency reachable from any state. It uses a **two-phase transition protocol** — first validating that all modules can transition, then commanding the actual transition. Certain state changes require explicit GCS gating permissions: takeoff permission for disable→standby, execute premission for standby→premission, and permission to engage for premission→mission. SHMM aggregates ModuleState from all modules and broadcasts SystemState intent system-wide.

---

## 6. DMM — Data Management

![DMM — Data Management](images/6%20DMM%20—%20Data%20Management%20Infographic.png)

DMM manages the full mission data lifecycle using an **orchestrator/processor architecture**. The Log Manager (orchestrator) coordinates 4 processor nodes via a uniform 5-service interface (configure → start_session → collect → stop_session → poll → get_summary): the Video Manager handles RGB+thermal RTSP streaming and local recording, the Thermal Video Logger records thermal video with telemetry, the Image Processor captures geo-tagged images with EXIF metadata, and the FC Log Downloader retrieves .ulg flight logs via MAVLink. Completed mission data (flight.json, payload.json, sensor artifacts) is zipped and uploaded to the cloud API. A DM State Manager handles lifecycle gating through SHMM.

---

## 7. Shared Libraries & Message Architecture

![Shared Libraries & Message Architecture](images/7%20Shared%20Libraries%20&%20Message%20Architecture%20Infographic.png)

The shared foundation comprises three layers. **odp_common** (core C++ library) provides ~20 enums and ~20 structs defining system states, events, module/component IDs, and domain types, plus a BaseStateManager abstract HSM class and rotation utilities (Euler↔Quaternion↔RotationMatrix). **odp_common_ros** defines 12 custom ROS 2 messages (SystemState, ModuleState, ComponentCheck, ApproachParams, MissionItem, Waypoint, ObjectTrack2D/3D, etc.) and 5 services (MissionStart/Abort/Pause/Continue/Pull), with Type Adapters providing zero-copy bridges between C++ types and ROS messages. **ros2_object_msgs** is a lightweight standalone package (4 ObjectTrack messages) enabling external consumers to use tracking data without the full odp_common dependency.

---

## 8. DevOps & Deployment Pipeline

![DevOps & Deployment Pipeline](images/8%20DevOps%20&%20Deployment%20Pipeline%20Infographic.png)

The DevOps pipeline spans four stages. **Development** uses VS Code dev containers with multi-arch support (amd64 + arm64). **CI/CD** via GitHub Actions builds multi-arch Docker images and pushes to Azure Container Registry, triggering downstream builds via repository_dispatch. The **Docker Image Stack** layers NVIDIA CUDA/L4T → uxv-cuda-base (ROS 2 + OpenCV 4.10 with CUDA + TensorRT + VPI 3 + hsmcpp) → uxv-main runtime image (colcon-built workspace + launch files + AI models). **Deployment** uses Ansible to target NVIDIA Jetson boards with strategies including remote build, registry pull, and tar transfer, plus network tuning (DDS socket buffers up to 2 GB), PTP time synchronization, and systemd auto-start.

---

*Generated from [Nest-UXV-Main-details.md](Nest-UXV-Main-details.md) for quick architecture revision.*
