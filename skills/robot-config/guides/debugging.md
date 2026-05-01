# Debugging

Reference for troubleshooting MoveIt Pro runtime issues.

### Log Location

MoveIt Pro outputs logs via Docker's standard logging. Use `docker logs` to access them:

```bash
# View recent driver logs (controllers, hardware interfaces, ros2_control)
docker logs moveit_pro-drivers-1 --tail 50

# View recent agent_bridge logs (objectives, move_group, planning)
docker logs moveit_pro-agent_bridge-1 --tail 50

# Search for errors in drivers
docker logs moveit_pro-drivers-1 2>&1 | grep -i "error\|fatal\|failed" | tail -20

# Search for objective execution errors
docker logs moveit_pro-agent_bridge-1 2>&1 | grep "objective_server_node" | tail -20

# Follow logs in real time
docker logs moveit_pro-drivers-1 --follow
docker logs moveit_pro-agent_bridge-1 --follow
```

**Important:** `docker compose exec` requires the correct project context. MoveIt Pro runs as the `moveit_pro` compose project. Running `docker compose` from your workspace directory will fail because it only sees your override file. Use `docker exec` with explicit container names (`moveit_pro-drivers-1`, `moveit_pro-agent_bridge-1`) instead — this works regardless of your current directory.

The backend is fully ready when you see: `You can start planning now!`

### Log Prefixes

| Prefix | Source |
|--------|--------|
| `[objective_server_node]` | Objective/behavior tree execution |
| `[ros2_control_node]` | Controller manager, hardware interfaces |
| `[MujocoSystem]` | MuJoCo simulation |
| `[controller_manager]` | Controller loading/switching |

### Common Runtime Errors

**Missing Behavior**:
```
[objective_server_node] [error] Failed to create behavior tree for Objective `My Objective`.
Reason: Missing manifest for element_ID: SomeBehaviorName
```
The behavior name in the objective XML doesn't match any registered behavior. Check that the correct behavior loader plugins are listed in config.yaml.

**Missing Package at Launch**:
```
[launch] Caught exception: "package 'some_package' not found, searching: [...]"
```
The workspace hasn't been rebuilt. Run `moveit_pro build`.

**Port/Parameter Errors in Objectives**:
```
[objective_server_node] [error] ... Failed to get required values from input data ports: ...
```
A required port in the behavior tree XML is missing or has the wrong name.

**Stale Plugin Crashes**: `pluginlib` discovers plugins on `AMENT_PREFIX_PATH`, including previously-built-but-currently-skipped packages. If a plugin's `.so` doesn't exist at runtime, loading will crash. A clean rebuild fixes this.

**Build fails immediately after `moveit_pro down`**: Running `moveit_pro build user_workspace` right after `moveit_pro down` sometimes fails because the build container can't start while the previous containers are still cleaning up. Simply retry the build command — it will succeed on the second attempt.

### Stopping and Restarting

```bash
moveit_pro down                   # Stop and remove all containers
moveit_pro build user_workspace   # Rebuild (faster than full `moveit_pro build`)
moveit_pro run                    # Relaunch
```

When switching between configs (e.g., base → sim), you must also reconfigure:
```bash
moveit_pro down
moveit_pro configure -w <workspace_dir> -c <new_config_package>
moveit_pro build user_workspace
moveit_pro run
```

**Clear the config cache** when changing SRDF, controller config, or other cached parameters:
```bash
rm -rf ~/.config/moveit_pro/<config_package_name>
```

### Triggering Objectives from CLI

```bash
# Run an objective
docker exec moveit_pro-agent_bridge-1 bash -c "source /opt/overlay_ws/install/setup.bash && \
  source \${HOME}/user_ws/install/setup.bash && \
  ros2 action send_goal --feedback /do_objective \
  moveit_studio_sdk_msgs/action/DoObjectiveSequence \
  \"{objective_name: 'My Objective Name'}\""

# Cancel a running objective
docker exec moveit_pro-agent_bridge-1 bash -c "source /opt/overlay_ws/install/setup.bash && \
  ros2 service call /cancel_objective moveit_studio_sdk_msgs/srv/CancelObjective {}"
```
