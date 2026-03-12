---
name: moveit-pro-cli
description: Guide for using the MoveIt Pro platform programmatically and via CLI. Covers the moveit_pro command, running/saving objectives, camera streams, RViz, dev containers, building workspaces, and ACM updates. Use when the user asks about MoveIt Pro commands, running objectives, accessing cameras, debugging containers, building workspaces, or programmatic robot control.
---

# MoveIt Pro

MoveIt Pro is a Docker-based robotic manipulation platform. The `moveit_pro` CLI manages configuration, building, running, and debugging. This skill covers programmatic, non-interactive usage.

## When to Use

- Running or configuring MoveIt Pro
- Executing or saving objectives programmatically
- Viewing camera streams or launching RViz
- Working inside dev containers with ROS 2
- Building workspaces, running tests
- Updating collision matrices (ACM) or URDF workflows

## Agent Guidelines

### Non-Interactive Commands

Many `moveit_pro` commands prompt for input if flags are missing. Always provide all flags to avoid prompts an agent cannot navigate.

**Bad** (will prompt):
```bash
moveit_pro configure
moveit_pro run
moveit_pro new config
moveit_pro setup_workspace
```

**Good** (fully non-interactive):
```bash
moveit_pro configure -c my_config -w /path/to/workspace
moveit_pro run -c my_config -w /path/to/workspace --no-browser
moveit_pro new config -n my_pkg -b lab_sim --no-input
moveit_pro setup_workspace https://github.com/org/repo.git
```

### Blocking Commands

These commands block the terminal until stopped with Ctrl+C or `moveit_pro down`. Run them in a background shell or separate terminal.

- `moveit_pro run` — Starts the full application (runtime + web UI). Blocks until stopped.
- `moveit_pro run --headless` — Starts only the runtime (no web frontend). Still blocks.
- `moveit_pro rviz` — Opens the RViz GUI. Blocks until closed.

To stop a running instance from another terminal:
```bash
moveit_pro down
```

### Interactive Shells (Use docker exec Instead)

`moveit_pro dev` and `moveit_pro shell` open interactive bash sessions an agent cannot use. Instead, run commands directly via `docker exec`:

```bash
# Find the running MoveIt Pro container
CONTAINER=$(docker ps --filter "name=moveit_pro" --format "{{.Names}}" | head -1)

# Run commands non-interactively
docker exec "$CONTAINER" bash -c "source /opt/overlay_ws/install/setup.bash && ros2 topic list"
docker exec "$CONTAINER" bash -c "source /opt/overlay_ws/install/setup.bash && source ~/user_ws/install/setup.bash && ros2 node list"
```

### Environment Variables

Pre-set these to bypass first-run prompts:
```bash
export MOVEIT_HOST_USER_WORKSPACE="/path/to/workspace"
export MOVEIT_CONFIG_PACKAGE="lab_sim"
export MOVEIT_LICENSE_KEY="XXXXXX-XXXXXX-XXXXXX-XXXXXX-XXXXXX-XXXXXX"
```

## CLI Reference

**Global options:** `--version`, `--verbose / -v`, `--help`

**Configuration** (stored at `~/.config/moveit_pro/moveit_pro_config.9.yaml`):
```bash
moveit_pro configure -c my_config -w PATH           # Set config package and workspace
moveit_pro setup_workspace URL_OR_PATH         # Setup workspace (always pass identifier)
moveit_pro envfile                             # Generate .env file
moveit_pro envfile --all                       # Generate .env with all variables
```

**Building** (`moveit_pro build` with no subcommand builds everything):
```bash
moveit_pro build                               # Build Docker image + workspace
moveit_pro build user_workspace                # Build workspace only (colcon build)
moveit_pro build user_image                    # Build Docker image only
moveit_pro build all                           # Build both explicitly
```
Options: `--user-workspace / -w PATH`, `--colcon-args ARGS`
```bash
moveit_pro build user_workspace --colcon-args "--packages-select my_package"
```

**Running:**
```bash
moveit_pro run -c my_config -w PATH --no-browser     # Start without browser (recommended for agents)
moveit_pro run --headless / -h                  # Start without frontend
moveit_pro run --no-drivers / -n                # Start without hardware drivers
moveit_pro run --only-drivers / -o              # Start drivers only
moveit_pro run --list / -l                      # List available config packages
moveit_pro run -v                               # Run with verbose output
```

