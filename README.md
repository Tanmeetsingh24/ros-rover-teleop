# Mars rover teleoperation

ROS teleoperation of a six-motor rover, with motor commands running on Arduino. The firmware subscribes to `cmd_vel` (`geometry_msgs/Twist`) and converts linear/angular velocity into left/right wheel PWM.

The Arduino sketch is based on open ROS–Arduino teleop examples (original sketch authors noted in `robomuse_motor_control.ino`). This repo is the rover-side integration: motor pins, wheel geometry, and the ROS serial node on the robot.

## Contents

| Path | Role |
| --- | --- |
| `robomuse_motor_control.ino` | Arduino ROS node: `cmd_vel` → differential drive PWM |
| `AR_Tags/` | AR-tag assets used while testing onboard tag recognition for navigation experiments |

## Stack

ROS · `rosserial_arduino` · Arduino · differential drive

Teleop is intended to come from a ROS teleop source on the operator machine (`cmd_vel`), not from a custom ground-station UI in this repository.
