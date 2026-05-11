# MAVLink Messages for PX4 Drone Simulator (sim.py)

## Overview

This document lists all MAVLink messages that `sim.py` must send and receive to simulate a PX4 autopilot for the UXV autonomy stack. The simulator communicates via MAVLink over UDP (port 14540) through mavrouter to mavros, which exposes ROS 2 topics consumed/published by the repository's modules.

### Architecture

```
sim.py (PX4 sim) ←── MAVLink/UDP:14540 ──→ mavrouter ←──→ mavros ←──→ ROS 2 Topics ←──→ UXV Modules
```

### Modules in This Repository

| Module | Full Name | Description |
|--------|-----------|-------------|
| **DMM** | Drone Mission Module | Flight data logging, image geo-tagging, FC log download |
| **HLPM** | High Level Planning Module | Global path planning, path translation to setpoints |
| **IPM** | Input Processing Module | Camera streaming, time synchronization |
| **ODP** | Object Detection & Tracking | Object localization using drone pose + GPS |
| **SHMM** | System Health Monitoring | Health checks (no direct mavros dependency) |

---

## Messages sim.py Must SEND (→ mavrouter → mavros → ROS 2)

### 1. HEARTBEAT

| Property | Details |
|----------|---------|
| **MAVLink ID** | 0 |
| **Send Rate** | 1 Hz |
| **ROS 2 Topic** | `/mavros/state` |
| **ROS 2 Msg Type** | `mavros_msgs/msg/State` |
| **Consumers** | DMM (`log_manager_node`, `fc_log_downloader_node`, `dm_state_manager`) |
| **Purpose** | Publishes vehicle connected/armed status and flight mode. DMM uses arm→disarm transitions to trigger flight data collection and FC log downloads. The state manager monitors this topic for liveness. |
| **Key Fields** | `type` = `MAV_TYPE_QUADROTOR` (2), `autopilot` = `MAV_AUTOPILOT_PX4` (12), `base_mode` = armed flag (bit 7), `custom_mode` = PX4 flight mode enum, `system_status` = `MAV_STATE_STANDBY` or `MAV_STATE_ACTIVE` |
| **Sim Behavior** | Start disarmed in MANUAL mode. Toggle armed state via COMMAND_LONG or keyboard. Cycle mode on command. |

---

### 2. LOCAL_POSITION_NED

| Property | Details |
|----------|---------|
| **MAVLink ID** | 32 |
| **Send Rate** | 30–50 Hz |
| **ROS 2 Topic** | `/mavros/local_position/pose` (position part) |
| **ROS 2 Msg Type** | `geometry_msgs/msg/PoseStamped` |
| **Consumers** | HLPM (`global_path_planning_ros`, `path_translator_ros`), ODP (`object_localization_ros`), IPM (dual-cam recording), state managers (HLPM, ODP) |
| **Purpose** | Provides ego-vehicle local position in NED frame. HLPM uses it for path planning reference and to compute error between current position and setpoints. ODP uses it for object localization relative to drone position. |
| **Key Fields** | `time_boot_ms` (ms since boot), `x` (North, meters), `y` (East, meters), `z` (Down, meters — negative = above home), `vx, vy, vz` (m/s in NED) |
| **Sim Behavior** | Updated every tick from kinematics model. Position integrates velocity; velocity changes toward active setpoint with acceleration limits. |

---

### 3. ATTITUDE_QUATERNION

| Property | Details |
|----------|---------|
| **MAVLink ID** | 61 |
| **Send Rate** | 30–50 Hz |
| **ROS 2 Topic** | `/mavros/local_position/pose` (orientation part) |
| **ROS 2 Msg Type** | `geometry_msgs/msg/PoseStamped` → `pose.orientation` (quaternion x,y,z,w) |
| **Consumers** | Same as LOCAL_POSITION_NED (mavros combines both into one PoseStamped) |
| **Purpose** | Provides drone orientation. mavros uses LOCAL_POSITION_NED + ATTITUDE_QUATERNION to construct the complete PoseStamped with both position and orientation. |
| **Key Fields** | `time_boot_ms`, `q1` (w), `q2` (x), `q3` (y), `q4` (z), `rollspeed`, `pitchspeed`, `yawspeed` (rad/s) |
| **Sim Behavior** | Heading derived from velocity vector direction (yaw = atan2(vy, vx)). Roll/pitch kept at 0 for simple point-mass model. Quaternion computed from Euler (0, 0, yaw). |

