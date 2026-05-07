# QGroundControl (QGC) for Drones

## What QGroundControl is

**QGroundControl (QGC)** is a ground control station (GCS) for PX4-based drones.  
It talks to the vehicle primarily over **MAVLink** and gives you operational visibility and control.

In your architecture:

- **PX4** = autopilot and low-level control
- **ROS 2 stack** = autonomy/offboard logic
- **QGC** = operator interface for mission setup, telemetry, safety checks, and tuning

---

## What we can do with QGC

With QGC, you can:

1. **Connect to vehicles** (USB, telemetry radio, UDP in SITL)
2. **Check flight readiness** (GPS lock, EKF status, battery, sensors, mode)
3. **Calibrate and configure** sensors, RC, flight modes, geofence, failsafes
4. **Plan missions** (waypoints, takeoff/land, survey patterns) and upload to PX4
5. **Monitor live telemetry** (position, attitude, battery, mode, link health)
6. **Tune PX4 parameters** and verify behavior changes
7. **Review logs** for post-flight debugging and tuning
8. **Flash/update firmware** and apply airframe setup (when needed)

---

## What QGC does not replace

QGC is not a full autonomy framework. It does **not** replace:

- your ROS 2 mission planner/swarm logic,
- custom offboard controllers,
- topic-level debugging tools (RViz2/Foxglove).

Use QGC for **operations + vehicle state visibility**; use ROS 2 tools for **algorithm/developer introspection**.

---

## Integration with PX4 + ROS 2 simulator topics

You already have PX4 ROS 2 topics from simulation.  
Use a **QGC-first workflow** for flight visualization, and keep ROS 2 visual tools as secondary.

### QGC-first visualization workflow

1. Start PX4 SITL + simulator (Gazebo, etc.).
2. Ensure PX4 is publishing MAVLink UDP endpoint(s).
3. Open QGC and connect to the SITL vehicle.
4. Use QGC Fly/Map/Widgets to watch:
   - current position and heading,
   - arming + mode transitions,
   - mission progress,
   - battery and link status.

This gives you a fast “is the drone flying correctly?” operational view.

---

## How to visualize flight from ROS 2 topics

When you need deeper debugging than QGC provides, use ROS 2 tools in parallel:

- **RViz2**: trajectory path, TF frames, pose/odometry overlays, markers
- **Foxglove / rqt_plot**: velocity/attitude/altitude trends over time

### Practical mapping (conceptual)

- PX4 ROS 2 `vehicle_odometry` / pose topics -> RViz2 Pose/Odometry display
- Position history -> Path trail in RViz2
- Attitude + rates + velocity -> Foxglove or plot tools
- QGC telemetry -> operator-level map + mission state + safety status

Use these together:

- **QGC answers**: “Is mission execution and vehicle state healthy?”
- **ROS 2 tools answer**: “Why is control behavior drifting/oscillating?”

---

## Common integration issues when visualizing simulated flight

1. **Frame mismatch (NED vs ENU)**  
   PX4 commonly uses NED conventions; many ROS tools assume ENU.  
   If not converted correctly, the trajectory appears mirrored/rotated.

2. **Namespace mix-ups in multi-drone SITL**  
   Wrong drone namespace causes one drone to appear as another or no data to render.

3. **Timestamp inconsistencies**  
   Clock/source mismatch can cause stale-looking or jittery visualization.

4. **Assuming QGC shows all ROS 2 internals**  
   QGC shows MAVLink-visible vehicle state, not all custom ROS 2 internal topics.

---

## Recommended usage pattern for your setup

For PX4 + ROS 2 simulator workflows:

1. Use **QGC first** to confirm arm/mode/mission/overall flight behavior.
2. Add **RViz2/Foxglove** when debugging trajectory tracking, frame transforms, estimator/control issues.
3. Keep role separation clear:
   - QGC for operations and mission supervision
   - ROS 2 tools for engineering/debug analysis

This gives you both quick operator visibility and deep technical diagnosis.
