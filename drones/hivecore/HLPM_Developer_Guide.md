# HLPM Developer Guide

## Overview

The **High Level Planning Module (HLPM)** is responsible for global path planning, state management, and path translation for UXV missions. It consumes position data from MAVROS, plans waypoint paths, and outputs velocity/position setpoints back to MAVROS at 30 Hz.

---

## 1. Code Organization

### Architecture: Core/ROS Split

All business logic lives in `cpp/core/` (pure C++, no ROS dependency). ROS2 wrappers in `cpp/ros/` handle pub/sub, parameters, and message conversions. This ensures testability and portability.

```
src/hlpm/
├── cpp/
│   ├── core/                              ← Platform-agnostic C++ libraries
│   │   ├── global_path_planning/          ← Guidance strategies (flightplan + terminal Bézier)
│   │   ├── hlp_state_manager/             ← Hierarchical state machine for HLPM states
│   │   └── path_translator/               ← Follows waypoint path → velocity/position setpoints
│   └── ros/                               ← ROS2 wrappers (subscribers, publishers, services)
│       ├── global_path_planning_ros/      ← ROS2 wrapper for GPP
│       ├── hlp_state_manager_ros/         ← ROS2 wrapper for state manager
│       ├── path_translator_ros/           ← ROS2 wrapper for path translator
│       └── hlpm_bringup_ros/             ← Launch config — composable container assembly
```

### Package Summary (7 packages)

| Package | Layer | Purpose |
|---|---|---|
| `global_path_planning` | Core | Guidance strategies: flightplan + terminal (Bézier curves) |
| `hlp_state_manager` | Core | Hierarchical state machine for HLPM state transitions |
| `path_translator` | Core | Follows waypoint path, outputs velocity/position setpoints |
| `global_path_planning_ros` | ROS | ROS2 wrapper — subscriptions, publishers, services for GPP |
| `hlp_state_manager_ros` | ROS | ROS2 wrapper — state intent handling for HSM |
| `path_translator_ros` | ROS | ROS2 wrapper — control loop thread, setpoint publishing |
| `hlpm_bringup_ros` | ROS | Master launch — assembles all components into composable container |

### File Inventory

**Core: `global_path_planning` (9 files)**

- `include/global_path_planning/global_path_planning.hpp` — `GlobalPathPlanning` orchestrator class
- `include/global_path_planning/global_path_planning_types.hpp` — `ApproachGeometry` struct
- `include/global_path_planning/global_path_planning_utils.hpp` — Inline utilities (orientation, distance, conversions)
- `include/global_path_planning/flightplan_guidance/flightplan_guidance.hpp` — `FlightplanGuidance` class
- `include/global_path_planning/flightplan_guidance/flightplan_guidance_compile_params.hpp` — Compile-time threshold
- `include/global_path_planning/terminal_guidance/terminal_guidance.hpp` — `TerminalGuidance` class
- `include/global_path_planning/terminal_guidance/terminal_guidance_compile_params.hpp` — 6 Bézier params
- `src/global_path_planning.cpp` — Orchestrator implementation
- `src/flightplan_guidance/flightplan_guidance.cpp` — Mission waypoint translation (GNSS→ENU, NED→ENU)
- `src/terminal_guidance/terminal_guidance.cpp` — Bézier approach path computation

**Core: `hlp_state_manager` (3 files + config)**

- `include/hlp_state_manager/hlp_state_manager.hpp` — `HLPStateManager` class
- `src/hlp_state_manager.cpp` — HSM check validation logic
- `config/state_checks.yaml` — State-specific check definitions

**Core: `path_translator` (3 files)**

- `include/path_translator/path_translator.hpp` — `PathTranslator` class
- `include/path_translator/path_translator_runtime_params.hpp` — `PTCalibrationParams`, `YawControlMode` enum
- `src/path_translator.cpp` — Waypoint following + velocity output

**ROS wrappers** — each has: header, source, config YAML, QoS YAML, and launch file.

---

## 2. Class Hierarchy & Inheritance

