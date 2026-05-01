# Creating a Sim Config (Physics Simulation)

> **MANDATORY: Task tracking.** Before starting this guide, create a task for each numbered step below using `TaskCreate`. Mark each task `in_progress` when you start it and `completed` when you finish it. Do not skip steps. If a step cannot be completed, stop and ask the user — do not silently move on.

This guide covers creating a sim config that adds physics simulation to an existing base config. The sim config inherits from the base config via `based_on_package` and overrides only what's needed for physics.

**Prerequisites:**
- A working base config (`<robot>_base_config`) that builds, runs, and passes verification

**The sim config adds:**
- Physics simulation via `picknik_mujoco_ros/MujocoSystem` (replaces `mock_components/GenericSystem`)
- MuJoCo XML scene and robot model files
- Convex-decomposed mesh assets for physics collision
- Sim-specific objectives (e.g., Reset Simulation)
- `MujocoBehaviorsLoader` behavior plugin

## Step 1: Create the Sim Config Package

Create a new package alongside the base config:

```
my_robot_ws/
└── src/
    ├── <robot>_base_config/    # Already exists and verified
    └── <robot>_sim/            # New — inherits from base
        ├── CMakeLists.txt
        ├── package.xml
        ├── config/
        │   └── config.yaml
        ├── description/
        │   ├── <robot>.urdf.xacro      # New xacro with MujocoSystem
        │   ├── <robot>.ros2_control.xacro  # MujocoSystem hardware interface
        │   └── mujoco/
        │       ├── scene.xml            # MuJoCo scene (environment, lighting, camera)
        │       ├── <robot>.xml          # MuJoCo robot model (bodies, joints, meshes)
        │       └── assets/              # Convex-decomposed meshes for physics
        ├── launch/
        │   └── agent_bridge.launch.xml
        ├── objectives/
        │   └── reset_simulation.xml
        └── waypoints/
            └── waypoints.yaml
```

## Step 2: config.yaml with Inheritance

The sim config.yaml uses `based_on_package` to inherit from the base config. Only override what changes:

```yaml
based_on_package: "<robot>_base_config"

hardware:
  robot_description:
    urdf:
      package: "<robot>_sim"
      path: "description/<robot>.urdf.xacro"
    urdf_params:
      - use_fake_hardware: "false"
      - mujoco_model: "description/mujoco/scene.xml"
      - mujoco_viewer: false

objectives:
  behavior_loader_plugins:
    core:
      - "moveit_pro::behaviors::CoreBehaviorsLoader"
      - "moveit_pro::behaviors::MTCCoreBehaviorsLoader"
      - "moveit_pro::behaviors::VisionBehaviorsLoader"
      - "moveit_pro::behaviors::ConverterBehaviorsLoader"
      - "moveit_pro::behaviors::MujocoBehaviorsLoader"
  objective_library_paths:
    mujoco_objectives:
      package_name: "moveit_pro_objectives"
      relative_path: "objectives/mujoco"
    sim_objectives:
      package_name: "<robot>_sim"
      relative_path: "objectives"
  waypoints_file:
    package_name: "<robot>_sim"
    relative_path: "waypoints/waypoints.yaml"
```

**Key changes from base:**
- `based_on_package` — inherits everything not explicitly overridden
- `use_fake_hardware: "false"` — MujocoSystem is real hardware, not fake
- `mujoco_model` — path to the MuJoCo scene XML
- `MujocoBehaviorsLoader` — loads MuJoCo-specific behaviors
- `mujoco_objectives` — built-in MuJoCo objectives (e.g., Reset Simulation)

## Step 3: URDF Xacro with MujocoSystem

The sim xacro is a **new file** in the sim config package that replaces `mock_components/GenericSystem` with `picknik_mujoco_ros/MujocoSystem`. It is NOT a separate ros2_control xacro — the ros2_control block goes directly in the main xacro.

Key differences from the base config xacro:
- Do NOT include the base config's `ros2_control.xacro` — the MujocoSystem block replaces it entirely
- If the gripper macro has `include_ros2_control`, set it to `"false"` (MujocoSystem handles all joints)
- The `ros2_control` block includes ALL joints: arm, gripper command joint (if any), and gripper passive joints
- Arm joints get `command_interface` (position) + `state_interface` (position, velocity, effort)
- Gripper command joint (if any) gets `command_interface` + `state_interface`
- Gripper passive joints (if any) get only `state_interface`
- Initial values must match the MuJoCo keyframe (e.g., home pose)
- Pass through any `namespace` or other xacro args that the base config's robot macro expects

