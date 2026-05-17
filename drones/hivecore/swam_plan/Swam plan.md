
Below is a cleaner, production-oriented version of the solution.

---

# Problem statement

We are writing mission code for a drone swarm.

The mission planner receives:

1. A list of global setpoints:
    

```text
latitude, longitude, altitude
```

These represent the **center of the swarm** at each mission step.

2. A list of drone offsets that define the swarm topology:
    

```text
x, y, z
```

These offsets are expressed in the **local coordinate frame of the swarm**.

For every global setpoint, we need to compute the global target position of every drone in the swarm.

The orientation of the swarm frame is determined dynamically. For a given setpoint, the swarm should “face” the next setpoint. In other words, the local forward direction of the swarm frame should align with the direction of travel from the current setpoint to the next setpoint.

The required output for each drone is again:

```text
latitude, longitude, altitude
```

---

# Recommended solution

Use a combination of:

```text
GeographicLib / geodesy  -> for latitude/longitude/altitude conversion
tf2                      -> for coordinate frame transforms and rotations
```

Do **not** use raw latitude and longitude directly as Cartesian `x/y` coordinates. Latitude and longitude are angular quantities on the Earth’s surface, not linear metric coordinates.

The correct flow is:

```text
Global geodetic coordinates
        |
        | lat/lon/alt -> local Cartesian
        v
Mission ENU frame
        |
        | tf2 transform
        v
Swarm local frame
        |
        | per-drone offsets
        v
Drone target positions in mission ENU
        |
        | local Cartesian -> lat/lon/alt
        v
Drone global target positions
```

---

# Coordinate system choice

For mission planning in ROS 2, I recommend using an **ENU** frame internally.

```text
ENU frame:
x = East
y = North
z = Up
```

This works well with ROS-style Cartesian math and `tf2`.

However, many drone flight controllers and MAVLink-based systems commonly use **NED**:

```text
NED frame:
x = North
y = East
z = Down
```

So the mission code should clearly separate:

```text
Mission planning frame: ENU
Flight-controller interface frame: NED, if required
```

A production system should not mix these implicitly. Every conversion should happen in a named function with tests.

---

# Frame definitions

A clean frame design could be:

```text
earth / WGS84
    |
    | GeographicLib LocalCartesian origin
    v
mission_enu
    |
    | translation = current swarm center in ENU
    | rotation    = yaw toward next setpoint
    v
swarm_frame
    |
    | drone topology offsets
    v
drone_i_target
```

Where:

```text
mission_enu
```

is a local Cartesian frame initialized at a known mission origin.

```text
swarm_frame
```

is a moving frame whose origin is the current swarm center.

The `x` axis of `swarm_frame` is chosen as the forward direction of the swarm.

For example:

```text
swarm_frame:
x = forward, toward next setpoint
y = left
z = up
```

This matches the common ROS body-frame convention.

---

# Dependency example

For ROS 2 C++:

```cmake
find_package(rclcpp REQUIRED)
find_package(tf2 REQUIRED)
find_package(tf2_ros REQUIRED)
find_package(tf2_geometry_msgs REQUIRED)
find_package(geometry_msgs REQUIRED)
find_package(GeographicLib REQUIRED)
```

Example `package.xml` dependencies:

```xml
<depend>rclcpp</depend>
<depend>tf2</depend>
<depend>tf2_ros</depend>
<depend>tf2_geometry_msgs</depend>
<depend>geometry_msgs</depend>
```

Depending on your platform, GeographicLib may be installed as a system dependency rather than a ROS package.

---

# Data models

A production project should avoid passing around raw tuples everywhere. Use explicit types.

```cpp
#pragma once

#include <string>
#include <vector>

struct GeoPoint
{
  double latitude_deg;
  double longitude_deg;
  double altitude_m;
};

struct EnuPoint
{
  double east_m;
  double north_m;
  double up_m;
};

struct SwarmOffset
{
  std::string drone_id;

  // Offset in swarm_frame.
  // x = forward
  // y = left
  // z = up
  double x_m;
  double y_m;
  double z_m;
};

struct DroneGlobalTarget
{
  std::string drone_id;
  GeoPoint target;
};
```

---

# Geodetic conversion using GeographicLib

This class owns the conversion between WGS84 geodetic coordinates and the local mission ENU frame.