```
hsmcpp::HierarchicalStateMachine        (3rd-party HSM library)
  └── common::BaseStateManager          (odp_common — template HSM for ODP)
        └── hlpm::HLPStateManager       (core — HLP-specific state logic)

rclcpp::Node                            (ROS2 base)
  ├── hlpm::GlobalPathPlanningROS       (ROS wrapper for GPP)
  ├── hlpm::HLPStateManagerROS          (ROS wrapper for HSM)
  └── hlpm::PathTranslatorROS           (ROS wrapper for PT)

hlpm::GlobalPathPlanning                (standalone — owns:)
  ├── std::unique_ptr<TerminalGuidance>
  └── std::unique_ptr<FlightplanGuidance>

hlpm::PathTranslator                    (standalone)
```

---

## 3. C++ Design Patterns

### Strategy Pattern ⭐ (Primary)

`GlobalPathPlanning` orchestrates two interchangeable guidance strategies:

- `TerminalGuidance` — Bézier curve approach to a target
- `FlightplanGuidance` — Follow predefined mission waypoints
- Selected at runtime via `GuidanceMode` enum and `setGuidanceMode()`
- `calculatePath()` dispatches to the active strategy

### Template Method Pattern

`BaseStateManager` (from odp-common) defines the HSM structure template and declares pure virtual hooks:

- `initializeStateMaps()`, `isValidEntity()`, `processEntityChecks()`, `printStateSummary()`
- `HLPStateManager` implements these hooks for HLP-specific behavior

### Observer/Callback Pattern

- `HLPStateManager` takes `std::function<void(StateID)> statusCallback` — notifies ROS layer of state changes
- All ROS nodes use subscription callbacks

### Adapter Pattern

Each `*ROS` class adapts between ROS2 messages and core library types:

- `geometry_msgs::msg::PoseStamped` ↔ `common::Pose`
- `mavros_msgs::msg::HomePosition` ↔ `common::Point3DGnss` + `common::Point3D`
- `odp_common_ros::msg::MissionItemArray` ↔ `common::MissionPlan`
- `mavros_msgs::msg::PositionTarget` ← `common::Point3D` velocity/position

### Facade Pattern

`GlobalPathPlanning` is a facade over both guidance strategies — external code interacts only with this unified interface.

### Composable Component Pattern (ROS2)

All 3 ROS nodes registered with `RCLCPP_COMPONENTS_REGISTER_NODE()` macro. The bringup launch uses `ComposableNodeContainer` with `component_container_mt` (multi-threaded).

### Non-copyable/Non-movable Idiom

All core classes explicitly `= delete` copy/move constructors/operators.

---

## 4. Macros Used

### Compile-Time Configuration Macros (via CMake `-D` flags)

**Terminal Guidance** (`terminal_guidance_compile_params.hpp`):

| Macro | Default | Maps To Constant |
|---|---|---|
| `CONFIG_TG_MIN_BEZIER_SAMPLES` | 1 | `kMinBezierSamples` |
| `CONFIG_TG_MAX_BEZIER_SAMPLES` | 20 | `kMaxBezierSamples` |
| `CONFIG_TG_METERS_PER_SAMPLE` | 5.0f | `kMetersPerSample` |
| `CONFIG_TG_BEZIER_CONTROL_FACTOR` | 0.5f | `kBezierControlFactor` |
| `CONFIG_TG_FOLLOW_UP_DISTANCE` | 10.0f | `kFollowUpDistance` |
| `CONFIG_TG_WP_REACHED_THRESHOLD` | 2.0f | `kWaypointReachedThreshold` |

**Flightplan Guidance** (`flightplan_guidance_compile_params.hpp`):

| Macro | Default | Maps To Constant |
|---|---|---|
| `CONFIG_FPG_WAYPOINT_REACHED_THRESHOLD_M` | 2.0f | `kWaypointReachedThresholdMeters` |

**Pattern used:**

```cpp
#ifdef CONFIG_TG_MIN_BEZIER_SAMPLES
static constexpr int kMinBezierSamples = CONFIG_TG_MIN_BEZIER_SAMPLES;
#else
static constexpr int kMinBezierSamples = 1;
#endif
```

### ROS2 Component Registration Macros

```cpp
RCLCPP_COMPONENTS_REGISTER_NODE(hlpm::GlobalPathPlanningROS)   // global_path_planning_ros.cpp:572
RCLCPP_COMPONENTS_REGISTER_NODE(hlpm::PathTranslatorROS)       // path_translator_ros.cpp:382
RCLCPP_COMPONENTS_REGISTER_NODE(hlpm::HLPStateManagerROS)      // hlp_state_manager_ros.cpp:345
```

