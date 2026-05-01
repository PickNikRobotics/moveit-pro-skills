# Verification & Testing

> **MANDATORY: Task tracking.** Before starting verification, create a task for each step below using `TaskCreate`. Mark each task `in_progress` when you start it and `completed` when you finish it. Do not skip steps. If a step cannot be completed, stop and ask the user — do not silently move on. A config is not complete until every task is marked completed.

After `moveit_pro run` reaches `You can start planning now!`, verify the config systematically before considering it complete. Do not skip these steps — a config that launches is not necessarily a config that works.

All verification commands run inside the Docker container via `docker exec`. Which container to use depends on the command:
- **drivers** (`moveit_pro-drivers-1`): Controller queries (`ros2 control list_controllers`), joint state echo, hardware-level checks
- **agent_bridge** (`moveit_pro-agent_bridge-1`): Objective execution (`ros2 action send_goal /do_objective ...`), action server queries, move_group checks

Both containers need environment setup (CycloneDDS, workspace sourcing) before ROS 2 CLI commands will work.

**Important:** Do NOT use `docker compose exec` — it requires the correct project context and will fail with "service has neither an image nor a build context" if run from your workspace directory. MoveIt Pro runs as the `moveit_pro` compose project, but your workspace only has an override file. Use `docker exec` with explicit container names (`moveit_pro-drivers-1`, `moveit_pro-agent_bridge-1`) instead.

**Important: Restart the ROS2 daemon before running CLI commands.** CycloneDDS discovery over localhost does not work for new CLI processes unless the daemon is restarted. Run this once per container session before any other `ros2` commands:

```bash
docker exec -u <username> moveit_pro-drivers-1 bash -c "\
  export CYCLONEDDS_URI=/home/<username>/.ros/cyclonedds.xml && \
  export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp && \
  source /opt/overlay_ws/install/setup.bash && \
  source /home/<username>/user_ws/install/setup.bash && \
  ros2 daemon stop && ros2 daemon start"
```

Replace `<username>` with the user configured in your Dockerfile (default `moveit-pro-user`, or your host username if customized). After the daemon restart, subsequent commands in the same container will discover topics correctly.

For all verification commands below, prefix with the environment setup:
```bash
docker exec -u <username> moveit_pro-drivers-1 bash -c "\
  export CYCLONEDDS_URI=/home/<username>/.ros/cyclonedds.xml && \
  export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp && \
  source /opt/overlay_ws/install/setup.bash && \
  source /home/<username>/user_ws/install/setup.bash && \
  <command>"
```

### Step 1: Verify Controllers Loaded

Check that all expected controllers are loaded and in the correct state:

```bash
docker exec -u <username> moveit_pro-drivers-1 bash -c "\
  export CYCLONEDDS_URI=/home/<username>/.ros/cyclonedds.xml && \
  export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp && \
  source /opt/overlay_ws/install/setup.bash && \
  source /home/<username>/user_ws/install/setup.bash && \
  ros2 control list_controllers"
```

Expected output should show:
- `joint_state_broadcaster` — **active**
- `joint_trajectory_admittance_controller` — **active**
- `robotiq_gripper_controller` (or your end effector controller) — **active**
- `joint_velocity_controller` — **inactive** (activated on demand for joint teleop)
- `velocity_force_controller` — **inactive** (activated on demand for pose teleop)

If any controller is missing, check the ros2_control yaml and config.yaml `controllers_active/inactive_at_startup` lists.

### Step 2: Verify Joint States

Confirm the robot is publishing joint states and the joint names match your config:

```bash
docker exec -u <username> moveit_pro-drivers-1 bash -c "\
  export CYCLONEDDS_URI=/home/<username>/.ros/cyclonedds.xml && \
  export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp && \
  source /opt/overlay_ws/install/setup.bash && \
  source /home/<username>/user_ws/install/setup.bash && \
  ros2 topic echo /joint_states --once"
```

Verify:
- All expected arm joints appear in the `name` list
- End effector joints (if any) appear
- Position values match your expected home/initial pose
- The topic is publishing (not hanging)

### Step 3: Run a Test Objective

Run the simplest objective to verify the behavior tree execution pipeline works. Use "Move to Home" as the universal first test (every config should have this):

```bash
docker exec -u <username> moveit_pro-agent_bridge-1 bash -c "\
  export CYCLONEDDS_URI=/home/<username>/.ros/cyclonedds.xml && \
  export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp && \
  source /opt/overlay_ws/install/setup.bash && \
  source /home/<username>/user_ws/install/setup.bash && \
  ros2 action send_goal /do_objective \
  moveit_studio_sdk_msgs/action/DoObjectiveSequence \
  '{objective_name: \"Move to Home\"}'"
```

**Note:** Objectives run via the `agent_bridge` container (where the objective server lives), not `drivers`.

If your config has end effector objectives (Open/Close Gripper, Activate/Deactivate Vacuum, etc.), test those too. If these succeed, the objective server, controller manager, and controllers are all working together.

### Step 4: Test Teleop (Joint Jog)

Joint jog is the simplest teleop mode and should work for any config with a `joint_velocity_controller`:

1. Open the MoveIt Pro web UI (http://localhost)
2. Select the **Teleop** tab
3. Choose **Joint Jog** mode
4. Try moving each joint individually

If joint jog fails:
- Check that `joint_velocity_controller` loaded (Step 1)
- Verify `joint_jog.yaml` has `controllers: ['joint_velocity_controller']`
- Check logs for controller switching errors

### Step 5: Visual Inspection

Open the MoveIt Pro web UI (http://localhost) and ask the user to confirm the following — do not consider this step complete until they have eyes on the UI:

1. **End effector orientation**: If using a gripper, the fingers should point **away** from the arm. If they point toward the wrist, the mounting rotation is wrong.
2. **No gaps or overlaps**: The end effector base should sit flush against the arm's tool flange. A gap means the Z offset is too large; overlap means too small or missing.
3. **Home pose looks reasonable**: The robot should not be in a fully extended or obviously impossible pose.
4. **No collision warnings**: No toast/alert errors visible in the UI.
5. **Robot is visible**: If the 3D view is empty or shows only a grid, the URDF failed to load — check the logs.

Also ask the user to test **IMarker teleop** (drag-and-drop in the 3D view) — it exercises the pose-jog pipeline end-to-end.

### Step 6: Test Motion Planning

Try planning and executing a motion to verify the full pipeline:

```bash
docker exec -u <username> moveit_pro-agent_bridge-1 bash -c "\
  export CYCLONEDDS_URI=/home/<username>/.ros/cyclonedds.xml && \
  export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp && \
  source /opt/overlay_ws/install/setup.bash && \
  source /home/<username>/user_ws/install/setup.bash && \
  ros2 action send_goal --feedback /do_objective \
  moveit_studio_sdk_msgs/action/DoObjectiveSequence \
  \"{objective_name: 'Move to Waypoint'}\""
```

Or use the web UI to drag the interactive marker and plan a motion. If planning fails:
- Check kinematics.yaml uses `pose_ik_plugin/PoseIKPlugin`
- Verify the manipulator chain in the SRDF is correct (base_link to tip_link)
- Check joint limits in joint_limits.yaml match the URDF

### Verification Complete

When all steps above are marked as completed tasks, the config is verified. If any step was skipped or could not be completed, the config is NOT verified — flag it to the user.
