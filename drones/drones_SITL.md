# Drone SITL with PX4, Gazebo, and ROS 2

This document describes a simulation setup for **multiple drones** where:

- **PX4 autopilot** runs in **SITL** on a laptop
- **Gazebo** simulates the drone bodies, sensors, actuators, and world
- a **ROS 2 autonomy stack** plays the role of the **Jetson Orin Nano companion computer**

The autonomy stack is responsible for high-level planning and for sending **control intent** to PX4, while PX4 handles state estimation and low-level flight control.

The goal of this note is to explain:

- how the full simulated system is structured
- how it maps to the real drone architecture
- how **Gazebo, PX4, and the ROS 2 autonomy stack** are connected
- what communication protocols are involved
- how to think about the setup for **multi-drone simulation**

## What SITL means

**SITL** stands for **Software In The Loop**.

In this setup, PX4 does **not** run on a physical flight controller. Instead, it runs as a **Linux process on the laptop**. Gazebo simulates the drone body, sensors, motors, and environment.

So compared to a real drone:

| Real drone | Simulation |
|---|---|
| PX4 on flight controller hardware | PX4 SITL process on laptop |
| Real IMU, GPS, barometer, etc. | Gazebo simulated sensors |
| Real motors and real vehicle dynamics | Gazebo physics engine |
| Jetson autonomy stack | ROS 2 autonomy stack on laptop or another computer |

## System architecture

In your case, the full system looks like this:

```text
Gazebo world
  ├─ drone_1 model + sensors + physics
  ├─ drone_2 model + sensors + physics
  └─ ...

For each drone:
Gazebo sensors/physics <-> PX4 SITL instance <-> PX4 estimator/controllers <-> Gazebo actuators

ROS 2 autonomy stack <-> ROS 2 / DDS <-> PX4 bridge <-> PX4 SITL
                                      ^
                                      optional QGroundControl via MAVLink
```

### Role of each component

#### 1. Gazebo

Gazebo simulates:

- drone dynamics
- motors/propellers
- sensors such as IMU, GPS, barometer, camera, lidar
- environment, wind, obstacles, and terrain

Gazebo acts like the **physical drone and the world**.

#### 2. PX4 SITL

Each drone has its own **PX4 SITL instance**.

Each PX4 instance:

- receives simulated sensor data from Gazebo
- runs EKF2/state estimation
- runs flight-control loops
- generates actuator/motor outputs
- sends those outputs back to Gazebo
- accepts high-level commands from the autonomy stack
- publishes state, odometry, and status

PX4 SITL plays the role of the **flight controller**.

#### 3. ROS 2 autonomy stack

This plays the role of the **Jetson Orin Nano companion computer**.

It performs high-level tasks such as:

- mission logic
- path planning
- obstacle avoidance
- swarm coordination
- behavior planning
- generation of high-level control intent for PX4

## How the simulated system mimics the real drone

### Real system

```text
Jetson Orin Nano (ROS 2 autonomy)
   <-> companion communication link
PX4 on flight controller
   <-> sensors + ESCs + motors + airframe
```

### Simulated system

```text
ROS 2 autonomy on laptop
   <-> ROS 2 / DDS bridge
PX4 SITL on laptop
   <-> Gazebo sensors + actuators + physics
```

So the **hardware changes**, but the **software boundary stays similar**:

- the autonomy stack still behaves like a companion computer
- PX4 still behaves like the low-level autopilot
- Gazebo behaves like the real vehicle and sensor environment

## Communication paths in the system

There are **two main communication links**.

### 1. Gazebo <-> PX4 SITL

This link represents the **physical vehicle interaction**.

- **Gazebo -> PX4:** simulated sensor data such as IMU, GPS, barometer, etc.
- **PX4 -> Gazebo:** actuator outputs such as motor commands

This is how PX4 "believes" it is flying a real drone.

Usually, you do **not** implement this communication yourself. If you use the standard PX4 SITL + Gazebo setup, this simulator link is already provided by the PX4/Gazebo integration.

### 2. ROS 2 autonomy stack <-> PX4

This link represents the **companion computer to flight controller link**.

Your autonomy stack sends high-level intent to PX4, and PX4 sends back state and status.

Examples of autonomy-to-PX4 intent:

- desired position
- desired velocity
- trajectory setpoints
- mode changes
- arm/disarm
- takeoff/land

Examples of PX4-to-autonomy feedback:

- odometry
- attitude
- local/global position
- battery/status
- navigation state

