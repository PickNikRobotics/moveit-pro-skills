---
name: ros2-cli
description: Guide for working with ROS 2 systems via CLI. Covers discovery, topics, services, actions, parameters, launch files, colcon builds, package management, debugging, and common message types. Use when the user asks about ROS 2 commands, inspecting nodes/topics/services, building packages, launching nodes, or debugging a ROS 2 system.
---

# ROS 2 CLI

Reference for non-interactive ROS 2 command-line usage. Applies to any ROS 2 system — host, container, or remote.

## Agent Guidelines

### Source the Workspace First

Before running any `ros2` or `colcon` command, source the workspace setup file:

```bash
source /opt/ros/${ROS_DISTRO}/setup.bash        # Base ROS 2 install
source ~/ws/install/setup.bash                   # Overlay workspace (if applicable)
```

Chain with `&&` to keep commands non-interactive:
```bash
source /opt/ros/humble/setup.bash && ros2 topic list
```

### Non-Interactive Only

All commands in this skill are non-interactive. Never use:
- `ros2 run` for long-running GUI tools without backgrounding
- Interactive Python shells (`ros2 run ... --ros-args` that drop to REPL)
- `colcon build` without `--event-handlers=console_cohesion+` if you need structured output

### Timeouts

Many `ros2` commands wait indefinitely for discovery. Use `--spin-time` or `--timeout` where available, or wrap with a shell timeout:
```bash
timeout 10 ros2 topic echo /joint_states --once
```

## Discovery & Introspection

### List Everything

```bash
ros2 node list                     # All active nodes
ros2 topic list                    # All active topics
ros2 topic list -t                 # Topics with their message types
ros2 service list                  # All active services
ros2 action list                   # All active actions
ros2 action list -t                # Actions with their types
```

### Node Details

```bash
ros2 node info /my_node            # Subscribers, publishers, services, actions
```

### Interface Inspection

```bash
ros2 interface show geometry_msgs/msg/Twist      # Message fields
ros2 interface show std_srvs/srv/SetBool          # Service request/response
ros2 interface show control_msgs/action/FollowJointTrajectory  # Action goal/result/feedback
ros2 interface list                                # All available interfaces
ros2 interface packages                            # Packages that provide interfaces
```

## Topics

### Subscribe (Read Data)

```bash
ros2 topic echo /joint_states --once               # Single message, then exit
ros2 topic echo /joint_states --once --no-arr       # Truncate arrays
ros2 topic echo /joint_states -n 5                  # First 5 messages, then exit
ros2 topic echo /cmd_vel --field linear.x           # Single field only
```

### Publish

```bash
# Publish once
ros2 topic pub --once /cmd_vel geometry_msgs/msg/Twist \
  "{linear: {x: 0.5, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.3}}"

# Publish at 10 Hz (runs until killed — background or use timeout)
timeout 5 ros2 topic pub /cmd_vel geometry_msgs/msg/Twist \
  "{linear: {x: 0.5}, angular: {z: 0.3}}" -r 10
```

### Topic Info

```bash
ros2 topic type /joint_states                      # Message type
ros2 topic info /joint_states                      # Type, publisher/subscriber count
ros2 topic info /joint_states --verbose             # Detailed QoS info per endpoint
```

## Services

### Call a Service

```bash
ros2 service call /service_name std_srvs/srv/SetBool "{data: true}"

ros2 service call /controller_manager/list_controllers \
  controller_manager_msgs/srv/ListControllers {}

ros2 service call /compute_fk moveit_msgs/srv/GetPositionFK \
  "{header: {frame_id: 'base_link'}, fk_link_names: ['tool0'], robot_state: {joint_state: {name: ['joint_1'], position: [0.0]}}}"
```

### Service Info

```bash
ros2 service type /service_name                    # Service type
ros2 service find std_srvs/srv/SetBool             # Find services of a given type
```

## Actions

### Send a Goal

```bash
ros2 action send_goal /follow_joint_trajectory \
  control_msgs/action/FollowJointTrajectory \
  "{trajectory: {joint_names: ['joint_1'], points: [{positions: [1.0], time_from_start: {sec: 2}}]}}"

# With feedback printing
ros2 action send_goal /navigate_to_pose nav2_msgs/action/NavigateToPose \
  "{pose: {header: {frame_id: 'map'}, pose: {position: {x: 1.0, y: 2.0}}}}" --feedback
```

### Action Info

```bash
ros2 action info /action_name                      # Type, clients, servers
ros2 action type /action_name                      # Action type
```

## Parameters

```bash
ros2 param list /my_node                           # All parameters on a node
ros2 param get /my_node param_name                 # Get value
ros2 param set /my_node param_name value           # Set value
ros2 param describe /my_node param_name            # Type, description, constraints
ros2 param dump /my_node                           # Dump all params as YAML
ros2 param load /my_node params.yaml               # Load params from file
```

## Launch Files

### Running Launch Files

```bash
ros2 launch my_package my_launch.launch.py
ros2 launch my_package my_launch.launch.py arg1:=value1 arg2:=value2

# Show arguments a launch file accepts
ros2 launch my_package my_launch.launch.py --show-args

# Launch with debug logging
ros2 launch my_package my_launch.launch.py --debug
```

### Common Launch Arguments

```bash
# ROS args passed to all nodes in the launch
ros2 launch my_package my_launch.launch.py \
  --ros-args --log-level DEBUG

# Remap a topic for all nodes
ros2 launch my_package my_launch.launch.py \
  --ros-args -r /old_topic:=/new_topic
```

## Building with colcon

### Build

