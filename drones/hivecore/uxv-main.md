# NestOS Edge UXV — uxv-main Repository Guide

## What This Repository Does

**uxv-main** is the top-level orchestration repository for NestAI's **HiveCore Autonomy Software Stack**, designed to run on the Mission Computer of an Unmanned eXtended Vehicle (UXV/drone). It does **not** contain the module source code itself — instead, it aggregates multiple ROS 2 packages from separate Git repositories via `vcstool`, provides Docker-based build and deployment infrastructure, and offers a unified YAML-driven launch system to bring up the entire autonomy stack.

The stack processes camera feeds (RGB + thermal), performs AI-based object detection and tracking, manages flight planning and terminal guidance, monitors system health, and handles data recording/upload — all running as ROS 2 nodes inside Docker containers on NVIDIA-accelerated hardware (Jetson for field deployment, NVIDIA PCs for development/simulation).

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         uxv-main (this repo)                        │
│  Orchestration · Docker · Launch files · Configuration profiles     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐        │
│  │   IPM    │──▶│   ORTM   │──▶│   HLPM   │   │   SHMM   │        │
│  │ (Input   │   │ (Object  │   │ (High-   │   │ (State & │        │
│  │Processing│   │  Recog & │   │  Level   │   │  Health  │        │
│  │ Module)  │   │ Tracking)│   │ Planning)│   │ Monitor) │        │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘        │
│       │                                            ▲               │
│       │         ┌──────────────────────┐           │               │
│       └────────▶│        DMM           │───────────┘               │
│                 │  (Data Management)   │                           │
│                 │  Video Mgr · Logger  │                           │
│                 │  Uploader · Thermal  │                           │
│                 └──────────────────────┘                           │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Shared: odp-common (messages) · ros2_object_msgs           │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Module Breakdown & Data Flow

### 1. IPM — Input Processing Module (`src/ipm`)
- **Repo:** `nestos.edge.uxv.input-processing-module`
- **ROS packages:** `image_streaming_ros`, `ipm_bringup_ros`, `ip_state_manager_ros`
- **Nodes:** `camera_rgb`, `camera_thermal`, `ip_state_manager_component`
- **Purpose:** Captures and publishes camera frames (V4L2, RTSP, or video file). Supports undistortion, scaling, and temporal noise reduction (VPI/CUDA). Publishes full-res and AI-ready scaled images.
- **Key topics published:**
  - `/ipm/image_streaming/camera/gimbal/image_scaled` → consumed by ORTM + DMM
  - `/ipm/image_streaming/camera/thermal/image_scaled` → consumed by ORTM + DMM

### 2. ORTM — Object Recognition & Tracking Module (`src/odp.object-recognition-and-tracking-module`)
- **Repo:** `nestos.edge.uxv.object-recognition-and-tracking-module`
- **ROS packages:** `ortm_bringup_ros`, `object_recognition_ros`, `multi_object_tracking_ros`, `object_localization_ros`, `ort_state_manager_ros`
- **Nodes:** `object_recognition_component`, `multi_object_tracking_component`, `object_localization_component`, `ort_state_manager_component`
- **Purpose:** Runs TensorRT-accelerated inference (RGB/thermal/blend models via `.engine` files) to detect objects, tracks them across frames (multi-object tracking), and localizes them in 3D.
- **Data flow:** IPM scaled images → detection → tracking → localization → HLPM

### 3. HLPM — High-Level Planning Module (`src/hlpm`)
- **Repo:** `nestos.edge.uxv.high-level-planning-module`
- **ROS packages:** `hlpm_bringup_ros`, `hlp_state_manager_ros`, `terminal_guidance_ros`, `path_translator_ros`
- **Nodes:** `hlp_state_manager_component`, `terminal_guidance_component`, `path_translator_component`
- **Components:** Global path planner, local path planner, terminal guidance, path translator
- **Purpose:** Consumes tracked object data and flight state, produces guidance commands for the flight controller.

