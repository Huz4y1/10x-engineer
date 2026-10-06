---
tags: [robotics, ros2, deep-dive]
---

# ROS 2 in practice

The index is [[21 — ROBOTICS]]. This is the software layer — how the boxes in that diagram actually talk to each other.

Control theory: [[22 — CONTROL SYSTEMS]] · Estimation: [[PID and Kalman filters]] · Hardware: [[20 — EMBEDDED]]

---

## What ROS actually is

ROS is **not** an operating system. It's a **message-passing system plus a build system plus a pile of existing robot drivers.**

The idea: a robot is many small programs that need to talk. The camera program, the wheel program, the planning program. Instead of wiring them together directly, each one publishes messages to a named channel, and anyone interested subscribes.

```
[camera node] --publishes--> /image_raw --subscribes--> [detector node]
                                                              |
[wheel node] <--subscribes-- /cmd_vel <---------publishes-----+
```

**Why this is worth the complexity:** you can kill the detector, restart it, or swap it for a different one, and nothing else notices. You can record every message and replay the whole run at your desk. That last part — `ros2 bag` — is the single biggest reason people accept ROS.

> **If you're building one small robot with one microcontroller, you don't need ROS.** Its value appears when you have many sensors, several processes, and a need to debug what happened after the fact.

---

## The four communication patterns

| Pattern | Shape | Use for |
|---|---|---|
| **Topic** | Fire and forget, many-to-many | Sensor streams, commands — **90% of everything** |
| **Service** | Request → response, blocking | "Reset the odometry", quick queries |
| **Action** | Long request, with feedback and cancel | "Drive to the kitchen" |
| **Parameter** | Named config value, changeable at runtime | Gains, thresholds, frame names |

> **Default to topics.** Services block, and a service call that hangs takes your node with it. If it takes more than a moment, it should be an action.

---

## A publisher

```python
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist

class Driver(Node):
    def __init__(self):
        super().__init__("driver")
        self.pub = self.create_publisher(Twist, "/cmd_vel", 10)
        self.timer = self.create_timer(0.1, self.tick)      # 10 Hz

    def tick(self):
        msg = Twist()
        msg.linear.x = 0.2       # m/s forward
        msg.angular.z = 0.0      # rad/s turn
        self.pub.publish(msg)

def main():
    rclpy.init()
    node = Driver()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.shutdown()
```

## A subscriber

```python
from rclpy.node import Node
from sensor_msgs.msg import LaserScan

class Watcher(Node):
    def __init__(self):
        super().__init__("watcher")
        self.create_subscription(LaserScan, "/scan", self.on_scan, 10)

    def on_scan(self, msg: LaserScan):
        import math
        valid = [r for r in msg.ranges if math.isfinite(r) and r > msg.range_min]
        if valid and min(valid) < 0.3:
            self.get_logger().warn("obstacle close")
```

> ⚠️ **Laser ranges contain `inf` and `NaN`** for "nothing detected" and "bad reading". `min(msg.ranges)` on raw data gives you `NaN`, and `NaN < 0.3` is `False` — so your emergency stop silently never fires. Filter first. ([[Numerical methods in practice]])

> ⚠️ **Never block inside a callback.** `time.sleep`, a long computation or a blocking service call stops the executor and your other callbacks stop running. Do work in a timer, or use a separate callback group.

---

## QoS — the thing that silently breaks everything

ROS 2 lets each topic choose delivery guarantees. **If the publisher and subscriber disagree, they simply never connect** — no error, no warning, just silence.

```python
from rclpy.qos import QoSProfile, ReliabilityPolicy, DurabilityPolicy, HistoryPolicy

sensor_qos = QoSProfile(
    reliability=ReliabilityPolicy.BEST_EFFORT,     # sensors: drop, don't queue
    history=HistoryPolicy.KEEP_LAST, depth=1,
)

latched = QoSProfile(
    durability=DurabilityPolicy.TRANSIENT_LOCAL,   # late subscribers get the last message
    reliability=ReliabilityPolicy.RELIABLE,
    history=HistoryPolicy.KEEP_LAST, depth=1,
)
```

| Data | Reliability | Why |
|---|---|---|
| Camera, LiDAR, IMU | `BEST_EFFORT`, depth 1 | A stale frame is worse than a dropped one |
| Commands, goals | `RELIABLE` | Must arrive |
| Map, static config | `TRANSIENT_LOCAL` | Nodes starting later still get it |

> ⚠️ **"My subscriber gets nothing" is a QoS mismatch far more often than a bug.** Check with `ros2 topic info /scan --verbose` — it shows both sides' profiles.

---

## Transforms (tf2) — where everything is

Every sensor sees the world from its own position. tf2 tracks the relationships and converts between them.

```
map -> odom -> base_link -> laser
                         -> camera_link
```

```python
import rclpy
from tf2_ros import Buffer, TransformListener
import tf2_geometry_msgs      # import registers the geometry_msgs conversions

self.buf = Buffer()
self.listener = TransformListener(self.buf, self)

try:
    tf = self.buf.lookup_transform("map", "laser", rclpy.time.Time())   # Time() = latest
    point_in_map = tf2_geometry_msgs.do_transform_point(point_in_laser, tf)
except Exception as e:
    self.get_logger().warn(f"transform unavailable: {e}")
```