```cpp
#pragma once

#include <GeographicLib/LocalCartesian.hpp>

class GeoConverter
{
public:
  explicit GeoConverter(const GeoPoint & origin)
  : origin_(origin),
    local_cartesian_(
      origin.latitude_deg,
      origin.longitude_deg,
      origin.altitude_m)
  {
  }

  EnuPoint toEnu(const GeoPoint & geo) const
  {
    double east = 0.0;
    double north = 0.0;
    double up = 0.0;

    local_cartesian_.Forward(
      geo.latitude_deg,
      geo.longitude_deg,
      geo.altitude_m,
      east,
      north,
      up);

    return EnuPoint{
      .east_m = east,
      .north_m = north,
      .up_m = up
    };
  }

  GeoPoint toGeo(const EnuPoint & enu) const
  {
    double latitude = 0.0;
    double longitude = 0.0;
    double altitude = 0.0;

    local_cartesian_.Reverse(
      enu.east_m,
      enu.north_m,
      enu.up_m,
      latitude,
      longitude,
      altitude);

    return GeoPoint{
      .latitude_deg = latitude,
      .longitude_deg = longitude,
      .altitude_m = altitude
    };
  }

private:
  GeoPoint origin_;
  GeographicLib::LocalCartesian local_cartesian_;
};
```

In production, the origin should be chosen carefully. Common choices are:

```text
mission home position
takeoff position
first mission setpoint
fixed surveyed reference point
```

For a swarm mission, the first swarm-center setpoint is often acceptable, but for repeatable missions a fixed known origin is better.

---

# Computing swarm yaw from current and next setpoint

Assume we use `mission_enu`:

```text
x = East
y = North
z = Up
```

Assume we define `swarm_frame` as:

```text
x = forward
y = left
z = up
```

We want the swarm-frame `x` axis to point from the current setpoint to the next setpoint.

In ENU, that direction is:

```text
dx = next.east  - current.east
dy = next.north - current.north
```

The yaw angle from ENU `x` axis is:

```cpp
yaw = atan2(dy, dx)
```

This means:

```text
yaw = 0        -> swarm forward points East
yaw = pi / 2   -> swarm forward points North
```

Code:

```cpp
#include <cmath>
#include <stdexcept>

double computeYawInMissionEnu(
  const EnuPoint & current,
  const EnuPoint & next)
{
  const double dx = next.east_m - current.east_m;
  const double dy = next.north_m - current.north_m;

  constexpr double kMinDistanceMeters = 0.05;

  const double distance_xy = std::hypot(dx, dy);

  if (distance_xy < kMinDistanceMeters) {
    throw std::runtime_error(
      "Cannot compute swarm yaw: current and next setpoints are too close");
  }

  return std::atan2(dy, dx);
}
```

This is an important production detail. If two consecutive setpoints are identical or extremely close, heading becomes undefined. You should either:

```text
reuse the previous valid yaw
skip the duplicate setpoint
use a mission-defined default yaw
raise a mission validation error
```

For real drone missions, I would prefer validating the mission before execution and rejecting ambiguous setpoints.

---

# Building the tf2 transform

This transform places the `swarm_frame` inside `mission_enu`.

```cpp
#include <geometry_msgs/msg/transform_stamped.hpp>
#include <tf2/LinearMath/Quaternion.h>
#include <tf2_geometry_msgs/tf2_geometry_msgs.hpp>

geometry_msgs::msg::TransformStamped makeMissionToSwarmTransform(
  const EnuPoint & swarm_center_enu,
  double yaw_rad,
  const rclcpp::Time & stamp)
{
  geometry_msgs::msg::TransformStamped transform;

  transform.header.stamp = stamp;
  transform.header.frame_id = "mission_enu";
  transform.child_frame_id = "swarm_frame";

  transform.transform.translation.x = swarm_center_enu.east_m;
  transform.transform.translation.y = swarm_center_enu.north_m;
  transform.transform.translation.z = swarm_center_enu.up_m;

  tf2::Quaternion q;
  q.setRPY(0.0, 0.0, yaw_rad);
  q.normalize();

  transform.transform.rotation = tf2::toMsg(q);

  return transform;
}
```

This transform means:

```text
A point expressed in swarm_frame can be transformed into mission_enu.
```

For example, a drone offset:

```text
x = 10 m forward
y = 3 m left
z = 0 m
```

will be rotated according to the mission heading and translated to the swarm center.

---

# Transforming drone offsets