### 4. SHMM — State & Health Monitoring Module (`src/shmm`)
- **Repo:** `nestos.edge.uxv.state-and-health-management-module`
- **ROS packages:** `shmm_bringup_ros`, `system_state_manager_ros`
- **Nodes:** `system_state_manager_component`
- **Purpose:** Orchestrates module lifecycle. Publishes system-wide state/intent that all other modules subscribe to for coordinated state transitions (idle → active → engaged, etc.).
- **Key topics:**
  - `/smm/system_state_manager/state/intent` → subscribed by all module state managers

### 5. DMM — Data Management Module (`src/dmm`)
- **Repo:** `nestos.edge.uxv.data-management-module`
- **ROS packages:** `dm_state_manager_ros`, `video_manager_ros`, `drone_mission_logger`, `thermal_video_logger_node`
- **Nodes:** `dm_state_manager_component`, `video_manager_component`, `thermal_video_logger_node`, `log_manager_node`, `uploader_node`, `file_uploader_node`, `fc_log_downloader_node`, `image_processor_node`
- **Purpose:** Records video (RGB + thermal), manages mission logs, downloads flight controller logs, and uploads data to cloud storage.

### 6. Shared Libraries
- **`odp.odp-common`** (`nestos.edge.uxv.common`) — shared ROS message types (`ModuleState`, `SystemState`, `ComponentCheckArray`, etc.) and utilities
- **`odp.ros2_object_msgs`** (`nestos.edge.uxv.ros2_object_msgs`) — object detection/tracking message definitions

---

## Inter-Module Dependencies (Data Flow)

```
SHMM (system state/intent)
  │
  ├──▶ IPM state manager (module lifecycle)
  ├──▶ ORTM state manager
  ├──▶ HLPM state manager
  └──▶ DMM state manager

IPM (camera images)
  ├──▶ ORTM (object recognition input)
  └──▶ DMM / Video Manager (recording)

ORTM (detections & tracks)
  ├──▶ HLPM (planning & guidance input)
  └──▶ DMM (logging overlays)

Each module state manager
  └──▶ SHMM (health reports back)
```

---

## Repository File Map

### Entry Points — Where to Find What

| What | Where | Description |
|------|-------|-------------|
| **Source repo manifest** | `uxv-dev.repos` | VCS tool file listing all sub-repos to clone into `src/` and `devops/` |
| **Release manifests** | `milestones/release-v*.repos` | Pinned versions for specific releases |
| **Dockerfile** | `Dockerfile` | Multi-stage build: bind-mounts `src/`, runs `colcon build`, bakes install tree |
| **Docker Compose** | `compose/docker-compose.yaml` | Multi-UAV deployment with profiles (`uav_001`, `uav_002`, `dev`, `mediamtx`) |
| **Compose config** | `compose/.env.example` | Template for all Docker Compose settings |
| **Per-UAV env** | `compose/env/uxv_001.env`, `uxv_002.env` | ROS_DOMAIN_ID and FastDDS discovery settings per UAV instance |
| **Docker entrypoint** | `docker-entrypoint.sh` | Sources ROS workspaces, configures FastDDS, exec's CMD |

### Launch System

| What | Where | Description |
|------|-------|-------------|
| **Production launch** | `launch/uxv_main.yaml` | Minimal production stack (IPM + ORTM + DMM video/logging) |
| **Dev launch (all modules)** | `launch/uxv_dev.yaml` | Full stack with all 5 modules — simple YAML includes |
| **Unified launcher (Option A)** | `launch/uxv_dev-full.launch.py` | Python launch file reading a single config YAML to control all modules |
| **Per-module launcher (Option B)** | `launch/uxv_dev-modules.launch.py` | Python launch file reading per-module config dirs (`params/`, `qos/`) |
| **Bag playback** | `launch/cams_from_bag.yml` | Replays recorded camera topics + runs ORTM pipeline |
| **DDS configs** | `launch/fastdds_*.xml`, `launch/cyclonedds*.xml` | FastDDS/CycloneDDS profiles for localhost, network, and simulation |
| **Dual cam recording** | `launch/uxv_main_with_dual_cam_rec.yaml` | Production + dual camera bag recording |
| **Mono cam recording** | `launch/uxv_main_with_mono_cam_rec.yaml` | Production + single camera bag recording |