> **`rclpy.time.Time()` means "latest available".** Asking for a specific timestamp before it exists raises `ExtrapolationException` — the most common tf error, and it usually means your clocks disagree.

> ⚠️ **The tf tree must have exactly one root and no cycles.** Two nodes publishing the same parent→child transform makes the tree flicker and your robot appear to teleport. `ros2 run tf2_tools view_frames` draws it.

---

## The command line — where you actually debug

```bash
ros2 node list
ros2 topic list
ros2 topic echo /scan                 # print messages
ros2 topic hz /scan                   # is it publishing, and how fast
ros2 topic info /scan --verbose       # QoS on both sides
ros2 topic pub /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.1}}"

ros2 service list && ros2 service call /reset std_srvs/srv/Empty
ros2 param list && ros2 param get /driver max_speed
ros2 interface show sensor_msgs/msg/LaserScan

ros2 bag record -a                    # record EVERYTHING
ros2 bag play rosbag2_2026_01_01/     # replay it at your desk
ros2 bag info rosbag2_2026_01_01/

rqt_graph                             # see who talks to whom
ros2 run tf2_tools view_frames
```

> **`ros2 bag record -a` before every real-world test.** Hardware failures are rarely reproducible. A bag file turns a one-off mystery into something you can debug a hundred times.

> **Debug order:** `node list` (is it running?) → `topic hz` (is it publishing?) → `topic echo` (is the data sane?) → `topic info --verbose` (do the QoS profiles match?) → `view_frames` (is tf connected?).

---

## Packages and building

```bash
ros2 pkg create --build-type ament_python my_robot --dependencies rclpy sensor_msgs

colcon build --symlink-install         # --symlink-install: edit Python without rebuilding
source install/setup.bash              # EVERY new terminal
ros2 run my_robot driver
ros2 launch my_robot bringup.launch.py
```

> ⚠️ **`source install/setup.bash` in every terminal, after every build.** "Package not found" is almost always a forgotten source. Put the workspace source line in your `~/.bashrc` ([[Bash]]).

```python
# launch/bringup.launch.py
from launch import LaunchDescription
from launch_ros.actions import Node

def generate_launch_description():
    return LaunchDescription([
        Node(package="my_robot", executable="driver", name="driver",
             parameters=[{"max_speed": 0.5}]),
        Node(package="my_robot", executable="watcher", name="watcher"),
    ])
```

> **A launch file is how you start a robot.** Ten `ros2 run` terminals is fine for a demo and unusable for a real system.

---

## Simulation

| Tool | For |
|---|---|
| **Gazebo** | Full physics, sensors, the standard |
| Webots | Lighter, easier to start |
| **PX4 SITL** | Flight controller in the loop ([[23 — AEROSPACE]]) |
| Isaac Sim | GPU physics, photoreal, for [[12 — REINFORCEMENT LEARNING]] |

> **Develop in simulation, but distrust it.** Simulated sensors have no dropouts, no mud on the lens, and perfect timing. The **sim-to-real gap** is where robotics projects fail — real friction, real latency, real noise. Test on hardware early and often.

> **Set `use_sim_time: true` on every node when simulating**, so they follow the simulator's clock rather than wall time. Mixing the two produces tf errors that make no sense.

---

## Safety

```python
from geometry_msgs.msg import Twist

# A watchdog: stop if commands stop arriving
def tick(self):
    if (self.get_clock().now() - self.last_cmd).nanoseconds > 5e8:   # 500 ms
        self.pub.publish(Twist())      # all zeros = stop
```

> ⚠️ **A robot that keeps its last command when the controller dies will drive into a wall.** Every actuator needs a timeout that defaults to *stopped*. This is not optional on anything with a motor.

> **A physical emergency stop that cuts motor power in hardware.** Software cannot be trusted to stop software. Test it before every session ([[20 — EMBEDDED]]).

---

## Common mistakes

| Mistake | Symptom |
|---|---|
| QoS mismatch | Subscriber receives nothing, no error |
| Forgot `source install/setup.bash` | "Package not found" |
| Blocking inside a callback | Everything else stops |
| `min()` on raw laser ranges | `NaN` — safety check never fires |
| Two publishers on one tf | Robot appears to teleport |
| `use_sim_time` unset in sim | Constant tf extrapolation errors |
| No command watchdog | **Robot keeps driving after the controller dies** |
| Service call for a slow operation | Node hangs |
| Not recording a bag | Unreproducible hardware bug |
| Trusting simulation | Sim-to-real gap |
| Using ROS for a one-chip robot | Enormous overhead for nothing |

## Related

[[21 — ROBOTICS]] · [[22 — CONTROL SYSTEMS]] · [[PID and Kalman filters]] · [[20 — EMBEDDED]] · [[11 — COMPUTER VISION]] · [[12 — REINFORCEMENT LEARNING]] · [[Sensors overview]] · [[23 — AEROSPACE]] · [[Numerical methods in practice]] · [[Bash]]