**Testing:**
```bash
moveit_pro test -c my_config -w PATH                 # Build and run colcon test
moveit_pro test --colcon-args "--packages-select my_package"
moveit_pro test --no-drivers --headless         # Test without drivers, headless
```

**Stopping and utilities:**
```bash
moveit_pro down                                # Stop all containers
moveit_pro down --rmi                          # Stop + remove Docker images
moveit_pro rviz                                # Launch RViz visualizer (blocks terminal)
moveit_pro logs                                # Launch log viewer
moveit_pro export_logs                         # Export logs to ZIP
moveit_pro update_acm URDF_PATH                # Update Allowed Collision Matrix
```

**Creating packages** (always use `--no-input` for non-interactive):
```bash
moveit_pro new config -n my_robot_config -b lab_sim -o /path/to/output --no-input
```
Options: `--output-dir/-o`, `--package-name/-n`, `--based-on/-b`, `--no-input`

## Creating Objectives

Objectives are behavior trees in XML format (BehaviorTree.CPP v4). Each objective is a tree of behavior nodes that execute sequentially, in parallel, or with fallback logic.

### Structure

```xml
<root BTCPP_format="4" main_tree_to_execute="My Objective">
  <BehaviorTree ID="My Objective">
    <Control ID="Sequence" name="TopLevelSequence">
      <!-- Behavior nodes go here -->
    </Control>
  </BehaviorTree>
</root>
```

### Key Patterns

- **Sequence** — runs children in order, fails if any child fails
- **Fallback** — tries children in order, succeeds on first success
- **Parallel** — runs children concurrently
- **SubTree** — reuses an existing objective as a node (with port remapping)
- **Decorators** — wrap nodes for retries, conditions, timeouts

### Best Practices

- **Reuse existing objectives** as SubTree nodes instead of rebuilding logic
- **Validate behaviors exist** before adding them — each node's `ID` must match a registered behavior plugin
- **Check port schemas** — nodes have typed InputPorts and OutputPorts; use the blackboard for data flow between nodes
- **Preserve metadata** when editing — keep Metadata nodes, `main_tree_to_execute`, tree IDs, and `_collapsed` attributes

### Objective Files

Objectives are `.xml` files in the config package's `objectives/` directory. On the host, the user workspace is mounted into the container, so edits on either side are reflected:

- **Host:** `~/moveit_pro/<workspace>/src/<config_package>/objectives/`
- **Container:** `~/user_ws/src/<config_package>/objectives/`

The workspace path is set via `moveit_pro configure -w PATH` and stored in `~/.config/moveit_pro/moveit_pro_config.9.yaml`.

The REST API (port 3200) also provides CRUD operations for objectives — see Swagger docs at `http://localhost:3200/docs` when MoveIt Pro is running.

## Running Objectives Programmatically

MoveIt Pro must be running (`moveit_pro run`) before executing objectives.

### Via docker exec (inside container, non-interactive)

```bash
CONTAINER=$(docker ps --filter "name=moveit_pro" --format "{{.Names}}" | head -1)

# Run an existing objective by name
docker exec "$CONTAINER" bash -c '
  source /opt/overlay_ws/install/setup.bash &&
  ros2 action send_goal /do_objective moveit_studio_sdk_msgs/action/DoObjectiveSequence \
    "{objective_name: '\''Close Gripper'\''}"
'

# Run a custom objective via XML
docker exec "$CONTAINER" bash -c '
  source /opt/overlay_ws/install/setup.bash &&
  ros2 action send_goal /do_objective moveit_studio_sdk_msgs/action/DoObjectiveSequence '\''
objective_xml_string: |
  <root BTCPP_format="4" main_tree_to_execute="my_objective">
    <BehaviorTree ID="my_objective">
      <Control ID="Sequence" name="TopLevelSequence">
        <Action ID="AlwaysSuccess"/>
      </Control>
    </BehaviorTree>
  </root>
'\''
'

# Run with port remapping (move to a specific waypoint)
docker exec "$CONTAINER" bash -c '
  source /opt/overlay_ws/install/setup.bash &&
  ros2 action send_goal /do_objective moveit_studio_sdk_msgs/action/DoObjectiveSequence '\''
objective_xml_string: |
  <root BTCPP_format="4" main_tree_to_execute="Move To Waypoint Remapped">
    <BehaviorTree ID="Move To Waypoint Remapped">
      <Control ID="Sequence" name="TopLevelSequence">
        <SubTree
          ID="Move to Waypoint"
          waypoint_name="Look at Table"
          joint_group_name="manipulator"
          controller_names="joint_trajectory_controller"
          use_all_planners="false"
        />
      </Control>
    </BehaviorTree>
  </root>
'\''
'
```