```xml
<xacro:arg name="mujoco_model" default="description/mujoco/scene.xml" />
<xacro:arg name="mujoco_viewer" default="false" />

<!-- Set include_ros2_control="false" on the gripper — MujocoSystem handles it -->
<xacro:robotiq_gripper
    name="RobotiqGripperHardwareInterface"
    prefix="" parent="link_6"
    use_fake_hardware="false"
    include_ros2_control="false"
    com_port="/dev/ttyUSB0">
    <origin xyz="0 0 0" rpy="0 0 ${pi}" />
</xacro:robotiq_gripper>

<ros2_control name="<robot>_mujoco_simulation" type="system">
  <!-- Arm joints with command + state interfaces -->
  <joint name="joint_1">
    <command_interface name="position">
      <param name="min">${-2*pi}</param>
      <param name="max">${2*pi}</param>
    </command_interface>
    <state_interface name="position">
      <param name="initial_value">0.0</param>
    </state_interface>
    <state_interface name="velocity"/>
    <state_interface name="effort"/>
  </joint>
  <!-- ... repeat for all arm joints ... -->

  <!-- Gripper: finger_joint is commanded -->
  <joint name="finger_joint">
    <command_interface name="position"/>
    <state_interface name="position">
      <param name="initial_value">0.0</param>
    </state_interface>
    <state_interface name="velocity"/>
  </joint>
  <!-- Gripper passive joints: state-only -->
  <joint name="left_inner_knuckle_joint">
    <state_interface name="position"/>
    <state_interface name="velocity"/>
  </joint>
  <!-- ... repeat for all passive gripper joints ... -->

  <hardware>
    <plugin>picknik_mujoco_ros/MujocoSystem</plugin>
    <param name="mujoco_model">$(arg mujoco_model)</param>
    <param name="mujoco_model_package"><robot>_sim</param>
    <param name="render_publish_rate">10</param>
    <param name="tf_publish_rate">60</param>
    <param name="mujoco_viewer">$(arg mujoco_viewer)</param>
  </hardware>
</ros2_control>
```

**Gripper joint names:** For Robotiq 2F-140, the passive joints are: `left_inner_finger_joint`, `left_inner_knuckle_joint`, `right_outer_knuckle_joint`, `right_inner_finger_joint`, `right_inner_knuckle_joint`. For Robotiq 2F-85, the joint names differ (e.g., `robotiq_85_left_knuckle_joint`). Check the kinova_sim example for 2F-85 patterns.

## Step 4: MuJoCo XML Files

The MuJoCo model is split into two files:
- `description/mujoco/assets/robot_description.xml` — the robot model (bodies, joints, meshes, actuators, contact exclusions)
- `description/mujoco/scene.xml` — the environment (includes robot, adds ground, lighting, keyframes)

### 4a: Generate robot_description.xml

This is a multi-step pipeline:

1. **Extract rendered URDF** from the running base config. You need a container with the base config's packages installed. Options:
   - If MoveIt Pro is running the base config: `docker exec` into the drivers or agent_bridge container
   - Use `moveit_pro dev` to start a dev container (interactive, exits when the shell closes)
   - Or run directly: `docker run --rm --entrypoint "" -v ~/my_ws/src:/home/$USER/user_ws/src -u $(id -u):$(id -g) <dev_image> sleep infinity` and `docker exec` into it

   ```bash
   # Inside a running MoveIt Pro container with the base config:
   ros2 topic echo /robot_description --once --field data > robot_description.urdf
   ```

   If the base config isn't running, you can also render the URDF on the host with xacro:
   ```bash
   # Inside the dev container (after sourcing the workspace):
   xacro <path_to_base_config>/description/<robot>.urdf.xacro use_fake_hardware:=true > robot_description.urdf
   ```

2. **Inject MuJoCo compiler options** into the URDF's `<mujoco>` tag:
   ```xml
   <mujoco>
     <compiler fusestatic="false" discardvisual="false"/>
   </mujoco>
   ```
   Add this as a child of the `<robot>` tag. `fusestatic="false"` keeps static geometry separate; `discardvisual="false"` preserves visual meshes.

