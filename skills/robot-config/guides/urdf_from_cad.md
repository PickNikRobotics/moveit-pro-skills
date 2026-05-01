# Building a URDF from CAD/STL Files

This guide covers constructing a URDF from raw mesh files when no existing robot description package is available. After completing this guide, proceed to [Creating a MoveIt Pro Config](existing_urdf_config.md) to build the full MoveIt Pro configuration.

### Prerequisites

- STL or DAE mesh files for each link (visual and/or collision)
- Knowledge of the robot's kinematic structure (DH parameters or joint axes/origins)
- Joint types (revolute, prismatic, fixed) and limits

### Step-by-Step

#### 1. Organize Meshes

```
<package_name>/
├── meshes/
│   ├── visual/           # Detailed meshes for rendering (.dae preferred, .stl ok)
│   │   ├── base_link.dae
│   │   ├── link1.dae
│   │   └── ...
│   └── collision/        # Simplified meshes for collision checking (.stl preferred)
│       ├── base_link.stl
│       ├── link1.stl
│       └── ...
```

Tips:
- Collision meshes should be simplified (convex hulls or low-poly) for performance
- If you only have one set of meshes, use them for both visual and collision
- Ensure mesh origins align with the joint frames (or adjust with `<origin>` in URDF)
- Scale: ROS uses meters. If your STLs are in mm, add `scale="0.001 0.001 0.001"` in the URDF mesh tag

#### 2. Build the URDF

Create the URDF from scratch. For each link in the kinematic chain:

```xml
<link name="link_1">
  <visual>
    <geometry>
      <mesh filename="package://<package>/meshes/visual/link1.dae"/>
    </geometry>
    <origin xyz="0 0 0" rpy="0 0 0"/>
  </visual>
  <collision>
    <geometry>
      <mesh filename="package://<package>/meshes/collision/link1.stl"/>
    </geometry>
    <origin xyz="0 0 0" rpy="0 0 0"/>
  </collision>
  <inertial>
    <mass value="1.0"/>
    <inertia ixx="0.01" iyy="0.01" izz="0.01" ixy="0" ixz="0" iyz="0"/>
  </inertial>
</link>
```

For each joint:

```xml
<joint name="joint_1" type="revolute">
  <parent link="base_link"/>
  <child link="link_1"/>
  <origin xyz="0 0 0.1" rpy="0 0 0"/>
  <axis xyz="0 0 1"/>
  <limit lower="-3.14" upper="3.14" velocity="3.15" effort="100"/>
</joint>
```

Key points:
- Every link needs `<inertial>` even for visualization-only (MoveIt will warn otherwise)
- Joint `<origin>` is the transform from parent to child frame at joint position zero
- Double-check axis directions — wrong axis = robot moves in unexpected directions
- Use XACRO macros for repetitive patterns (common on serial manipulators)

#### 3. Validate the URDF

```bash
# Inside the container or with ROS installed:
check_urdf my_robot.urdf
urdf_to_graphiz my_robot.urdf     # Generates a visual kinematic tree
```

Visually verify in RViz or the MoveIt Pro UI that:
- Links are positioned correctly relative to each other
- Joints rotate around the correct axes
- Meshes are oriented properly (no flipped/rotated parts)

#### 4. Generate Collision Matrix

If you have access to the MoveIt Setup Assistant:
```bash
ros2 launch moveit_setup_assistant setup_assistant.launch.py
```

Otherwise, manually create the SRDF by:
1. Listing all link pairs that are adjacent (connected by a joint) -> `reason="Adjacent"`
2. Testing various joint configurations for unexpected collisions between non-adjacent links -> `reason="Never"`
3. Pay special attention to the zero/home pose

#### 5. Proceed to Creating a MoveIt Pro Config

Once you have a valid URDF with meshes, proceed to [Creating a MoveIt Pro Config](existing_urdf_config.md) from Step 2 onward.

#### 6. Optimize Meshes

See https://docs.picknik.ai/how_to/configuration_tutorials/optimizing_robot_model_meshes/

**Decimation** — Reduce polygon count (target: under 100K faces / 10MB):
```bash
moveit_pro dev decimate_mesh body.stl body_decimated.stl
```

**Convex Decomposition** — For MuJoCo physics simulation:
1. Export mesh as .obj from Blender (Forward=Y, Up=Z)
2. `pip install coacd && coacd -i body.obj -o body_decomp.obj -c 7`
3. Use Blender `separate_hulls.py` to extract individual .stl convex hulls
4. Reference original .obj as visual, individual .stl hulls as collision in MuJoCo XML

### Mesh-Specific Gotchas

- **Wrong scale**: Robot appears tiny or enormous -> check STL units vs meters
- **Wrong origin**: Mesh appears offset from joint -> adjust `<origin>` in `<visual>` and `<collision>` tags
- **Missing inertials**: MoveIt/Gazebo will warn or crash -> add reasonable `<inertial>` to every non-fixed link
- **Complex collision meshes**: Planning is slow -> use simplified convex hulls for collision geometry
- **Mesh file not found at runtime**: The package must be built and installed so `package://` URIs resolve -> `moveit_pro build`
