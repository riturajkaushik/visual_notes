# NestOS Edge UXV-Main — Complete Architecture & Component Reference

> **Copilot prompt used:** *I need to understand this repository - how various components are organized, which depedns on what, working and need to each component, waht are the entry points. For your reference, this repo is for autonomy stack of the drones used for warefare (attack, surveillance etc). The codes in this repo runs in Jetson Nano Orin. This repo is the parent repo and uses other repors using VCS. I need to know everythong about these external repos pulled by this repo as well - their structure, entrypoints, relative denpendencies etc (repos are already pull and the code is available for your reference). Also I need a global infographic picture to undertand the entire repo and then per component inforgarphics. SO provide the Google Nano Banana prompts for the image generations as well  - keep the style modern and worthy for technical presentation. All these information should be saved in a file called Nest-UXV-Main-details.md*

> **Repository:** `nestai-internal/nestos.edge.uxv.uxv-main`
> **Platform:** NVIDIA Jetson Nano Orin · ROS 2 (Humble / Jazzy)
> **Purpose:** HiveCore Autonomy Stack for Unmanned Aerial Vehicles (UAVs) — surveillance, attack, reconnaissance missions

---

## Table of Contents

1. [High-Level Architecture Overview](#1-high-level-architecture-overview)
2. [Repository Structure](#2-repository-structure)
3. [External Repositories (VCS)](#3-external-repositories-vcs)
4. [Module Deep-Dives](#4-module-deep-dives)
   - 4.1 [IPM — Input Processing Module](#41-ipm--input-processing-module)
   - 4.2 [ORTM — Object Recognition & Tracking Module](#42-ortm--object-recognition--tracking-module)
   - 4.3 [HLPM — High-Level Planning Module](#43-hlpm--high-level-planning-module)
   - 4.4 [SHMM — State & Health Management Module](#44-shmm--state--health-management-module)
   - 4.5 [DMM — Data Management Module](#45-dmm--data-management-module)
   - 4.6 [ODP-Common — Shared Types & Messages](#46-odp-common--shared-types--messages)
   - 4.7 [ros2_object_msgs — Standalone Object Tracking Messages](#47-ros2_object_msgs--standalone-object-tracking-messages)
   - 4.8 [DevOps — Build, Deploy & CI/CD](#48-devops--build-deploy--cicd)
5. [System State Machine & Lifecycle](#5-system-state-machine--lifecycle)
6. [Cross-Module Data Flow & Topic Map](#6-cross-module-data-flow--topic-map)
7. [Entry Points & Launch System](#7-entry-points--launch-system)
8. [Docker & Deployment Architecture](#8-docker--deployment-architecture)
9. [Dependency Graph](#9-dependency-graph)
10. [Infographic Prompts for Image Generation](#10-infographic-prompts-for-image-generation)

---

## 1. High-Level Architecture Overview

The UXV-Main stack is a **modular ROS 2 autonomy system** designed for military UAV operations. It runs on NVIDIA Jetson Orin inside a Docker container and communicates with external systems (MAVROS/PX4 flight controller, Ground Control Station) over DDS middleware.

### The Five Core Modules

| Module | Abbreviation | Role |
|--------|-------------|------|
| **Input Processing Module** | IPM | Camera ingestion — captures RGB & thermal video, publishes scaled frames |
| **Object Recognition & Tracking Module** | ORTM | Perception — detects objects (TensorRT), tracks them (SORT), geo-locates (3D projection) |
| **High-Level Planning Module** | HLPM | Path planning — terminal guidance (Bézier curves), flightplan following, MAVROS setpoints |
| **State & Health Management Module** | SHMM | Orchestrator — hierarchical state machine, module health aggregation, lifecycle gating |
| **Data Management Module** | DMM | Mission data — video recording, mission logging, FC log download, cloud upload |

### Shared Libraries

| Package | Role |
|---------|------|
| **odp_common** | Core C++ types, state machine base class, rotation utilities |
| **odp_common_ros** | Custom ROS 2 messages (12 msgs, 5 srvs), type adapters |
| **ros2_object_msgs** | Lightweight standalone object tracking messages |

### External Systems

| System | Connection |
|--------|-----------|
| **MAVROS / PX4** | Flight controller interface — pose, GPS, state, setpoints |
| **Ground Control Station (GCS)** | Mission plans, approach parameters, target points, permissions |
| **MediaMTX** | RTSP video relay server |
| **Cloud API** | Mission data upload endpoint |

---

## 2. Repository Structure

```
nestos.edge.uxv.uxv-main/
├── src/                                # All autonomy module source code (VCS-managed)
│   ├── ipm/                            # Input Processing Module
│   ├── odp.object-recognition-and-tracking-module/  # ORTM
│   ├── hlpm/                           # High-Level Planning Module
│   ├── shmm/                           # State & Health Management Module
│   ├── dmm/                            # Data Management Module
│   ├── odp.odp-common/                 # Shared types & ROS msgs
│   └── odp.ros2_object_msgs/           # Standalone tracking msgs
├── devops/                             # DevOps repository (VCS-managed)
│   └── devops/                         # Docker base images, CI/CD, Ansible deploy
├── launch/                             # ROS 2 launch files & DDS configs
│   ├── uxv_dev-full.launch.py          # Option A: single-file config launch
│   ├── uxv_dev-modules.launch.py       # Option B: per-module config launch
│   ├── uxv_main.yaml                   # Legacy YAML launch (direct includes)
│   ├── cyclonedds.xml / fastdds_*.xml  # DDS middleware configs
│   └── mediamtx.yml                    # RTSP server config
├── launch_profiles/                    # Runtime configuration profiles
│   ├── example-config.yaml             # Option A config template
│   ├── example-config/                 # Option B config directory
│   │   ├── base.yaml                   # Module selection + launch file mapping
│   │   ├── params/                     # Per-module parameter overrides
│   │   └── qos/                        # Per-module QoS overrides
│   ├── shared/                         # Shared profiles (e.g., ortm-sim.yaml)
│   └── local/                          # .gitignore'd personal configs
├── compose/                            # Docker Compose orchestration
│   ├── docker-compose.yaml             # Multi-UAV deployment (uav_001, uav_002)
│   └── env/                            # Per-instance env files
├── systemd/                            # Systemd service for auto-start
│   ├── uxv-main.service
│   └── install-service.sh
├── Dockerfile                          # Runtime image build
├── docker-entrypoint.sh                # Container entrypoint (sources ROS workspaces)
├── build_runtime_image.sh              # Build helper via docker compose
├── create-container.sh                 # Dev container creation
├── run-container.sh                    # Dev container run
├── start-uxv.sh / stop-uxv.sh         # Production start/stop scripts
├── uxv-dev.repos                       # VCS repository manifest
├── vars.sh                             # Container image/name variables
├── docs/
│   ├── INTERFACES.md                   # Auto-generated topic/service reference
│   ├── LAUNCH_CONFIGURATIONS.md        # Auto-generated parameter reference
│   └── ci-workflow.md                  # CI documentation
└── scripts/                            # Utility scripts (profiling, streaming)
```

---

## 3. External Repositories (VCS)

All external repos are pulled via `vcs import < uxv-dev.repos`:

| Local Path | Repository | Branch | Purpose |
|-----------|-----------|--------|---------|
| `devops/devops` | `nestos.edge.uxv.devops` | main | Docker base images, CI/CD pipelines, Ansible deployment |
| `src/dmm` | `nestos.edge.uxv.data-management-module` | main | Mission data lifecycle — recording, logging, upload |
| `src/ipm` | `nestos.edge.uxv.input-processing-module` | main | Camera ingestion and image preprocessing |
| `src/odp.object-recognition-and-tracking-module` | `nestos.edge.uxv.object-recognition-and-tracking-module` | main | AI perception — detection, tracking, localization |
| `src/hlpm` | `nestos.edge.uxv.high-level-planning-module` | main | Autonomous path planning and MAVROS control |
| `src/shmm` | `nestos.edge.uxv.state-and-health-management-module` | main | System state orchestration and health monitoring |
| `src/odp.ros2_object_msgs` | `nestos.edge.uxv.ros2_object_msgs` | main | Lightweight object tracking message definitions |
| `src/odp.odp-common` | `nestos.edge.uxv.common` | main | Shared C++ types, ROS messages, state machine base |

---

## 4. Module Deep-Dives

### 4.1 IPM — Input Processing Module

**Purpose:** Sensor ingestion layer — captures frames from RGB and thermal cameras, applies hardware-accelerated preprocessing, and publishes scaled images for downstream AI consumption.

![IPM — Input Processing Module](images/2%20IPM%20—%20Input%20Processing%20Module%20Infographic.png)

#### Directory Structure
```
src/ipm/
├── cpp/
│   ├── core/
│   │   ├── image_streaming/         # V4L2/RTSP/GStreamer capture + VPI processing
│   │   └── ip_state_manager/        # HSM lifecycle state machine
│   └── ros/
│       ├── image_streaming_ros/     # ROS 2 composable node wrapper
│       ├── ip_state_manager_ros/    # State manager ROS wrapper
│       └── ipm_bringup_ros/         # Bringup package (launch + config)
```

#### ROS 2 Packages

| Package | Type | Output |
|---------|------|--------|
| `image_streaming` | cmake | `libimage_streaming.so` — frame capture engine |
| `ip_state_manager` | cmake | `libip_state_manager.so` — HSM state machine |
| `image_streaming_ros` | ament_cmake | Composable node `ipm::ImageStreamingROS` + standalone exe |
| `ip_state_manager_ros` | ament_cmake | Composable node `ipm::IPStateManagerROS` |
| `ipm_bringup_ros` | ament_cmake | Launch/config only |

#### Key Components

| Node Name | Plugin | Role |
|-----------|--------|------|
| `camera_rgb` | `ipm::ImageStreamingROS` | RGB camera capture → publishes raw + scaled images |
| `camera_thermal` | `ipm::ImageStreamingROS` | Thermal camera capture → publishes raw + scaled images |
| `ip_state_manager_component` | `ipm::IPStateManagerROS` | Module lifecycle management, health checks |

#### Core Capabilities
- **Video Sources:** V4L2 (hardware), RTSP (network), video file playback, Jetson GStreamer (hardware-accelerated NvArgus)
- **Image Processing:** NVIDIA VPI-based undistortion, temporal noise reduction (TNR), resolution scaling
- **Time Sync:** Optional MAVROS PX4 autopilot timesync (`/mavros/timesync_status`)
- **Architecture:** All 3 nodes run as **composable components** in a single multi-threaded container for zero-copy IPC

#### Topics Published
| Topic | Type | Consumers |
|-------|------|-----------|
| `/ipm/image_streaming/camera/gimbal/image` | `sensor_msgs/Image` | External (raw RGB) |
| `/ipm/image_streaming/camera/gimbal/image_scaled` | `sensor_msgs/Image` | ORTM, DMM |
| `/ipm/image_streaming/camera/thermal/image` | `sensor_msgs/Image` | External (raw thermal) |
| `/ipm/image_streaming/camera/thermal/image_scaled` | `sensor_msgs/Image` | ORTM, DMM |
| `/ipm/ip_state_manager/module_state` | `odp_common_ros/ModuleState` | SHMM, cameras |
| `/ipm/component/check` | `odp_common_ros/ComponentCheckArray` | IP State Manager |

#### Topics Subscribed
| Topic | Type | Source |
|-------|------|--------|
| `/smm/system_state_manager/state/intent` | `odp_common_ros/SystemState` | SHMM |
| `/mavros/timesync_status` | `mavros_msgs/TimesyncStatus` | MAVROS (optional) |

#### Key Dependencies
`fmt`, `OpenCV`, `GStreamer`, `NVIDIA VPI 3`, `hsmcpp`, `odp_common`, `cv_bridge`, `mavros_msgs`

---

### 4.2 ORTM — Object Recognition & Tracking Module

**Purpose:** Perception pipeline — detects objects in camera imagery using GPU inference, tracks them across frames, and projects 2D tracks to geolocated 3D positions.

#### Directory Structure
```
src/odp.object-recognition-and-tracking-module/
├── cpp/
│   ├── core/
│   │   ├── object_recognition/       # TensorRT inference engine
│   │   ├── multi_object_tracking/    # SORT tracker (Kalman + Hungarian)
│   │   ├── object_localization/      # 2D→3D geo-projection
│   │   └── ort_state_manager/        # HSM lifecycle state machine
│   └── ros/
│       ├── object_recognition_ros/   # Detection node
│       ├── multi_object_tracking_ros/# Tracking node
│       ├── object_localization_ros/  # Localization node
│       ├── ort_state_manager_ros/    # State manager node
│       └── ortm_bringup_ros/         # Bringup (launch + config)
```

#### ROS 2 Packages

| Package | Type | Output |
|---------|------|--------|
| `object_recognition` | cmake | `libobject_recognition.so` — TensorRT detector |
| `multi_object_tracking` | cmake | `libmulti_object_tracking.so` — SORT tracker |
| `object_localization` | cmake | `libobject_localization.so` — geo-projection |
| `ort_state_manager` | cmake | `libort_state_manager.so` — HSM state machine |
| `object_recognition_ros` | ament_cmake | Composable node `ortm::ObjectRecognitionROS` |
| `multi_object_tracking_ros` | ament_cmake | Composable node `ortm::MultiObjectTrackingROS` |
| `object_localization_ros` | ament_cmake | Composable node `ortm::ObjectLocalizationROS` |
| `ort_state_manager_ros` | ament_cmake | Composable node `ortm::OrtStateManagerROS` |
| `ortm_bringup_ros` | ament_cmake | Launch/config only |

#### Key Components

| Node Name | Plugin | Role |
|-----------|--------|------|
| `object_recognition_component` | `ortm::ObjectRecognitionROS` | GPU inference → 2D detections (excludable: `component_or`) |
| `multi_object_tracking_component` | `ortm::MultiObjectTrackingROS` | SORT tracker → 2D tracks with persistent IDs (excludable: `component_mot`) |
| `object_localization_component` | `ortm::ObjectLocalizationROS` | 2D→3D projection → geolocated tracks (excludable: `component_ol`) |
| `ort_state_manager_component` | `ortm::OrtStateManagerROS` | Module lifecycle management (always launched) |

#### Perception Pipeline
```
Camera Image → [Object Recognition] → 2D Detections → [Multi-Object Tracking] → 2D Tracks → [Object Localization] → 3D Geolocated Tracks
                  (TensorRT/YOLO)       (Detection2DArray)    (SORT/Kalman/Hungarian)  (ObjectTrack2DArray)  (ray-ground intersect)  (ObjectTrack3DArray)
```

#### Inference Modes
- **Mode 0:** RGB only
- **Mode 1:** Thermal only
- **Mode 2:** Blended (RGB + Thermal)

#### Localization Pipeline
`pixel coord → camera direction → FLU body frame → gimbal transform → base frame → ENU map → WGS84 GNSS`

#### Topics Published
| Topic | Type | Consumers |
|-------|------|-----------|
| `/ortm/object_recognition/detections/objects` | `vision_msgs/Detection2DArray` | MOT |
| `/ortm/multi_object_tracking/tracks/objects` | `ros2_object_msgs/ObjectTrack2DArray` | OL, DMM, SHMM (health) |
| `/ortm/object_localization/tracks/objects` | `ros2_object_msgs/ObjectTrack3DArray` | SHMM (health) |
| `/ortm/ort_state_manager/module_state` | `odp_common_ros/ModuleState` | OR, MOT, OL, SHMM |

#### Topics Subscribed
| Topic | Type | Source |
|-------|------|--------|
| `/ipm/image_streaming/camera/gimbal/image_scaled` | `sensor_msgs/Image` | IPM |
| `/ipm/image_streaming/camera/thermal/image_scaled` | `sensor_msgs/Image` | IPM |
| `/mavros/local_position/pose` | `geometry_msgs/PoseStamped` | MAVROS |
| `/mavros/global_position/global` | `sensor_msgs/NavSatFix` | MAVROS |
| `/smm/system_state_manager/state/intent` | `odp_common_ros/SystemState` | SHMM |

#### Key Dependencies
`inference_tensorrt_cpp` (TensorRT), `OpenCV`, `Eigen3`, `GeographicLib`, `hsmcpp`, `odp_common`

---

### 4.3 HLPM — High-Level Planning Module

**Purpose:** Autonomous path planning — generates flight paths (terminal guidance Bézier curves, flightplan waypoint following) and translates them to MAVROS setpoints for the flight controller.

#### Directory Structure
```
src/hlpm/
├── cpp/
│   ├── core/
│   │   ├── global_path_planning/    # Terminal guidance + flightplan guidance
│   │   ├── hlp_state_manager/       # HSM lifecycle state machine
│   │   └── path_translator/         # Waypoint following → MAVROS setpoints
│   └── ros/
│       ├── global_path_planning_ros/
│       ├── hlp_state_manager_ros/
│       ├── path_translator_ros/
│       └── hlpm_bringup_ros/        # Bringup (launch + config)
```

#### ROS 2 Packages

| Package | Type | Output |
|---------|------|--------|
| `global_path_planning` | cmake | `libglobal_path_planning.so` — path generation |
| `hlp_state_manager` | cmake | `libhlp_state_manager.so` — HSM state machine |
| `path_translator` | cmake | `libpath_translator.so` — setpoint generation |
| `global_path_planning_ros` | ament_cmake | Composable node `hlpm::GlobalPathPlanningROS` |
| `hlp_state_manager_ros` | ament_cmake | Composable node `hlpm::HLPStateManagerROS` |
| `path_translator_ros` | ament_cmake | Composable node `hlpm::PathTranslatorROS` |
| `hlpm_bringup_ros` | ament_cmake | Launch/config only |

#### Key Components

| Node Name | Plugin | Role |
|-----------|--------|------|
| `hlp_state_manager_component` | `hlpm::HLPStateManagerROS` | Module lifecycle + UAV role filtering (always launched) |
| `global_path_planning_component` | `hlpm::GlobalPathPlanningROS` | Path generation engine (excludable: `component_gpp`) |
| `path_translator_component` | `hlpm::PathTranslatorROS` | Real-time setpoint output at 30 Hz (excludable: `component_pt`) |

#### Guidance Strategies
1. **Terminal Guidance:** Generates smooth Bézier-curve approach paths to a swarm-provided target point using configurable geometry (elevation angle θ, approach azimuth ψ, radius)
2. **Flightplan Guidance:** Follows GCS-uploaded mission plans (GNSS waypoints converted to local NED via GeographicLib), tracking ingress/egress progress with pause/continue/pull support

#### Path Translator Modes
- **Position mode:** Direct waypoint position setpoints
- **Velocity mode:** Velocity setpoints at configurable speed
- **Yaw modes:** `face_forward`, `pose_yaw`, `fixed_yaw`

#### Topics Published
| Topic | Type | Consumers |
|-------|------|-----------|
| `/hlpm/hlp_state_manager/module_state` | `odp_common_ros/ModuleState` | GPP, PT, SHMM |
| `/hlpm/global_path_planning/global_path` | `geometry_msgs/PoseArray` | Path Translator |
| `/mavros/setpoint_raw/local` | `mavros_msgs/PositionTarget` | MAVROS (flight controller) |

#### Topics Subscribed
| Topic | Type | Source |
|-------|------|--------|
| `/mavros/local_position/pose` | `geometry_msgs/PoseStamped` | MAVROS |
| `/mavros/home_position/home` | `mavros_msgs/HomePosition` | MAVROS |
| `/platform/uav_interface/terminal/approach_params` | `odp_common_ros/ApproachParams` | Platform/GCS |
| `/platform/uav_interface/terminal/target_point` | `geometry_msgs/Point` | Platform/GCS |
| `/platform/uav_interface/mission/plan` | `odp_common_ros/MissionItemArray` | Platform/GCS |
| `/smm/system_state_manager/state/intent` | `odp_common_ros/SystemState` | SHMM |

#### Services Provided
| Service | Type |
|---------|------|
| `/hlpm/global_path_planning/mission/pause` | `MissionPause` |
| `/hlpm/global_path_planning/mission/continue` | `MissionContinue` |
| `/hlpm/global_path_planning/mission/pull` | `MissionPull` |

#### Key Dependencies
`GeographicLib`, `Eigen3`, `hsmcpp`, `odp_common`, `mavros_msgs`, `flex_msgs`

---

### 4.4 SHMM — State & Health Management Module

**Purpose:** Central state orchestrator — runs a hierarchical state machine that aggregates module health, enforces transition protocols, and broadcasts system intent to all modules.

#### Directory Structure
```
src/shmm/
├── cpp/
│   ├── core/
│   │   └── system_state_manager/    # Pure C++ HSM (extends BaseStateManager)
│   └── ros/
│       ├── system_state_manager_ros/# ROS 2 composable node wrapper
│       └── shmm_bringup_ros/        # Bringup (launch + config)
```

#### ROS 2 Packages

| Package | Type | Output |
|---------|------|--------|
| `system_state_manager` | cmake | `libsystem_state_manager.so` — HSM core |
| `system_state_manager_ros` | ament_cmake | Composable node `smm::SystemStateManagerROS` |
| `shmm_bringup_ros` | ament_cmake | Launch/config only |

#### Key Component

| Node Name | Plugin | Role |
|-----------|--------|------|
| `system_state_manager_component` | `smm::SystemStateManagerROS` | The system brain — collects all module states, publishes intent |

#### Two-Phase Transition Protocol
1. **Validation Phase:** Asks all modules "can you transition?" → modules report `is_transition_ready=true`
2. **Transition Phase:** Commands modules to actually transition → modules confirm new state

#### GCS Gating
Certain transitions require explicit GCS permission:
- `disable → standby`: `take_off_permission_granted`
- `standby → premission`: `execute_premission`
- `premission → mission`: `permission_to_engage_granted`

#### Topics Published
| Topic | Type | Consumers |
|-------|------|-----------|
| `/smm/system_state_manager/state/intent` | `odp_common_ros/SystemState` | ALL module state managers |

#### Topics Subscribed
| Topic | Type | Source |
|-------|------|--------|
| `/ipm/ip_state_manager/module_state` | `odp_common_ros/ModuleState` | IPM |
| `/ortm/ort_state_manager/module_state` | `odp_common_ros/ModuleState` | ORTM |
| `/hlpm/hlp_state_manager/module_state` | `odp_common_ros/ModuleState` | HLPM |
| `/dmm/dm_state_manager/module_state` | `odp_common_ros/ModuleState` | DMM |
| `/gcs/ground_control_station/module_state` | `odp_common_ros/ModuleState` | GCS (external) |

#### Key Dependencies
`hsmcpp`, `yaml-cpp`, `odp_common`

---

### 4.5 DMM — Data Management Module

**Purpose:** Mission data lifecycle manager — handles video recording/streaming, mission logging, flight controller log download, and cloud upload using an orchestrator/processor architecture.

#### Directory Structure
```
src/dmm/
├── python/
│   ├── core/odp_cloud/                    # Cloud upload client (git submodule)
│   └── ros/
│       ├── drone_mission_logger/          # Python ROS 2 package (4 nodes)
│       └── drone_mission_logger_msgs/     # Custom messages & services
├── cpp/
│   ├── core/
│   │   ├── dm_state_manager/              # HSM state machine
│   │   └── video_handler/                 # GStreamer/OpenCV RTSP engine
│   └── ros/
│       ├── dm_state_manager_ros/          # State manager composable node
│       ├── video_manager_ros/             # Video recording composable node
│       └── thermal_video_logger_node/     # Thermal video processor
```

#### ROS 2 Packages

| Package | Type | Output |
|---------|------|--------|
| `dm_state_manager` | cmake | `libdm_state_manager.so` — HSM core |
| `video_handler` | cmake | `libvideo_handler.so` — GStreamer pipeline |
| `dm_state_manager_ros` | ament_cmake | Composable node `dmm::DmStateManagerROS` |
| `video_manager_ros` | ament_cmake | Composable node `dmm::VideoManagerROS` |
| `thermal_video_logger_node` | ament_cmake | Standalone exe `thermal_video_logger_node_exe` |
| `drone_mission_logger` | ament_python | 4 Python node entry points |
| `drone_mission_logger_msgs` | ament_cmake | 2 msgs, 5 srvs |

#### Key Components

| Node Name | Language | Role |
|-----------|----------|------|
| `dm_state_manager_component` | C++ | Module lifecycle, calls log_manager services on state transitions |
| `video_manager_component` | C++ | RTSP streaming + local recording (RGB + thermal), AI overlay |
| `thermal_video_logger_node` | C++ | Thermal video recording with telemetry sampling |
| `log_manager_node` | Python | **Orchestrator brain** — configures processors, manages sessions |
| `image_processor_node` | Python | Camera image capture with geo-telemetry & EXIF |
| `fc_log_downloader_node` | Python | Downloads `.ulg` flight logs from PX4/ArduPilot via MAVLink |
| `uploader_node` | Python | Uploads zipped mission data to cloud API |
| `file_uploader_node` | Python | Rsync-based file upload action server |

#### Orchestrator/Processor Architecture
The `log_manager_node` orchestrates all "processor" nodes via a uniform 5-service interface:
```
configure → start_session → [flight data collection] → stop_session → is_processing (poll) → get_summary
```

#### Mission Data Pipeline
```
Sensors → Processor Nodes collect during flight → log_manager stops & finalizes
→ timestamped directory: flight.json + payload.json + sensor metadata + artifacts
→ zip → cloud upload
```

#### Custom Messages (drone_mission_logger_msgs)

| Definition | Purpose |
|-----------|---------|
| `FlightSession.msg` | Session ID, start time, flight directory |
| `EnrichedImage.msg` | Image with embedded GPS/heading/accuracy |
| `LoadMission.srv` | Load mission config YAML |
| `Configure.srv` | Configure a processor node |
| `StartSession.srv` | Start a data collection session |
| `IsProcessing.srv` | Check if processor is still working |
| `GetMediaSummary.srv` | Get image/video counts and sizes |
| `StartRecording.srv` | Start video recording for a stream |
| `StopRecording.srv` | Stop video recording |
| `StartDualRecording.srv` | Start RGB + thermal recording |
| `StopDualRecording.srv` | Stop dual recording |

#### Key Dependencies
`GStreamer`, `OpenCV`, `cv_bridge`, `hsmcpp`, `odp_common`, `mavros_msgs`, `yaml-cpp`, `pymavlink`

---

### 4.6 ODP-Common — Shared Types & Messages

**Purpose:** The foundational shared library providing C++ types, a state machine base class, rotation utilities, ROS 2 messages, and type adapters used by every module.

#### Structure
```
src/odp.odp-common/
├── cpp/
│   ├── core/odp_common/           # Pure C++ library
│   │   ├── shared_types.hpp       # ~20 enums + ~20 structs (namespace common::)
│   │   ├── shared_functions.hpp   # Rotation conversions, string↔enum converters
│   │   └── state_manager/         # BaseStateManager (abstract HSM base class)
│   └── ros/odp_common_ros/       # ROS 2 package
│       ├── msg/ (12 messages)     # SystemState, ModuleState, ComponentCheck, Waypoint, etc.
│       ├── srv/ (5 services)      # MissionStart/Abort/Pause/Continue/Pull
│       └── include/               # Type adapters (PoseStamped, NavSatFix, Detection2D, etc.)
```

#### Core Enums & Types (namespace `common::`)
- **States:** `initialize`, `disable`, `standby`, `premission`, `mission` (waypoint / terminal_guidance), `postmission`, `emergency`
- **Events:** `init_done`, `enable`, `prepare`, `start`, `step`, `finish`, `reset`, `emergency`, `recover`
- **Module IDs:** IP, SHM, ORT, HLP, DM, VP, GCS
- **Component IDs:** RGB/Thermal streaming, Object Recognition, MOT, OL, Video/Log Manager, GPP/LPP, Path Translator, FCH, SSM
- **Domain Types:** `UavRole` (striker/scout), `CameraType`, `TrackStatus`, `InferenceMode`, `MavCmd`, `MavFrame`

#### ROS 2 Messages (12)
`ApproachParams`, `ComponentCheck`, `ComponentCheckArray`, `MissionItem`, `MissionItemArray`, `ModuleState`, `SystemState`, `Waypoint`, `ObjectTrack2D`, `ObjectTrack2DArray`, `ObjectTrack3D`, `ObjectTrack3DArray`

#### ROS 2 Services (5)
`MissionStart`, `MissionAbort`, `MissionPause`, `MissionContinue`, `MissionPull`

#### Type Adapters (Zero-Copy Bridges)
| Adapter | C++ Type | ROS Message |
|---------|----------|-------------|
| `StampedPoseAdapter` | `common::PoseStamped` | `geometry_msgs/PoseStamped` |
| `NavSatFixAdapter` | `common::GnssStamped` | `sensor_msgs/NavSatFix` |
| `Detection2DArrayAdapter` | `common::Detection2DArrayStamped` | `vision_msgs/Detection2DArray` |
| `ObjectTrack2DArrayAdapter` | `common::ObjectTrack2DArrayStamped` | `odp_common_ros/ObjectTrack2DArray` |
| `ObjectTrack3DArrayAdapter` | `common::ObjectTrack3DArrayStamped` | `odp_common_ros/ObjectTrack3DArray` |

#### Key Dependencies
`hsmcpp`, `yaml-cpp`, `Eigen3`, `GeographicLib`

---

### 4.7 ros2_object_msgs — Standalone Object Tracking Messages

**Purpose:** Lightweight, standalone message package for object tracking — exists so external consumers can use tracking messages without the full `odp_common` dependency chain.

#### Messages (4)
| Message | Fields |
|---------|--------|
| `ObjectTrack2D` | `track_id`, `track_status`, `Detection2D`, `GeoPoint`, `pose_filtered`, `velocity_filtered` |
| `ObjectTrack2DArray` | `header`, `img_acquisition_time`, `ObjectTrack2D[]` |
| `ObjectTrack3D` | `track_id`, `track_status`, `Detection2D`, `GeoPoint`, `position_filtered`, `twist_filtered` |
| `ObjectTrack3DArray` | `header`, `img_acquisition_time`, `ObjectTrack3D[]` |

**Dependencies:** `std_msgs`, `geometry_msgs`, `geographic_msgs`, `vision_msgs`

---

### 4.8 DevOps — Build, Deploy & CI/CD

**Purpose:** Complete infrastructure for building Docker base images, CI/CD pipelines, Ansible deployment to Jetson boards, and developer tooling.

#### Structure
```
devops/devops/
├── .github/workflows/build-container.yml  # CI: multi-arch Docker build → ACR push
├── docker_creation/                       # uxv-cuda-base image build system
│   ├── VERSION (0.7.0)
│   ├── build_base_image.sh
│   └── dev-docker-cuda/Dockerfile         # CUDA + ROS 2 + OpenCV + TensorRT + VPI
├── deploy/                                # Ansible deployment system
│   ├── playbooks/ (full_setup, build, deploy, run, test, validate)
│   ├── roles/ (docker_build_remote, docker_registry, docker_compose, ptp_slave, ros_verify)
│   └── inventory/ (hosts, group_vars, host_vars)
├── developers_environment/                # VS Code dev containers, RViz visualizer
└── opencv_cuda_builder/                   # Standalone OpenCV+CUDA builder
```

#### Base Image Stack
```
NVIDIA CUDA / L4T JetPack
  └── uxv-cuda-base:0.7.0-{humble|jazzy}
        ├── ROS 2 (Humble or Jazzy)
        ├── OpenCV 4.10.0 (with CUDA)
        ├── TensorRT + inference_tensorrt_cpp
        ├── NVIDIA VPI 3
        ├── hsmcpp (state machine library)
        ├── cv_bridge (rebuilt for custom OpenCV)
        └── NestAI tracing libraries
              └── uxv-main:{version}-{distro}   (runtime image — this repo's Dockerfile)
```

#### CI/CD Pipeline
1. GitHub Actions builds multi-arch Docker images (`amd64` + `arm64`) on native runners
2. Pushes to Azure Container Registry (`nestaiinternal.azurecr.io`)
3. Merges multi-arch manifests
4. Triggers downstream `uxv-main` build via `repository_dispatch`

#### Ansible Deployment
- **Target:** NVIDIA Jetson boards
- **Strategies:** Remote build, registry pull, tar-based transfer
- **Includes:** Network tuning (DDS socket buffers up to 2 GB), PTP time sync, DDS discovery, ROS 2 verification

---

## 5. System State Machine & Lifecycle

The entire system follows a **hierarchical state machine** pattern. SHMM is the top-level orchestrator; each module has its own state manager that mirrors the system state.

### System States
```
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  EMERGENCY ←──────────────────────────────────────────────────────────┐  │
│      │                                                                │  │
│      ▼                                                                │  │
│  INITIALIZE ──→ DISABLE ──→ STANDBY ──→ PREMISSION ──→ MISSION ──→ POST │
│                   ▲                                     │  ├─ waypoint   │
│                   │                                     │  └─ terminal   │
│                   └─────────────────────────────────────┘   guidance     │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### State Descriptions
| State | Description |
|-------|-------------|
| `emergency` | Critical failure — all modules halt |
| `initialize` | System boot — modules loading, cameras initializing |
| `disable` | Modules loaded but not yet enabled — waiting for GCS takeoff permission |
| `standby` | Ready to fly — all modules healthy, waiting for mission |
| `premission` | Mission loaded — executing pre-mission checks and procedures |
| `mission:waypoint` | In-flight — following flightplan waypoints |
| `mission:terminal_guidance` | Terminal approach — Bézier-curve guidance to target |
| `postmission` | Mission complete — post-flight data collection and upload |

### Two-Phase Transition Protocol
```
SHMM publishes SystemState(validate_only=true)     → All modules check readiness
All modules respond ModuleState(is_transition_ready) → SHMM validates
SHMM publishes SystemState(validate_only=false)     → All modules execute transition
All modules report new state                         → SHMM confirms
```

---

## 6. Cross-Module Data Flow & Topic Map

### Primary Data Pipeline
```
┌─────────┐     ┌──────────────────────┐     ┌──────────────────────┐
│ CAMERAS │────▶│         IPM          │────▶│         ORTM         │
│ RGB +   │     │ image_streaming_ros  │     │ detect → track → loc │
│ Thermal │     │ (scaled images)      │     │ (3D geolocated objs) │
└─────────┘     └──────────────────────┘     └──────────────────────┘
                         │                            │
                         ▼                            ▼
                ┌──────────────────┐         ┌───────────────┐
                │       DMM       │         │     HLPM      │
                │ video recording │         │ path planning │
                │ mission logging │         │ → MAVROS      │
                └──────────────────┘         └───────────────┘
```

### State Management Backbone
```
          ┌────────────────────────────────────┐
          │              SHMM                  │
          │   system_state_manager_component   │
          │   (publishes SystemState intent)   │
          └─────┬──────┬──────┬──────┬────────┘
                │      │      │      │
        ┌───────▼─┐ ┌──▼───┐ ┌▼────┐ ┌▼───┐
        │ IPM SM  │ │ORT SM│ │HLP  │ │DM  │
        │         │ │      │ │SM   │ │SM  │
        └─────────┘ └──────┘ └─────┘ └────┘
        Each publishes ModuleState back to SHMM
```

### Complete Cross-Module Topic Map

| Topic | Publisher | Subscriber(s) |
|-------|-----------|---------------|
| `/ipm/.../gimbal/image_scaled` | `camera_rgb` | `object_recognition_component`, `video_manager_component` |
| `/ipm/.../thermal/image_scaled` | `camera_thermal` | `object_recognition_component`, `video_manager_component` |
| `/ortm/.../detections/objects` | `object_recognition_component` | `multi_object_tracking_component` |
| `/ortm/.../tracks/objects` (2D) | `multi_object_tracking_component` | `object_localization_component`, `video_manager_component` |
| `/ortm/.../tracks/objects` (3D) | `object_localization_component` | *(health-checked)* |
| `/hlpm/.../global_path` | `global_path_planning_component` | `path_translator_component` |
| `/mavros/setpoint_raw/local` | `path_translator_component` | MAVROS → Flight Controller |
| `/smm/.../state/intent` | `system_state_manager_component` | ALL state managers |
| `/ipm/.../module_state` | `ip_state_manager_component` | `system_state_manager_component` |
| `/ortm/.../module_state` | `ort_state_manager_component` | `system_state_manager_component` |
| `/hlpm/.../module_state` | `hlp_state_manager_component` | `system_state_manager_component` |
| `/dmm/.../module_state` | `dm_state_manager_component` | `system_state_manager_component` |
| `/gcs/.../module_state` | GCS (external) | `system_state_manager_component` |
| `/mavros/local_position/pose` | MAVROS | ORTM (OL), HLPM (GPP, PT) |
| `/mavros/global_position/global` | MAVROS | ORTM (OL), DMM (multiple) |
| `upload_trigger` / `upload_status` | `log_manager_node` ↔ `uploader_node` | bidirectional |

---

## 7. Entry Points & Launch System

### Production Entry Point
```bash
# Systemd service → start-uxv.sh → run-container.sh → Docker → docker-entrypoint.sh
#   → ros2 launch launch/uxv_dev-full.launch.py params_path:=/launch_profiles/shared/ortm-sim.yaml
```

### Launch System Options

| Option | Launch File | Config Source |
|--------|-----------|---------------|
| **A — Unified** | `uxv_dev-full.launch.py` | Single YAML file (e.g., `example-config.yaml`) |
| **B — Per-Module** | `uxv_dev-modules.launch.py` | Directory with `base.yaml` + `params/` + `qos/` |
| **Legacy** | `uxv_main.yaml` | Hard-coded package includes |

### Module Launch Identifiers
| Identifier | Module | Can Exclude |
|-----------|--------|-------------|
| `module_ip` | IPM | `component_is_rgb`, `component_is_thermal` |
| `module_ort` | ORTM | `component_or`, `component_mot`, `component_ol` |
| `module_hlp` | HLPM | `component_gpp`, `component_lpp`, `component_tg`, `component_pt` |
| `module_shm` | SHMM | `component_fch` |
| `module_dm` | DMM | `component_vm`, `component_lm` |

### Docker Compose Profiles
| Profile | Purpose |
|---------|---------|
| `uav_001` | UAV instance 1 (production) |
| `uav_002` | UAV instance 2 (production) |
| `dev` | Development with source mounts |
| `mediamtx` | RTSP video relay server |

---

## 8. Docker & Deployment Architecture

### Image Layer Stack
```
NVIDIA CUDA 12.x / L4T JetPack 6                         ← NVIDIA base
  └── uxv-cuda-base:0.7.0-jazzy                          ← DevOps builds this
        ├── ROS 2 Jazzy
        ├── OpenCV 4.10.0 (CUDA + GStreamer)
        ├── TensorRT + inference_tensorrt_cpp
        ├── NVIDIA VPI 3
        ├── hsmcpp, tracing libs
        └── cv_bridge (rebuilt)
              └── uxv-main:latest-jazzy                   ← This repo's Dockerfile
                    ├── colcon build (all src/ packages)
                    ├── launch/ files
                    └── AI model files (.engine, .onnx)
```

### Container Runtime Configuration
- **Network:** Host mode (for DDS discovery)
- **GPU:** NVIDIA runtime with all devices visible
- **DDS:** FastDDS (default) or CycloneDDS, configurable via XML profiles
- **Volumes:** launch/, launch_profiles/, bags/, videos/, mission_data/

### Deployment Flow
```
Developer → git push → GitHub Actions CI → Docker build (amd64 + arm64)
  → Azure Container Registry → Ansible deploy → Jetson Orin
                                                    └── systemd auto-start
```

---

## 9. Dependency Graph

### Module-Level Dependencies
```
         ┌──────────────────────────────────────┐
         │           odp_common (core)           │ ◀── Every module depends on this
         │      odp_common_ros (messages)         │
         │      ros2_object_msgs (tracking)       │
         └──────────────────────────────────────┘
                          ▲
          ┌───────┬───────┼───────┬──────────┐
          │       │       │       │          │
        ┌─┴──┐  ┌─┴──┐ ┌─┴──┐ ┌─┴───┐  ┌───┴──┐
        │ IPM│  │ORTM│ │HLPM│ │SHMM │  │ DMM  │
        └─┬──┘  └─┬──┘ └─┬──┘ └──┬──┘  └──┬───┘
          │       │       │       │         │
          │       ▼       │       │         │
          └──────▶ ORTM subscribes to IPM images
                  │       │       │         │
                  └───────┼───────┼────────▶│ DMM subscribes to IPM images + ORTM tracks
                          │       │         │
                          └──────▶│ HLPM sends setpoints to MAVROS
                                  │         │
                                  └────────▶│ SHMM aggregates all module states
```

### External System Dependencies
| External System | Packages Used |
|----------------|--------------|
| MAVROS / PX4 | `mavros_msgs` — pose, GPS, state, setpoints, vehicle info |
| TensorRT | `inference_tensorrt_cpp` — GPU inference for object detection |
| NVIDIA VPI 3 | Image undistortion, TNR, scaling on Jetson |
| GStreamer | Camera capture pipelines, video recording |
| hsmcpp | Hierarchical state machine library (all state managers) |
| GeographicLib | GNSS ↔ local coordinate conversions |
| OpenCV | Image processing, video I/O |
| Eigen3 | Matrix/transform math |

---

## 10. Infographic Prompts for Image Generation

### 10.1 Global System Architecture Infographic

**Prompt:**
```
Create a modern, clean technical architecture infographic for an autonomous military drone software stack called "NestOS UXV-Main". Use a dark navy (#0a1628) background with electric blue (#00a8ff) accent lines and white text. The layout should be landscape (16:9).

Center the diagram around 5 large rounded-rectangle module blocks arranged in a horizontal flow from left to right: IPM (Input Processing, icon: camera), ORTM (Object Recognition & Tracking, icon: crosshair target), HLPM (Path Planning, icon: route/waypoints), SHMM (State Management, icon: heartbeat pulse), DMM (Data Management, icon: database cloud).

Show data flow arrows between modules: IPM→ORTM (labeled "Scaled Images"), IPM→DMM (labeled "Video Feeds"), ORTM→DMM (labeled "Object Tracks"), HLPM→MAVROS (labeled "Setpoints"). Show SHMM in the center with bidirectional arrows to ALL modules (labeled "SystemState / ModuleState").

At the top, show external systems: "Ground Control Station (GCS)" and "MAVROS / PX4 Flight Controller" connected with dashed lines.

At the bottom, show the infrastructure layer: "Docker Container" → "NVIDIA Jetson Orin" → "uxv-cuda-base (ROS2 + CUDA + TensorRT)".

Style: flat design, no gradients, thin connector lines (2px), modern sans-serif font (Inter or similar), module blocks have subtle glow effect. Include the NestAI logo placeholder in top-left corner. Add a legend box in bottom-right for line types (solid=data, dashed=external, dotted=health). Make it presentation-ready for a defense tech demo.
```

---

### 10.2 IPM — Input Processing Module Infographic

**Prompt:**
```
Create a technical component diagram infographic for the "Input Processing Module (IPM)" of a drone autonomy stack. Dark theme with navy background (#0a1628), cyan accents (#00e5ff), white text.

Show a vertical flow: At top, two camera icons (RGB Camera, Thermal Camera) feeding into a processing block labeled "ImageStreamingROS" containing sub-blocks: "V4L2/GStreamer Capture" → "VPI Processing (Undistortion, TNR)" → "Resolution Scaling". Two output arrows: "Full-Res Image" and "Scaled Image (AI-Ready)".

To the right, show a state machine block "IP State Manager" with states flowing vertically: initialize → disable → standby → premission → mission → postmission, with an emergency state connected from any point.

Show bidirectional health check arrows between cameras and state manager (labeled "ComponentCheck"). Show an upward arrow from state manager to "SHMM" (labeled "ModuleState").

Bottom section: show a "Composable Node Container" box encompassing all 3 nodes (camera_rgb, camera_thermal, ip_state_manager_component) with a note "Zero-Copy IPC".

Style: flat technical diagram, monospaced labels for topic names, thin connector lines, subtle grid pattern in background. Presentation-quality for engineering review.
```

---

### 10.3 ORTM — Perception Pipeline Infographic

**Prompt:**
```
Create a technical pipeline diagram infographic for the "Object Recognition & Tracking Module (ORTM)" of a drone autonomy stack. Dark theme (#0a1628 background), orange-red accent (#ff4d4d) for detection, green (#00ff88) for tracking, blue (#00a8ff) for localization.

Show a horizontal pipeline with 3 large stages connected by thick arrows:

Stage 1 — "Object Recognition" (orange-red accent): Icon of neural network. Input: "Camera Images (RGB/Thermal)" from IPM. Processing: "TensorRT GPU Inference (YOLO)". Output: "2D Bounding Box Detections".

Stage 2 — "Multi-Object Tracking" (green accent): Icon of connected dots. Processing: "SORT Algorithm: Kalman Filter Prediction + Hungarian Assignment (IoU)". Output: "2D Tracks with Persistent IDs".

Stage 3 — "Object Localization" (blue accent): Icon of globe pin. Inputs: "2D Tracks + Drone Pose + GPS". Processing: "Ray-Ground Intersection: Pixel → Camera → Body → Map → WGS84". Output: "3D Geolocated Object Tracks".

Below the pipeline, show "ORT State Manager" with the same HSM lifecycle states. Show the composable container enclosing all 4 nodes.

Add a small inset showing inference modes: RGB Only, Thermal Only, Blended.

Style: clean technical, left-to-right flow, modern sans-serif, subtle gradient on stage backgrounds matching accent colors. Defense/tech presentation quality.
```

---

### 10.4 HLPM — Path Planning Infographic

**Prompt:**
```
Create a technical infographic for the "High-Level Planning Module (HLPM)" of a drone autonomy stack. Dark navy background (#0a1628) with purple (#a855f7) and teal (#14b8a6) accents.

Show two guidance modes side by side in the upper section:

Left — "Flightplan Guidance" (teal): Show a top-down map view with waypoint markers (numbered 1-5) connected by straight line segments. Labels: "GCS Mission Plan", "GNSS→Local NED", "Ingress/Egress tracking". Icons: waypoint pins, compass.

Right — "Terminal Guidance" (purple): Show a 3D perspective view of a Bézier curve approach path spiraling down to a target point. Labels: "Approach Geometry (θ, ψ, radius)", "Bézier Curve Interpolation". Icons: target crosshair, curved arrow.

Below both, show "Path Translator" as a wide block: Input "PoseArray (waypoints)" → Processing "30Hz Control Loop" → Output "MAVROS PositionTarget Setpoints". Show yaw mode options: face_forward, pose_yaw, fixed_yaw.

At bottom, show the 3-node composable container with state manager. Add small external connection icons for MAVROS and Platform/GCS.

Style: technical with subtle 3D perspective on the Bézier illustration, clean flat design elsewhere. Engineering presentation quality.
```

---

### 10.5 SHMM — State Orchestration Infographic

**Prompt:**
```
Create a technical infographic for the "State & Health Management Module (SHMM)" — the central orchestrator of a drone autonomy stack. Dark navy background (#0a1628), gold/amber (#f59e0b) accents.

Center: A large circular state machine diagram showing states as connected nodes in a ring: EMERGENCY (red) → INITIALIZE → DISABLE → STANDBY → PREMISSION → MISSION (with sub-states: WAYPOINT, TERMINAL_GUIDANCE) → POSTMISSION → back to STANDBY. Use arrows showing valid transitions. Emergency connects from all states.

Around the state machine, show 5 module icons (IPM, ORTM, HLPM, DMM, GCS) positioned like satellites. Each has an inbound arrow (labeled "ModuleState") and an outbound arrow (labeled "SystemState Intent"). Use the two-phase protocol visualization: Phase 1 "Validate" (yellow dashed), Phase 2 "Transition" (gold solid).

At top, show GCS gating requirements for key transitions:
- disable→standby: "Takeoff Permission"
- standby→premission: "Execute Premission"
- premission→mission: "Permission to Engage"

Bottom: Show the timeout/staleness mechanism as a clock icon with "Re-publish intent if stale".

Style: Command-center aesthetic, circular state diagram as the hero element, clean lines, modern font. Suitable for defense stakeholder briefing.
```

---

### 10.6 DMM — Data Management Infographic

**Prompt:**
```
Create a technical infographic for the "Data Management Module (DMM)" of a drone autonomy stack. Dark navy background (#0a1628), emerald green (#10b981) accents for data flow, sky blue (#38bdf8) for cloud.

Show the orchestrator/processor architecture as a hub-spoke diagram:

Center hub: "Log Manager (Orchestrator)" with a state machine: IDLE → CONFIGURING → READY → RUNNING → STOPPING.

Spokes radiating outward to 4 processor nodes:
- "Thermal Video Logger" (camera icon) — records thermal video + telemetry
- "Image Processor" (image icon) — captures geo-tagged images with EXIF
- "FC Log Downloader" (download icon) — downloads .ulg flight logs via MAVLink
- "Video Manager" (film icon) — RGB+thermal RTSP streaming and local recording

All processors connected via uniform 5-service interface: configure → start → [collect] → stop → poll → summary.

At the top, show data sources: "IPM (Camera Feeds)", "ORTM (Object Tracks)", "MAVROS (Telemetry)".

At the bottom, show the output pipeline: "Mission Directory" (folder icon with flight.json, payload.json, sensor data) → "ZIP" → "Cloud Upload" (cloud icon with upward arrow).

Add a "DM State Manager" sidebar showing the HSM lifecycle with connection to SHMM.

Style: clean data-flow diagram, hub-spoke layout, icons in circular badges, subtle glow on the hub. Technical presentation quality.
```

---

### 10.7 Shared Libraries & Message Architecture Infographic

**Prompt:**
```
Create a technical infographic showing the "Shared Libraries & Message Architecture" of a drone autonomy stack called NestOS UXV-Main. Dark navy background (#0a1628), white and light blue accents.

Show a layered architecture diagram with 3 horizontal layers:

Top Layer — "ROS 2 Messages & Services" (light blue #60a5fa): Show 12 message types arranged in a grid: SystemState, ModuleState, ComponentCheck, ComponentCheckArray, ApproachParams, MissionItem, MissionItemArray, Waypoint, ObjectTrack2D/3D, ObjectTrack2DArray/3DArray. Show 5 services: MissionStart/Abort/Pause/Continue/Pull.

Middle Layer — "Type Adapters (Zero-Copy Bridge)" (cyan #22d3ee): Show 5 adapter blocks connecting ROS msg types to C++ types with bidirectional arrows.

Bottom Layer — "Core C++ Library (odp_common)" (white/gray): Show key components: "SharedTypes (20 enums + 20 structs)", "BaseStateManager (HSM abstract class)", "Rotation Utilities (Euler↔Quat↔RotMat)", "String↔Enum Converters".

On the right side, show "ros2_object_msgs" as a separate lighter-weight package branching off with just the 4 ObjectTrack messages — labeled "For external consumers (no odp_common dependency)".

Show arrows from each of the 5 modules (IPM, ORTM, HLPM, SHMM, DMM) converging into the bottom layer to indicate universal dependency.

Style: layered architecture with clear horizontal bands, technical font, minimal decoration, engineering documentation quality.
```

---

### 10.8 DevOps & Deployment Pipeline Infographic

**Prompt:**
```
Create a technical infographic for the "DevOps & Deployment Pipeline" of a drone autonomy stack. Dark navy background (#0a1628), gradient accents from blue (#3b82f6) to purple (#8b5cf6).

Show a left-to-right pipeline with 4 major stages:

Stage 1 — "Development" (blue): VS Code icon + Dev Container badges (amd64, arm64, CUDA). Show source repos feeding into VCS import.

Stage 2 — "CI/CD" (blue-purple gradient): GitHub Actions icon → "Multi-Arch Build (amd64 + arm64)" → "Azure Container Registry" with image tags: latest-jazzy, 0.7.0-jazzy, sha-jazzy. Show downstream dispatch trigger arrow.

Stage 3 — "Docker Image Stack" (purple): Layered block diagram showing: NVIDIA CUDA base → uxv-cuda-base (ROS2 + OpenCV + TensorRT + VPI + hsmcpp) → uxv-main (colcon build + launch files + AI models).

Stage 4 — "Deployment" (purple-pink): Ansible icon → NVIDIA Jetson Orin board illustration. Show deployment strategies: "Remote Build", "Registry Pull", "Tar Transfer". Below: network tuning (DDS buffers), PTP time sync, systemd auto-start.

At bottom, show a small RViz visualizer screenshot placeholder with a drone 3D model for the developer tools section.

Style: horizontal pipeline with clear stage separation, gradient color progression, modern tech startup aesthetic suitable for investor/stakeholder deck.
```

---

*Document generated on 2026-05-11. For the most up-to-date interface details, see `docs/INTERFACES.md` and `docs/LAUNCH_CONFIGURATIONS.md`.*