### Test Macros

- `FRIEND_TEST(...)` — extensive GTest friend tests for white-box testing (17 in GPP ROS, 14 in PT ROS, 3 in SM ROS)
- `CONFIG_DIR` / `PARAMS_DIR` — CMake-defined test directory paths

---

## 5. Dependencies from `odp-common`

### From `odp_common/shared_types.hpp` (core types)

- **Structs**: `Point3D`, `Point3DGnss`, `Pose`, `Quaternion`, `MissionItem`, `MissionPlan`, `MiddlewareCheck`, `CoreCheck`
- **Enums**: `StateID`, `EventID`, `ModuleID`, `ComponentID`, `UavRole`, `MessageType`, `EndpointRole`, `CheckType`, `MissionStatus`, `WaypointType`, `MavCmd`, `MavFrame`, `CoordinateFrame`

### From `odp_common/shared_functions.hpp` (utilities)

- **`common::from_eul::toQuat()`** — Euler→Quaternion (used in GPP utils for orientation from waypoint direction)
- **`common::from_quat::toEul()`** — Quaternion→Euler (used in path_translator for yaw extraction)
- **`common::from_string::toComponentID/toCheckType/toMessageType/toEndpointRole()`** — YAML config parsing in state manager
- **`common::to_string::fromStateID()`** — Logging state names

### From `odp_common/state_manager/base_state_manager.hpp` (state machine base)

- **`BaseStateManager`** — base class for `HLPStateManager`
  - Provides: HSM initialization, `setupStateMachine()`, `handleSystemIntent()`, `loadChecks()` (YAML-based check loading)
  - Template method hooks for subclasses

### From `odp_common_ros/` (ROS messages/services)

- **Messages**: `ModuleState`, `SystemState`, `ComponentCheck`, `ComponentCheckArray`, `ApproachParams`, `MissionItem`, `MissionItemArray`, `Waypoint`
- **Services**: `MissionPause`, `MissionContinue`, `MissionPull`

---

## 6. Information Flow to MAVROS

### Inbound: MAVROS → HLPM

| Topic | Message Type | Consumed By | Purpose |
|---|---|---|---|
| `/mavros/local_position/pose` | `geometry_msgs/PoseStamped` | GPP ROS + PT ROS | Current local ENU position |
| `/mavros/home_position/home` | `mavros_msgs/HomePosition` | GPP ROS | GNSS origin for LocalCartesian projection |
| `/mavros/global_position/global` | (referenced in state checks) | HSM middleware check | GPS availability verification |

### Outbound: HLPM → MAVROS

| Topic | Message Type | Published By | Rate | Purpose |
|---|---|---|---|---|
| `/mavros/setpoint_raw/local` | `mavros_msgs/PositionTarget` | PT ROS | **30 Hz** | Velocity/position setpoint sent to FCU |

### PositionTarget Details

- Uses `FRAME_LOCAL_NED` coordinate frame
- **Velocity mode** (default): sends `velocity.x/y/z`, ignores position/acceleration
- **Position mode**: sends `position.x/y/z`, ignores velocity/acceleration
- Always sets `yaw`, ignores `yaw_rate`
- Uses `type_mask` bit flags to tell MAVROS which fields to use

### Complete Data Flow Pipeline

```
                    ┌─────────────────────────────────────────────────┐
                    │                    MAVROS                        │
                    │  /mavros/local_position/pose (PoseStamped)      │
                    │  /mavros/home_position/home  (HomePosition)     │
                    └──────┬──────────────────┬───────────────────────┘
                           │                  │
                           ▼                  ▼
                 ┌──────────────────────────────────┐
                 │     GlobalPathPlanningROS         │
                 │  • Receives ego pose + home pos   │
                 │  • Receives mission plan           │
                 │  • Receives target point + params  │
                 │  • Calls calculatePath()           │
                 │  • Strategy: Flightplan OR Terminal │
                 └──────────────┬───────────────────┘
                                │
                   /hlpm/global_path_planning/global_path (PoseArray)
                                │
                                ▼
                 ┌──────────────────────────────────┐
                 │       PathTranslatorROS           │
                 │  • Background thread @ 30 Hz      │
                 │  • Follows waypoints sequentially  │
                 │  • Generates velocity OR position  │
                 │    setpoint per waypoint            │
                 └──────────────┬───────────────────┘
                                │
                   /mavros/setpoint_raw/local (PositionTarget)
                                │
                                ▼
                    ┌───────────────────────────────────┐
                    │  MAVROS → FCU (MAVLink             │
                    │  SET_POSITION_TARGET_LOCAL_NED)     │
                    └───────────────────────────────────┘

  State Management (aside):
    HLPStateManagerROS orchestrates transitions via:
    • Subscribes: smm/system_state_manager/state/intent (SystemState)
    • Subscribes: hlpm/component/check (ComponentCheckArray)
    • Publishes:  hlpm/hlp_state_manager/module_state (ModuleState)
```

