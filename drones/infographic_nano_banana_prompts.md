# Nano Banana Pro Prompts: Drone Software Stack Concept Diagrams

Use these prompts directly with **Nano Banana Pro** to generate presentation-grade conceptual diagrams that emphasize **component connectivity**, **data links**, and **message semantics**.

## Shared style directive (prepend to every prompt)

```text
Create a clean, professional technical infographic for an engineering audience. Use a white background, muted enterprise color palette, consistent iconography, and crisp vector style. Prefer horizontal 16:9 slide composition. Use clear directional arrows with labeled links. Include a small legend for link type and message family. Keep text concise, readable at presentation distance, and avoid decorative clutter.
```

---

## Prompt 1 — End-to-end drone software stack (single drone)

```text
Design a conceptual architecture diagram titled “Drone Software Stack: End-to-End Data and Control Flow”.

Show these components as distinct layers:
1) Gazebo simulator (vehicle physics, sensors, actuators, world)
2) PX4 SITL autopilot (EKF2/state estimation, control loops, commander)
3) ROS 2 autonomy stack (mission planner, path planner, obstacle avoidance, behavior logic)
4) QGroundControl (operator UI)
5) Optional MAVLink router

Show directional links and label each with transport/protocol:
- Gazebo <-> PX4 SITL: simulator sensor/actuator integration link
- ROS 2 stack <-> PX4: DDS/uXRCE-DDS bridge path
- PX4 <-> QGC: MAVLink (UDP/TCP/serial)
- PX4 <-> MAVLink router <-> multiple consumers

Annotate message families on links:
- ROS 2 to PX4: OffboardControlMode, TrajectorySetpoint, VehicleCommand
- PX4 to ROS 2: vehicle_odometry, position, attitude, vehicle_status, battery
- PX4 to QGC: heartbeat, attitude, position, status, mission progress
- QGC to PX4: arm/disarm, mode change, mission commands, parameter ops

Add one callout: “Autonomy sends intent; PX4 performs estimation + low-level control”.
```

---

## Prompt 2 — SITL closed-loop runtime flow

```text
Create a runtime loop diagram titled “SITL Closed Loop: Planner to Physics to Feedback”.

Visualize this cyclic flow with numbered steps:
1) ROS 2 planner publishes intent setpoints
2) DDS bridge delivers intent to PX4
3) PX4 estimates state (EKF2) and computes actuator outputs
4) Gazebo updates vehicle physics and environment interaction
5) Gazebo produces simulated sensor data
6) PX4 updates state estimate
7) PX4 publishes odometry/status back to ROS 2 and telemetry to QGC

For each edge, label:
- Link type (DDS bridge, simulator integration, MAVLink)
- Example messages (TrajectorySetpoint, VehicleCommand, sensor stream, vehicle_odometry, heartbeat)

Add two side notes:
- “PX4 internal bus: uORB (inside PX4 only)”
- “QGC gives operator-level health and mission visibility”
```

---

## Prompt 3 — PX4 internal vs external interfaces

```text
Generate a split-view diagram titled “PX4 Interfaces: Internal uORB vs External Protocols”.

Left panel: PX4 internal modules
- Sensors input
- EKF2 estimator
- Controllers
- Commander/navigation
- Actuator outputs
Connect modules via internal bus labeled “uORB”.

Right panel: PX4 external interfaces
- ROS 2 autonomy via DDS/uXRCE-DDS bridge
- QGroundControl via MAVLink
- Optional companion apps/loggers via MAVLink router
- Gazebo SITL integration link

On each external interface, annotate representative message categories:
- Control intent, telemetry, health/status, mission/params, command/ack

Add a top banner:
“PX4 does not control from raw sensors directly; it controls from estimated state.”
```

---

## Prompt 4 — MAVLink topology and message routing

