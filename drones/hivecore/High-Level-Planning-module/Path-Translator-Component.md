# Path Translator Component (`path_translator_ros`)

> Repo Commit: 70a3fc3b2046b0cd9a5eaae002306357c85fbd47
> Date: 2026-05-18

> Copilot prompt:  
## Overview

`path_translator_ros` is a ROS 2 composable component that acts as a bridge between high-level path planning and low-level drone control. It receives a waypoint path and the drone's current pose, then continuously publishes position or velocity setpoints to guide the drone along the path via MAVROS.

The package is a thin ROS 2 wrapper around the core `PathTranslator` library (`hlpm::PathTranslator`), which handles all waypoint tracking logic independently of ROS.

**Node name:** `path_translator_component`  
**Namespace:** `hlpm`  
**Component class:** `hlpm::PathTranslatorROS`

---

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│              PathTranslatorROS (ROS 2 Node)               │
│                                                          │
│  ┌────────────────┐       ┌───────────────────────────┐  │
│  │  Subscribers   │       │   Core PathTranslator     │  │
│  │  - ego_pose    │──────▶│   - setPath()             │  │
│  │  - path        │       │   - setEgoPosition()      │  │
│  │  - module_state│       │   - update()              │  │
│  └────────────────┘       │   - getVelocity()         │  │
│                           │   - getNextWaypoint()     │  │
│  ┌────────────────┐       │   - getYaw()              │  │
│  │  Publishers    │◀──────│   - isPathComplete()      │  │
│  │  - setpoint    │       └───────────────────────────┘  │
│  │  - comp_check  │                                      │
│  └────────────────┘       ┌───────────────────────────┐  │
│                           │   Main Loop Thread        │  │
│                           │   (runs at loop_rate_hz)  │  │
│                           └───────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

The component runs a dedicated background thread (`mainLoop`) at a configurable rate. Each iteration:
1. Checks if path translation is active (gated by module state).
2. Calls `PathTranslator::update()` to advance waypoint tracking.
3. Publishes a `PositionTarget` setpoint based on the configured control mode.

---

## Functionalities

### Control Modes

The `control_mode` parameter selects what type of setpoint is published:

| Mode | Value | Description |
|------|-------|-------------|
| Position | 0 | Publishes target position; velocity/acceleration fields are ignored by the flight controller. |
| Velocity | 1 | Publishes velocity command; position/acceleration fields are ignored. |

### Yaw Control Modes

The `yaw_control_mode` parameter determines how yaw is computed:

| Mode | Value | Description |
|------|-------|-------------|
| Face Forward | 0 | Yaw = `atan2(dy, dx)` from ego toward the next waypoint. |
| Pose Yaw | 1 | Yaw extracted from the orientation quaternion of the current path pose. |
| Fixed Yaw | 2 | Constant yaw angle from the `fixed_yaw` parameter. |

### State Gating

The component only publishes setpoints when the module is in an **active** state. State transitions are received via the `module_state` topic:

| Active States (publishing enabled) | Inactive States (publishing disabled) |
|-------------------------------------|---------------------------------------|
| `state_premission` | `state_initialize` |
| `substate_way_point` | `state_disable` |
| `substate_terminal_guidance` | `state_standby` |
| `state_postmission` | `state_emergency` |

### Component Health Checks

On every module state callback, the component publishes diagnostic checks:

| Check Name | Meaning |
|------------|---------|
| `hlp_pt_library_initialized` | Core PathTranslator library initialized successfully. |
| `hlp_pt_is_path_available` | A path is loaded and not yet complete. |
| `hlp_pt_is_path_complete` | All waypoints in the current path have been reached. |

---

## ROS 2 Interface

### Subscribed Topics

| Topic (default) | Message Type | Description |
|-----------------|--------------|-------------|
| `/mavros/local_position/pose` | `geometry_msgs/msg/PoseStamped` | Current drone pose in the local frame. |
| `/hlpm/global_path_planning/global_path` | `geometry_msgs/msg/PoseArray` | Waypoint path to follow. |
| `/hlpm/hlp_state_manager/module_state` | `odp_common_ros/msg/ModuleState` | Module state for activation gating. |

### Published Topics

| Topic (default) | Message Type | Description |
|-----------------|--------------|-------------|
| `/mavros/setpoint_raw/local` | `mavros_msgs/msg/PositionTarget` | Position or velocity setpoint in `FRAME_LOCAL_NED`. |
| `/hlpm/component/check` | `odp_common_ros/msg/ComponentCheckArray` | Component health check diagnostics. |

---

## Configuration Parameters

Parameters are declared in `config/path_translator_params.yaml`:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `ego_pose_topic` | string | `/mavros/local_position/pose` | Topic for ego pose subscription. |
| `path_topic` | string | `/hlpm/global_path_planning/global_path` | Topic for path subscription. |
| `module_state_topic` | string | `/hlpm/hlp_state_manager/module_state` | Topic for module state subscription. |
| `setpoint_topic` | string | `/mavros/setpoint_raw/local` | Topic for setpoint publication. |
| `component_check_topic` | string | `/hlpm/component/check` | Topic for component check publication. |
| `loop_rate_hz` | double | `30.0` | Control loop frequency [Hz]. |
| `max_speed` | double | `3.0` | Maximum travel speed [m/s]. |
| `waypoint_reached_threshold` | double | `2.0` | Distance to consider a waypoint reached [m]. |
| `control_mode` | int | `1` | Setpoint type: 0 = position, 1 = velocity. |
| `yaw_control_mode` | int | `0` | Yaw mode: 0 = face_forward, 1 = pose_yaw, 2 = fixed_yaw. |
| `fixed_yaw` | double | `0.0` | Fixed yaw angle [rad] (used when `yaw_control_mode = 2`). |

QoS settings can be overridden via `config/path_translator_qos_settings.yaml`.

---

## Dependencies

### ROS 2 Packages

| Package | Purpose |
|---------|---------|
| `rclcpp` | ROS 2 C++ client library. |
| `rclcpp_components` | Composable node registration and loading. |
| `geometry_msgs` | `PoseStamped` and `PoseArray` message types. |
| `mavros_msgs` | `PositionTarget` message type for drone setpoints. |

### Internal Project Packages

| Package | Purpose |
|---------|---------|
| `path_translator` | Core waypoint-following library (`hlpm::PathTranslator`). Contains all path tracking logic independent of ROS. |
| `odp_common` | Shared types (`Point3D`, `Pose`, `StateID`, `ComponentID`). |
| `odp_common_ros` | ROS message definitions (`ModuleState`, `ComponentCheck`, `ComponentCheckArray`). |

### Build System

| Tool | Purpose |
|------|---------|
| `ament_cmake` | ROS 2 CMake build tool. |
| `ament_cmake_gtest` | Unit testing (test dependency). |

---

## Launch

The package provides a launch file at `launch/path_translator.launch.py` that starts the component inside a `ComposableNodeContainer`:

```bash
ros2 launch path_translator_ros path_translator.launch.py
```

---

## File Structure

```
path_translator_ros/
├── CMakeLists.txt
├── package.xml
├── README.md
├── Path-Translator-Component.md    # This file
├── config/
│   ├── path_translator_params.yaml
│   └── path_translator_qos_settings.yaml
├── include/path_translator_ros/
│   └── path_translator_ros.hpp
├── launch/
│   └── path_translator.launch.py
├── src/
│   └── path_translator_ros.cpp
└── test/
    └── test_path_translator_ros.cpp
```