---

## 7. State Machine Flow

```
state_initialize → state_disable → state_standby → state_premission
                                                        ↓
                                                   state_mission (parent)
                                                    ├── substate_way_point (entry)
                                                    └── substate_terminal_guidance
                                                        ↓
                                                   state_postmission → state_disable (cycle)

Any state → state_emergency → state_disable (recovery)
```

Each transition requires:

- **Middleware checks**: ROS topic connections verified (subscriber/publisher alive)
- **Core checks**: Logic conditions met (e.g., mission plan received, home position available)
- Filtered by **UAV role** (striker vs. scout)

---

## 8. How-To: Extending HLPM

### 8.1 How to Add a New Publisher (Publish a New Topic)

**Example:** Publish a diagnostics topic `/hlpm/path_translator/diagnostics` of type `std_msgs/Float64`.

**Step 1: Declare publisher in the ROS header** (`path_translator_ros.hpp`):

```cpp
#include <std_msgs/msg/float64.hpp>

class PathTranslatorROS : public rclcpp::Node {
  // ... existing members ...
  rclcpp::Publisher<std_msgs::msg::Float64>::SharedPtr diagnostics_pub_;
};
```

**Step 2: Add topic name parameter** in `config/path_translator_params.yaml`:

```yaml
path_translator_component:
  ros__parameters:
    topics:
      # ... existing topics ...
      diagnostics_pub: "/hlpm/path_translator/diagnostics"
```

**Step 3: Create publisher in constructor** (`path_translator_ros.cpp`):

```cpp
// Declare parameter
this->declare_parameter<std::string>("topics.diagnostics_pub",
    "/hlpm/path_translator/diagnostics");

// Create publisher
diagnostics_pub_ = this->create_publisher<std_msgs::msg::Float64>(
    this->get_parameter("topics.diagnostics_pub").as_string(),
    rclcpp::QoS(10));
```

**Step 4: Publish in appropriate callback or loop**:

```cpp
void PathTranslatorROS::update() {
  // ... existing logic ...
  auto msg = std_msgs::msg::Float64();
  msg.data = some_diagnostic_value;
  diagnostics_pub_->publish(msg);
}
```

**Step 5 (optional): Register as middleware check** in `config/state_checks.yaml` if this publisher is required for a state transition:

```yaml
state_disable:
  hlpm:
    path_translator:
      - name: "pt_diagnostics_pub"
        role: "publisher"
        type: "diagnostics"
        topic: "/hlpm/path_translator/diagnostics"
```

---

### 8.2 How to Subscribe to a New Topic

**Example:** Subscribe to `/sensors/lidar/scan` of type `sensor_msgs/LaserScan` in GPP ROS.

**Step 1: Declare subscriber in the ROS header** (`global_path_planning_ros.hpp`):

```cpp
#include <sensor_msgs/msg/laser_scan.hpp>

class GlobalPathPlanningROS : public rclcpp::Node {
  rclcpp::Subscription<sensor_msgs::msg::LaserScan>::SharedPtr lidar_sub_;
  void lidarCallback(const sensor_msgs::msg::LaserScan::SharedPtr msg);
};
```

**Step 2: Add topic name parameter** in `config/global_path_planning_params.yaml`:

```yaml
global_path_planning_component:
  ros__parameters:
    topics:
      lidar_sub: "/sensors/lidar/scan"
```

**Step 3: Create subscription in constructor** (`global_path_planning_ros.cpp`):

```cpp
this->declare_parameter<std::string>("topics.lidar_sub", "/sensors/lidar/scan");

lidar_sub_ = this->create_subscription<sensor_msgs::msg::LaserScan>(
    this->get_parameter("topics.lidar_sub").as_string(),
    rclcpp::QoS(10),
    std::bind(&GlobalPathPlanningROS::lidarCallback, this, std::placeholders::_1));
```