```text
Design a network topology diagram titled “MAVLink Communication Topology (Real and Sim)”.

Show one MAVLink source node (PX4 real FCU or PX4 SITL or synthetic bridge) feeding:
- QGroundControl
- Companion application
- Logger/analytics
through an optional MAVLink router node.

Label transport options on links:
- Serial UART/USB
- UDP
- TCP

Label MAVLink message families:
- Heartbeat/identity
- Pose/motion telemetry
- Health/status
- Command/control
- Mission/parameter protocol

Include explicit SYSID/COMPID labels and a warning callout:
“ID mismatches commonly break discovery and correct routing.”
```

---

## Prompt 5 — ROS 2 <-> MAVLink conversion map

```text
Create a bidirectional mapping infographic titled “ROS 2 and MAVLink Conversion Paths”.

Use two mirrored lanes:
Lane A: ROS 2 -> MAVLink (for QGC visibility without full PX4 SITL)
Lane B: MAVLink -> ROS 2 (for autonomy/analytics consumers)

For each lane, create mapping tables:
- Pose/Odometry <-> attitude/position telemetry
- Velocity topics <-> velocity telemetry
- Battery/status topics <-> health/status messages
- Armed/mode/nav state <-> heartbeat mode/state bits
- Command status <-> ack/status pathways

Add engineering constraint badges:
- Frame conventions (ENU vs NED)
- Unit consistency
- Timestamp monotonicity
- Stable publish rates
- Deterministic SYSID/COMPID
```

---

## Prompt 6 — Multi-drone scaling architecture

```text
Generate a multi-instance architecture diagram titled “Scaling to N Drones in SITL”.

Show three drones as example instances (drone_1, drone_2, drone_3), each with:
- Gazebo model
- PX4 SITL instance
- Namespaced ROS 2 I/O endpoints

Draw per-drone interfaces:
- /drone_i/fmu/in/trajectory_setpoint
- /drone_i/fmu/in/offboard_control_mode
- /drone_i/fmu/in/vehicle_command
- /drone_i/fmu/out/vehicle_odometry

Add identity and networking labels for each drone:
- namespace
- SYSID/COMPID
- simulator/MAVLink ports
- TF frame prefix

Add one central ROS 2 swarm coordinator and optional shared QGC/MAVLink routing view.
Emphasize isolation boundaries and anti-collision of IDs/topics/ports.
```

---

## Prompt 7 — Operations view (QGC-first) + engineering view (ROS 2 tools)

```text
Create a dual-audience diagram titled “QGC Operations View vs ROS 2 Engineering View”.

Left side (Operator / Mission Control):
- QGroundControl screens for map, mode, arming state, mission progress, battery, link health
- Data source via MAVLink from PX4

Right side (Developer / Debug):
- RViz2/Foxglove plots for trajectory, TF frames, attitude/rates, velocity trends
- Data source via ROS 2 topics

Center bridge:
- PX4 SITL as common truth source between operational monitoring and engineering introspection

Add caption:
“QGC answers mission health; ROS 2 tools answer root-cause behavior.”
```

---

## Prompt 8 — Failure modes and validation checkpoints

```text
Design a diagnostic flowchart infographic titled “Common Integration Failures and Validation Checks”.

Include failure clusters:
1) Missing/invalid heartbeat
2) ENU-NED frame mismatch
3) Wrong SYSID/COMPID or namespace collisions
4) Timestamp instability / stale updates
5) Missing global position or status fields

For each failure cluster, show:
- Observable symptom in QGC/ROS 2
- Likely root cause
- Validation check
- Corrective action

Append a compact checklist panel:
- heartbeat stable
- IDs consistent
- frame transform verified with known motion
- units/signs verified (altitude, yaw, velocity axes)
- command path and ack path validated
```

---

## Suggested presentation order

1. Prompt 1 (full stack context)  
2. Prompt 2 (runtime loop)  
3. Prompt 3 (PX4 interface boundaries)  
4. Prompt 4 (MAVLink topology)  
5. Prompt 5 (ROS 2 <-> MAVLink conversion)  
6. Prompt 6 (multi-drone scale-out)  
7. Prompt 7 (operations vs engineering views)  
8. Prompt 8 (failure/debug playbook)