3. **Flatten mesh paths** — MuJoCo needs all meshes in one directory:
   ```bash
   # Inside the dev container:
   ros2 run moveit_studio_utils_py flatten_meshes robot_description.urdf
   ```
   This copies all referenced meshes into the current directory and updates the URDF paths.

4. **Convert non-STL meshes** — MuJoCo only loads STL and OBJ. If meshes are DAE (COLLADA), convert them:
   ```bash
   # On the host (not in container) — create a venv:
   python3 -m venv ~/mesh_tools_venv
   ~/mesh_tools_venv/bin/pip install trimesh numpy pycollada
   ```
   Then convert:
   ```python
   import trimesh, glob
   for dae in sorted(glob.glob('*.dae')):
       stl = dae.replace('.dae', '.stl')
       scene = trimesh.load(dae, force='scene')
       meshes = [g for g in scene.geometry.values() if hasattr(g, 'vertices')]
       combined = trimesh.util.concatenate(meshes)
       combined.export(stl)
   ```
   After conversion, update the URDF to reference `.stl` instead of `.dae`:
   ```bash
   sed -i 's/\.dae/.stl/g' robot_description.urdf
   ```
   Then delete the original DAE files.

5. **Run urdf_to_mjcf** to generate the MJCF XML:
   ```bash
   # Inside the dev container:
   ros2 run moveit_studio_utils_py urdf_to_mjcf robot_description.urdf
   ```
   This produces `robot_description.xml` with: asset declarations, auto-generated actuators (position, kp=1000), gravity compensation, and the full body tree.

   **Do not hand-write or reorganize the body tree.** The body hierarchy, inertials, and joint transforms are generated from the rendered URDF and must match exactly. Hand-writing the MJCF body tree leads to subtle transform errors that cause simulation instability. Only modify actuators, contact exclusions, and defaults after generation.

   **Actuator names must match ros2_control joint names exactly.** The MujocoSystem plugin matches actuators to ros2_control joints by actuator `name`, not by the `joint` attribute. The `urdf_to_mjcf` tool sets both to the joint name (e.g., `name="j1" joint="j1"`). If you rename actuators (e.g., `name="j1_actuator"`), MujocoSystem will crash with: `Actuator name 'j1_actuator' does not match name of any joints in the model.`

6. **Tune the generated XML:**

   **Actuators — use `dampratio` (not explicit `kv`).** The `urdf_to_mjcf` tool generates default `kp=1000, dampratio=1` which is too weak for most arms. The recommended approach is uniform `kp` with `dampratio`:

   ```xml
   <position joint="joint_1" name="joint_1" ctrlrange="-6.2832 6.2832" kp="5000" dampratio="3"/>
   ```

   MuJoCo's `dampratio` auto-scales velocity damping based on each joint's effective inertia. This is critical because robot arms have vastly different inertias across joints (heavy shoulder vs. light wrist). With explicit `kv`, it is very easy to under-damp wrist joints, causing **high-frequency oscillation** — the joints vibrate at 3-5 rad/s around the target position even at rest. This oscillation appears as "other joints moving" during teleop and can be mistaken for a controller coupling issue.

   **Recommended starting gains:**
   - Most 6-7 DOF arms: `kp="5000" dampratio="3"` uniformly on all joints
   - Gripper joints: `kp="100" dampratio="3"`
   - Increase `kp` if joints are sluggish or can't track fast trajectories
   - Increase `dampratio` if joints oscillate (try 4-5)
   - Decrease `dampratio` if motions feel overdamped/slow (try 2)

   **Do not use per-joint `kp`/`kv` tuning** unless you have a specific reason. Per-joint kv tuning requires getting the damping right for each joint's inertia independently — too low causes oscillation, too high causes sluggishness. `dampratio` handles this automatically.

   Set `ctrlrange` to match the joint limits from the URDF/SRDF. Remove the `forcelimited="false"` that `urdf_to_mjcf` adds by default.

   **`actuatorfrcrange` can cause saturation.** The `urdf_to_mjcf` tool sets `actuatorfrcrange` from the URDF's `<limit effort="...">` values. Wrist joints on smaller arms often have low effort limits (e.g., 28 Nm) that cause actuator force saturation in sim — the actuator can't apply enough force to track the commanded position, leading to joint drift and instability. **Increase `actuatorfrcrange` on wrist joints to at least ±150 Nm for simulation**, or remove the attribute entirely to allow unlimited force. The real hardware effort limits still apply on the physical config.

   **Contact exclusions** — Mirror the SRDF `disable_collisions` entries using MuJoCo `<contact><exclude>` syntax. At minimum, exclude:
   - Adjacent arm links (parent-child pairs)
   - Near-neighbor arm links (skip-one pairs)
   - Arm-to-gripper mounting links
   - All gripper internal link pairs
   ```xml
   <contact>
     <exclude body1="base_link" body2="link_1"/>
     <exclude body1="link_1" body2="link_2"/>
     <!-- ... -->
   </contact>
   ```

