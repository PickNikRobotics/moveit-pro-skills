# Available Robot URDFs for ROS 2

A reference of publicly available URDF/description packages for building MoveIt Pro configurations. Sourced from the [PickNik Hardware Ecosystem](https://picknik.ai/hardware-ecosystem/) and community repositories.

> **Note**: Universal Robots and Kinova are excluded — those are already well-covered in MoveIt Pro example configs.

---

## Robot Arms

### Tier 1: MoveIt Pro Configs Available

These have existing MoveIt Pro configuration packages or official PickNik support.

| Manufacturer | Models | Description Package | Driver / Config |
|---|---|---|---|
| Franka Robotics | FR3, FR3v2, FP3, FER, FR3 Duo | [franka_description](https://github.com/frankaemika/franka_description) | [franka_ros2](https://github.com/frankaemika/franka_ros2), [MoveIt Pro config](https://docs.picknik.ai/hardware_guides/franka_fr3_hardware_setup_guide/) |
| UFactory | xArm 5/6/7, Lite 6 | Included in [xarm_ros2](https://github.com/xArm-Developer/xarm_ros2) | [MoveIt Pro config](https://github.com/PickNikRobotics/moveit_pro_xarm_configs) |
| Hello Robot | Stretch | [stretch_description](https://github.com/hello-robot/stretch_ros2) | [MoveIt Pro config](https://github.com/PickNikRobotics/stretch_pro_config) |
| FANUC | Various | Included in driver | [fanuc_driver](https://github.com/FANUC-CORPORATION/fanuc_driver), [MoveIt Pro guide](https://docs.picknik.ai/hardware_guides/fanuc_hardware_setup_guide/) |
| ABB | Various | [abb_ros2](https://github.com/PickNikRobotics/abb_ros2/tree/rolling) | Included |

### Tier 2: ROS 2 Driver + Description Available

These have ROS 2 drivers with URDF descriptions but no pre-built MoveIt Pro config. Good candidates for building new configs.

| Manufacturer | Models | Description / Driver Repo |
|---|---|---|
| KUKA | LBR iiwa 7/14, LBR Med | [lbr_fri_ros2_stack](https://github.com/lbr-stack/lbr_fri_ros2_stack) |
| KUKA | KR Cybertech, LBR iisy | [kuka_drivers](https://github.com/kroshu/kuka_drivers) |
| KUKA | KR3, KR5, KR6, KR10, KR16, KR120, KR150, KR210 | [kuka_experimental](https://github.com/ros-industrial/kuka_experimental) (ROS 1, URDFs reusable) |
| Doosan | M-series (m0609, m1013, m1509, m0617, a0509, a0912, h2515, h2017) | [doosan-robot2](https://github.com/doosan-robotics/doosan-robot2) |
| Denso | COBOTTA, VS-060 | [denso_robot_ros2](https://github.com/DENSORobot/denso_robot_ros2) |
| Flexiv | Rizon 4, Rizon 10 | [flexiv_ros2](https://github.com/flexivrobotics/flexiv_ros2) |
| Techman | TM5, TM12, TM14 | [tmr_ros2](https://github.com/TechmanRobotInc/tmr_ros2) |
| Elite Robots | EC66, CS-series | [Elite_Robots_CS_ROS2_Driver](https://github.com/Elite-Robots/Elite_Robots_CS_ROS2_Driver) |
| Aubo | i5, i10 | [aubo_ros2_driver](https://github.com/AuboRobot/aubo_ros2_driver) |
| Fairino | FR5, FR10 | [frcobot_ros2](https://github.com/FAIR-INNOVATION/frcobot_ros2) |
| Kawasaki | RS-series | [khi_ros2](https://github.com/Kawasaki-Robotics/khi_ros2) |
| Yaskawa Motoman | GP-series | [motoros2](https://github.com/Yaskawa-Global/motoros2) |
| Mitsubishi | MELFA RV-series | [melfa_ros2_driver](https://github.com/Mitsubishi-Electric-Asia/melfa_ros2_driver) |
| Neuromeka | Indy7 | [indy-ros2](https://github.com/neuromeka-robotics/indy-ros2) |
| Tormach | ZA6 | [tormach_za_ros2_drivers](https://github.com/tormach/tormach_za_ros2_drivers/) |
| Trossen/Interbotix | WidowX, ViperX, PincherX | [interbotix_ros_manipulators](https://github.com/Interbotix/interbotix_ros_manipulators) |
| Elephant Robotics | myCobot 280/320 | [mycobot_ros2](https://github.com/elephantrobotics/mycobot_ros2) |
| Comau | edo | [edo-ROS2](https://github.com/comau-na/edo-ROS2) |
| Duatic AG | DynaArm | [dynaarm_driver](https://github.com/Duatic/dynaarm_driver) |
| Schunk | LWA 4P | [schunk_svh_library](https://github.com/SCHUNK-SE-Co-KG/schunk_svh_library) |
| AgileX | Piper | [Piper_ros](https://github.com/agilexrobotics/Piper_ros) |
| Hyundai | Various | [hdr_ros2_driver](https://github.com/hyundai-robotics/hdr_ros2_driver) |
| Stäubli | TX2-series | [Staubli_ROS2](https://github.com/IvoD1998/Staubli_ROS2) |

### Tier 3: Community / Hobby / Open Source

| Robot | Description / Driver Repo |
|---|---|
| Annin Robotics AR4 | [ar4_ros_driver](https://github.com/ycheng517/ar4_ros_driver) |
| Standard Open Arm SO-ARM100 | [ros2_so_arm100](https://github.com/JafarAbdi/ros2_so_arm100) |
| Dorna 2 | [r2d2](https://github.com/pinorobotics/r2d2) |
| Hanwha HCR-3A | [hanwha_robot_arm](https://github.com/pondinesh006/hanwha_robot_arm) |
| Mecademic | [mecademic-ros2](https://github.com/myersjm/mecademic-ros2/) |

---

## Grippers & End Effectors

| Manufacturer | Models | Repo |
|---|---|---|
| Robotiq | 2F-85, 2F-140, Hand-E | [ros2_robotiq_gripper](https://github.com/PickNikRobotics/ros2_robotiq_gripper) |
| Robotiq | ePick (vacuum) | [ros2_epick_gripper](https://github.com/PickNikRobotics/ros2_epick_gripper) |
| Schunk | EGK series | [schunk_egu_egk_gripper](https://github.com/SCHUNK-SE-Co-KG/schunk_egu_egk_gripper) |
| OnRobot | RG2, RG6 | [onrobot-ros2](https://github.com/ABC-iRobotics/onrobot-ros2) |

---

## Mobile Bases / AGVs

| Manufacturer | Models | Repo |
|---|---|---|
| Clearpath | Husky | [husky](https://github.com/husky/husky) |
| Clearpath | Ridgeback | [ridgeback](https://github.com/ridgeback/ridgeback) |
| Husarion | Panther | [panther_ros](https://github.com/husarion/panther_ros) |
| Robotnik | RB Vogui | [rbvogui_sim](https://github.com/RobotnikAutomation/rbvogui_sim) |
| Robotnik | RB Theron | [rb_theron_sim](https://github.com/RobotnikAutomation/rb_theron_sim/) |
| Neobotix | EMRox | [neobotix](https://github.com/neobotix) |
| Omron | LD/HD series | [Omron_AMR_ROS2](https://github.com/OmronAPAC/Omron_AMR_ROS2) |
| PAL Robotics | TIAGo Pro | [tiago_robot](https://github.com/pal-robotics/tiago_robot) |
| MiR | MiR100/200 | [mir_robot](https://github.com/mintar/mir_robot/tree/ros2) |
| Avular | Origin One | [origin_one docs](https://avular-robotics.github.io/origin_one/1.0-2/) |

---

## Depth Sensors (with URDF models)

| Manufacturer | Models | Repo |
|---|---|---|
| Intel | RealSense D435, D455, L515 | [realsense-ros](https://github.com/IntelRealSense/realsense-ros) |
| StereoLabs | ZED, ZED 2, ZED Mini | [zed-ros2-wrapper](https://github.com/stereolabs/zed-ros2-wrapper) |
| Zivid | Zivid 2/2+ | [zivid-ros](https://github.com/zivid/zivid-ros) |
| Orbbec | Astra, Femto | [OrbbecSDK_ROS2](https://github.com/orbbec/OrbbecSDK_ROS2) |
| Luxonis | OAK-D series | [depthai-ros](https://github.com/luxonis/depthai-ros) |
| Mechmind | PRO-S | [mecheye_ros2_interface](https://github.com/MechMindRobotics/mecheye_ros2_interface) |
| Photoneo | PhoXi | [phoxi_camera](https://github.com/photoneo/phoxi_camera) |

---

## Force/Torque Sensors

| Manufacturer | Models | Repo |
|---|---|---|
| Robotiq | FT-300 | [rq_fts_ros2_driver](https://github.com/panagelak/rq_fts_ros2_driver) |
| Bota Systems | SensONE | [bota_driver](https://gitlab.com/botasys/bota_driver/) |

---

## Notes

- **ROS 1 URDFs are reusable in ROS 2** — the URDF format is unchanged. Only the build system (ament vs catkin) and launch files differ.
- **Tier 1** robots are the fastest path to a working MoveIt Pro setup.
- **Tier 2** robots need a MoveIt Pro config package built around their existing URDF — this is the primary use case for the CLAUDE.md guide.
- **Tier 3** robots may need additional work (mesh cleanup, URDF fixes) but are good for learning.
- Most driver repos include a `*_description` package with URDFs, meshes, and sometimes MoveIt configs. Check the repo structure for `urdf/`, `meshes/`, `description/`, or `*_moveit_config/` directories.
