# Creating a MoveIt Pro Config from an Existing URDF

> **MANDATORY: Task tracking.** Before starting this guide, create a task for each numbered step below using `TaskCreate`. Mark each task `in_progress` when you start it and `completed` when you finish it. Do not skip steps. If a step cannot be completed, stop and ask the user — do not silently move on.

This guide covers creating a complete MoveIt Pro configuration package when you already have a URDF or XACRO description of your robot. If you don't have a URDF yet, see [Building a URDF from CAD/STL Files](urdf_from_cad.md) first.

### Prerequisites

- A valid URDF or XACRO file for the robot
- Joint names, link names, and kinematic chain known
- MoveIt Pro installed and licensed

### Step-by-Step

#### 0. Get the Robot Description Package

Before creating your config, you need the robot's URDF/description package in your workspace. Common approaches:

```bash
# Clone into your workspace src/ directory
cd ~/my_robot_ws/src/
git clone https://github.com/<org>/<robot_description_repo>.git
```

**Only copy what you need.** Many robot repos contain dozens of packages (drivers, examples, demos) but you only need the `*_description` package for the URDF and meshes. Either clone the whole repo and add `COLCON_IGNORE` files to packages you don't need, or copy just the description package directory.

**ROS 1 packages are reusable.** The URDF/xacro format and mesh files are identical between ROS 1 and ROS 2. If the description package is ROS 1 (catkin), you need to convert its `package.xml` to format 3 and switch the build type to `ament_cmake`:
- Change `<buildtool_depend>catkin</buildtool_depend>` to `<buildtool_depend>ament_cmake</buildtool_depend>`
- Add `<export><build_type>ament_cmake</build_type></export>`
- Update `CMakeLists.txt` to use `find_package(ament_cmake REQUIRED)` and `ament_package()` instead of catkin macros. For a description-only package (no compiled code), a minimal CMakeLists.txt that just installs directories is sufficient.

**Binary package conflicts:** If `rosdep install` installs a binary version of the same package, your source copy must override it. Add the package name to `allow-overriding` in `colcon-defaults.yaml`:
```yaml
build:
  allow-overriding:
    - my_robot_description
```

#### 1. Scaffold the Package

```bash
moveit_pro new config
```

Or manually create the directory structure:

```
<package_name>/
├── CMakeLists.txt
├── package.xml
├── config/
│   ├── config.yaml
│   ├── control/<robot>_ros2_control.yaml
│   └── moveit/
│       ├── <robot>.srdf
│       ├── kinematics.yaml
│       ├── joint_limits.yaml
│       ├── joint_jog.yaml
│       ├── pose_jog.yaml
│       └── sensors_3d.yaml
├── description/
│   ├── <robot>.urdf.xacro
│   └── <robot>.ros2_control.xacro
├── launch/
│   └── agent_bridge.launch.xml
├── objectives/
│   └── move_to_home.xml
└── waypoints/
    └── waypoints.yaml
```

#### 2. Robot Description (description/)

**`<robot>.urdf.xacro`** — Wraps the existing URDF and adds ros2_control.

Some robot descriptions provide a plain URDF file you can include directly:
```xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro" name="my_robot">
    <!-- World frame — required by MoveIt Pro for TF resolution -->
    <link name="world" />
    <joint name="world_fixed" type="fixed">
        <parent link="world" />
        <child link="base_link" />
        <origin xyz="0 0 0" rpy="0 0 0" />
    </joint>

    <xacro:include filename="$(find my_robot_description)/urdf/my_robot.urdf" />
    <xacro:include filename="my_robot.ros2_control.xacro" />
    <xacro:my_robot_ros2_control name="MyRobotSystem" />
</robot>
```

Many robot descriptions instead expose a **xacro macro** that must be included and then invoked. Check the source package's xacro files — if the robot's links and joints are wrapped in a `<xacro:macro>` block, you need to call the macro:
```xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro" name="my_robot">
    <!-- World frame — required by MoveIt Pro for TF resolution -->
    <link name="world" />
    <joint name="world_fixed" type="fixed">
        <parent link="world" />
        <child link="base_link" />
        <origin xyz="0 0 0" rpy="0 0 0" />
    </joint>

    <!-- Include the macro definition file -->
    <xacro:include filename="$(find my_robot_description)/xacro/macro.my_robot.xacro" />
    <!-- Invoke the macro to generate the robot links and joints -->
    <xacro:my_robot namespace="" />

    <xacro:include filename="my_robot.ros2_control.xacro" />
    <xacro:my_robot_ros2_control name="MyRobotSystem" />
</robot>
```

**The `world` frame is required.** MoveIt Pro expects a `world` link as the TF root. If your source robot description doesn't define one (or defines it in its own top-level xacro that you're not using), you must add it in your wrapper. Without it, IMarker teleop will fail with "Timed out waiting for a TF between base_link and world" and IK queries may silently fail. Some robot macros define `world` internally — check before adding a duplicate, which will cause a URDF error.

Note: When using a macro-based description, you typically write your own `ros2_control.xacro` with `mock_components/GenericSystem` rather than using the source package's ros2_control (which often pulls in real hardware driver plugins).