### 4b: Create scene.xml

The scene file includes the robot and adds the simulation environment:

```xml
<mujoco model="<robot> scene">
  <option timestep="0.001"/>
  <include file="assets/robot_description.xml"/>

  <statistic center="0.3 0 0.5" extent="1.5"/>
  <default>
    <geom solref=".004 1"/>
    <joint damping="50"/>
  </default>

  <visual>
    <headlight diffuse="0.6 0.6 0.6" ambient="0.3 0.3 0.3" specular="0 0 0"/>
    <rgba haze="0.15 0.25 0.35 1"/>
    <global azimuth="120" elevation="-20"/>
  </visual>

  <asset>
    <texture type="skybox" builtin="gradient" rgb1="0.3 0.5 0.7" rgb2="0 0 0" width="512" height="3072"/>
    <texture type="2d" name="groundplane" builtin="checker" mark="edge"
             rgb1="0.2 0.3 0.4" rgb2="0.1 0.2 0.3" markrgb="0.8 0.8 0.8"
             width="300" height="300"/>
    <material name="groundplane" texture="groundplane" texuniform="true"
              texrepeat="5 5" reflectance="0.2"/>
  </asset>

  <worldbody>
    <light pos="0 0 1.5" dir="0 0 -1" directional="true"/>
    <geom name="floor" size="0 0 0.05" type="plane" material="groundplane"/>
  </worldbody>

  <!-- Keyframe: must match the home pose from SRDF and initial_value in xacro -->
  <keyframe>
    <key name="default"
         qpos="0 0 1.5708 0 1.5708 0 0 0 0 0 0 0"
         ctrl="0 0 1.5708 0 1.5708 0 0 0 0 0 0 0"/>
  </keyframe>
</mujoco>
```

**`timestep` and `damping` are critical for stability.** The default MuJoCo timestep (0.002s) combined with high actuator gains can cause oscillation. Use `timestep="0.001"` for robot arms. The `<joint damping="50"/>` default adds velocity-proportional damping to all joints, providing baseline oscillation suppression. Increase to 100 if wrist joints still oscillate; decrease to 20 if motions feel sluggish. This passive damping works alongside the actuator `dampratio` — both contribute to stability.

**Keyframe values:** `qpos` and `ctrl` must have one entry per joint, in the order they appear in the MJCF body tree. Match the home pose from the SRDF `group_state`. Gripper joints are typically all 0 (open).

### 4c: Verify the model before proceeding

Test that MuJoCo can load and simulate the model:
```bash
# Inside the dev container:
python3 -c "
import mujoco
model = mujoco.MjModel.from_xml_path('scene.xml')
data = mujoco.MjData(model)
mujoco.mj_resetDataKeyframe(model, data, 0)
for _ in range(1000):
    mujoco.mj_step(model, data)
print(f'Joints: {model.njnt}, Actuators: {model.nu}, Bodies: {model.nbody}')
print(f'Max velocity after 1s: {max(abs(data.qvel)):.6f}')
"
```
If max velocity is near 0, the model is stable. If it's large or NaN, check actuator gains and contact exclusions.

**Also verify after launching in MoveIt Pro.** The standalone MuJoCo test above runs without controllers. The full system with JTAC/JVC controllers can introduce additional dynamics. After launching, check joint stability at rest:
```bash
ros2 topic echo /joint_states --once
```
All arm joint velocities should be near 0 (< 0.01 rad/s). If velocities are 1+ rad/s while positions are near the home pose, the joints are **oscillating** — this means the actuator gains need more damping. Increase `dampratio` (try 3-5) or increase the `<joint damping="..."/>` default in scene.xml (try 50-100).