```bash
colcon build                                        # Build entire workspace
colcon build --packages-select my_pkg               # Build one package
colcon build --packages-up-to my_pkg                # Build package + dependencies
colcon build --symlink-install                       # Symlink instead of copy (faster iteration)
colcon build --cmake-args -DCMAKE_BUILD_TYPE=Release
colcon build --cmake-args -DCMAKE_BUILD_TYPE=Debug
colcon build --packages-select my_pkg --cmake-args -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
```

### Test

```bash
colcon test                                         # Run all tests
colcon test --packages-select my_pkg                # Test one package
colcon test --packages-select my_pkg --ctest-args -R test_name  # Run specific test
colcon test-result --verbose                        # Show test results summary
colcon test-result --all                            # Include passed tests
```

### Clean

```bash
rm -rf build/ install/ log/                         # Full clean (no colcon clean by default)
colcon build --packages-select my_pkg --cmake-clean-first  # Clean rebuild of one package
```

## Package Management

### Inspect Packages

```bash
ros2 pkg list                                       # All installed packages
ros2 pkg list | grep moveit                         # Filter by name
ros2 pkg prefix my_package                          # Install prefix path
ros2 pkg xml my_package                             # Print package.xml
ros2 pkg executables my_package                     # List executables in a package
```

### package.xml Dependency Types

| Tag | Meaning |
|-----|---------|
| `<depend>` | Build + export + exec dependency (shorthand) |
| `<build_depend>` | Needed at build time only |
| `<exec_depend>` | Needed at runtime only |
| `<test_depend>` | Needed for testing only |
| `<buildtool_depend>` | Build system dependency (e.g., `ament_cmake`) |

### Finding Package Files

```bash
# Find where a package is installed
ros2 pkg prefix my_package

# List share files (launch, config, etc.)
ls $(ros2 pkg prefix my_package)/share/my_package/

# Find a specific file type
find $(ros2 pkg prefix my_package)/share -name "*.launch.py"
```

## Debugging

### Log Levels

```bash
# Set log level for a node at launch
ros2 run my_package my_node --ros-args --log-level DEBUG

# Set log level for a specific logger
ros2 run my_package my_node --ros-args --log-level my_package:=DEBUG

# Change log level at runtime
ros2 service call /my_node/set_logger_level rcl_interfaces/srv/SetLoggerLevel \
  "{logger_name: 'my_package', level: 10}"
# Levels: DEBUG=10, INFO=20, WARN=30, ERROR=40, FATAL=50
```

### Topic Diagnostics

```bash
ros2 topic hz /joint_states                         # Publishing rate
ros2 topic bw /camera/image_raw                     # Bandwidth usage
ros2 topic delay /joint_states                      # Message delay (requires header)
```

Use `timeout` to avoid indefinite blocking:
```bash
timeout 5 ros2 topic hz /joint_states
```

### TF Debugging

```bash
# View TF tree (generates frames.pdf)
ros2 run tf2_tools view_frames

# Look up a specific transform
ros2 run tf2_ros tf2_echo base_link tool0

# Monitor TF for issues
ros2 run tf2_tools tf2_monitor
```

### System Diagnostics

```bash
ros2 doctor                                         # Check system health
ros2 doctor --report                                # Detailed diagnostic report
ros2 wtf                                            # Alias for ros2 doctor
```

### Component / Lifecycle

```bash
ros2 lifecycle list /my_lifecycle_node              # Show available transitions
ros2 lifecycle get /my_lifecycle_node               # Current state
ros2 lifecycle set /my_lifecycle_node activate       # Trigger transition

ros2 component list /my_container                   # List loaded components
```

## Common Message Types

### geometry_msgs

| Type | Key Fields |
|------|-----------|
| `Twist` | `linear.{x,y,z}`, `angular.{x,y,z}` |
| `Pose` | `position.{x,y,z}`, `orientation.{x,y,z,w}` |
| `PoseStamped` | `header`, `pose` |
| `Transform` | `translation.{x,y,z}`, `rotation.{x,y,z,w}` |
| `TransformStamped` | `header`, `child_frame_id`, `transform` |
| `Wrench` | `force.{x,y,z}`, `torque.{x,y,z}` |
| `Point` | `x`, `y`, `z` |
| `Quaternion` | `x`, `y`, `z`, `w` |

### sensor_msgs

| Type | Key Fields |
|------|-----------|
| `JointState` | `name[]`, `position[]`, `velocity[]`, `effort[]` |
| `Image` | `header`, `height`, `width`, `encoding`, `data` |
| `PointCloud2` | `header`, `height`, `width`, `fields[]`, `data` |
| `LaserScan` | `ranges[]`, `angle_min`, `angle_max`, `range_min`, `range_max` |
| `Imu` | `orientation`, `angular_velocity`, `linear_acceleration` |
| `CameraInfo` | `K` (intrinsic matrix), `D` (distortion), `width`, `height` |

### std_msgs

| Type | Key Fields |
|------|-----------|
| `String` | `data` (string) |
| `Bool` | `data` (bool) |
| `Int32` / `Float64` / etc. | `data` (numeric) |
| `Header` | `stamp.{sec,nanosec}`, `frame_id` |
| `ColorRGBA` | `r`, `g`, `b`, `a` |

### trajectory_msgs

| Type | Key Fields |
|------|-----------|
| `JointTrajectory` | `joint_names[]`, `points[].positions[]`, `points[].time_from_start` |
| `JointTrajectoryPoint` | `positions[]`, `velocities[]`, `accelerations[]`, `effort[]`, `time_from_start` |

### YAML Syntax for CLI

When passing messages on the command line, use YAML:
```bash
# Nested message
"{linear: {x: 1.0, y: 0.0, z: 0.0}}"

# Arrays
"{name: ['j1', 'j2'], position: [0.1, 0.2]}"

# Stamped messages (let ROS fill the timestamp)
"{header: {frame_id: 'base_link'}, pose: {position: {x: 1.0}}}"
```
