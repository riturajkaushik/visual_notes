# MAVLink in Real and Simulated Drones

## What MAVLink is and why it exists

**MAVLink** (Micro Air Vehicle Link) is a lightweight messaging protocol used to connect drone components and tools that need to exchange:

- telemetry and health data,
- commands and mode changes,
- mission items and status,
- parameters and configuration,
- acknowledgments and system state.

Its main purpose is **interoperability**:

- autopilot <-> ground station (QGC),
- autopilot <-> companion computer,
- simulator/autopilot <-> external tools,
- one network <-> multiple consumers via routers.

In practice, MAVLink gives you a common “wire language” across hardware, simulators, and software stacks.

---

## MAVLink in real drones vs simulated drones

## Real drone path

Typical flow:

1. Sensors feed the autopilot (PX4/ArduPilot).
2. Autopilot estimates state and runs control.
3. Autopilot publishes MAVLink telemetry to GCS/companion/radio.
4. GCS sends back commands, mission updates, or parameter requests via MAVLink.

Common links:

- serial telemetry radio,
- USB serial,
- UDP/TCP over companion network.

## Simulated drone path

You generally have one of these:

1. **SITL autopilot present**: simulator <-> SITL autopilot, and autopilot emits MAVLink (same as real workflow for external tools).
2. **No autopilot, synthetic state publisher**: simulator publishes ROS 2 or custom states; a bridge emits MAVLink so QGC/tools can consume it.

Key point: external MAVLink consumers (QGC, routers, loggers) often do not care whether data came from real FCU, SITL, or a synthetic bridge, as long as protocol behavior is valid.

---

## What MAVLink carries (conceptually)

Message families you usually care about:

1. **Identity/liveness**: heartbeat, system type, mode bits.
2. **Pose/motion**: attitude, local/global position, velocity.
3. **Health/status**: battery, GPS fix, EKF/arming status, errors.
4. **Command/control**: arm/disarm, mode changes, takeoff/land, generic commands.
5. **Mission/protocol**: waypoint upload/download and progress.
6. **Parameter protocol**: read/write parameter values.
7. **Time/sync/logging**: timestamps and optional log/control signals.

---

## Transport and topology basics

MAVLink messages can travel over:

- **Serial** (UART/USB),
- **UDP** (very common in SITL and companion setups),
- **TCP** (less common for flight-critical telemetry, but used in some architectures).

Typical topology:

- one producer (autopilot/bridge),
- one or more consumers (QGC, onboard app, logger),
- optional router (e.g., mavlink-router style fan-out).

### What MAVRouter is

**MAVRouter** (commonly implemented with tools like `mavlink-router`) is a MAVLink traffic router that sits between endpoints and forwards MAVLink packets between links.

Its main role is to decouple one MAVLink source from many clients:

- one FCU/SITL stream in, many consumers out (QGC, companion apps, loggers),
- multiple input/output links bridged across serial and UDP/TCP paths,
- centralized routing point instead of every tool connecting directly to the autopilot.

Typical use cases:

1. **Real drone**: fan out a single telemetry radio/serial stream to both GCS and onboard services.
2. **SITL/simulation**: expose one simulated MAVLink source to QGC plus test/analytics tools at the same time.
3. **Mixed tooling**: run logging/monitoring without disrupting the primary GCS connection.

Important identifiers:

- **SYSID**: vehicle/system identity,
- **COMPID**: component identity inside that system.

Incorrect IDs are a common reason QGC or tools do not behave as expected.

---

## ROS 2 -> MAVLink conversion (publishing MAVLink from ROS 2 topics)

Use this direction when your source of truth is ROS 2 topics and you want QGC/MAVLink tools to see a vehicle.

### Why convert this direction

- You have simulator-generated PX4-like state topics.
- You are not running full PX4 SITL but still want QGC telemetry/visualization.

### Conceptual mapping