**Step 4: Implement callback**:

```cpp
void GlobalPathPlanningROS::lidarCallback(
    const sensor_msgs::msg::LaserScan::SharedPtr msg) {
  // Convert ROS msg → core type if needed, then pass to core object
  // gpp_->setScanData(...);
}
```

**Step 5 (optional): Register as middleware check** if needed for state transitions.

---

### 8.3 How to Create a New Service

**Example:** Add a service `/hlpm/global_path_planning/mission/reset` using a new `MissionReset.srv`.

**Step 1: Define service type** (in `odp_common_ros/srv/MissionReset.srv`):

```
---
bool success
string message
```

**Step 2: Add to `odp_common_ros/CMakeLists.txt`**:

```cmake
rosidl_generate_interfaces(${PROJECT_NAME}
  # ... existing ...
  "srv/MissionReset.srv"
)
```

**Step 3: Declare service in the ROS header** (`global_path_planning_ros.hpp`):

```cpp
#include <odp_common_ros/srv/mission_reset.hpp>

class GlobalPathPlanningROS : public rclcpp::Node {
  rclcpp::Service<odp_common_ros::srv::MissionReset>::SharedPtr mission_reset_srv_;
  void missionResetCallback(
      const std::shared_ptr<odp_common_ros::srv::MissionReset::Request> request,
      std::shared_ptr<odp_common_ros::srv::MissionReset::Response> response);
};
```

**Step 4: Create service in constructor** (`global_path_planning_ros.cpp`):

```cpp
mission_reset_srv_ = this->create_service<odp_common_ros::srv::MissionReset>(
    "/hlpm/global_path_planning/mission/reset",
    std::bind(&GlobalPathPlanningROS::missionResetCallback, this,
              std::placeholders::_1, std::placeholders::_2));
```

**Step 5: Implement handler**:

```cpp
void GlobalPathPlanningROS::missionResetCallback(
    const std::shared_ptr<odp_common_ros::srv::MissionReset::Request> request,
    std::shared_ptr<odp_common_ros::srv::MissionReset::Response> response) {
  gpp_->reset();   // delegate to core logic
  response->success = true;
  response->message = "Mission reset successfully";
}
```

---

### 8.4 How to Create a New HLPM Component (Full Service)

**Follow the existing pattern to add e.g. an `obstacle_avoidance` service:**

#### 1. Create core package (`cpp/core/obstacle_avoidance/`)

```
obstacle_avoidance/
├── CMakeLists.txt
├── package.xml
├── include/obstacle_avoidance/
│   └── obstacle_avoidance.hpp
└── src/
    └── obstacle_avoidance.cpp
```

**Header (`obstacle_avoidance.hpp`):**

```cpp
#pragma once
#include <odp_common/shared_types.hpp>
#include <vector>

namespace hlpm {

class ObstacleAvoidance {
public:
  ObstacleAvoidance();
  ~ObstacleAvoidance();

  // Delete copy/move (follow existing pattern)
  ObstacleAvoidance(const ObstacleAvoidance&) = delete;
  ObstacleAvoidance& operator=(const ObstacleAvoidance&) = delete;
  ObstacleAvoidance(ObstacleAvoidance&&) = delete;
  ObstacleAvoidance& operator=(ObstacleAvoidance&&) = delete;

  void setEgoPose(const common::Pose& pose);
  void setObstacles(const std::vector<common::Point3D>& obstacles);
  std::vector<common::Point3D> getAvoidancePath() const;

private:
  common::Pose ego_pose_;
  std::vector<common::Point3D> obstacles_;
};

}  // namespace hlpm
```

**CMakeLists.txt:**

```cmake
cmake_minimum_required(VERSION 3.14)
project(obstacle_avoidance)

find_package(ament_cmake REQUIRED)
find_package(odp_common REQUIRED)
find_package(Eigen3 REQUIRED)

add_library(${PROJECT_NAME} STATIC
  src/obstacle_avoidance.cpp
)
target_include_directories(${PROJECT_NAME} PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>
)
target_link_libraries(${PROJECT_NAME} PUBLIC odp_common::odp_common Eigen3::Eigen)

ament_export_targets(${PROJECT_NAME}Targets HAS_LIBRARY_TARGET)
install(TARGETS ${PROJECT_NAME} EXPORT ${PROJECT_NAME}Targets)
install(DIRECTORY include/ DESTINATION include)
ament_package()
```