### Configuration Profiles

| What | Where | Description |
|------|-------|-------------|
| **Unified config example** | `launch_profiles/example-config.yaml` | Single YAML controlling modules, launch files, params, and QoS (Option A) |
| **Per-module config example** | `launch_profiles/example-config/` | Directory with `base.yaml` + `params/*.yaml` + `qos/*.yaml` (Option B) |
| **Shared profiles** | `launch_profiles/shared/` | Reusable configs (e.g., `ortm-sim.yaml` for simulation) |
| **Local overrides** | `launch_profiles/local/` | Git-ignored directory for personal/machine-specific configs |

### Infrastructure & Deployment

| What | Where | Description |
|------|-------|-------------|
| **Prerequisites installer** | `install-pre-requisites.sh` | Installs Docker + NVIDIA container toolkit |
| **Image builder** | `build_runtime_image.sh` | Wraps `docker compose build` for the runtime image |
| **Legacy container scripts** | `create-container.sh`, `run-container.sh` | Manual Docker container lifecycle (Jetson deployment) |
| **Start/stop scripts** | `start-uxv.sh`, `stop-uxv.sh` | Wrap container run with logging |
| **Systemd service** | `systemd/uxv-main.service` | Auto-start on boot (depends on `docker.service` + `mavros-package.service`) |
| **CI/CD** | `.github/workflows/build-application-image.yml` | Builds multi-arch (amd64+arm64) × multi-distro (humble+jazzy) images, pushes to Azure ACR |
| **CI docs** | `docs/ci-workflow.md` | Detailed CI workflow documentation |

### Documentation

| What | Where | Description |
|------|-------|-------------|
| **README** | `README.md` | Setup, build, run instructions |
| **Interface reference** | `docs/INTERFACES.md` | All ROS topics, message types, QoS settings per node |
| **Launch config reference** | `docs/LAUNCH_CONFIGURATIONS.md` | All configurable parameters per node, with defaults |
| **CI docs** | `docs/ci-workflow.md` | CI pipeline documentation |

### Utility Scripts

| What | Where | Description |
|------|-------|-------------|
| **Update repos** | `update-repos.sh` | Pulls latest from all sub-repos |
| **MediaMTX launcher** | `launch-mediamtx.sh` | Starts RTSP server container for video streaming |
| **Mock publishers** | `launch/mock_publisher.py`, `launch/mavros_mock_publisher.py` | Test data generators |
| **Video inspection** | `scripts/inspect_video_timestamps.py` | Debug video timestamp alignment |
| **Package versions** | `scripts/get_package_versions.py` | List installed ROS package versions |
| **Profiling** | `scripts/nsys-profile.sh`, `scripts/setup-profiling-env.sh` | NVIDIA Nsight Systems profiling |
| **Streaming scripts** | `scripts/rgb-stream.sh`, `scripts/ir-stream.sh` | Camera stream utilities |

---

## Launch Configuration System

The launch system supports two approaches for configuring the stack:

### Option A — Single Unified Config File
- **Launch file:** `launch/uxv_dev-full.launch.py`
- **Config file:** One YAML (e.g., `launch_profiles/example-config.yaml`)
- The YAML has 3 sections:
  1. **General** — `log_level`, `launch_modules` list, `exclude_components` list
  2. **Launch file selection** — which ROS package and launch file per module
  3. **Parameter & QoS overrides** — per-node `ros__parameters` and `qos_overrides`

### Option B — Per-Module Config Directory
- **Launch file:** `launch/uxv_dev-modules.launch.py`
- **Config dir:** Directory with `base.yaml` + `params/<module>_overrides.yaml` + `qos/<module>_overrides.yaml`
- Same information as Option A, but split across files for easier version control

### Module Identifiers
| ID | Module |
|----|--------|
| `module_ip` | Input Processing Module (IPM) |
| `module_ort` | Object Recognition & Tracking Module (ORTM) |
| `module_hlp` | High-Level Planning Module (HLPM) |
| `module_dm` | Data Management Module (DMM) |
| `module_shm` | State & Health Monitoring Module (SHMM) |

