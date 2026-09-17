# ROS rover teleoperation

ROS teleoperation firmware for a **six-wheel differential rover** (RoboMuse platform). An Arduino on the robot runs a `rosserial` node that subscribes to `cmd_vel` and drives the left and right motor banks over dual H-bridge channels.

The sketch is adapted from open ROS–Arduino teleop examples (original authors credited in `robomuse_motor_control.ino`). This repo is the **rover-side integration**: motor pin mapping, wheel geometry constants, and the serial ROS node — teleop input comes from a standard ROS source publishing `geometry_msgs/Twist`, not from a custom ground-station UI here.

---

## System overview

```
  Operator machine                         Robot (Arduino)
  ┌─────────────────────┐                 ┌──────────────────────────┐
  │  teleop_twist_      │   USB serial    │  rosserial node          │
  │  keyboard / joystick│ ──────────────► │  subscribes: cmd_vel     │
  │  (or nav stack)     │                 │  drives: left + right PWM│
  └─────────────────────┘                 └───────────┬──────────────┘
                                                      │
                                           ┌──────────┴──────────┐
                                           │  H-bridge × 2       │
                                           │  EN_L / IN1_L / IN2_L│──► 3× left motors
                                           │  EN_R / IN1_R / IN2_R│──► 3× right motors
                                           └─────────────────────┘
```

Each side of the rover has **three mechanically coupled wheels** driven as a single left or right channel — the firmware implements standard **differential-drive kinematics**, not per-wheel control.

---

## How `cmd_vel` becomes motor PWM

On each incoming `geometry_msgs/Twist` message the callback reads:

- `linear.x` — forward/back speed (m/s)
- `angular.z` — yaw rate (rad/s)

These are converted to left/right wheel angular velocities using the rover geometry:

```
ω_r = (v / r) + (ω · L) / (2 · r)
ω_l = (v / r) − (ω · L) / (2 · r)
```

| Symbol | Meaning              | Value in sketch |
| ------ | -------------------- | --------------- |
| `v`    | Linear velocity      | `msg.linear.x`  |
| `ω`    | Angular velocity     | `msg.angular.z` |
| `r`    | Wheel radius (m)     | `0.0325`        |
| `L`    | Track width (m)      | `0.295`         |

The main loop applies `MotorL(w_l × 10)` and `MotorR(w_r × 10)` each cycle, mapping the computed rad/s values to PWM duty on the enable pins. Direction is set via the H-bridge IN pins (one direction line per side is used; the complementary line is left commented in the sketch).

---

## Hardware pin map

| Pin  | Function              |
| ---- | --------------------- |
| 5    | Left enable (`EN_L`)  |
| 6    | Right enable (`EN_R`) |
| 7    | Left direction (`IN1_L`) |
| 8    | Right direction (`IN1_R`) |
| 11   | Left direction (`IN2_L`, unused in current logic) |
| 13   | Right direction (`IN2_R`, unused in current logic) |

Motors are driven through dual H-bridge modules — one channel per side, each side powering three wheels.

---

## Repository contents

| Path | Role |
| ---- | ---- |
| `robomuse_motor_control.ino` | Arduino ROS node — `cmd_vel` subscriber, differential-drive math, PWM output |
| `AR_Tags/recognise.py` | Early OpenCV thresholding experiment for onboard vision work; not a complete AR-tag pipeline |

---

## Stack

- **ROS 1** with [`rosserial_arduino`](http://wiki.ros.org/rosserial_arduino)
- **Arduino** (Uno-class board on the rover)
- **OpenCV** (optional, for vision experiments in `AR_Tags/`)

---

## Running teleop

### 1. Flash the Arduino

1. Install the **ROS Arduino libraries** (`ros_lib`) into your Arduino IDE — generated from your ROS workspace:

   ```bash
   rosrun rosserial_arduino make_libraries.py .
   ```

   Copy the generated `ros_lib` folder into your Arduino sketchbook `libraries/` directory.

2. Open `robomuse_motor_control.ino`, select your board and port, and upload.

### 2. Start the serial bridge on the robot

With ROS running on a machine connected to the Arduino over USB:

```bash
rosrun rosserial_python serial_node.py /dev/ttyUSB0
```

Adjust the port to match your setup (`/dev/ttyACM0`, etc.).

### 3. Publish velocity commands from the operator side

Keyboard teleop example:

```bash
rosrun teleop_twist_keyboard teleop_twist_keyboard.py
```

Any node that publishes to `cmd_vel` will work — joystick teleop, a navigation stack, or a custom Python node.

---

## Notes

- Wheel radius and track width are hard-coded in the sketch — recalibrate `wheel_rad` and `wheel_sep` if the chassis geometry changes.
- The PWM scaling factor (`× 10`) was tuned empirically for this platform; adjust if motors stall or overspeed.
- `AR_Tags/recognise.py` references a local `gradient.png` and demonstrates OpenCV threshold modes — it was a stepping stone toward onboard tag recognition, not production navigation code.