---

### 4. GLOBAL_POSITION_INT

| Property | Details |
|----------|---------|
| **MAVLink ID** | 33 |
| **Send Rate** | 5–10 Hz |
| **ROS 2 Topic** | `/mavros/global_position/global` |
| **ROS 2 Msg Type** | `sensor_msgs/msg/NavSatFix` |
| **Consumers** | DMM (`log_manager_node`, `image_processor_node`, `thermal_video_logger_node`), ODP (`object_localization_ros`), IPM (dual-cam recording), state managers (DMM, HLPM, ODP) |
| **Purpose** | Global GPS coordinates for geo-tagging images and thermal video, mission logging, and computing object locations in geographic coordinates. |
| **Key Fields** | `time_boot_ms`, `lat` (degrees × 10⁷), `lon` (degrees × 10⁷), `alt` (mm AMSL), `relative_alt` (mm above home), `vx, vy, vz` (cm/s, NED), `hdg` (heading, cdeg 0-35999, UINT16_MAX if unknown) |
| **Sim Behavior** | Converts local NED position to lat/lon using flat-earth approximation: `lat = home_lat + (north_m / 111319.5)`, `lon = home_lon + (east_m / (111319.5 * cos(home_lat)))`. Altitude = home_alt - z (since z is down). |

---

### 5. GPS_RAW_INT

| Property | Details |
|----------|---------|
| **MAVLink ID** | 24 |
| **Send Rate** | 5 Hz |
| **ROS 2 Topic** | `/mavros/gpsstatus/gps1/raw` |
| **ROS 2 Msg Type** | `mavros_msgs/msg/GPSRAW` |
| **Consumers** | DMM (`log_manager_node`, `image_processor_node`, `thermal_video_logger_node`), state manager (DMM) |
| **Purpose** | Raw GPS sensor data including fix quality metrics. Used to assess GPS health (fix type, satellite count, dilution of precision) and for raw position metadata in images/video. |
| **Key Fields** | `time_usec` (µs since epoch), `fix_type` (0=no fix, 2=2D, 3=3D), `lat` (degE7), `lon` (degE7), `alt` (mm AMSL), `eph` (HDOP in cm, lower=better), `epv` (VDOP in cm), `vel` (groundspeed cm/s), `cog` (course over ground, cdeg), `satellites_visible` (count) |
| **Sim Behavior** | Mirror lat/lon/alt from GLOBAL_POSITION_INT. Always report: `fix_type=3` (3D fix), `satellites_visible=12`, `eph=80` (0.8m HDOP), `epv=120` (1.2m VDOP). |

---

### 6. VFR_HUD

| Property | Details |
|----------|---------|
| **MAVLink ID** | 74 |
| **Send Rate** | 4 Hz |
| **ROS 2 Topic** | `/mavros/vfr_hud` |
| **ROS 2 Msg Type** | `mavros_msgs/msg/VfrHud` |
| **Consumers** | DMM (`image_processor_node`, `thermal_video_logger_node`) |
| **Purpose** | Flight instrument data for image/video metadata overlays. Provides heading, speed, and altitude in human-readable form. Used to embed flight telemetry into captured imagery. |
| **Key Fields** | `airspeed` (m/s), `groundspeed` (m/s), `heading` (degrees 0-360, North=0), `throttle` (0-100%), `alt` (meters AMSL), `climb` (m/s, positive=ascending) |
| **Sim Behavior** | Derived from kinematics: `groundspeed = sqrt(vx²+vy²)`, `airspeed = groundspeed` (no wind model), `heading = atan2(vy,vx) converted to 0-360`, `climb = -vz` (NED→climb), `alt = home_alt - z`. |