#### 2. Create ROS wrapper (`cpp/ros/obstacle_avoidance_ros/`)

```
obstacle_avoidance_ros/
├── CMakeLists.txt
├── package.xml
├── include/obstacle_avoidance_ros/
│   └── obstacle_avoidance_ros.hpp
├── src/
│   └── obstacle_avoidance_ros.cpp
├── config/
│   ├── obstacle_avoidance_params.yaml
│   └── obstacle_avoidance_qos.yaml
└── launch/
    └── obstacle_avoidance.launch.py
```

**Header (`obstacle_avoidance_ros.hpp`):**

```cpp
#pragma once
#include "obstacle_avoidance/obstacle_avoidance.hpp"
#include <rclcpp/rclcpp.hpp>
#include <geometry_msgs/msg/pose_stamped.hpp>
#include <geometry_msgs/msg/pose_array.hpp>
#include <sensor_msgs/msg/point_cloud2.hpp>

namespace hlpm {

class ObstacleAvoidanceROS : public rclcpp::Node {
public:
  explicit ObstacleAvoidanceROS(const rclcpp::NodeOptions& options);

private:
  std::unique_ptr<ObstacleAvoidance> oa_;

  // Subscribers
  rclcpp::Subscription<geometry_msgs::msg::PoseStamped>::SharedPtr ego_pose_sub_;
  rclcpp::Subscription<sensor_msgs::msg::PointCloud2>::SharedPtr pointcloud_sub_;

  // Publishers
  rclcpp::Publisher<geometry_msgs::msg::PoseArray>::SharedPtr avoidance_path_pub_;

  // Callbacks
  void egoPoseCallback(const geometry_msgs::msg::PoseStamped::SharedPtr msg);
  void pointcloudCallback(const sensor_msgs::msg::PointCloud2::SharedPtr msg);
};

}  // namespace hlpm
```

**Source (`obstacle_avoidance_ros.cpp`):**

```cpp
#include "obstacle_avoidance_ros/obstacle_avoidance_ros.hpp"
#include <rclcpp_components/register_node_macro.hpp>

namespace hlpm {

ObstacleAvoidanceROS::ObstacleAvoidanceROS(const rclcpp::NodeOptions& options)
  : Node("obstacle_avoidance_component", options)
  , oa_(std::make_unique<ObstacleAvoidance>()) {

  // Declare parameters
  this->declare_parameter<std::string>("topics.ego_pose_sub",
      "/mavros/local_position/pose");
  this->declare_parameter<std::string>("topics.pointcloud_sub",
      "/sensors/lidar/points");
  this->declare_parameter<std::string>("topics.avoidance_path_pub",
      "/hlpm/obstacle_avoidance/path");

  // Create subscribers
  ego_pose_sub_ = this->create_subscription<geometry_msgs::msg::PoseStamped>(
      this->get_parameter("topics.ego_pose_sub").as_string(),
      rclcpp::QoS(10),
      std::bind(&ObstacleAvoidanceROS::egoPoseCallback, this,
                std::placeholders::_1));

  pointcloud_sub_ = this->create_subscription<sensor_msgs::msg::PointCloud2>(
      this->get_parameter("topics.pointcloud_sub").as_string(),
      rclcpp::QoS(10),
      std::bind(&ObstacleAvoidanceROS::pointcloudCallback, this,
                std::placeholders::_1));

  // Create publishers
  avoidance_path_pub_ = this->create_publisher<geometry_msgs::msg::PoseArray>(
      this->get_parameter("topics.avoidance_path_pub").as_string(),
      rclcpp::QoS(10));
}

void ObstacleAvoidanceROS::egoPoseCallback(
    const geometry_msgs::msg::PoseStamped::SharedPtr msg) {
  common::Pose pose;
  pose.position = {static_cast<float>(msg->pose.position.x),
                   static_cast<float>(msg->pose.position.y),
                   static_cast<float>(msg->pose.position.z)};
  oa_->setEgoPose(pose);
}

void ObstacleAvoidanceROS::pointcloudCallback(
    const sensor_msgs::msg::PointCloud2::SharedPtr msg) {
  // Convert pointcloud → obstacle points, pass to core
  // oa_->setObstacles(obstacles);
  // Publish avoidance path
}

}  // namespace hlpm

RCLCPP_COMPONENTS_REGISTER_NODE(hlpm::ObstacleAvoidanceROS)
```

