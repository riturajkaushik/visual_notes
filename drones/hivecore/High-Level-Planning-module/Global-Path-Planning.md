# Global Path Planning ROS

## Package Overview

`global_path_planning_ros` is a ROS 2 composable node that provides state-driven global path planning for unmanned aerial vehicles (UAVs). It wraps the core `GlobalPathPlanning` library, which implements two guidance modes:

- **Terminal Guidance** — computes a path toward a target point using approach geometry parameters (elevation angle, azimuth, radius).
- **Flightplan Guidance** — computes a path following a mission plan (waypoints) received from a ground control station.

The node switches between these modes based on the vehicle's mission state, publishing the computed path as a `PoseArray` of waypoints.

---

## Component: `GlobalPathPlanningROS`

**Namespace:** `hlpm`  
**Node Name:** `global_path_planning`  
**Plugin:** Registered as a composable component via `rclcpp_components`

### Subscribers

| Topic (default) | Message Type | Description |
|---|---|---|
| `/mavros/local_position/pose` | `geometry_msgs/PoseStamped` | Current vehicle pose in local ENU frame |
| `/mavros/home_position/home` | `mavros_msgs/HomePosition` | Home position (GNSS + local coordinates) |
| `/hlpm/hlp_state_manager/module_state` | `odp_common_ros/ModuleState` | Mission state machine transitions |
| `/platform/uav_interface/terminal/approach_params` | `odp_common_ros/ApproachParams` | Terminal approach geometry (elevation, azimuth, radius) |
| `/platform/uav_interface/terminal/target_point` | `geometry_msgs/Point` | Target point for terminal guidance |
| `/platform/uav_interface/mission/plan` | `odp_common_ros/MissionItemArray` | Mission plan (waypoints) from GCS |

### Publishers

| Topic (default) | Message Type | Description |
|---|---|---|
| `/hlpm/global_path_planning/global_path` | `geometry_msgs/PoseArray` | Computed global path (position + orientation per waypoint) |
| `/hlpm/component/check` | `odp_common_ros/ComponentCheckArray` | Component health/status checks |

### Service Servers

| Service (default) | Type | Description |
|---|---|---|
| `/hlpm/global_path_planning/mission/pause` | `MissionPause` | Pauses the mission; publishes an empty path |
| `/hlpm/global_path_planning/mission/continue` | `MissionContinue` | Resumes mission from the current waypoint index |
| `/hlpm/global_path_planning/mission/pull` | `MissionPull` | Returns the full original mission plan |

### Parameters

All topic and service names are configurable via ROS parameters declared at node startup:

- `ego_pose_topic`, `home_position_topic`, `module_state_topic`
- `approach_params_topic`, `target_point_topic`, `mission_plan_topic`
- `global_path_topic`, `component_check_topic`
- `mission_pause_service`, `mission_continue_service`, `mission_pull_service`

### State-Driven Behavior

The node reacts to mission state transitions received on the module state topic:

| State | Action |
|---|---|
| `substate_terminal_guidance` | Switches to terminal guidance mode; calculates and publishes terminal path |
| `state_standby`, `state_premission`, `substate_way_point`, `state_postmission` | Switches to flightplan guidance mode; calculates and publishes mission path |
| `state_emergency`, `state_initialize` | Ignored (no path published) |

Target point updates only trigger path recalculation when the node is in `substate_terminal_guidance`.

### Component Health Checks

The node publishes diagnostic checks covering:
- Ingress path status and reach
- Mission plan calculation and receipt
- Terminal path calculation and finish
- Egress reach status
- Home position availability

---

## Coordinate Frames

| Context | Frame | Notes |
|---|---|---|
| Published path (`global_path`) | `map` | All path waypoints are published in the `map` frame |
| Ego pose input | Local ENU | Received from MAVROS local position (East-North-Up) |
| Home position | GNSS + Local ENU | Stores both lat/lon/alt (geodetic) and local ENU position |
| Mission waypoints | Per-waypoint `MavFrame` | Each waypoint carries its own frame identifier; supports both global (GNSS) and local NED frame waypoints |

> **Note:** This node does not perform tf2 transformations. Frame conversions between GNSS, local NED, and local ENU are handled internally by the core `GlobalPathPlanning` library.

The node is forced to use two different local frames, namely ENU and NED, because different parts of the system follow different conventions:

 - **ENU (East-North-Up)** — the ROS/MAVROS standard. MAVROS automatically converts the autopilot's native frame to ENU before publishing /mavros/local_position/pose. This is what the ROS ecosystem expects.
 
 - **NED (North-East-Down)** — the MAVLink/PX4/ArduPilot standard. Mission waypoints coming from a ground control station via MAVLink use NED (or global geodetic frames). The MavFrame field on each waypoint reflects this.

So ENU comes from the ROS side (MAVROS), and NED comes from the autopilot/GCS side (MAVLink protocol). The core GlobalPathPlanning library handles the conversion internally and outputs everything in the map frame, so downstream consumers don't need to worry about it.

This is a very common pattern in UAV stacks — ROS standardized on ENU, while aviation/autopilot ecosystems standardized on NED. Unifying at the boundary would require modifying either MAVROS conventions or the MAVLink mission protocol, both of which are external standards.

---

## Dependencies

### Build & Runtime

| Dependency | Purpose |
|---|---|
| `rclcpp` | ROS 2 C++ client library |
| `rclcpp_components` | Composable node registration |
| `geometry_msgs` | `PoseStamped`, `PoseArray`, `Point` messages |
| `mavros_msgs` | `HomePosition` message |
| `global_path_planning` | Core path planning algorithm library |
| `odp_common` | Shared types (`StateID`, `Pose`, `Point3D`, `MissionItem`, etc.) |
| `odp_common_ros` | Custom messages/services (`ModuleState`, `ApproachParams`, `MissionItemArray`, `ComponentCheckArray`, `MissionPause`/`Continue`/`Pull`) |

### Test

- `ament_cmake_gtest`, `ament_lint_auto`, `ament_lint_common`, `yaml-cpp`

### Execution

- `ros2launch`

---

## Launch & Configuration

- **Launch file:** `launch/global_path_planning.launch.py`
- **Parameters config:** `config/global_path_planning_params.yaml`
- **QoS settings:** `config/global_path_planning_qos_settings.yaml`

The launch file loads the composable node with its parameter and QoS configuration files.