```cpp
#include <geometry_msgs/msg/point_stamped.hpp>
#include <tf2_geometry_msgs/tf2_geometry_msgs.hpp>

EnuPoint transformDroneOffsetToMissionEnu(
  const SwarmOffset & offset,
  const geometry_msgs::msg::TransformStamped & mission_to_swarm_tf)
{
  geometry_msgs::msg::PointStamped point_in_swarm;
  point_in_swarm.header.stamp = mission_to_swarm_tf.header.stamp;
  point_in_swarm.header.frame_id = "swarm_frame";

  point_in_swarm.point.x = offset.x_m;
  point_in_swarm.point.y = offset.y_m;
  point_in_swarm.point.z = offset.z_m;

  geometry_msgs::msg::PointStamped point_in_mission;

  tf2::doTransform(
    point_in_swarm,
    point_in_mission,
    mission_to_swarm_tf);

  return EnuPoint{
    .east_m = point_in_mission.point.x,
    .north_m = point_in_mission.point.y,
    .up_m = point_in_mission.point.z
  };
}
```

---

# Full planner function

This function takes:

```text
current swarm center in lat/lon/alt
next swarm center in lat/lon/alt
list of drone offsets in swarm_frame
```

and returns:

```text
global lat/lon/alt target for each drone
```

```cpp
#include <vector>

std::vector<DroneGlobalTarget> computeDroneTargetsForSetpoint(
  const GeoConverter & geo_converter,
  const GeoPoint & current_swarm_center_geo,
  const GeoPoint & next_swarm_center_geo,
  const std::vector<SwarmOffset> & swarm_offsets,
  const rclcpp::Time & stamp)
{
  const EnuPoint current_center_enu =
    geo_converter.toEnu(current_swarm_center_geo);

  const EnuPoint next_center_enu =
    geo_converter.toEnu(next_swarm_center_geo);

  const double yaw_rad =
    computeYawInMissionEnu(current_center_enu, next_center_enu);

  const auto mission_to_swarm_tf =
    makeMissionToSwarmTransform(
      current_center_enu,
      yaw_rad,
      stamp);

  std::vector<DroneGlobalTarget> targets;
  targets.reserve(swarm_offsets.size());

  for (const auto & offset : swarm_offsets) {
    const EnuPoint drone_target_enu =
      transformDroneOffsetToMissionEnu(offset, mission_to_swarm_tf);

    const GeoPoint drone_target_geo =
      geo_converter.toGeo(drone_target_enu);

    targets.push_back(
      DroneGlobalTarget{
        .drone_id = offset.drone_id,
        .target = drone_target_geo
      });
  }

  return targets;
}
```

---

# Example usage inside a ROS 2 node

```cpp
#include <rclcpp/rclcpp.hpp>

class SwarmMissionPlannerNode : public rclcpp::Node
{
public:
  SwarmMissionPlannerNode()
  : Node("swarm_mission_planner"),
    geo_converter_(
      GeoPoint{
        .latitude_deg = 60.1699,
        .longitude_deg = 24.9384,
        .altitude_m = 20.0
      })
  {
  }

  void example()
  {
    const GeoPoint current_center{
      .latitude_deg = 60.1699,
      .longitude_deg = 24.9384,
      .altitude_m = 50.0
    };

    const GeoPoint next_center{
      .latitude_deg = 60.1703,
      .longitude_deg = 24.9392,
      .altitude_m = 50.0
    };

    const std::vector<SwarmOffset> topology{
      SwarmOffset{
        .drone_id = "drone_1",
        .x_m = 0.0,
        .y_m = 0.0,
        .z_m = 0.0
      },
      SwarmOffset{
        .drone_id = "drone_2",
        .x_m = -5.0,
        .y_m = 5.0,
        .z_m = 0.0
      },
      SwarmOffset{
        .drone_id = "drone_3",
        .x_m = -5.0,
        .y_m = -5.0,
        .z_m = 0.0
      }
    };

    const auto targets =
      computeDroneTargetsForSetpoint(
        geo_converter_,
        current_center,
        next_center,
        topology,
        this->now());

    for (const auto & target : targets) {
      RCLCPP_INFO(
        this->get_logger(),
        "Drone %s target: lat=%.8f lon=%.8f alt=%.2f",
        target.drone_id.c_str(),
        target.target.latitude_deg,
        target.target.longitude_deg,
        target.target.altitude_m);
    }
  }

private:
  GeoConverter geo_converter_;
};
```

---

# ENU to NED conversion

If your flight controller expects NED, convert only at the interface boundary.