### Component Exclusion
Individual components within modules can be disabled via `exclude_components`:

| ID | Component | Module |
|----|-----------|--------|
| `component_is_rgb` | RGB Camera | IPM |
| `component_is_thermal` | Thermal Camera | IPM |
| `component_or` | Object Recognition | ORTM |
| `component_mot` | Multi-Object Tracking | ORTM |
| `component_ol` | Object Localization | ORTM |
| `component_vm` | Video Manager | DMM |
| `component_lm` | Log Manager | DMM |
| `component_fch` | Flight Controller Handler | SHMM |
| `component_gpp` | Global Path Planner | HLPM |
| `component_lpp` | Local Path Planner | HLPM |
| `component_tg` | Terminal Guidance | HLPM |
| `component_pt` | Path Translator | HLPM |

---

## Docker & Deployment Architecture

### Image Hierarchy
```
NVIDIA L4T / Ubuntu + CUDA
  └─▶ uxv-cuda-base (built by devops repo)
        └─▶ uxv-main (built by this repo's Dockerfile)
              - colcon build artifacts baked in
              - AI model files (.engine, .onnx) copied
```

### Deployment Modes

1. **Docker Compose (PC/Laptop)** — `compose/docker-compose.yaml`
   - Profiles: `uav_001`, `uav_002` (multi-UAV), `dev` (interactive), `mediamtx` (RTSP)
   - NVIDIA GPU runtime, host networking, FastDDS discovery

2. **Legacy Shell Scripts (Jetson)** — `create-container.sh` + `run-container.sh`
   - Direct `docker run` with bind mounts for source, cameras, bags, videos
   - Used for on-device development and field deployment

3. **Systemd Service** — `systemd/uxv-main.service`
   - Auto-start after boot, depends on Docker + MAVROS
   - Used for autonomous field operation

### DDS / Middleware
- Default: **FastDDS** (`rmw_fastrtps_cpp`)
- Alternative: **CycloneDDS** (`rmw_cyclonedds_cpp`)
- Profiles:
  - `fastdds_localhost.xml` — single-machine
  - `fastdds_network.xml` — multi-machine
  - `fastdds_sim_template.xml` — simulation (discovery server mode)
  - `cyclonedds_localhost.xml` / `cyclonedds.xml` — CycloneDDS variants
- Per-UAV instance gets its own `ROS_DOMAIN_ID` and FastDDS discovery server port

---

## CI/CD Pipeline

- **Workflow:** `.github/workflows/build-application-image.yml` (manual trigger)
- Builds **4 variants** in parallel: `{amd64, arm64} × {humble, jazzy}`
- Uses `vcstool` to clone sources from a `.repos` manifest (pinned releases or `main`)
- Pushes to Azure Container Registry via OIDC (Workload Identity Federation)
- Creates multi-arch manifests: `uxv-main:latest-<distro>`, `uxv-main:<sha7>-<distro>`

---

## Quick Reference — Common Workflows

| Task                       | Command                                                                       |
| -------------------------- | ----------------------------------------------------------------------------- |
| Clone all source repos     | `vcs import < uxv-dev.repos`                                                  |
| Update all source repos    | `./update-repos.sh`                                                           |
| Build runtime Docker image | `./build_runtime_image.sh`                                                    |
| Start stack (compose)      | `cd compose && docker compose up -d`                                          |
| Start dev shell (compose)  | `docker compose run --rm --profile dev uxv-dev bash`                          |
| Start stack (legacy)       | `./create-container.sh`                                                       |
| Build inside container     | `colcon build --symlink-install && source install/setup.bash`                 |
| Launch full stack          | `ros2 launch launch/uxv_dev-full.launch.py`                                   |
| Launch with custom profile | `ros2 launch launch/uxv_dev-full.launch.py params_path:=/path/to/config.yaml` |
| Replay from bag            | `ros2 launch launch/cams_from_bag.yml bag_path:=/path/to/bag`                 |