---

### 7. AUTOPILOT_VERSION

| Property | Details |
|----------|---------|
| **MAVLink ID** | 148 |
| **Send Rate** | On request + once at connection |
| **ROS 2 Topic** | `/mavros/vehicle_info` |
| **ROS 2 Msg Type** | `mavros_msgs/msg/VehicleInfo` |
| **Consumers** | DMM (`log_manager_node`, `fc_log_downloader_node`), state manager (DMM) |
| **Purpose** | Identifies the autopilot type and firmware version. DMM uses this to determine whether the FC is PX4 or ArduPilot, which dictates the log download method (PX4 uses MAVLink FTP for .ulg files, ArduPilot uses different protocol). |
| **Key Fields** | `capabilities` (bitmask: MAV_PROTOCOL_CAPABILITY_*), `flight_sw_version` (semantic version packed as uint32), `middleware_sw_version`, `os_sw_version`, `board_version`, `vendor_id`, `product_id`, `uid` |
| **Sim Behavior** | Send proactively at startup and respond to `MAV_CMD_REQUEST_AUTOPILOT_CAPABILITIES`. Report as PX4 v1.14.0 with standard capabilities. |

---

### 8. HOME_POSITION

| Property | Details |
|----------|---------|
| **MAVLink ID** | 242 |
| **Send Rate** | Every 5 seconds (periodic) |
| **ROS 2 Topic** | `/mavros/home_position/home` |
| **ROS 2 Msg Type** | `mavros_msgs/msg/HomePosition` |
| **Consumers** | HLPM (`global_path_planning_ros`) |
| **Purpose** | Defines the geographic reference point (home position) for the path planner. HLPM uses this as the NED coordinate frame origin when planning global paths and converting between geographic and local coordinates. |
| **Key Fields** | `latitude` (degE7), `longitude` (degE7), `altitude` (mm AMSL), `x, y, z` (local position of home in NED — always 0,0,0), `q[4]` (orientation quaternion of home approach), `approach_x, approach_y, approach_z` (landing approach vector) |
| **Sim Behavior** | Send configured home position (from CLI args). Static — does not change during flight. Local position (x,y,z) always (0,0,0). Approach vector = (0,0,-1) (straight down). |

---

### 9. TIMESYNC