| ROS 2 data | MAVLink output purpose |
|---|---|
| Pose/Odometry | Vehicle attitude + position telemetry |
| Velocity/Twist | Speed/velocity status |
| Battery state | Battery/health status |
| Armed/mode/nav state | Heartbeat mode/state bits + system status |
| GPS-like fix/global pose (if available) | Global map position and GPS status |

### Minimum set for QGC visibility

At minimum, publish equivalents of:

1. heartbeat/system identity,
2. attitude,
3. position (local or global),
4. basic status/battery.

If heartbeat is missing or invalid, QGC usually will not show a valid active vehicle.

### Engineering details that matter

1. **Frames**: ROS often ENU, MAVLink often NED conventions in many PX4 contexts.
2. **Units**: keep strict unit mapping and scaling.
3. **Rates**: publish at stable rates (too slow = stale UI, too fast = unnecessary load).
4. **Time**: consistent timestamps to avoid jitter or “time going backward” behavior.
5. **SYSID/COMPID**: deterministic IDs to avoid collisions in multi-vehicle setups.

---

## MAVLink -> ROS 2 conversion (consuming MAVLink into ROS 2 topics)

Use this direction when your source of truth is MAVLink (real FCU/SITL/bridge) and ROS 2 nodes need structured topic data.

### Why convert this direction

- Build autonomy or analytics in ROS 2 while vehicle is controlled/observed over MAVLink.
- Normalize real and simulated inputs into the same ROS 2 interface.

### Conceptual mapping

| MAVLink input purpose | ROS 2 topic representation |
|---|---|
| Heartbeat and mode/state | vehicle state/status topic |
| Attitude + pose telemetry | pose/odometry topic |
| Velocity telemetry | twist/velocity topic |
| Battery/system health | battery/health topic |
| Command acknowledgments | command status/ack topics |
| Mission progress/state | mission status topics |

### Engineering details that matter

1. Preserve sequence and freshness (drop stale or out-of-order data safely).
2. Map mode/state enums clearly (document any approximations).
3. Keep frame transforms explicit in one place (do not mix conventions ad hoc).
4. Choose ROS 2 QoS profiles intentionally for telemetry vs control-relevant topics.

---

## Practical implementation patterns for both directions

## Pattern A: MAVROS-centered

- Good when you want established ROS <-> MAVLink integration behavior.
- Fastest path for many prototypes.
- Requires understanding plugin/topic conventions.

## Pattern B: MAVSDK + custom ROS 2 node

- Good when you need higher-level APIs for command/control and custom ROS interfaces.
- Cleaner for application-level logic; less protocol boilerplate than raw MAVLink.

## Pattern C: Custom MAVLink serializer/deserializer bridge

- Maximum control over message set, rates, IDs, and behavior.
- Best for highly custom simulators and strict interface contracts.
- Highest maintenance and protocol-responsibility burden.

---

## Real vs sim conversion caveats

1. **Semantic gaps**: simulated states may not include all autopilot semantics QGC expects.
2. **Protocol completeness**: telemetry-only bridges may not fully support params/mission workflows.
3. **Timing realism**: simulation clocks and publish rates can differ from onboard timing behavior.
4. **Multi-vehicle collisions**: ID and namespace discipline is mandatory.

---

## Bidirectional conversion checklist

1. Confirm heartbeat appears and stays stable.
2. Confirm consistent SYSID/COMPID per vehicle/component.
3. Validate frame mapping (ENU/NED) with a known motion test.
4. Validate units and signs for altitude, yaw, and velocity axes.
5. Validate timestamp monotonicity and update rates.
6. Validate battery/status fields are populated.
7. Validate command path:
   - ROS 2 command -> MAVLink command/ack
   - MAVLink command/status -> ROS 2 state update
8. Validate behavior under packet loss or delayed updates.

---

## Purpose summary (one paragraph)

MAVLink is the standard interoperability layer that lets flight stacks, ground stations, companion apps, and simulation infrastructure exchange vehicle telemetry, control intent, and management protocols in a shared format. In real drones it links autopilot and external systems; in simulation it enables the same toolchain behavior, even when the state source is synthetic, as long as message semantics, timing, IDs, and frame conventions are handled correctly.