### Via Websocket (roslibpy) - No ROS Required

Connect to rosbridge on port 3201 from the host machine. No ROS installation needed.

```python
import roslibpy
from roslibpy import ActionClient
import time

client = roslibpy.Ros(host='localhost', port=3201)
client.run()

action = ActionClient(client, '/do_objective',
    'moveit_studio_sdk_msgs/action/DoObjectiveSequence')

done = False

def on_result(result):
    global done
    error_code = result.get('values', {}).get('error_code', {})
    print(f"Error code: {error_code.get('val', 'unknown')}")
    done = True

def on_feedback(feedback):
    print(f"Feedback: {feedback}")

def on_error(error):
    global done
    print(f"Error: {error}")
    done = True

goal_id = action.send_goal(
    {"objective_name": "Close Gripper"},
    on_result, on_feedback, on_error
)

while not done:
    time.sleep(0.1)

client.terminate()
```

**Synchronous alternative** using the service interface:
```python
service = roslibpy.Service(client, '/execute_objective',
    'moveit_studio_sdk_msgs/srv/ExecuteObjective')
result = service.call(roslibpy.ServiceRequest({"objective_name": "Close Gripper"}))
```

**Cancel a running objective:**
```python
action.cancel_goal(goal_id)
```

Only one objective runs at a time. Sending a new goal cancels any running objective. If you lose the goal handle, you must cancel via the web UI or restart MoveIt Pro.

Install roslibpy (needs main branch for ROS 2 action support):
```bash
pip install git+https://github.com/RobotWebTools/roslibpy.git@main
```

## Camera Streams

Camera feeds are served as MJPEG streams by `web_video_server` on port 3202. MoveIt Pro must be running.

```
http://localhost:3202/stream?topic=<IMAGE_TOPIC>&qos_profile=sensor_data
```

If accessing through the nginx proxy (port 80):
```
http://localhost/video/stream?topic=<IMAGE_TOPIC>&qos_profile=sensor_data
```

Common image topics depend on your robot config. Examples:
- `/wrist_camera/color/image_raw`
- `/camera/color/image_raw`
- `/camera/depth/image_rect_raw`

To discover available image topics:
```bash
CONTAINER=$(docker ps --filter "name=moveit_pro" --format "{{.Names}}" | head -1)
docker exec "$CONTAINER" bash -c "source /opt/overlay_ws/install/setup.bash && ros2 topic list" | grep image
```

## RViz Visualization

Launch RViz for visual debugging (robot model, TF tree, camera feeds, motion planning):

```bash
moveit_pro rviz
```

This blocks the terminal. It runs `ros2 launch moveit_studio_agent developer_rviz.launch.py` inside a container with the robot description loaded.

## ROS 2 Introspection (via docker exec)

Run ROS 2 commands non-interactively in the container:

```bash
CONTAINER=$(docker ps --filter "name=moveit_pro" --format "{{.Names}}" | head -1)
SETUP="source /opt/overlay_ws/install/setup.bash && source ~/user_ws/install/setup.bash"

docker exec "$CONTAINER" bash -c "$SETUP && ros2 node list"
docker exec "$CONTAINER" bash -c "$SETUP && ros2 topic list"
docker exec "$CONTAINER" bash -c "$SETUP && ros2 topic echo /joint_states --once"
docker exec "$CONTAINER" bash -c "$SETUP && ros2 service list"
docker exec "$CONTAINER" bash -c "$SETUP && ros2 param list"
docker exec "$CONTAINER" bash -c "$SETUP && ros2 action list"
```

## Updating Allowed Collision Matrix (ACM)

Analyze a URDF to generate `disable_collision` entries for SRDF files:

