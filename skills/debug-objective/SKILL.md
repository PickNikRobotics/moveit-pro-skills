---
name: debug-objective
description: Debug MoveIt Pro Objectives (XML behavior trees) and Behaviors (C++ plugins) using runtime logs. Use when an Objective fails to load, fails at runtime, throws "Missing manifest" / "package not found" / port-validation errors, or when a Behavior is misbehaving and you need to inspect the agent log. Pairs well with the moveit-pro-cli skill, which covers triggering Objectives and managing the runtime.
---

# Debugging Objectives

When a MoveIt Pro Objective fails — fails to load, fails mid-execution, throws a port error, or just behaves unexpectedly — the answer is almost always in the agent runtime log. This skill covers where the log lives, how to read it, and the most common error patterns.

## Where the Log Lives

The MoveIt Pro backend (`agent_robot.app`) writes all output to `/tmp/agent_robot.log` inside the runtime container. The log is **overwritten each time** `moveit_pro run` starts, so capture it before stopping the stack.

Two ways to read it:

1. **`moveit_pro logs`** — opens the supported log viewer.
2. **`docker exec` directly** — better for `tail -f`, `grep`, and scripted use.

For host-side log archives (one file per run), check `~/.ros/log_moveit_pro/`.

## Reading the Log via `docker exec`

Find the running container, then exec into it:

```bash
CONTAINER=$(docker ps --filter "name=moveit_pro" --format "{{.Names}}" | head -1)

# Last 50 lines
docker exec "$CONTAINER" bash -c 'tail -50 /tmp/agent_robot.log'

# Errors only
docker exec "$CONTAINER" bash -c 'grep -iE "error|fatal|failed" /tmp/agent_robot.log | tail -20'

# Objective-server messages only
docker exec "$CONTAINER" bash -c 'grep "objective_server_node" /tmp/agent_robot.log | tail -20'

# Follow in real time
docker exec "$CONTAINER" bash -c 'tail -f /tmp/agent_robot.log'
```

If multiple MoveIt Pro containers are running (e.g., `moveit_pro-drivers-1`, `moveit_pro-agent_bridge-1`), the agent log lives in `agent_bridge`. Target it explicitly:

```bash
docker exec moveit_pro-agent_bridge-1 bash -c 'tail -50 /tmp/agent_robot.log'
```

Use `moveit_pro-drivers-1` for hardware/controller-level checks (e.g., `ros2 control list_controllers`).

## Common Error Patterns

### Missing Behavior

```text
[objective_server_node] [error] Failed to create behavior tree for Objective `My Objective`.
Reason: Missing manifest for element_ID: SomeBehaviorName
```

**Cause:** The Behavior name in the Objective XML doesn't match any registered Behavior. This can happen because:

1. The name is misspelled in the XML.
2. The Behavior plugin isn't loaded by the current robot configuration package — check the `behavior_loader_plugins` section in the config's `config.yaml`.
3. The package containing the Behavior hasn't been built in the workspace.

**Fix:** Confirm the Behavior is registered by checking the `register_*_behaviors.cpp` file in its source package, and verify that the active config's `behavior_loader_plugins` includes the matching loader (e.g., `moveit_pro::behaviors::CoreBehaviorsLoader`). If it's a workspace-local Behavior, rebuild with `moveit_pro build user_workspace`.

### Missing Package

```text
[launch] Caught exception: "package 'some_package' not found, searching: [...]"
```

**Cause:** The user workspace hasn't been built, or the package isn't installed.

**Fix:** Rebuild the workspace:

```bash
moveit_pro build user_workspace
```

If the package is part of a private repo or external dependency, verify it's checked out in the workspace and that its `package.xml` declares the correct dependencies.

### Port / Parameter Errors

```text
[objective_server_node] [error] ... Failed to get required values from input data ports: ...
```

**Cause:** A required port in the Objective XML is missing, has the wrong name, or is wired to a blackboard variable that was never set.

**Fix:** Check the Behavior's `providedPorts()` method (in its `.hpp` or `.cpp`) for the correct port names and types. Cross-reference against the catalog at https://picknik.ai/behaviors. If using the `create-objective` skill, its CRITICAL Node and Port Validation section covers this.

## Tips

- Errors from the objective server are prefixed with `[objective_server_node]`.
- ROS 2 control errors are prefixed with `[ros2_control_node]`.
- MuJoCo-specific messages come from `[MujocoSystem]`.
- The backend is fully ready when you see: `You can start planning now!`
- The log is overwritten on every run, so save a copy (`docker cp` or `moveit_pro export_logs`) before stopping the stack if you need to share it.

## Triggering and Cancelling Objectives

To execute or cancel an Objective while debugging, see the **moveit-pro-cli** skill — it covers ROS 2 CLI (`ros2 action send_goal /do_objective ...`), rclpy, and roslibpy.