Given ENU:

```text
x_enu = East
y_enu = North
z_enu = Up
```

NED is:

```text
x_ned = North = y_enu
y_ned = East  = x_enu
z_ned = Down  = -z_enu
```

Code:

```cpp
struct NedPoint
{
  double north_m;
  double east_m;
  double down_m;
};

NedPoint enuToNed(const EnuPoint & enu)
{
  return NedPoint{
    .north_m = enu.north_m,
    .east_m = enu.east_m,
    .down_m = -enu.up_m
  };
}

EnuPoint nedToEnu(const NedPoint & ned)
{
  return EnuPoint{
    .east_m = ned.east_m,
    .north_m = ned.north_m,
    .up_m = -ned.down_m
  };
}
```

For yaw, be very careful.

In the ENU swarm-frame definition above:

```text
yaw_enu = atan2(north_delta, east_delta)
```

For NED, where yaw is commonly measured clockwise from North:

```text
yaw_ned = atan2(east_delta, north_delta)
```

A useful relation is:

```cpp
double yawEnuToYawNed(double yaw_enu)
{
  return M_PI_2 - yaw_enu;
}
```

Normalize after conversion:

```cpp
double normalizeAngleRad(double angle)
{
  while (angle > M_PI) {
    angle -= 2.0 * M_PI;
  }

  while (angle < -M_PI) {
    angle += 2.0 * M_PI;
  }

  return angle;
}
```

Then:

```cpp
double yaw_ned = normalizeAngleRad(yawEnuToYawNed(yaw_enu));
```

---

# Should tf2 be used here?

Yes, but with a clear boundary.

Use `tf2` for:

```text
rotating topology offsets
translating local swarm-frame points into mission frame
maintaining named frame relationships
debugging frame relationships in ROS tools
publishing static or dynamic transforms if useful
```

Do not use `tf2` for:

```text
raw latitude/longitude math
WGS84 geodesic conversion
earth curvature handling
map projection
```

That is the job of `GeographicLib`, `geodesy`, or another well-defined geospatial library.

---

# Production recommendations

For a production drone system, I would enforce the following standards.

First, define frame conventions in one central document and mirror them in code comments:

```text
mission_enu:
  x = East
  y = North
  z = Up

swarm_frame:
  x = Forward
  y = Left
  z = Up

flight_controller_ned:
  x = North
  y = East
  z = Down
```

Second, validate missions before execution:

```text
No duplicate consecutive waypoints unless yaw is explicitly provided
No invalid latitude/longitude values
No impossible altitude values
No unsafe drone separation distances
No topology offsets that violate vehicle constraints
No frame ambiguity
```

Third, never silently guess heading when the current and next setpoints are identical. That should be rejected or explicitly handled.

Fourth, isolate coordinate conversion code from mission logic. Keep this kind of code in a module such as:

```text
swarm_navigation/geo_converter.hpp
swarm_navigation/frame_conversions.hpp
swarm_navigation/swarm_target_solver.hpp
```

Fifth, write unit tests for:

```text
lat/lon/alt -> ENU -> lat/lon/alt roundtrip
ENU -> NED -> ENU roundtrip
zero yaw transform
90-degree yaw transform
duplicate waypoint handling
known swarm topology outputs
```

A simple but important test:

```cpp
// If swarm yaw is zero, swarm x should align with ENU East.
SwarmOffset offset{
  .drone_id = "test",
  .x_m = 10.0,
  .y_m = 0.0,
  .z_m = 0.0
};

// Expected result:
// east increases by 10 m
// north unchanged
// up unchanged
```

Another test:

```cpp
// If swarm yaw is +90 degrees, swarm x should align with ENU North.
SwarmOffset offset{
  .drone_id = "test",
  .x_m = 10.0,
  .y_m = 0.0,
  .z_m = 0.0
};

// Expected result:
// east unchanged
// north increases by 10 m
// up unchanged
```

---

# Final recommendation

For your use case, the right production-grade approach is:

```text
Use GeographicLib to convert lat/lon/alt into a local ENU mission frame.

Use tf2 to define the transform from mission_enu to swarm_frame.

Use tf2::doTransform to rotate and translate each drone’s local swarm offset into mission_enu.

Convert each resulting ENU target back to lat/lon/alt.

Convert ENU to NED only at the flight-controller boundary if the autopilot requires NED.
```

This keeps the system mathematically correct, testable, and maintainable.