**Oscillation diagnosis checklist:**
1. High velocity (1+ rad/s) with small position error (< 0.1 rad) at rest → underdamped actuators → increase `dampratio`
2. Large position error growing over time → actuators too weak → increase `kp`
3. NaN in velocities → simulation diverged → check contact exclusions, reduce `kp`, increase `timestep`
4. Joints move when jogging a single joint → likely oscillation (not controller coupling) → check velocities at rest first

## Step 5: Mesh Assets for Physics

The `flatten_meshes` tool copies all URDF-referenced meshes into `description/mujoco/assets/`. For MuJoCo:
- **STL and OBJ** formats are supported; DAE is not
- Visual meshes (group 1, `contype="0" conaffinity="0"`) are used for rendering only
- Collision meshes (default group) are used for physics contact
- The `urdf_to_mjcf` tool creates paired visual/collision geoms automatically

If meshes are too dense for simulation performance, decimate them:
```bash
# Inside the dev container:
ros2 run moveit_studio_utils_py decimate_mesh input.stl output.stl --ratio 0.5
```

## Step 5b: Simulated Depth Camera

To add a simulated depth camera to the MuJoCo model, add a `<camera>` element and a corresponding `<site>` for the optical frame. See the [official docs](https://docs.picknik.ai/how_to/configuration_tutorials/create_robot_sim_config/migrate_to_mujoco_config/#simulated-depth-camera-sensor) for reference.

### Computing the camera orientation

**Do not manually reason through nested quaternion/euler transform chains.** This is error-prone and wastes iteration cycles. Instead, compute the camera orientation programmatically:

```python
# Inside a container with the sim config installed:
import mujoco
import numpy as np

model = mujoco.MjModel.from_xml_path('scene.xml')
data = mujoco.MjData(model)
mujoco.mj_resetDataKeyframe(model, data, 0)
mujoco.mj_forward(model, data)

# Get body positions/orientations at the home pose
cam_id = mujoco.mj_name2id(model, mujoco.mjtObj.mjOBJ_BODY, '<camera_body>')
tip_id = mujoco.mj_name2id(model, mujoco.mjtObj.mjOBJ_BODY, '<tool_tip_body>')

cam_pos = data.xpos[cam_id]
tip_pos = data.xpos[tip_id]
cam_rot = data.xmat[cam_id].reshape(3, 3)

# Look direction: camera toward tool tip, in camera body frame
look_world = (tip_pos - cam_pos)
look_world = look_world / np.linalg.norm(look_world)
look_body = cam_rot.T @ look_world

# Up direction: world +Z projected into camera body frame
up_body = cam_rot.T @ np.array([0, 0, 1])

# Compute camera axes
look = look_body / np.linalg.norm(look_body)
y_cam = up_body - np.dot(up_body, look) * look
y_cam = y_cam / np.linalg.norm(y_cam)
x_cam = np.cross(look, y_cam)

print(f'xyaxes="{x_cam[0]:.4f} {x_cam[1]:.4f} {x_cam[2]:.4f} {y_cam[0]:.4f} {y_cam[1]:.4f} {y_cam[2]:.4f}"')
```

### Camera and optical frame placement

The camera and optical frame site go inside the camera's parent body in the MJCF:

```xml
<body name="<camera_link>" ...>
  <!-- Camera: xyaxes defines right (X) and up (Y). Camera looks along -Z. -->
  <camera name="<camera_name>"
    pos="<offset_x> <offset_y> <offset_z>"
    fovy="<vertical_fov_degrees>"
    resolution="<width> <height>"
    xyaxes="<x1> <x2> <x3> <y1> <y2> <y3>"
    user="<depth_type> <fov_x_degrees> <range_min> <range_max>"/>
  <!-- Optical frame: SAME xyaxes but Y-axis NEGATED (180 deg rotation about X for ROS convention) -->
  <site name="<camera_name>_optical_frame"
    pos="<offset_x> <offset_y> <offset_z>"
    xyaxes="<x1> <x2> <x3> <-y1> <-y2> <-y3>"/>
</body>
```

**User parameters** (set via the `user` attribute):
- `user[0]` — depth type: `0` = RGB + Depth (most common), `1` = RGB only, `2` = 3D lidar
- `user[1]` — horizontal FOV in degrees
- `user[2]` — minimum range in meters
- `user[3]` — maximum range in meters

**Published ROS topics** (automatic, based on camera name):
- `/<camera_name>/color` — RGB image
- `/<camera_name>/depth` — depth image
- `/<camera_name>/points` — point cloud
- `/<camera_name>/camera_info` — camera intrinsics

### Critical: camera occlusion by robot geometry

The URDF camera mount point is often physically **inside** the robot mesh. The camera link origin may be behind or within the end effector housing. If the camera feed shows only white/blank, the camera is inside geometry.

**Fix:** Offset the camera position along the look direction in the `pos` attribute. The offset distance should be at least the distance from the camera link origin to the front of the end effector. Compute this from FK data — check the distance between the camera body and the tool tip body. Start with a generous offset and pull back if needed.

**A white/blank camera feed almost always means the camera is inside robot geometry.**

### Optical frame convention

Per the [MoveIt Pro docs](https://docs.picknik.ai/how_to/configuration_tutorials/create_robot_sim_config/migrate_to_mujoco_config/#simulated-depth-camera-sensor): the optical frame site must have the **Y-axis negated** compared to the camera's xyaxes. This applies the 180° rotation about X that converts from MuJoCo camera convention to ROS optical frame convention (Z forward, X right, Y down).

Example:
```xml
<!-- Camera -->
<camera ... xyaxes="-0.607 -0.795 0.000 0.555 -0.424 0.715"/>
<!-- Optical frame: same X-axis, negated Y-axis -->
<site   ... xyaxes="-0.607 -0.795 0.000 -0.555 0.424 -0.715"/>
```

### Take Snapshot objective

To capture a point cloud from the simulated camera and display it in the UI:

```xml
<Action ID="GetPointCloud"
  topic_name="/<camera_name>/points"
  message_out="{point_cloud}"
  publisher_timeout_sec="5.0"
  message_timeout_sec="5.0"/>
<Action ID="SendPointCloudToUI"
  point_cloud="{point_cloud}"
  pcd_topic="/pcd_pointcloud_captures"/>
```

`GetPointCloud` subscribes to the camera's point cloud topic. `SendPointCloudToUI` converts it to PCD format for the web UI. Optionally add `UpdatePlanningSceneService` to feed the point cloud into the collision octomap (requires `PointCloudServiceOctomapUpdater` plugin in sensors_3d.yaml).

## Step 6: Sim-Specific Objectives

**`objectives/reset_simulation.xml`** — resets the sim to the default MuJoCo keyframe:

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<root BTCPP_format="4" main_tree_to_execute="Reset Simulation">
  <BehaviorTree ID="Reset Simulation"
    _description="Reset the MuJoCo simulation to the default keyframe.">
    <Control ID="Sequence" name="TopLevelSequence">
      <Action ID="SwitchController"
        activate_asap="true"
        automatic_deactivation="true"
        controller_list_action_name="/controller_manager/list_controllers"
        controller_switch_action_name="/controller_manager/switch_controller"
        strictness="1"
        deactivate_controllers="joint_trajectory_admittance_controller;velocity_force_controller;joint_velocity_controller"
      />
      <Action ID="ResetMujocoKeyframe"
        response_timeout="10.000000"
        keyframe_name="default"
      />
    </Control>
  </BehaviorTree>
</root>
```

The `SwitchController` step deactivates trajectory controllers before reset to avoid conflicts. The `ResetMujocoKeyframe` action resets all joint positions and velocities to the named keyframe in the MuJoCo model. The keyframe name must match one defined in `scene.xml`.

## Step 7: Build, Run, and Verify

**Do not consider the sim config complete until you have built it, run it, and verified it yourself.** Do not hand off to the user after creating files — iterate through build/run/verify until it works.

### 7a: Configure and Build

```bash
moveit_pro configure -w <workspace_dir> -c <robot>_sim
moveit_pro build
```

Fix any build errors before proceeding. Common build issues:
- Missing `exec_depend` in package.xml (e.g., `picknik_mujoco_ros`)
- CMakeLists.txt not installing new directories (e.g., `objectives/`)

**Important:** If you override `ros2_control.config` in the sim config.yaml, the sim yaml **completely replaces** the base config's controller yaml — it does NOT merge. Your sim ros2_control yaml must include ALL controller definitions (controller_manager types, ALL controller parameters), not just the fields you want to change. Copy the base config's yaml as a starting point and modify it.

### 7b: Clear config cache

Delete the config cache to ensure fresh config is loaded (especially important if you previously ran the base config or a broken sim config):

```bash
rm -rf ~/.config/moveit_pro/<robot>_sim
```

### 7c: Run

```bash
moveit_pro run
```

Watch the logs for:
- `[MujocoSystem]: ParseXML: Error opening file` — scene.xml not found in install dir, rebuild needed
- `Failed to initialize hardware 'doosan_mujoco_simulation'` — MuJoCo model loading failed
- `missing state interfaces` / `missing command interfaces` — joint mismatch between xacro and MJCF
- `You can start planning now!` from move_group — **this means it's working**

The drivers container should NOT crash. If `ros2_control_node` dies, check the error above it.

### 7d: Verify with ROS commands

Follow the [Verification & Testing](verification.md) guide. At minimum:

1. **Controllers**: `ros2 control list_controllers` — all expected controllers loaded
2. **Joint states**: `ros2 topic echo /joint_states --once` — joint names and values match home pose
3. **MuJoCo rendering**: `ros2 topic list | grep render` — MuJoCo render topic publishing

### 7e: Visual verification

Open the MoveIt Pro web UI (http://localhost) and ask the user to confirm:
- Robot is visible in the 3D viewport (not a blank scene)
- Robot is in the home pose (not collapsed/exploded)
- No error toasts in the UI
- Gripper is attached and oriented correctly

### 7f: Test Reset Simulation objective

In the web UI (or via ROS), run the "Reset Simulation" objective and verify it completes successfully. This confirms the MuJoCo keyframe works end-to-end.

If any step fails, fix the issue and re-run from the appropriate step. Do not consider the config complete until all verification passes.

## Common Issues

**DAE meshes not loading:** MuJoCo does not support COLLADA (.dae). Convert to STL using trimesh+pycollada (see Step 4a.4). Common with Doosan robots.

**`moveit_pro dev` exits immediately:** The dev container's entrypoint needs env vars (`USERNAME`, `USER_UID`, `RMW_IMPLEMENTATION`). Use the `moveit_pro dev` CLI which handles these, or run `docker run --entrypoint "" -u 1000:1000 <image> sleep infinity` to bypass the entrypoint for one-off commands.

**`moveit_pro envfile` produces unusable output:** It dumps the entire shell environment. Instead, create a clean `.env` manually with only the `MOVEIT_*` variables (see the doosan_ws/.env for an example).

**Joints oscillating at rest / joints move when jogging a single joint:** The most common sim config issue. Symptoms: joint velocities are 1-5+ rad/s while positions are near the target, or moving one joint during teleop causes other joints to vibrate. Cause: actuator velocity damping is too low relative to stiffness, especially on lightweight wrist joints. **Fix:** Switch from explicit `kp`/`kv` to `kp`/`dampratio`. Use `kp="5000" dampratio="3"` uniformly on all joints. The `dampratio` auto-scales damping per joint inertia, preventing the underdamping that manual `kv` tuning causes on light joints. Also increase passive joint damping in scene.xml defaults (`<joint damping="50"/>`). **Always check joint velocities at rest after launching** (`ros2 topic echo /joint_states --once`) — if any arm joint velocity exceeds 0.01 rad/s, the actuators need more damping.

**Path tolerance violated / Controller aborted sub-trajectory:** Physics sim actuators lag behind commanded positions, causing the trajectory controller to abort. Two fixes needed:
1. **Relax path tolerance:** Create a sim-specific `config/control/<robot>_ros2_control.yaml` that sets `default_path_tolerance: 1.0` on the `joint_trajectory_admittance_controller`. This must be a **complete** copy of the base config's ros2_control yaml (not just the override) because `ros2_control.config` replaces the entire file. Add `ros2_control.config` to the sim config.yaml pointing to the new file.
2. **If still failing after path tolerance:** Check actuator gains — the joints may not be tracking accurately enough. Increase `kp` (try 8000-10000) while keeping `dampratio` at 3+.

**Simulation unstable / joints exploding:** Ensure contact exclusions cover all adjacent link pairs. Check that `gravcomp="1"` is set on all bodies (urdf_to_mjcf does this automatically). If using explicit `kv`, switch to `dampratio`. Increase `timestep` if needed (try 0.0005).

**Gripper passive joints drifting:** This is normal for the Robotiq 140 mimic joints in MuJoCo — the finger linkage isn't perfectly constrained without MuJoCo equality constraints. Small drift (<0.1 rad) is acceptable. For tighter coupling, add `<equality><joint>` constraints in the MJCF.