**Watch for hidden xacro arg dependencies:** Some robot macros include sub-macros (e.g., Gazebo plugins, transmission definitions) that reference `$(arg ...)` xacro arguments. These are resolved at the **top-level file scope**, not inside the macro. If you include the macro file directly rather than the robot's own top-level xacro, those args won't be declared and xacro will fail with a `KeyError`. **Always check what `$(arg ...)` references exist in the included files** (including transitively-included files like gazebo or transmission macros) and declare matching `<xacro:arg name="..." default="..."/>` elements in your wrapper xacro.

**Watch for unwanted simulator dependencies:** Many robot description packages unconditionally include Gazebo, Ignition, or MuJoCo plugin macros that reference packages not in your workspace (e.g., a Gazebo controller config package). These will cause `PackageNotFoundError` at runtime even though you don't need the simulator. **Read the robot macro's includes** — if you see `<xacro:include>` for gazebo/ignition/mujoco config files, and those files reference packages you don't have, comment out the invocation of that macro in your local copy of the robot description.

**Always add a `grasp_link`** — even for arm-only configs (no gripper). Place it at the tool flange face, offset from the last link's frame origin. This is needed for correct IMarker placement (the IMarker appears at the SRDF chain tip, which should be at the flange face, not at the last joint center):

```xml
<link name="grasp_link" />
<joint name="grasp_link_joint" type="fixed">
    <parent link="<tool_flange_link>" />
    <child link="grasp_link" />
    <origin xyz="0 0 <offset_to_flange_face>" rpy="0 0 0" />
</joint>
```

To determine the Z offset: check the last link's inertial center-of-mass Z value and visual mesh extent. If the inertial Z is ~0.076m, the flange face is likely ~0.09–0.10m from the frame origin. Always verify visually after building — the IMarker should appear at the tool tip, not embedded in the wrist.

**`<robot>.ros2_control.xacro`** — Defines hardware interface for each joint:
- Use `mock_components/GenericSystem` for simulation
- List every joint with `<command_interface>` (position) and `<state_interface>` (position, velocity)

#### 3. SRDF (config/moveit/<robot>.srdf)