```bash
moveit_pro update_acm /path/to/robot.urdf
moveit_pro update_acm robot.urdf -o output.srdf
moveit_pro update_acm robot.urdf --samples 5000 --padding 0.1 --threshold 0.9 -v
moveit_pro update_acm robot.urdf --include-never
```

Options: `--output/-o PATH`, `--samples/-n NUM` (default 1000), `--padding/-p FLOAT` (default 0.0), `--threshold/-t FLOAT` (default 0.95), `--include-never`, `--verbose/-v`

Works with a running container (docker exec) or starts an ephemeral one.

## Common Workflows

### First-Time Setup
```bash
export MOVEIT_LICENSE_KEY="XXXXXX-XXXXXX-XXXXXX-XXXXXX-XXXXXX-XXXXXX"
moveit_pro setup_workspace https://github.com/PickNikRobotics/moveit_pro_example_ws.git
moveit_pro configure -c lab_sim -w ~/moveit_pro_example_ws
moveit_pro build
moveit_pro run -c lab_sim --no-browser
```

### Development Cycle
```bash
moveit_pro build user_workspace --colcon-args "--packages-select my_package"
moveit_pro test --colcon-args "--packages-select my_package"
moveit_pro run -c my_config --no-browser
```

### Multi-Machine Setup
```bash
# Machine 1 (drivers only):
moveit_pro run -c my_config --only-drivers

# Machine 2 (application without drivers):
moveit_pro run -c my_config --no-drivers
```

## Ports Reference

| Port | Service | Protocol |
|------|---------|----------|
| 80 | Web UI (nginx) | HTTP |
| 3200 | REST API (Swagger at /docs) | HTTP |
| 3201 | Rosbridge | WebSocket |
| 3202 | Video streams | HTTP (MJPEG) |
| 3203 | AI/LLM server | HTTP |

## Key Paths

| Path | Description |
|------|-------------|
| `~/.config/moveit_pro/` | User configuration |
| `~/.ros/log_moveit_pro/` | Log files |
| `/opt/moveit_pro/` | System install directory |
| `~/user_ws/` | User workspace (inside containers) |

## Launch Modes

All `moveit_pro run` variants **block the terminal** until stopped. Use a separate terminal or background shell for other work.

| Mode | Command | What runs | Web UI |
|------|---------|-----------|--------|
| Default | `moveit_pro run -c my_config` | Runtime + frontend | Opens browser automatically at `http://localhost` |
| No browser | `moveit_pro run -c my_config --no-browser` | Runtime + frontend | Available at `http://localhost`, browser not auto-opened |
| Headless | `moveit_pro run -c my_config --headless` | Runtime only | None — use the REST API (port 3200) or rosbridge (port 3201) |
| Drivers only | `moveit_pro run -c my_config --only-drivers` | Hardware drivers only | None |
| No drivers | `moveit_pro run -c my_config --no-drivers` | Runtime + frontend, no drivers | Opens browser |

```bash
moveit_pro run -c my_config -w /path/to/workspace              # Opens browser automatically
moveit_pro run -c my_config -w /path/to/workspace --no-browser # No auto-open, UI still at http://localhost
moveit_pro run -c my_config --headless                         # No frontend at all
```

To stop from another terminal:
```bash
moveit_pro down          # Stop all containers
moveit_pro down --rmi    # Stop and remove Docker images
```

## Migration Guides

When upgrading MoveIt Pro versions, check the release notes for migration notes and breaking changes. Major versions (8.0.0, 9.0.0) contain the most significant migration guides. Release notes are published at:

- **Release Notes:** https://docs.picknik.ai/release-notes/
- **Individual release:** `https://docs.picknik.ai/release-notes/YYYY/MM/DD/VERSION/`

Look for "Migration Notes", "Migration Guide", and "Breaking Changes" sections within each release.

## Troubleshooting

- **"requires a built docker image"** - Run `moveit_pro build user_image`
- **"requires an active user workspace"** - Run `moveit_pro configure -c my_config -w PATH`
- **Build failures** - Run commands in container: `docker exec "$CONTAINER" bash -c "cd ~/user_ws && colcon build"`
- **Port in use** - Run `moveit_pro down` to free ports 3200-3203
- **Complete reset** - `moveit_pro down --rmi`
- **Docs** - https://docs.picknik.ai/ or `moveit_pro COMMAND --help`
