# MoveIt Pro Skills

Agent skills for working with [MoveIt Pro](https://docs.picknik.ai/) — the Docker-based robotic manipulation platform by PickNik Robotics.

> [!IMPORTANT]
> This repository is a legacy snapshot for MoveIt Pro 9 and earlier. MoveIt Pro
> 10 and later ship version-matched skills with the product, and workspace setup
> installs them under `~/moveit_pro/.claude/skills/` for Claude Code and
> `~/moveit_pro/.agents/skills/` for Codex and other compatible agents. Do not
> install this snapshot for current MoveIt Pro releases.

## Skills

- **moveit-pro-cli** — Guide for using the MoveIt Pro platform programmatically and via CLI
- **ros2-cli** — Guide for working with ROS 2 systems via CLI
- **create-objective** — Build or modify MoveIt Pro Behavior Trees: Objectives (XML) and custom Behavior Nodes (C++). Accepts an inline description or a markdown SOP document.
- **robot-config** — Build a MoveIt Pro robot configuration package (mock, sim, or physical) from a URDF, CAD, or STLs and verify it builds, launches, and behaves correctly.
- **debug-objective** — Debug MoveIt Pro Objectives (XML behavior trees) and Behaviors (C++ plugins) using runtime logs. Covers log location, common error patterns ("Missing manifest", "package not found", port errors), and ready-to-paste `docker exec` commands.
- **create-behavior** — Author a new MoveIt Pro Behavior plugin (C++): header, source, registration, CMake, and test wiring. Pairs with `create-objective` for the XML side.

## Legacy MoveIt Pro 9 Installation

Install every skill in this repo (recommended):

```bash
npx skills add PickNikRobotics/moveit-pro-skills -y
```

The `-y` flag skips the interactive multi-select and installs all skills. Add `-g` for a global install, or `--agent claude-code` to scope to a specific agent.

Install a specific skill:

```bash
npx skills add PickNikRobotics/moveit-pro-skills --skill moveit-pro-cli
```

List available skills before installing:

```bash
npx skills add PickNikRobotics/moveit-pro-skills --list
```