#### 3. Register in bringup (`hlpm_bringup_ros/launch/hlpm.launch.py`)

```python
ComposableNode(
    package='obstacle_avoidance_ros',
    plugin='hlpm::ObstacleAvoidanceROS',
    name='obstacle_avoidance_component',
    parameters=[base_params, param_overrides],
)
```

#### 4. Add state checks (`hlp_state_manager/config/state_checks.yaml`)

```yaml
state_disable:
  hlpm:
    obstacle_avoidance:
      - name: "oa_ego_pose_sub"
        role: "subscriber"
        type: "mavros_pose"
        topic: "/mavros/local_position/pose"
      - name: "oa_path_pub"
        role: "publisher"
        type: "avoidance_path"
        topic: "/hlpm/obstacle_avoidance/path"
```

---

## 9. Key Conventions Summary

| Convention | Details |
|---|---|
| Namespace | `hlpm::` for all classes |
| Core classes | Non-copyable, non-movable, no ROS dependency |
| ROS classes | Inherit `rclcpp::Node`, own `std::unique_ptr<CoreClass>` |
| Topic names | Always parameterized (never hardcoded) |
| Component registration | `RCLCPP_COMPONENTS_REGISTER_NODE()` in every ROS source |
| Composable container | Multi-threaded (`component_container_mt`) |
| Config files | Separate `*_params.yaml` and `*_qos_settings.yaml` |
| State checks | YAML-driven middleware/core checks per state |
| Coordinate frames | Core uses ENU; MAVROS output uses LOCAL_NED frame |

---

## 10. ROS2 Topics Quick Reference

### All Subscriptions

| Node | Topic | Type |
|---|---|---|
| GPP ROS | `/mavros/local_position/pose` | `geometry_msgs/PoseStamped` |
| GPP ROS | `/mavros/home_position/home` | `mavros_msgs/HomePosition` |
| GPP ROS | `/hlpm/hlp_state_manager/module_state` | `odp_common_ros/ModuleState` |
| GPP ROS | `/platform/uav_interface/terminal/approach_params` | `odp_common_ros/ApproachParams` |
| GPP ROS | `/platform/uav_interface/terminal/target_point` | `geometry_msgs/Point` |
| GPP ROS | `/platform/uav_interface/mission/plan` | `odp_common_ros/MissionItemArray` |
| PT ROS | `/mavros/local_position/pose` | `geometry_msgs/PoseStamped` |
| PT ROS | `/hlpm/global_path_planning/global_path` | `geometry_msgs/PoseArray` |
| PT ROS | `/hlpm/hlp_state_manager/module_state` | `odp_common_ros/ModuleState` |
| HSM ROS | `smm/system_state_manager/state/intent` | `odp_common_ros/SystemState` |
| HSM ROS | `hlpm/component/check` | `odp_common_ros/ComponentCheckArray` |
| HSM ROS | `/platform/uav_interface/mission/plan` | `odp_common_ros/MissionItemArray` |

### All Publishers

| Node | Topic | Type |
|---|---|---|
| GPP ROS | `/hlpm/global_path_planning/global_path` | `geometry_msgs/PoseArray` |
| GPP ROS | `/hlpm/component/check` | `odp_common_ros/ComponentCheckArray` |
| PT ROS | `/mavros/setpoint_raw/local` | `mavros_msgs/PositionTarget` |
| PT ROS | `/hlpm/component/check` | `odp_common_ros/ComponentCheckArray` |
| HSM ROS | `hlpm/hlp_state_manager/module_state` | `odp_common_ros/ModuleState` |

### Services

| Node | Service | Type |
|---|---|---|
| GPP ROS | `/hlpm/global_path_planning/mission/pause` | `odp_common_ros/srv/MissionPause` |
| GPP ROS | `/hlpm/global_path_planning/mission/continue` | `odp_common_ros/srv/MissionContinue` |
| GPP ROS | `/hlpm/global_path_planning/mission/pull` | `odp_common_ros/srv/MissionPull` |