Critical decisions:
- **Planning group name**: Use `manipulator` — MoveIt Pro hardcodes this name in several places (joint_velocity_controller, JTAC, teleop objectives)
- **Chain**: Set `base_link` and `tip_link` for your kinematic chain. **If you have a gripper with a `grasp_link`, set `tip_link` to `grasp_link`** — this is required for pose jog (VFC) to work, as it expects `ee_frame` to be the tip of the planning group's chain. If there is no end effector, use the tool flange link (e.g., `link_6`, `tool0`). **How to identify the tool flange link**: find the last joint in the arm's kinematic chain — its child link is the tool flange. Not every robot defines a `tool0` or `tool_link`; some just use the final numbered link.
- **Collision pairs**: Run the MoveIt Setup Assistant or manually disable collisions between adjacent/never-colliding links. If Setup Assistant is not available, use this minimum checklist:
  1. All adjacent link pairs (connected by a joint) — `reason="Adjacent"`
  2. Near-neighbor pairs (separated by one joint) — `reason="Never"`
  3. `base_link` ↔ second link (often can't collide due to geometry) — `reason="Never"`
  4. If a gripper is attached: tool flange ↔ gripper base, and the link before the flange ↔ gripper base — see the End Effector section below for gripper-specific pairs
  5. **Test the zero/home pose** — if it reports self-collision, you're missing a `disable_collisions` entry

```xml
<group name="manipulator">
    <chain base_link="base_link" tip_link="tool_link"/>
</group>
<group_state name="home" group="manipulator">
    <!-- Set joint values to a known safe pose, NOT all zeros if that causes collision -->
</group_state>

<!-- End effector group: REQUIRED even for arm-only configs.
     MoveIt Pro's teleop UI crashes with "Couldn't find an end effector group"
     if this is missing. For arm-only (no gripper), create a minimal group
     referencing the tool flange link: -->
<group name="gripper">
    <link name="tool_link"/>
</group>
<end_effector name="gripper" parent_link="tool_link" group="gripper"/>
```

**The end effector group is always required.** Even if your robot has no physical gripper, you must define a minimal `gripper` group and `<end_effector>` in the SRDF. The MoveIt Pro web UI teleop mode requires it — without it, switching to teleop will crash with "Couldn't find an end effector group for the current planning group". For arm-only configs, use the tool flange link as the sole member. When you later add a real end effector, replace this stub with the full gripper group.

#### 3b. Kinematics (config/moveit/kinematics.yaml)

MoveIt Pro uses `PoseIKPlugin` (not KDL). The full configuration:

```yaml
manipulator:
  kinematics_solver: pose_ik_plugin/PoseIKPlugin
  target_tolerance: 0.001
  solve_mode: "optimize_distance"
  optimization_timeout: 0.005
```

The `solve_mode: "optimize_distance"` setting and `optimization_timeout` are important for IMarker teleop — without them, IK solutions may fail or be slow. The group key (`manipulator`) must match your SRDF planning group name.

#### 4. Controllers (config/control/<robot>_ros2_control.yaml)

MoveIt Pro expects these controllers to exist. They must be defined in your ros2_control yaml AND listed in either `controllers_active_at_startup` or `controllers_inactive_at_startup` in config.yaml:

| Controller | Type | Startup | Purpose |
|-----------|------|---------|---------|
| `joint_state_broadcaster` | `joint_state_broadcaster/JointStateBroadcaster` | **active** | Publishes /joint_states (required) |
| `joint_trajectory_admittance_controller` | `joint_trajectory_admittance_controller/JointTrajectoryAdmittanceController` | **active** | Trajectory execution with optional admittance |
| `velocity_force_controller` | `velocity_force_controller/VelocityForceController` | inactive | Pose-based teleoperation |
| `joint_velocity_controller` | `joint_velocity_controller/JointVelocityController` | inactive | Joint-based teleoperation |

The velocity/joint controllers claim the same hardware resources as the trajectory controller, so they cannot be active simultaneously. MoveIt Pro activates them on demand during teleop.

All three custom controllers (`JTAC`, `VFC`, `JVC`) must reference `planning_group_name: manipulator`.

JTAC works without a force/torque sensor — it falls back to behaving like a standard JTC. VFC should also work without an F/T sensor (set `ft_sensor_name: ""`), though pose jogging may require additional configuration to function correctly without one. If pose jog / IMarker doesn't work, check the VFC controller logs for errors.

Note: You can use a basic `joint_trajectory_controller/JointTrajectoryController` instead of JTAC at startup, but MoveIt Pro's teleop will still try to switch to JTAC, so it must at minimum be defined and inactive.

**Full controller parameter template** — replace joint names with your robot's joints. **All array parameters (`stop_accelerations`, `max_joint_velocity`, `max_joint_acceleration`, `max_cartesian_velocity`, `max_cartesian_acceleration`) must have one entry per arm joint.** The template below shows 6 entries for a 6-DOF arm — adjust for your robot's DOF count (e.g., 7 entries for a 7-DOF arm):

```yaml
controller_manager:
  ros__parameters:
    update_rate: 100

    joint_state_broadcaster:
      type: joint_state_broadcaster/JointStateBroadcaster

    joint_trajectory_admittance_controller:
      type: joint_trajectory_admittance_controller/JointTrajectoryAdmittanceController

    joint_velocity_controller:
      type: joint_velocity_controller/JointVelocityController

    velocity_force_controller:
      type: velocity_force_controller/VelocityForceController

joint_trajectory_admittance_controller:
  ros__parameters:
    planning_group_name: manipulator
    sensor_frames: ["<tool_flange_link>"]
    ee_frames: ["<tip_of_manipulator_chain>"]  # grasp_link if EE attached, or tool flange link if arm-only
    ft_sensor_name: ""
    stop_accelerations: [30.0, 30.0, 30.0, 30.0, 30.0, 30.0]
    ft_cutoff_frequency_ratio: 1.0
    ft_force_deadband: 0.0
    ft_torque_deadband: 0.0
    default_path_tolerance: 0.5  # Relax from tight default — prevents stitching failures after interrupted trajectories

joint_velocity_controller:
  ros__parameters:
    planning_group_name: manipulator
    command_interfaces: ["position"]
    max_joint_velocity:
      - 1.0
      - 1.0
      - 1.0
      - 1.0
      - 1.0
      - 1.0
    max_joint_acceleration:
      - 3.0
      - 3.0
      - 3.0
      - 3.0
      - 3.0
      - 3.0

velocity_force_controller:
  ros__parameters:
    planning_group_name: manipulator
    sensor_frame: <tip_of_manipulator_chain>  # grasp_link if EE attached, or tool flange link if arm-only
    ee_frame: <tip_of_manipulator_chain>      # must match SRDF chain tip
    # Note: VFC uses singular sensor_frame/ee_frame (NOT the plural form). JTAC uses the plural sensor_frames/ee_frames.
    ft_sensor_name: ""
    ft_force_deadband: 0.0
    ft_torque_deadband: 0.0
    stop_accelerations: [30.0, 30.0, 30.0, 30.0, 30.0, 30.0]
    max_joint_velocity:
      - 1.0
      - 1.0
      - 1.0
      - 1.0
      - 1.0
      - 1.0
    max_joint_acceleration:
      - 2.0
      - 2.0
      - 2.0
      - 2.0
      - 2.0
      - 2.0
    max_cartesian_velocity:
      - 0.25
      - 0.25
      - 0.25
      - 1.0
      - 1.0
      - 1.0
    max_cartesian_acceleration:
      - 2.0
      - 2.0
      - 2.0
      - 4.0
      - 4.0
      - 4.0
```

#### 5. Jog Configuration (config/moveit/)

The `planning_groups` and `controllers` arrays must have the same length — they are paired by index.

**`joint_jog.yaml`** — Joint-space teleop, uses Joint Velocity Controller:
```yaml
planning_groups: ['manipulator']
controllers: ['joint_velocity_controller']
```

**`pose_jog.yaml`** — Cartesian-space teleop, uses Velocity Force Controller:
```yaml
planning_groups: ['manipulator']
controllers: ['velocity_force_controller']
```

If your robot has no F/T sensor, pose jog via VFC won't work. You can point it to `joint_velocity_controller` as a fallback, but Cartesian jogging will be limited.

#### 6. Master Config (config/config.yaml)

This is the most complex file in the config package. **All sections shown below are required** — MoveIt Pro validates the config at startup and will refuse to launch if fields are missing. The template below is a complete, working starting point:

```yaml
hardware:
  simulated: true
  launch_control_node: true
  launch_robot_state_publisher: true
  robot_description:
    urdf:
      package: "<package>"
      path: "description/<robot>.urdf.xacro"
    srdf:
      package: "<package>"
      path: "config/moveit/<robot>.srdf"
    urdf_params:
      - use_fake_hardware: "true"

moveit_params:
  joint_group_name: "manipulator"    # Must match SRDF group name
  kinematics:
    package: "<package>"
    path: "config/moveit/kinematics.yaml"
  sensors_3d:
    package: "<package>"
    path: "config/moveit/sensors_3d.yaml"
  joint_limits:
    package: "<package>"
    path: "config/moveit/joint_limits.yaml"
  pose_jog:
    package: "<package>"
    path: "config/moveit/pose_jog.yaml"
  joint_jog:
    package: "<package>"
    path: "config/moveit/joint_jog.yaml"
  publish:
    planning_scene: true
    geometry_updates: true
    state_updates: true
    transforms_updates: true
  trajectory_execution:
    manage_controllers: true
    allowed_execution_duration_scaling: 2.0
    allowed_goal_duration_margin: 5.0
    allowed_start_tolerance: 0.01

ros2_control:
  config:
    package: "<package>"
    path: "config/control/<robot>_ros2_control.yaml"
  controllers_active_at_startup:
    - "joint_state_broadcaster"
    - "joint_trajectory_admittance_controller"
  controllers_inactive_at_startup:
    - "joint_velocity_controller"
    - "velocity_force_controller"

objectives:
  behavior_loader_plugins:
    core:
      - "moveit_pro::behaviors::CoreBehaviorsLoader"
      - "moveit_pro::behaviors::MTCCoreBehaviorsLoader"
      - "moveit_pro::behaviors::VisionBehaviorsLoader"
      - "moveit_pro::behaviors::ConverterBehaviorsLoader"
  objective_library_paths:
    core_objectives:
      package_name: "moveit_pro_objectives"
      relative_path: "objectives/core"
    motion_objectives:
      package_name: "moveit_pro_objectives"
      relative_path: "objectives/motion"
    perception_objectives:
      package_name: "moveit_pro_objectives"
      relative_path: "objectives/perception"
    <robot>_objectives:
      package_name: "<package>"
      relative_path: "objectives"
  waypoints_file:
    package_name: "<package>"
    relative_path: "waypoints/waypoints.yaml"
```

**Notes:**
- Every `moveit_params` sub-field (`kinematics`, `sensors_3d`, `joint_limits`, `pose_jog`, `joint_jog`) is **required** and must point to a file that exists in your package. MoveIt Pro will refuse to launch if any are missing.
- The `ros2_control.config` field (package + path to your controller yaml) is **required**.
- The `objectives` section must include `behavior_loader_plugins`, `objective_library_paths`, and `waypoints_file`. The core objective library paths (`core_objectives`, `motion_objectives`, `perception_objectives`) point to built-in MoveIt Pro objectives and should always be included. Add your own package's objectives as an additional entry.
- `waypoints_file` lives under `objectives`, not as a separate top-level section.

#### 7. Waypoints (waypoints/waypoints.yaml)

Must be a valid YAML list (not empty or comment-only). Initialize with at least one waypoint:

```yaml
- name: Home
  joint_group_names:
    - manipulator
  joint_state:
    header:
      frame_id: world
      stamp:
        sec: 0
        nanosec: 0
    name: [j1, j2, j3, j4, j5, j6]
    position: [0.0, 0.0, 0.0, 0.0, 0.0, 0.0]
    velocity: []
    effort: []
  multi_dof_joint_state:
    header:
      frame_id: ''
      stamp:
        sec: 0
        nanosec: 0
    joint_names: []
    transforms: []
    twist: []
    wrench: []
  description: ''
  favorite: false
```

MoveIt Pro will update this file when you save waypoints from the UI.

**Waypoint joint values are robot-specific.** The joint positions must be within your robot's joint limits and free of self-collision. Do not copy waypoint values from another robot's config. When creating waypoints for a new robot:

1. **Home**: Choose an "elbow up" or tucked configuration — not all-zeros, which is often a singularity or fully extended pose. Bend the elbow and wrist joints to a clearly non-singular pose.
2. **Ready** (or a second test waypoint): Choose a pose that is visibly different from Home — rotate the base joint and change the elbow angle. This ensures motion planning tests actually exercise the planner rather than no-op at the current position.
3. **Verify both poses visually** in the MoveIt Pro UI or RViz before using them in objectives. If either pose shows collision warnings, adjust the joint values.
4. **Joint count must match your robot** — a 7-DOF arm needs 7 values in the `position` array, a 6-DOF needs 6.

#### 8. Objectives (objectives/)

Every config should include at least one test objective to verify the motion planning pipeline works end-to-end. The "Move to Home" objective below retrieves a saved waypoint and plans/executes a motion to it:

**`objectives/move_to_home.xml`:**
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<root BTCPP_format="4" main_tree_to_execute="Move to Home">
  <BehaviorTree ID="Move to Home">
    <Control ID="Sequence" name="root">
      <Action
        ID="RetrieveWaypoint"
        waypoint_joint_state="{target_joint_state}"
        waypoint_name="Home"
        joint_group_name="manipulator"
      />
      <SubTree
        ID="Move to Joint State"
        target_joint_state="{target_joint_state}"
        joint_group_name="manipulator"
        controller_names="joint_trajectory_admittance_controller"
        controller_action_server="/joint_trajectory_admittance_controller/follow_joint_trajectory"
        execution_pipeline="jtac"
        velocity_scale_factor="0.5"
        acceleration_scale_factor="0.5"
      />
    </Control>
  </BehaviorTree>
  <TreeNodesModel>
    <SubTree ID="Move to Home">
      <MetadataFields>
        <Metadata runnable="true" />
        <Metadata subcategory="Motion" />
      </MetadataFields>
    </SubTree>
  </TreeNodesModel>
</root>
```

**Key points:**
- `RetrieveWaypoint` loads a named waypoint from `waypoints/waypoints.yaml`. The output port is `waypoint_joint_state` (not `waypoint`).
- `Move to Joint State` is a built-in SubTree (from `moveit_pro_objectives`) that handles planning and execution.
- `controller_names` and `controller_action_server` must match your active trajectory controller.
- `velocity_scale_factor` and `acceleration_scale_factor` (0.0–1.0) control execution speed — start conservative at 0.5.
- `runnable="true"` in the metadata makes the objective visible in the UI and callable from CLI.
- The `waypoint_name` must match a waypoint defined in `waypoints/waypoints.yaml`.
- **These objective files are reusable across robots** — they reference waypoint names, not joint values. The robot-specific part is in `waypoints.yaml`. You can copy `move_to_home.xml` and `move_to_ready.xml` to a new config without modification, as long as the waypoints file defines "Home" and "Ready" entries with valid poses for that robot.

**Testing from CLI:**
```bash
ros2 action send_goal --feedback /do_objective \
  moveit_studio_sdk_msgs/action/DoObjectiveSequence \
  "{objective_name: 'Move to Home'}"
```

#### 9. Launch File (launch/agent_bridge.launch.xml)

Standard boilerplate — rarely needs changes:

```xml
<launch>
  <include file="$(find-pkg-share moveit_studio_agent)/launch/studio_agent_bridge.launch.xml" />
  <include file="$(find-pkg-share moveit_studio_rest_api)/launch/rest_api.launch.xml" />
</launch>
```

#### 10. README

Create a `README.md` at the workspace root (not inside the config package). This is the public-facing documentation for the workspace. Include:

- Robot name and description (what arm, how many DOF)
- Link to the [MoveIt Pro product page](https://picknik.ai/moveit-pro/)
- Table of config packages in the workspace and what each one is for
- Where the robot description comes from (link to upstream repo)
- A screenshot of the robot in the MoveIt Pro web UI (taken after verification, with the trial dialog dismissed and the robot clearly visible)
- Setup instructions: clone (with `--recurse-submodules` if using submodules), configure, build, run
- Basic motion test command
- Stop command (`moveit_pro down`)

### Common Issues & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `agent_bridge.launch.xml not found` | Package not rebuilt after adding file | `moveit_pro build` |
| `Group 'manipulator' not found` | SRDF uses a different group name | Rename group to `manipulator` in SRDF and all config files |
| `Joint Velocity Controller parameters not found` | `joint_velocity_controller` not defined in ros2_control yaml | Add JVC to controller_manager and its params |
| `SwitchController: controller X not existing` | Controller not in ros2_control yaml | Add controller definition and register in config.yaml |
| `self-collision` at start | Missing `disable_collisions` in SRDF | Add the reported link pair to SRDF disable_collisions |
| `edit_waypoints service not available` | waypoints.yaml is empty or invalid | Initialize with a valid YAML list (at least one waypoint) |
| Teleop crashes: "Couldn't find an end effector group" | No `<end_effector>` defined in SRDF | Add a `gripper` group and `<end_effector>` to the SRDF — required even for arm-only configs. See SRDF section above. |
| Teleop jog doesn't work | joint_jog.yaml points to wrong controller | Set `controllers: ['joint_velocity_controller']` |
| `KDLKinematicsPlugin failed to load` | MoveIt Pro uses `PoseIKPlugin`, not KDL | Change kinematics.yaml solver to `pose_ik_plugin/PoseIKPlugin` |
| `sensor_frame` / `sensor_frames` parameter errors | JTAC/VFC frame parameters misconfigured | **JTAC** uses plural array form: `sensor_frames: ["<link>"]` and `ee_frames: ["<link>"]`. **VFC** uses singular form: `sensor_frame: <link>` and `ee_frame: <link>`. Both links must exist in the kinematic chain. |
| `No inverse kinematics solution found` (IMarker) | IK solver misconfigured or `ee_frame`/`ee_frames` not the chain tip | Ensure kinematics.yaml has `solve_mode: "optimize_distance"` and `optimization_timeout: 0.005`. Ensure SRDF chain tip = `grasp_link` and JTAC `ee_frames: ["grasp_link"]` / VFC `ee_frame: grasp_link`. |
| `Expected 'value' ... got '()' of type 'tuple'` | A YAML config file contains an empty list `[]` (e.g., `sensors: []`) | ROS2 parameters cannot represent empty lists — they become empty tuples in the launch system and crash. Remove the key entirely or give it a non-empty value |
| `parameter '' has invalid type: expected [double_array] got [not set]` | Controller (JTAC, VFC, or JVC) is missing required array parameters | JTAC needs `sensor_frames`, `ee_frames`, `ft_sensor_name`, `stop_accelerations`. VFC needs `sensor_frame`, `ee_frame`, `max_joint_velocity`, `max_joint_acceleration`. JVC needs `max_joint_velocity`, `max_joint_acceleration`. See controller template |
| `parameter 'state_publish_rate' has invalid type: expected [integer] got [double]` | Rate parameter is `100.0` instead of `100` | Use integer values (no decimal point) for `state_publish_rate` and `action_monitor_rate` |

**Critical: No empty lists in YAML config files.** ROS2 node parameters cannot represent empty lists. If a YAML file loaded as parameters contains `key: []`, the launch system converts it to an empty tuple `()` and crashes with the `Expected 'value'` error. Either remove the key or provide a valid non-empty value. This most commonly affects `sensors_3d.yaml` — if you have no 3D sensors, use:

```yaml
# Minimal sensors_3d.yaml for configs with no 3D sensors
octomap_resolution: 0.01
octomap_frame: "world"
```

Do **not** include `sensors: []`.

## Adding an End Effector

This section covers attaching an end effector to an existing arm config. The term "end effector" covers a wide range of tooling: parallel-jaw grippers, vacuum grippers, spray heads, welding torches, custom fixtures, and more. The steps below are written generically — you will need to read your specific end effector's URDF/xacro source files to determine the actual link names, joint names, joint types, and control interfaces.

### Before You Start: Read the End Effector Source

Before writing any config, open the end effector's xacro files and answer these questions:

1. **What is the macro name and what parameters does it accept?** Look for `<xacro:macro name="..." params="...">`.
2. **What links does it create?** Find all `<link name="...">` elements. Identify the base link (the one attached to the arm).
3. **What joints does it create?** Find all `<joint>` elements. Note which are `fixed`, `revolute`, `prismatic`, or `continuous`.
4. **Which joints are actuated vs. mimic?** Read the `ros2_control.xacro` — actuated joints have a `<command_interface>`. Mimic joints follow an actuated joint and should not be commanded directly.
5. **Does the macro add internal rotations?** Look for fixed joints with non-zero `rpy` values between the parent attachment point and the functional base. These compound with any rotation you apply externally.
6. **What are the joint limits and default positions?** Check `<limit>` tags and `initial_value` parameters in the ros2_control xacro.

For end effectors **without joints** (e.g., a fixed vacuum cup or spray head), several steps below simplify: there is no controller, no group states, no passive joints, and the SRDF group may contain only the links and `grasp_link`.

### Overview of Changes

Adding an end effector touches **every layer** of the config:

| File | What to add |
|------|-------------|
| `package.xml` | Dependency on end effector description package |
| `description/<robot>.urdf.xacro` | Include end effector macro, attach to tool flange, add `grasp_link` |
| `config/moveit/<robot>.srdf` | End effector group, end effector definition, named states (if applicable), passive joints (if applicable), collision pairs |
| `config/control/<robot>_ros2_control.yaml` | End effector controller in controller_manager + params (if actuated) |
| `config/config.yaml` | End effector controller in `controllers_active_at_startup` (if actuated), `urdf_params` for fake hardware |
| `objectives/` | Activate/deactivate objectives (e.g., open/close for grippers, on/off for vacuum) |

### Step 1: Add End Effector Description Dependency

The end effector's URDF package must be available in your workspace:
- Add as a git submodule, or
- Copy from another workspace into `src/`, or
- Clone directly into `src/`

**Do not use symlinks.** Symlinks do not survive Docker build context — the Docker build copies file contents, not symlink targets. Your build will succeed (colcon resolves the symlink on the host), but at runtime inside the container the package will be missing. Always copy the package.

**Binary package conflicts:** If `rosdep install` installs a binary version of the same package (e.g., `ros-humble-robotiq-description`), your source copy must override it. Add the package name to `allow-overriding` in `colcon-defaults.yaml`:
```yaml
build:
  allow-overriding:
    - robotiq_description
```

Add to `package.xml`:
```xml
<exec_depend>my_end_effector_description</exec_depend>
```

### Step 2: Attach End Effector in URDF XACRO

Include the end effector macro and attach it to the arm's tool flange link. The exact macro call depends entirely on your end effector — read the source xacro to determine the macro name and parameters.

```xml
<!-- Include the end effector macro definition -->
<xacro:include filename="$(find my_end_effector_description)/urdf/my_end_effector_macro.urdf.xacro" />

<!-- Invoke the macro, attaching to the arm's tool flange -->
<xacro:my_end_effector_macro
    name="MyEndEffectorHardwareInterface"
    prefix=""
    parent="<tool_flange_link>"
    use_fake_hardware="$(arg use_fake_hardware)">
    <origin xyz="0 0 0" rpy="0 0 ${pi}" />
</xacro:my_end_effector_macro>
```

**Key parameters:**
- `parent` — Your arm's tool flange link (e.g., `link_6`, `wrist3_link`, `tool0`). See Step 3 above for how to identify this.
- `origin` rotation — **Start with `rpy="0 0 ${pi}"` (180 Z rotation) as the default.** Most end effector URDF macros define their Z axis pointing into the mounting flange. Without this rotation, the tool will point back toward the arm. **However**, some macros add their own internal rotation via fixed joints between the parent attachment and the base. Read the macro source first — look for fixed joints with non-zero `rpy` values. Your externally-applied rotation compounds with any internal one, so you may need to adjust. **Always visually verify after building.**
- `origin` Z offset — The `xyz` Z value must place the end effector flush against the arm's **flange face**. Many arm URDFs place the last link's origin at the joint center, not at the flange surface. If the end effector appears embedded inside the wrist, add a positive Z offset. Check the arm's URDF for the last link's visual mesh or inertial origin to estimate the flange face position. **Always visually verify there is no overlap or gap.**
- `use_fake_hardware` — Pass through from config.yaml for sim/real switching. Not all end effector macros accept this parameter — check the macro definition.
- `prefix` — Use `""` for single-arm setups; use a prefix for dual-arm or multi-tool setups.

**Add a grasp/tool link** — a planning frame that represents where the tool acts on the world:

```xml
<link name="grasp_link" />
<joint name="grasp_link_joint" type="fixed">
    <parent link="<end_effector_base_link>" />
    <child link="grasp_link" />
    <origin xyz="0 0 <offset>" rpy="0 0 0" />
</joint>
```

The purpose of `grasp_link` is to give the motion planner a target frame. Where you place it depends on the type of end effector:
- **Parallel-jaw gripper**: Between the fingertips at the center of the grasp. Offset Z from the gripper base by approximately the finger length.
- **Vacuum gripper**: At the suction cup face.
- **Spray head / torch**: At the nozzle tip or focal point.
- **Custom fixture**: At whatever point should align with the target object.

Check the end effector's URDF or CAD to determine the appropriate offset distance from the base link.

### Step 3: Update SRDF

Add these sections to the SRDF. The content depends on what joints and links your end effector has.

**End effector planning group** — include the `grasp_link` and all non-fixed joints from the end effector. To determine this list, walk the URDF tree of the end effector and include every `revolute`, `prismatic`, or `continuous` joint:

```xml
<group name="gripper">
    <link name="grasp_link"/>
    <joint name="<ee_joint_1>"/>
    <joint name="<ee_joint_2>"/>
    <!-- ... include all non-fixed joints from the end effector -->
</group>
```

For end effectors with no movable joints (e.g., a fixed vacuum cup), the group may contain only the `grasp_link`.

**Named states** — define positions for discrete end effector states. For a parallel-jaw gripper this is typically open/close. For other end effectors, define whatever named states are meaningful (or omit if there are none). Only include joints that are actuated or that MoveIt should plan for — not mimic joints:

```xml
<group_state name="open" group="gripper">
    <joint name="<actuated_joint>" value="<open_value>"/>
</group_state>
<group_state name="close" group="gripper">
    <joint name="<actuated_joint>" value="<close_value>"/>
</group_state>
```

To find the correct open/close values: read the end effector's `ros2_control.xacro`. The `initial_value` in the actuated joint's `<state_interface>` is typically one end of the range. The `<limit>` tags in the URDF macro define the full range. These values vary between models — always check.

**End effector definition** — links the end effector group to the arm:
```xml
<end_effector name="gripper" parent_link="<tool_flange_link>" group="gripper"/>
```

**Passive joints** — if the end effector has mimic joints (joints that mechanically follow an actuated joint), declare them as passive so MoveIt does not try to plan for them independently:

```xml
<passive_joint name="<mimic_joint_1>"/>
<passive_joint name="<mimic_joint_2>"/>
<!-- ... one entry per mimic joint -->
```

To identify mimic joints: look for `<mimic joint="..." multiplier="..." offset="..."/>` tags inside joint definitions in the end effector's URDF xacro. If there are no mimic joints, skip this section.

**Collision pairs** — disable self-collision between end effector links and between end effector and arm wrist. The general principle is to disable pairs that either cannot physically collide due to the kinematic structure, or that are directly connected:

- All adjacent link pairs within the end effector (parent-child in the URDF) — `reason="Adjacent"`
- Tool flange link ↔ end effector base link (and any intermediate links) — `reason="Adjacent"`
- The link before the tool flange ↔ end effector base — `reason="Never"`
- Any link pairs within the end effector that geometrically cannot collide in any configuration — `reason="Never"`

For complex end effectors (e.g., multi-finger grippers with many links), **disable ALL internal link pairs exhaustively.** MoveIt's collision padding (typically 0.01m) causes false self-collision positives between nearby gripper links, especially in the closed position. Trying to selectively disable pairs leads to whack-a-mole — a new collision pair appears each time you fix one. The reliable approach is to generate all N*(N-1)/2 pairs for the gripper's links and disable them all. A simple script can generate these:

```python
gripper_links = ["base_link", "left_knuckle", "left_finger", ...]  # your gripper's links
for i, a in enumerate(gripper_links):
    for b in gripper_links[i+1:]:
        print(f'  <disable_collisions link1="{min(a,b)}" link2="{max(a,b)}" reason="Never"/>')
```

### Step 4: Add End Effector Controller

**Skip this step if your end effector has no actuated joints** (e.g., a fixed vacuum cup controlled by a separate I/O system outside ros2_control).

In `config/control/<robot>_ros2_control.yaml`, register the controller and add its parameters. The controller type depends on the end effector:

**For grippers using `GripperActionController`** (standard for single-DOF grippers):

```yaml
# In controller_manager ros__parameters:
<ee_controller_name>:
  type: position_controllers/GripperActionController
```

```yaml
# Controller parameters:
<ee_controller_name>:
  ros__parameters:
    default: true
    joint: <actuated_joint_name>
    allow_stalling: true
    stall_timeout: 0.05
    goal_tolerance: 0.02
```

**To find the actuated joint**: read the end effector's `ros2_control.xacro`. The joint with a `<command_interface>` that is NOT a mimic joint is the actuated one. There is typically only one for single-DOF grippers.

For end effectors with other control schemes (multi-DOF hands, pneumatic actuators, custom hardware), you will need to use the appropriate controller type for your hardware. The pattern above is specific to `GripperActionController`.

### Step 5: Update config.yaml

If the end effector has an actuated controller, add it to active startup:
```yaml
controllers_active_at_startup:
  - "joint_state_broadcaster"
  - "<arm_controller>"
  - "<ee_controller_name>"    # add this
```

If the end effector macro accepts a `use_fake_hardware` parameter, pass it through via `urdf_params` so the end effector uses mock hardware in sim:
```yaml
robot_description:
  urdf:
    package: "<package>"
    path: "description/<robot>.urdf.xacro"
  srdf:
    package: "<package>"
    path: "config/moveit/<robot>.srdf"
  urdf_params:
    - use_fake_hardware: "true"
```

**`urdf_params` for sim/hardware switching:** For sim configs, hardcode `use_fake_hardware: "true"`. Use the config inheritance system (`based_on_package`) to override to `"false"` in the hardware config. The `%>> hardware.simulated` substitution syntax exists but requires MoveIt Pro 8.8+ and may not work in all deployment contexts — hardcoding is more reliable.

### Step 6: Add End Effector Objectives

Create objectives to activate and deactivate the end effector. For grippers using `GripperActionController`, use `MoveGripperAction`:

**`objectives/open_gripper.xml`:**
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<root BTCPP_format="4" main_tree_to_execute="Open Gripper">
  <BehaviorTree ID="Open Gripper">
    <Action ID="MoveGripperAction"
      gripper_command_action_name="/<ee_controller_name>/gripper_cmd"
      position="<open_position_value>"
    />
  </BehaviorTree>
  <TreeNodesModel>
    <SubTree ID="Open Gripper">
      <MetadataFields>
        <Metadata runnable="true" />
        <Metadata subcategory="Gripper" />
      </MetadataFields>
    </SubTree>
  </TreeNodesModel>
</root>
```

**`objectives/close_gripper.xml`:** Same structure with `position="<close_position_value>"`.

The `gripper_command_action_name` must match `/<controller_name>/gripper_cmd`. The position values come from the joint limits you identified in "Before You Start."

For end effectors that are not grippers (vacuum, spray, etc.), you will need different behavior tree actions appropriate to your hardware — `MoveGripperAction` is specific to `GripperActionController`.

### End Effector Verification Steps

After building, always check these before moving on:
1. **Visual check**: Launch MoveIt Pro and verify the end effector is oriented correctly in the 3D view. For grippers, fingers should point **away** from the arm. If the tool points toward the wrist, adjust the rotation in the origin.
2. **Actuation** (if applicable): Run the activate/deactivate objectives (e.g., Open/Close Gripper) to verify the controller is working.
3. **Teleop**: Verify joint jog still works with the end effector attached.
4. **Collision**: Move the arm through various poses and check for unexpected collision warnings between end effector and arm links.

### End Effector Gotchas

- **Wrong orientation (most common issue)**: The default orientation (`rpy="0 0 0"`) is almost always wrong. Most end effector macros need `rpy="0 0 ${pi}"` to face the correct direction. **Always apply the 180 Z rotation as the starting point, then adjust based on visual verification.** Some macros add their own internal rotation — your external rotation compounds with it.
- **Mimic joints** (multi-finger grippers): Some grippers use mimic joints where only one joint is actuated and the others follow mechanically. If you see "multiple command interfaces" errors, check that only the actuated joint has a `<command_interface>` in the ros2_control xacro. Declare all mimic joints as `<passive_joint>` in the SRDF.
- **Separate hardware interfaces**: The end effector typically runs as a separate `ros2_control` `<system>` from the arm. They coexist in the same controller manager but are independently managed.
- **End effector description package not found**: The description package must be in your workspace or installed as a binary. A symlink works for dev; for Docker builds, add it as a submodule or copy.
- **Collision spam**: If MoveIt reports many self-collisions involving end effector links, you need more `disable_collisions` entries in the SRDF. Walk all link pairs systematically.
- **End effectors with no physical joints (vacuum grippers, etc.)**: Vacuum grippers have no physical moving joint, but in MoveIt Pro they typically still use a **virtual joint** + `GripperActionController` for on/off control. The pattern is:
  1. Add a `prismatic` or `revolute` joint in the URDF with a small range (e.g., 0 to 1) that represents "off" to "on". This joint has no physical geometry change — it's just a control interface.
  2. Add this virtual joint to the `ros2_control.xacro` with position command/state interfaces.
  3. Use `GripperActionController` with this joint, same as a jaw gripper.
  4. Create "Activate Vacuum" / "Deactivate Vacuum" objectives using `MoveGripperAction` with position 1.0 and 0.0.

  See `factory_sim/description/suction_tool.urdf` for a working example. If you prefer not to use a virtual joint (e.g., for a real robot using digital I/O), the actuation would use `SetIO` behavior instead, but for mock/sim configs the virtual joint approach is simpler and consistent.