## What communication protocol PX4 uses

PX4 uses **different protocols for different links**.

### Inside PX4

PX4 internally uses:

- **uORB**

This is PX4's internal publish/subscribe message bus. Estimators, controllers, sensors, and commander modules communicate through uORB.

Your ROS 2 nodes do **not** directly talk to uORB.

### PX4 to external systems

PX4 commonly uses:

- **MAVLink**
- **uXRCE-DDS / DDS bridge**

#### MAVLink

MAVLink is widely used for:

- QGroundControl
- telemetry radios
- mission commands
- companion computer communication
- MAVSDK or MAVROS-based control

#### ROS 2 integration path

For ROS 2, the usual communication path is:

```text
ROS 2 nodes <-> DDS <-> Micro XRCE-DDS Agent <-> PX4 uXRCE-DDS client <-> uORB
```

This means:

- your autonomy stack publishes/subscribes as ROS 2 topics
- the DDS bridge carries those messages to PX4
- PX4 maps them to and from internal uORB topics

So PX4 is **not directly a native ROS 2 node internally**, but it can communicate with ROS 2 through the **uXRCE-DDS bridge**.

## Do you need to implement a custom link?

### Between PX4 and Gazebo

**Usually no.**

If you use the standard PX4 SITL + Gazebo integration, the simulator link is already handled for you.

### Between PX4 and ROS 2 autonomy

**You need a bridge, but usually not a custom protocol implementation.**

Use the existing PX4 ROS 2 integration:

- PX4 uXRCE-DDS client
- Micro XRCE-DDS Agent
- ROS 2 topics/messages

So:

- **do not invent a custom communication protocol unless necessary**
- **use the standard PX4 ROS 2 bridge path**

## Recommended ROS 2 control interface for intent

Your autonomy stack should send PX4-supported setpoints and commands, not arbitrary intent messages.

Common message types include:

- `OffboardControlMode`
- `TrajectorySetpoint`
- `VehicleCommand`

Typical offboard flow:

1. Publish `OffboardControlMode`
2. Publish `TrajectorySetpoint` or another supported setpoint
3. Send `VehicleCommand` to arm and switch to offboard mode
4. Keep publishing setpoints so PX4 remains in offboard control

Typical feedback subscriptions:

- vehicle odometry
- local/global position
- attitude
- navigation state
- battery and vehicle status

## Multi-drone SITL setup

For **N drones**, you typically run:

- **N Gazebo vehicle instances** or one Gazebo world containing N models
- **N PX4 SITL instances**
- **ROS 2 nodes** either per drone or centrally with per-drone namespaces

Each drone should have unique:

- namespace
- PX4 instance
- simulator ports
- system ID
- topic names
- TF frame names

Example namespace structure:

```text
/drone_1/fmu/in/trajectory_setpoint
/drone_1/fmu/in/offboard_control_mode
/drone_1/fmu/in/vehicle_command
/drone_1/fmu/out/vehicle_odometry

/drone_2/fmu/in/trajectory_setpoint
/drone_2/fmu/out/vehicle_odometry
```

## Optional ground station path

You can also connect **QGroundControl** to PX4 SITL using **MAVLink UDP** for:

- telemetry display
- arming and mode inspection
- mission upload
- parameter tuning
- debugging

This is separate from the ROS 2 autonomy link and is often useful during simulation.

For a practical QGC-first workflow (including ROS 2 simulator topic visualization), see
[`Q-Ground-Control.md`](./Q-Ground-Control.md).

## End-to-end data flow

```text
Planner in ROS 2
  -> publishes intent/setpoints
  -> DDS bridge delivers them to PX4
  -> PX4 estimates state and runs control
  -> PX4 sends actuator commands to Gazebo
  -> Gazebo updates motion and physics
  -> Gazebo generates new sensor data
  -> PX4 updates state estimate
  -> PX4 publishes odometry/status back to ROS 2
```

## Short summary

In this simulated system:

- **Gazebo** acts like the drone body, sensors, motors, and world
- **PX4 SITL** acts like the flight controller
- **ROS 2 autonomy stack** acts like the Jetson companion computer
- **PX4 <-> Gazebo** uses the standard simulator integration
- **PX4 <-> ROS 2** uses the DDS bridge path, typically through **uXRCE-DDS**
- **PX4 internal modules** communicate through **uORB**
- **QGroundControl**, if used, typically talks to PX4 over **MAVLink**
