# PX4 Notes

Related note: for ground-station operations and QGC-first visualization guidance with PX4 ROS 2 simulator topics, see
[`Q-Ground-Control.md`](./Q-Ground-Control.md).

## What goes into PX4, and what comes out?

PX4 is a flight-control stack. For drones, it takes in:

- **Sensor data**
- **Pilot commands**
- **High-level commands** from a ground station, mission plan, or companion computer

PX4 processes that information, estimates the vehicle state, runs control loops, and outputs:

- **Actuator commands** to motors/servos
- **Telemetry and status information**

In short:

- **Inputs:** sensing + control intent
- **Outputs:** motor/servo signals + state/status information

## Typical sensor information PX4 receives on a drone

| Sensor | What it measures | Concrete example |
|---|---|---|
| **Accelerometer** | Linear acceleration | Detects the drone is tilting forward and speeding up |
| **Gyroscope** | Angular rate | Detects the drone is rotating 20 deg/s to the right |
| **Magnetometer** | Heading relative to Earth's magnetic field | Estimates that the drone is facing east |
| **Barometer** | Air pressure for altitude estimate | Estimates that the drone climbed from 10 m to 12 m |
| **GPS** | Global position, ground speed, time | Reports lat/lon and that the drone is moving north at 5 m/s |
| **Optical flow** | Motion over the ground from a downward camera | Detects that the drone is drifting left indoors |
| **Rangefinder / LiDAR** | Distance to the ground or obstacle | Measures the ground as 1.8 m below |
| **Airspeed sensor** | Airspeed, mostly for fixed-wing | Reports 17 m/s airspeed |
| **Battery monitor** | Voltage/current | Reports battery at 14.8 V |
| **RC receiver** | Pilot stick commands | Pilot raises throttle and yaws left |

### Concrete quadcopter example

In **position hold**, PX4 may use:

- **IMU** to estimate tilt and rotation
- **GPS** to estimate outdoor position
- **Barometer** to estimate altitude
- **Magnetometer** to estimate heading
- **Optical flow / rangefinder** for indoor or low-altitude stabilization

If wind pushes the drone 2 meters east, PX4 can detect the drift from GPS or optical flow, estimate the error, and command the motors to move it back.

## Does PX4 run state estimation?

Yes. PX4 runs onboard **state estimation**, and the standard estimator is **EKF2**, an **Extended Kalman Filter**.

It fuses sensor inputs such as:

- **IMU**
- **GPS**
- **Barometer**
- **Magnetometer**
- optionally **optical flow**, **rangefinder**, and other sensors

The estimator produces PX4's best estimate of:

- **Attitude**: roll, pitch, yaw
- **Position**
- **Velocity**
- often **sensor biases** and related internal states

So PX4 does not control the drone directly from raw sensor values. It first estimates the state, then uses that estimated state inside the controllers.

## What is control intent?

**Control intent** means the commanded behavior for the drone: what you want it to do, not what it is currently doing.

Examples:

- Pilot stick commands such as "go up" or "yaw left"
- A mission command such as "fly to waypoint A"
- An offboard command such as "hold 2 m altitude and move north at 1 m/s"
- A mode-level intent such as "stabilize", "position hold", "land", or "return to home"

PX4 converts that intent into lower-level targets. For example:

| Control intent | PX4 may turn it into |
|---|---|
| Roll stick right | Desired roll angle or roll rate |
| Fly to a waypoint | Desired position, velocity, and heading |
| Climb to 10 m | Desired altitude |
| Face north | Desired yaw |

Controllers then compare:

- **Desired state** from control intent
- **Estimated current state** from the estimator

The difference is the control error that PX4 tries to reduce through motor outputs.

## What types of control intent are possible in PX4?

PX4 does **not** accept arbitrary intent. It supports a defined set of command and setpoint types.

Common control-intent types include:

| Intent type | Example |
|---|---|
| **Manual stick intent** | Pilot commands roll/pitch/yaw/throttle |
| **Attitude setpoint** | Hold 10 deg roll, 0 deg pitch, yaw 90 deg |
| **Body-rate setpoint** | Rotate at 30 deg/s in yaw |
| **Thrust / torque setpoint** | Low-level actuator-oriented control |
| **Velocity setpoint** | Fly north at 2 m/s |
| **Position setpoint** | Go to x, y, z or a GPS waypoint |
| **Acceleration setpoint** | Accelerate forward at 1 m/s^2 |
| **Trajectory setpoint** | Position + velocity + acceleration together |
| **Mission/navigation commands** | Takeoff, land, RTL, waypoint following |
| **Vehicle commands / mode intent** | Arm, disarm, switch mode, enter offboard |

Which intents are valid depends on:

- **Vehicle type**: multicopter, fixed-wing, rover, VTOL
- **Flight mode**
- **Interface**: RC, MAVLink, ROS 2, offboard application, mission planner

So the answer is:

- **Not anything**
- **Only the supported PX4 command/setpoint interfaces**

## Where does PX4 run in a drone?

PX4 usually runs on the drone's **flight controller**, which is a small embedded computer.

### Typical hardware

- **Pixhawk-class flight controllers**
- Common processors: **STM32F4**, **STM32F7**, **STM32H7**

These boards connect to sensors and peripherals such as:

- IMU
- GPS
- Barometer
- Magnetometer
- ESCs / motors
- RC receiver
- Telemetry radio

### Typical operating system

On real flight controllers, PX4 commonly runs on:

- **NuttX**, a lightweight real-time operating system

In simulation or development, PX4 can also run on:

- **Linux**, for example on a desktop or laptop in SITL

### Concrete examples

- **Pixhawk 4 / Cube / Holybro flight controller** -> PX4 running on **NuttX**
- **Desktop/laptop SITL simulation** -> PX4 running as a **Linux process**
- **Raspberry Pi / NVIDIA Jetson companion computer** -> usually runs higher-level Linux software, while PX4 still runs on the flight controller

## One-line summary

PX4 takes in **sensor data** and **control intent**, runs **state estimation** and **control**, and outputs **motor/servo commands** plus **telemetry/status**.
