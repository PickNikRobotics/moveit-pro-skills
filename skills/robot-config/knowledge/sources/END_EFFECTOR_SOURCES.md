# Available End Effector URDFs for ROS 2

A reference of publicly available end effector URDF/description packages. Focus on grippers and tools that can be combined with robot arm configs in MoveIt Pro.

---

## Parallel Jaw Grippers

| Manufacturer | Model | Payload | URDF Available | Repo |
|---|---|---|---|---|
| Robotiq | 2F-85 (85mm stroke) | ~5 kg | Yes | [ros2_robotiq_gripper](https://github.com/PickNikRobotics/ros2_robotiq_gripper) |
| Robotiq | 2F-140 (140mm stroke) | ~2.5 kg | Yes (C3 model) | [robotiq_2finger_grippers](https://github.com/Danfoa/robotiq_2finger_grippers) |
| Robotiq | Hand-E (50mm stroke) | ~7 kg | Yes | [ros2_robotiq_gripper](https://github.com/PickNikRobotics/ros2_robotiq_gripper) |
| Schunk | EGK series | Varies | Driver only | [schunk_egu_egk_gripper](https://github.com/SCHUNK-SE-Co-KG/schunk_egu_egk_gripper) |
| OnRobot | RG2 | 2 kg | Yes | [onrobot-ros2](https://github.com/ABC-iRobotics/onrobot-ros2) |
| OnRobot | RG6 | 6 kg | Yes | [onrobot-ros2](https://github.com/ABC-iRobotics/onrobot-ros2) |

## Vacuum / Suction Grippers

| Manufacturer | Model | URDF Available | Repo |
|---|---|---|---|
| Robotiq | ePick | Yes | [ros2_epick_gripper](https://github.com/PickNikRobotics/ros2_epick_gripper) |
| Franka Robotics | Cobot Pump (for FR3) | Yes (in franka_description) | [franka_description](https://github.com/frankaemika/franka_description) |

## Multi-Finger / Dexterous Hands

| Manufacturer | Model | Fingers | URDF Available | Repo |
|---|---|---|---|---|
| Schunk | SVH 5-Finger Hand | 5 | Yes | [schunk_svh_library](https://github.com/SCHUNK-SE-Co-KG/schunk_svh_library) |
| PAL Robotics | Hey5 Hand | 5 | Yes | [pal_hey5](https://github.com/pal-robotics/pal_hey5) |
| Robotiq | 3-Finger Gripper | 3 | Community | — |
| Wonik Robotics | Allegro Hand | 4 | Yes (ROS 1, URDF reusable) | [allegro_hand_ros](https://github.com/simlabrobotics/allegro_hand_ros) |

## Integrated Arm+Gripper (gripper URDF included with arm)

| Manufacturer | Arm | Gripper | Repo |
|---|---|---|---|
| Franka Robotics | FR3, FER | Franka Hand (2-finger) | [franka_description](https://github.com/frankaemika/franka_description) |
| Trossen/Interbotix | WidowX, ViperX, PincherX | Integrated finger gripper | [interbotix_ros_manipulators](https://github.com/Interbotix/interbotix_ros_manipulators) |
| Elephant Robotics | myCobot | Integrated gripper options | [mycobot_ros2](https://github.com/elephantrobotics/mycobot_ros2) |
| ROBOTIS | OpenMANIPULATOR-X | Integrated gripper | [open_manipulator](https://github.com/ROBOTIS-GIT/open_manipulator) |
| Hello Robot | Stretch | DexWrist + gripper | [stretch_ros2](https://github.com/hello-robot/stretch_ros2) |

## Tool Changers (mechanical, no actuation URDF needed)

| Manufacturer | Notes |
|---|---|
| ATI | Industry standard, no ROS driver needed — just a fixed joint in URDF |
| Milibar | No ROS driver |
| TripleA | No ROS driver |

---

## Mounting Compatibility

Most cobots in the 3-10 kg payload class (including Fairino FR5) use the **ISO 9409-1** tool flange standard. Common patterns:

| Flange | Bolt Pattern | Common On |
|---|---|---|
| ISO 9409-1-50-4-M6 | 50mm PCD, 4x M6 | UR3, small cobots |
| ISO 9409-1-63-4-M6 | 63mm PCD, 4x M6 | Fairino FR5, UR5/10, most 5-10kg cobots |
| ISO 9409-1-100-6-M8 | 100mm PCD, 6x M8 | Larger industrial arms |
| ISO 9409-1-31.5-4-M5 | 31.5mm PCD, 4x M5 | Franka FR3 (custom) |

**Robotiq grippers** ship with adapter plates for most cobot flanges and are the most widely used with ROS.

---

## Recommended for Fairino FR5

The Fairino FR5 is a 5 kg payload, 6-DOF cobot likely using a ~63mm ISO flange. Compatible end effectors with existing ROS 2 URDFs:

### Best Options (URDF + ROS 2 driver + MoveIt integration):
1. **Robotiq 2F-85** — Most widely supported in MoveIt Pro, PickNik-maintained driver, proven URDF
2. **Robotiq Hand-E** — Higher payload grip, same driver stack
3. **Robotiq ePick** — Vacuum gripper, good for flat objects

### Good Options (URDF available, may need integration work):
4. **OnRobot RG2/RG6** — Good URDF, community driver
5. **Schunk EGK** — Industrial quality, driver available but URDF may need creation from CAD

### For Testing/Learning (no physical hardware needed):
6. **Robotiq 2F-85 in simulation** — Use the URDF with mock hardware, no physical gripper needed. This is the best path for testing the config setup process.

---

## Combining Arm + Gripper in XACRO

To attach a gripper to an arm, create a top-level XACRO that includes both:

```xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro" name="fairino5_with_gripper">
    <!-- Arm -->
    <xacro:include filename="$(find fairino_sim)/description/fairino5.urdf.xacro" />

    <!-- Gripper -->
    <xacro:include filename="$(find robotiq_description)/urdf/robotiq_2f_85_macro.urdf.xacro" />
    <xacro:robotiq_gripper name="gripper" prefix="" parent="wrist3_link">
        <origin xyz="0 0 0" rpy="0 0 0" />
    </xacro:robotiq_gripper>

    <!-- ros2_control for gripper -->
    <!-- Add command/state interfaces for gripper joints -->
</robot>
```

Key steps:
1. Identify the arm's tool flange link (e.g., `wrist3_link`)
2. Attach gripper via a fixed joint with appropriate offset
3. Add gripper joints to ros2_control XACRO
4. Update SRDF with gripper group and end effector definition
5. Add gripper controller to ros2_control config
6. Update config.yaml if adding gripper-specific behaviors

See the [Franka configs](https://github.com/frankaemika/franka_description) for a well-documented example of arm+hand composition.
