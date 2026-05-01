# Mobile-base + wholebody planning in MoveIt Pro

How to set up a planning group that combines the mobile base with the manipulator(s) so MoveIt can plan coordinated motions involving both, and how to switch between arm-only and wholebody groups at runtime.

## The three layers that have to line up

Wholebody planning is a 3-layer setup; missing or mismatched config in any layer breaks the rest.

### Layer 1 — URDF / SRDF: declare a planar base + a wholebody-rooted chain

There are two valid patterns in the MoveIt Pro ecosystem; both are fine.

**Pattern A — `<virtual_joint type="planar">` in the SRDF** (the docs' canonical recipe):

```xml
<virtual_joint name="position" type="planar" parent_frame="odom" child_link="base_footprint" />
<joint_property joint_name="position" property_name="motion_model" value="diff_drive" />
<!-- For a holonomic / Mecanum base: motion_model="holonomic" -->
```

Plus dummy 1-DoF joints inside the URDF's `<ros2_control>` tag for `linear_x_joint`, `linear_y_joint`, `rotational_yaw_joint`, kept detached from the kinematic chain so they're invisible to TF and visualization but claimable by ros2_control.

**Pattern B — three real URDF joints decomposing the planar motion** (what the in-repo `phoebe_sim` reference uses):

```xml
<link name="world"/>
<link name="virtual_rail_link_1"/>
<joint name="linear_x_joint" type="prismatic"><parent link="world"/><child link="virtual_rail_link_1"/>...</joint>
<link name="virtual_rail_link_2"/>
<joint name="linear_y_joint" type="prismatic"><parent link="virtual_rail_link_1"/>...</joint>
<link name="base_link"/>
<joint name="rotational_yaw_joint" type="continuous"><parent link="virtual_rail_link_2"/>...</joint>
```

These are real URDF joints, not virtual ones — they appear in TF and are claimable by ros2_control normally.

In either pattern, the SRDF then defines the wholebody group as a chain rooted at `world`:

```xml
<group name="wholebody">
  <chain base_link="world" tip_link="grasp_link"/>
</group>
```

A chain rooted at `world` automatically picks up the planar base DoFs (whichever pattern they came from) plus the manipulator chain.

### Layer 2 — ros2_control: a JTC for the planar joint variables

A second `JointTrajectoryController` claims the three planar joints (`linear_x_joint`, `linear_y_joint`, `rotational_yaw_joint`) and chains its output to the actual base drive controller (e.g., `DiffDriveController`, holonomic / mecanum drive controller, etc.). The arm's JTC stays separate. See `phoebe_sim/config/control/dual_arm.ros2_control.yaml` for a working example.

### Layer 3 — Objectives: per-Behavior `planning_group_name` port

There is no global "active group" in MoveIt Pro. Every planning Behavior (`SetupMTCPlanToPose`, `SetupMTCPlanToJointState`, `PlanPath`, etc.) takes a `planning_group_name` port that selects the group for *that* Behavior call.

Common patterns:

- **Static per-objective:** hard-code the port — e.g., `planning_group_name="manipulator"` for arm-only objectives, `planning_group_name="wholebody"` for objectives that should move the base too.
- **Runtime-switched within an objective:** drive the port from a blackboard variable that gets set upstream (e.g., from a perception result or a higher-level decision).

## Reference workspace: phoebe_sim

The PickNik `phoebe_ws` (https://github.com/PickNikRobotics/phoebe_ws) is the canonical mobile-base reference example shipped with `moveit_pro_example_ws`. It uses Pattern B (3 real URDF joints) and demonstrates:

- The URDF structure: `phoebe_sim/description/phoebe_sim.urdf.xacro`.
- The SRDF group taxonomy: `phoebe_sim/config/moveit/phoebe_sim.srdf` defines `mobile_base`, `left_manipulator`, `right_manipulator`, `arms_only`, `arms_lift_only`, `manipulator` (a true wholebody chain rooted at `world` for both arms), `left_mobile_manipulator` / `right_mobile_manipulator` (single-arm wholebody), plus gripper and lift groups.
- The ros2_control wiring: `phoebe_sim/config/control/dual_arm.ros2_control.yaml`.

Per-objective group selection (4 of phoebe's groups are actually used by shipped objectives):

| Group | Example objective | Purpose |
|---|---|---|
| `left_manipulator` | `objectives/single-tip_path_admittance_move_push_bottle.xml` | left arm only |
| `right_manipulator` | `objectives/record_admittance_trajectory.xml` | right arm only |
| `arms_only` | `objectives/multi-tip_path_move_push_bottle.xml` | both arms simultaneously, no base |
| `mobile_base` | `objectives/mobile_base_waypoint.xml` | base only (3 planar DoFs) |
| `manipulator` | _(none — defined in SRDF as the true wholebody chain, but not exercised by any shipped objective)_ | both arms + base, fully coordinated |

**Honest caveat — phoebe demonstrates the SRDF mechanism for wholebody but does not exercise it in any shipped BT.** Phoebe's BTs plan arm motion and base motion as separate objectives. If a project wants single-objective, fully-coordinated wholebody plans, the planning is on relatively new ground for MoveIt Pro reference content; expect to do some testing.

## Considerations to nail down before designing your group taxonomy

1. **What's the actual base kinematics?** Diff-drive vs. omnidirectional / Mecanum vs. 4-wheel-steering all use different `motion_model` values in the SRDF and different drive controllers in ros2_control. The wholebody group's chain doesn't care, but the controller layer must match.
2. **Which IK solver covers wholebody?** Stock KDL handles 6/7-DoF arm chains well but can struggle with the redundancy of (3 planar + arm) high-DoF groups. `pick_ik` (PickNik's, often the default in MoveIt Pro) and `bio_ik` are the practical choices. `kinematics.yaml` needs an entry for the wholebody group; the arm-only entry doesn't transfer.
3. **Controller activation per group.** When switching between an arm-only objective and a wholebody one, the *active* controller set must include the base JTC for wholebody and not for arm-only (or include both safely). MoveIt Pro has a `SetActiveController` Behavior; check phoebe's objectives for the pattern.
4. **Whether wholebody planning is the goal, or just coordinated arm-and-base actions.** The phoebe pattern (separate objectives for arm and base, both running in sequence or in parallel) sidesteps the IK-solver and singularity questions of true single-plan wholebody, at the cost of less-coordinated motion. Consider whether the use case demands true wholebody plans or whether sequenced arm + base motions are sufficient.

## Pitfalls observed elsewhere

- `hangar_sim` defines a `manipulator` group whose chain is rooted at `ridgeback_base_link` (the mobile-base body), NOT at `world`. **This is not wholebody planning** — the planner treats the base as fixed; mobile motion is managed separately (Nav2-style). Easy to mistake for a wholebody example because of the "manipulator" name and the presence of a mobile robot in the URDF.
- Naming the chain root matters more than the group name. `<chain base_link="world" ...>` is wholebody-capable; `<chain base_link="<some_robot_link>" ...>` is not, regardless of what the group is called.

## Related references

- Official mobile-base configuration tutorial: `~/moveit_pro_dev/src/docs/docs/how_to/configuration_tutorials/configure_mobile_base/configure_mobile_base.md`. Describes Pattern A (virtual_joint).
- Nav2 integration: `~/moveit_pro_dev/src/docs/docs/how_to/configuration_tutorials/add_nav2/`.