| Property | Details |
|----------|---------|
| **MAVLink ID** | 111 |
| **Send Rate** | On request (response to mavros TIMESYNC) |
| **ROS 2 Topic** | `/mavros/timesync_status` |
| **ROS 2 Msg Type** | `mavros_msgs/msg/TimesyncStatus` |
| **Consumers** | IPM (`image_streaming_ros`) |
| **Purpose** | Time synchronization protocol between autopilot and companion computer. IPM uses the computed time offset to accurately timestamp camera frames in autopilot time, ensuring temporal consistency between images and telemetry. |
| **Key Fields** | `tc1` (companion timestamp in ns — copied from request), `ts1` (autopilot timestamp in ns — sim's current time) |
| **Protocol** | mavros sends TIMESYNC with `tc1=0, ts1=companion_time`. sim.py responds with `tc1=received_ts1, ts1=sim_current_time`. mavros computes offset from round-trip. |
| **Sim Behavior** | When receiving TIMESYNC with tc1=0, respond with tc1=received_ts1 and ts1=current boot time in nanoseconds. |

---

## Messages sim.py Must RECEIVE (← mavros ← ROS 2 ← UXV modules)

### 1. SET_POSITION_TARGET_LOCAL_NED

| Property | Details |
|----------|---------|
| **MAVLink ID** | 84 |
| **ROS 2 Topic** | `/mavros/setpoint_raw/local` |
| **ROS 2 Msg Type** | `mavros_msgs/msg/PositionTarget` |
| **Publisher** | HLPM (`path_translator_ros`) |
| **Purpose** | Position/velocity/acceleration setpoints commanding the drone where to fly. HLPM's path translator converts global path waypoints into local NED setpoints and streams them to the autopilot at ~30 Hz. |
| **Key Fields** | `time_boot_ms`, `coordinate_frame` (MAV_FRAME_LOCAL_NED=1), `type_mask` (bitmask: bits set = fields IGNORED), `x, y, z` (position meters NED), `vx, vy, vz` (velocity m/s), `afx, afy, afz` (accel m/s²), `yaw` (rad), `yaw_rate` (rad/s) |
| **type_mask interpretation** | Bit 0: ignore x, Bit 1: ignore y, Bit 2: ignore z, Bit 3: ignore vx, Bit 4: ignore vy, Bit 5: ignore vz, Bit 6-8: ignore accel, Bit 9: ignore force, Bit 10: ignore yaw, Bit 11: ignore yaw_rate |
| **Sim Behavior** | Parse type_mask to determine control mode (position, velocity, or mixed). If position fields valid: set as target position, compute velocity to reach it. If velocity fields valid: directly set velocity. Respect max_velocity (default 5 m/s) and max_acceleration (default 2 m/s²). |

---

### 2. HEARTBEAT (from mavros)

| Property | Details |
|----------|---------|
| **MAVLink ID** | 0 |
| **Source** | mavros node (acts as GCS, system_id=255 or configurable) |
| **Purpose** | mavros sends periodic heartbeats as a MAVLink GCS. sim.py uses this to confirm mavros is connected and communicating. |
| **Key Fields** | `type` = MAV_TYPE_GCS (6), `autopilot` = MAV_AUTOPILOT_INVALID (8) |
| **Sim Behavior** | Track timestamp of last received mavros heartbeat. Log connection/disconnection events. No state change required. |

---

### 3. COMMAND_LONG (optional but recommended)

| Property | Details |
|----------|---------|
| **MAVLink ID** | 76 |
| **Source** | mavros (internal or relaying service calls) |
| **Purpose** | Vehicle commands sent by mavros for arm/disarm, mode changes, and capability requests. Although no mavros services are called in this repo currently, mavros itself sends these during initialization. |
| **Relevant Commands** | |
| `MAV_CMD_COMPONENT_ARM_DISARM` (400) | param1: 1=arm, 0=disarm. param2: 21196=force |
| `MAV_CMD_REQUEST_AUTOPILOT_CAPABILITIES` (520) | param1: 1=request version. Respond with AUTOPILOT_VERSION |
| `MAV_CMD_DO_SET_MODE` (176) | param1: base_mode, param2: custom_mode |
| **Sim Behavior** | Process command, update internal state (armed, mode), respond with COMMAND_ACK (result=MAV_RESULT_ACCEPTED or MAV_RESULT_DENIED). |

---

## Summary Table

### Outbound (sim.py → PX4 → mavros → ROS 2)

| # | MAVLink Message | ID | Rate | ROS 2 Topic | Consuming Modules |
|---|----------------|----|----- |-------------|-------------------|
| 1 | HEARTBEAT | 0 | 1 Hz | `/mavros/state` | DMM |
| 2 | LOCAL_POSITION_NED | 32 | 30 Hz | `/mavros/local_position/pose` | HLPM, ODP, IPM |
| 3 | ATTITUDE_QUATERNION | 61 | 30 Hz | `/mavros/local_position/pose` | HLPM, ODP, IPM |
| 4 | GLOBAL_POSITION_INT | 33 | 5 Hz | `/mavros/global_position/global` | DMM, ODP, IPM |
| 5 | GPS_RAW_INT | 24 | 5 Hz | `/mavros/gpsstatus/gps1/raw` | DMM |
| 6 | VFR_HUD | 74 | 4 Hz | `/mavros/vfr_hud` | DMM |
| 7 | AUTOPILOT_VERSION | 148 | on-req | `/mavros/vehicle_info` | DMM |
| 8 | HOME_POSITION | 242 | 0.2 Hz | `/mavros/home_position/home` | HLPM |
| 9 | TIMESYNC | 111 | on-req | `/mavros/timesync_status` | IPM |

### Inbound (ROS 2 → mavros → mavrouter → sim.py)

| # | MAVLink Message | ID | Source Module | Purpose |
|---|----------------|----|----|---------|
| 1 | SET_POSITION_TARGET_LOCAL_NED | 84 | HLPM (path_translator) | Flight setpoints |
| 2 | HEARTBEAT | 0 | mavros | Connection tracking |
| 3 | COMMAND_LONG | 76 | mavros | Arm/disarm/mode commands |

---

## Detailed Consumer Breakdown by Component

### DMM — Drone Mission Module

| Component | File | Topics Consumed | What It Does |
|-----------|------|-----------------|--------------|
| `log_manager_node` | `python/ros/.../log_manager_node.py` | state, global_position, gpsstatus, vehicle_info | Monitors arm/disarm to start/stop flight data collection. Records GPS + raw GPS into mission logs. Uses vehicle_info to identify PX4. |
| `fc_log_downloader_node` | `python/ros/.../fc_log_downloader_node.py` | state, vehicle_info | Detects arm→disarm transition to trigger FC log (.ulg) download. Uses vehicle_info to confirm PX4 autopilot. |
| `image_processor_node` | `python/ros/.../image_processor_node.py` | global_position, gpsstatus, vfr_hud | Enriches captured images with GPS coordinates, raw GPS quality, heading, and altitude metadata. |
| `thermal_video_logger_node` | `cpp/ros/.../thermal_video_logger_node.cpp` | global_position, gpsstatus, vfr_hud | Embeds GPS, heading, altitude into thermal video stream metadata. |
| `dm_state_manager` | config: `dm_state_manager_params.yaml` | state, global_position, gpsstatus, vehicle_info | Health monitoring — checks these topics are alive/publishing. |

### HLPM — High Level Planning Module

| Component | File | Topics Consumed | Topics Published | What It Does |
|-----------|------|-----------------|-----------------|--------------|
| `global_path_planning_ros` | `cpp/ros/.../global_path_planning_ros.cpp` | local_position/pose, home_position | — | Uses ego pose + home as reference frame for computing global path waypoints. |
| `path_translator_ros` | `cpp/ros/.../path_translator_ros.cpp` | local_position/pose | setpoint_raw/local | Converts global path waypoints to local NED setpoints. Reads current pose for error computation and streams setpoints to PX4. |
| `hlp_state_manager` | config: `hlp_state_manager_params.yaml` | local_position/pose, global_position, setpoint_raw/local | — | Health monitoring — verifies these topics are alive. |

### IPM — Input Processing Module

| Component | File | Topics Consumed | What It Does |
|-----------|------|-----------------|--------------|
| `image_streaming_ros` | `cpp/ros/.../image_streaming_ros.cpp` | timesync_status | Uses autopilot↔companion time offset to correctly timestamp camera frames. Only active when `do_timesync: true` in config. |

### ODP — Object Detection & Tracking

| Component | File | Topics Consumed | What It Does |
|-----------|------|-----------------|--------------|
| `object_localization_ros` | config: `object_localization_params.yaml` | local_position/pose, global_position | Uses drone position (local + GPS) to compute detected object geographic coordinates via projection. |

---

## Notes

- **No mavros service calls** exist anywhere in this codebase — all interaction is via topics only.
- **SHMM** has zero direct mavros dependencies.
- The existing `launch/mavros_mock_publisher.py` only mocks 2 topics at the ROS 2 level — `sim.py` replaces this entirely at the MAVLink level, providing a complete PX4 simulation.
- mavrouter must be configured to route between sim.py (UDP:14540) and mavros (typically UDP:14541 or serial).
- PX4 `system_id` = 1, `component_id` = 1 (autopilot). mavros typically uses `system_id` = 255 (GCS).
