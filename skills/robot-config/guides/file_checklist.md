# Complete File Checklist

Use this checklist when creating a new config package.

- [ ] `CMakeLists.txt` — installs all directories
- [ ] `package.xml` — declares dependencies
- [ ] `config/config.yaml` — master config, group name = `manipulator`
- [ ] `config/control/<robot>_ros2_control.yaml` — all 4 controllers defined (JSB, JTAC, JVC, VFC)
- [ ] `config/moveit/<robot>.srdf` — group named `manipulator`, collision pairs disabled, `gripper` group + `<end_effector>` defined (required for teleop even on arm-only configs)
- [ ] `config/moveit/kinematics.yaml` — solver = `pose_ik_plugin/PoseIKPlugin` (NOT KDL), with `solve_mode: "optimize_distance"` and `optimization_timeout: 0.005`
- [ ] `config/moveit/joint_limits.yaml` — limits for every joint (position and velocity limits from the URDF; `max_acceleration` from the manufacturer's datasheet — if unavailable, start conservative and tune based on testing)
- [ ] `config/moveit/joint_jog.yaml` — controller = `joint_velocity_controller`
- [ ] `config/moveit/pose_jog.yaml` — controller = `velocity_force_controller`
- [ ] `config/moveit/sensors_3d.yaml` — minimal if no 3D sensors (see note below about empty lists)
- [ ] `description/<robot>.urdf.xacro` — includes URDF + ros2_control xacro
- [ ] `description/<robot>.ros2_control.xacro` — mock_components for sim
- [ ] `launch/agent_bridge.launch.xml` — standard two-include boilerplate
- [ ] `objectives/move_to_home.xml` — arm movement test objective (uses `RetrieveWaypoint` + `Move to Joint State`)
- [ ] `waypoints/waypoints.yaml` — valid YAML list with at least one waypoint
- [ ] **Build**: `moveit_pro build` succeeds (ask user permission before running)
- [ ] **Run**: `moveit_pro run` launches and reaches `You can start planning now!` (ask user permission before running)
- [ ] **README.md** at workspace root — robot description, MoveIt Pro link, config package table, setup/build/run/stop instructions

### If adding an end effector:
- [ ] Gripper description package in workspace (submodule, symlink, or copy)
- [ ] `package.xml` — `exec_depend` on gripper description package
- [ ] `description/<robot>.urdf.xacro` — gripper macro included, attached to tool flange, `grasp_link` added
- [ ] `config/moveit/<robot>.srdf` — `gripper` group, `end_effector`, open/close states, passive joints, collision pairs
- [ ] `config/control/` — `robotiq_gripper_controller` (or equivalent) in controller_manager + params
- [ ] `config/config.yaml` — gripper controller in `controllers_active_at_startup`, `urdf_params` for fake hardware
- [ ] `objectives/open_gripper.xml` and `close_gripper.xml`
- [ ] **Build and run**: Rebuild with `moveit_pro build`, relaunch with `moveit_pro run`, verify gripper appears correctly in 3D view (ask user permission before running)
