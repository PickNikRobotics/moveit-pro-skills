---
name: create-objective
description: Build or modify MoveIt Pro Behavior Trees — Objectives (XML) and custom Behavior Nodes (C++). Use when the user asks to create, edit, or scaffold an Objective, or to add a new Behavior Node to a MoveIt Pro workspace. Accepts an inline description or a path to an SOP (Standard Operating Procedure) document (.md or .csv).
argument-hint: [inline description OR path to an SOP document (.md or .csv)]
---

You are building Behavior Trees for a MoveIt Pro robotics application. This workspace uses BehaviorTree.CPP with MoveIt Pro's plugin system.

## Your Task

$ARGUMENTS

## Determine the Workflow

This skill supports two workflows based on what the user provides:

### Workflow 1: Inline Description

The user describes the Objective directly in the invocation (e.g., `/create-objective Move to a waypoint, pick up an object, and place it on the table`). These are typically simple Objectives with straightforward sequential logic.

**Action:** Proceed directly to **Step 1: Discover Available Nodes**.

### Workflow 2: SOP (Standard Operating Procedure) Document

The user provides a path to an SOP document (e.g., `/create-objective path/to/sop.md` or `path/to/sop.csv`). SOPs describe complex, multi-phase workflows with fallbacks, conditional stages, and global interrupts. They typically come from domain experts who know the process but not Behavior Tree concepts.

Both **markdown** and **CSV** are accepted. Markdown SOPs are usually structured as numbered phases with sub-steps; CSV SOPs are usually one row per step with columns for phase, sub-phase, parameters, and fallback targets. Read the file as-is and infer the structure — modern LLMs parse tabular CSV SOPs as readily as prose markdown ones.

**Action:** Read the full SOP document, then follow **Interpreting an SOP** below before proceeding to Step 1.

## Interpreting an SOP (Standard Operating Procedure)

Read the full SOP first. Before searching for Behavior Nodes or writing any XML, build a mental model of the Behavior Tree structure by mapping SOP elements to BT concepts. See `example-sop-driver-reliability-testing.md` in this skill's directory for a worked markdown example.

### Mapping SOP Elements to BT Concepts

These mappings apply to both markdown and CSV SOPs — the surface form differs but the underlying concepts are the same.

- **Numbered top-level sections / distinct Phase column values** (e.g., "1. Setup", "2. Test Loop", "3. Teardown") → top-level named Sequences.
- **Sub-sections / Sub-phase column values** (e.g., "2a. Capture at each location") → nested named Sequences within the parent phase.
- **"Repeat N times"** → `Repeat` decorator wrapping a Sequence.
- **"For each X"** → `ForEach` decorator.
- **"If X fails, retry up to N times"** → `RetryUntilSuccessful` decorator.
- **"If still failing, log and skip"** → `ForceSuccess` wrapping the `RetryUntilSuccessful` (so the parent Sequence continues).
- **Parameters section / domain-specific CSV columns** (e.g., Coverage, Category) → SubTree input ports. When multiple rows share a Sub-phase but differ in these columns, the pattern is a parameterized SubTree.
- **Conditional steps** ("If X → Y") → `IfThenElse` or `Fallback` nodes.
- **Fallback columns / "on failure" branches** → local `Fallback` nodes if specific to a step, or `Parallel` monitoring branches if the same fallback applies to many steps (a global interrupt).
- **Continuation markers** (e.g., a `^` Process ID in a CSV row) → keep as part of the previous step; they are description continuations, not new nodes.

### Planning the BT Structure

After reading the SOP, outline the tree structure before writing XML:

1. **List the top-level phases** in order — these become the children of the root Sequence.
2. **Identify reusable patterns** — repeated Sub-phases with different parameters become SubTree XML files.
3. **Identify global interrupts** — fallbacks that span many steps become `Parallel` monitoring branches.
4. **Identify local fallbacks** — fallbacks on specific steps become `Fallback` nodes or `RetryUntilSuccessful` decorators.
5. **Identify parameters** — values that vary between uses of the same pattern become SubTree ports.

Present this outline to the user for review before proceeding to Step 1.

### Partial SOP Revisions

When a user updates only part of an SOP (e.g., adds a new phase, or modifies a fallback), update only the affected Behavior Tree files rather than regenerating everything.

1. **Identify what changed** — compare the revised SOP against the existing BT structure. Which phases, sub-phases, or fallbacks were added, removed, or modified?
2. **Map changes to XML files** — if SubTrees were properly extracted, each phase or reusable pattern has its own XML file. A change to one phase should only require editing that phase's SubTree XML.
3. **Leave unchanged files alone** — do not regenerate or reformat XML files that are not affected by the SOP revision.

| SOP Change | BT Impact |
|---|---|
| **Step added/removed within a phase** | Edit only that phase's SubTree XML. Add or remove the node. |
| **New phase added** | Create a new SubTree XML for the phase. Add a `<SubTree>` reference in the parent Objective at the correct position. |
| **Phase removed** | Remove the `<SubTree>` reference from the parent Objective. Ask the user before deleting the SubTree XML file — other Objectives may reference it. |
| **Fallback logic changed** | Edit the fallback's Sequence or the interrupt handler. If a fallback was added to new steps, check whether it should be a global interrupt (`Parallel`) or local (per-step `Fallback`). |
| **Global interrupt added/removed** | Edit the `Parallel` monitoring wrapper in the top-level Objective. |
| **Parameters changed** | Edit the port values on the affected SubTree references or the SubTree's port declarations. |
| **Steps reordered within a phase** | Edit that phase's SubTree XML to reorder the children of the Sequence. |

A well-structured Objective with phases extracted into SubTrees makes partial revisions clean and safe — changing one phase means editing one file. Invest in SubTree extraction upfront to make future revisions low-risk.

## Architecture Overview

There are two kinds of artifacts you may need to create:

1. **Objectives** — XML files that define Behavior Tree structure (flow control, sequencing, subtree composition).
2. **Behavior Nodes** — C++ classes that implement leaf-node actions (the actual robot logic).

Most tasks only require a new Objective XML. Only create a new C++ Behavior Node if the required functionality doesn't already exist. See [Behavior Trees concepts](https://docs.picknik.ai/concepts/behavior_trees/behavior_trees/) for background on the architecture.

## Step 1: Discover Available Nodes

Before writing anything, thoroughly search for existing Behavior Nodes that already do what's needed. There are three sources:

### Source A: Built-in BehaviorTree.CPP nodes

BehaviorTree.CPP ships with built-in control, decorator, and action nodes. Fetch documentation from https://www.behaviortree.dev/ to check what's available (e.g., control nodes like `Sequence`, `Fallback`, `IfThenElse`; decorators like `Repeat`, `RetryUntilSuccessful`, `Delay`, `ForceSuccess`; built-in actions like `SetBlackboard`, `Script`).

### Source B: MoveIt Pro Behavior Nodes

MoveIt Pro provides Behavior Node libraries loaded via plugins. Use these sources:

- https://picknik.ai/behaviors — searchable Behavior catalog with port definitions.
- https://docs.picknik.ai/ — full documentation including tutorials and migration guides.

The active plugins for a given application are listed in the `behavior_loader_plugins` section of its `config/config.yaml`. Check the target application's config to see which plugin loaders are active. Common MoveIt Pro plugin loaders include:

- `moveit_pro::behaviors::CoreBehaviorsLoader` — core motion planning, waypoints, gripper actions, planning scene management.
- `moveit_pro::behaviors::MTCCoreBehaviorsLoader` — MoveIt Task Constructor behaviors for pick/place, approach/retreat.
- `moveit_pro::behaviors::VisionBehaviorsLoader` — point cloud processing, perception, segmentation.
- `moveit_pro::behaviors::ConverterBehaviorsLoader` — data type conversion utilities.
- `moveit_pro::behaviors::NavBehaviorsLoader` — navigation behaviors.
- `moveit_pro::behaviors::MPCBehaviorsLoader` — model predictive control behaviors.
- `moveit_pro::behaviors::MujocoBehaviorsLoader` — MuJoCo simulation behaviors.

### Source C: Workspace-Local Behavior Nodes

This workspace contains custom Behavior packages with their own nodes:

- Search `src/*/objectives/*.xml` for Objectives that do similar things — reuse them as SubTrees.
- Search C++ behavior headers in `src/*/include/` to find available Action nodes and their ports.
- Check `register_behaviors.cpp` files to see what node IDs are registered.
- Check `behavior_plugin.yaml` files to find workspace-local plugin loaders (e.g., `example_behaviors::ExampleBehaviorsLoader`, `franka_behaviors::FrankaBehaviorsLoader`).

### Discovery Priority

1. First check if a built-in BehaviorTree.CPP node or existing MoveIt Pro Behavior already does what's needed.
2. Then check if an existing Objective XML in the workspace can be reused as a SubTree.
3. Consider whether the task can be accomplished by composing existing nodes with control flow (`Repeat`, `ForEach`, `Script`, etc.) rather than writing new C++ code.
4. Only create a new C++ Behavior Node as a last resort.

### CRITICAL: Confirm Before Creating Custom C++ Behavior Nodes

**NEVER create a custom C++ Behavior Node without first asking the user for confirmation.** Before proposing a custom node, you MUST:

1. Explain what functionality you believe is missing from existing nodes.
2. Describe the specific approach you'd take using only existing nodes and XML composition.
3. Explain why you think that approach is insufficient.
4. Ask the user whether they agree a custom node is needed, or if there are existing Behaviors you may have missed.

Most tasks can be accomplished purely with XML by composing existing BehaviorTree.CPP built-in nodes, MoveIt Pro Behaviors, and control flow. Prefer verbose XML with `Repeat`/`ForEach`/`Script` loops over writing new C++. A longer XML file that reuses existing nodes is better than a shorter one that requires a new C++ Behavior.

## Step 2: Write the Objective XML

Place new Objective XML files in the appropriate `objectives/` directory for the target application (e.g., `src/factory_sim/objectives/`, `src/lab_sim/objectives/`). Ask the user which application if unclear.

### CRITICAL: Node and Port Validation

Hallucinated node IDs and port names are the most common source of broken Objectives. Before using any Behavior Node in XML:

- **Verify the node exists.** Every Action, Decorator, or SubTree ID you reference must correspond to a real registered Behavior or existing Objective. Do not invent node IDs. If you have not confirmed a node ID against the Behavior catalog, an existing Objective, or a `register_*_behaviors.cpp` file, do not use it.
- **Verify every port.** Only use input, output, and bidirectional ports that are actually defined for a node. Check the node's `providedPorts()` in its header file, the catalog at https://picknik.ai/behaviors, or an existing Objective that uses the same node. Do not guess port names or types.
- **Verify type compatibility.** Ensure connected ports have compatible types. A `geometry_msgs::msg::PoseStamped` output cannot connect to a `std::string` input. When in doubt, check both sides.

If a workspace exposes batch validation tooling (e.g., `validate_behavior` / `validate_behaviors_batch` and `get_behavior_details` / `get_behavior_details_batch`), prefer it over manual lookup.

### XML Format

```xml
<root BTCPP_format="4" main_tree_to_execute="Tree Name Here">
  <BehaviorTree ID="Tree Name Here" _description="Human-readable description">
    <Control ID="Sequence" name="root">
      <!-- Actions, SubTrees, Decorators go here -->
    </Control>
  </BehaviorTree>
  <TreeNodesModel>
    <SubTree ID="Tree Name Here">
      <MetadataFields>
        <Metadata subcategory="Category Name" />
        <Metadata runnable="true" />
      </MetadataFields>
      <!-- Declare any ports exposed by this subtree -->
      <inout_port name="param_name" default="" />
    </SubTree>
  </TreeNodesModel>
</root>
```

### Key XML Patterns

- **Blackboard variables**: Use `{variable_name}` syntax to pass data between nodes: `output="{my_var}"` then `input="{my_var}"`.
- **SubTree references**: `<SubTree ID="Existing Objective Name" _collapsed="true" port_name="value" />`.
- **Control nodes**: `Sequence` (all must succeed), `Fallback` (first success wins), `IfThenElse`.
- **Decorators**: `<Decorator ID="Repeat" num_cycles="3">`, `<Decorator ID="RetryUntilSuccessful" num_attempts="2">`.
- **Comments**: Use XML comments `<!-- description -->` and the `name` attribute on nodes for readability.
- Set `runnable="true"` in `MetadataFields` if the Objective can be executed directly (top-level Objective).
- Set `_collapsed="true"` on SubTree references for UI readability.

### CRITICAL: Editing Existing Objectives

When modifying an existing Objective (rather than creating a new one), always read the full XML first. You MUST preserve the following unless the user explicitly asks to change them:

- The `main_tree_to_execute` attribute on the `<root>` element.
- The `ID` attribute on the `<BehaviorTree>` element (must match `main_tree_to_execute`).
- All `<MetadataFields>` and `<Metadata>` elements and their attributes.
- The `_collapsed` attribute on SubTree references.
- XML comments and formatting where possible.
- The `<TreeNodesModel>` section and its port declarations.

Mismatches between `main_tree_to_execute` and the primary `BehaviorTree ID` are a common failure mode — double-check both after any edit.

### Structuring Trees for Readability

A flat Sequence with many children is hard to read, debug, and reuse. Structure trees using **named nested Sequences** for logical grouping and **SubTrees** for reusable parameterized patterns.

#### Named Nested Sequences

When a Sequence has more than 4–5 direct children, look for logical phases and group them into named nested Sequences. Give each nested Sequence a descriptive `name` attribute.

**Anti-pattern — flat Sequence with 8 children:**

```xml
<Control ID="Sequence">
  <Action ID="GetCurrentPlanningScene" ... />
  <Action ID="GetRobotJointState" ... />
  <Action ID="UnpackJointStateMessage" ... />
  <Action ID="CreateJointState" ... />
  <Action ID="GeneratePointToPointTrajectory" ... />
  <Action ID="SwitchController" ... />
  <Action ID="ValidateTrajectory" ... />
  <Action ID="ExecuteTrajectory" ... />
</Control>
```

**Better — grouped by logical phase:**

```xml
<Control ID="Sequence" name="InterpolateToJointState">
  <Control ID="Sequence" name="GetCurrentState">
    <Action ID="GetCurrentPlanningScene" ... />
    <Action ID="GetRobotJointState" ... />
    <Action ID="UnpackJointStateMessage" ... />
  </Control>
  <Control ID="Sequence" name="PlanTrajectory">
    <Action ID="CreateJointState" ... />
    <Action ID="GeneratePointToPointTrajectory" ... />
    <Action ID="ValidateTrajectory" ... />
  </Control>
  <Control ID="Sequence" name="Execute">
    <Action ID="SwitchController" ... />
    <Action ID="ExecuteTrajectory" ... />
  </Control>
</Control>
```

**Guidelines:**

- Group 2–4 related steps into a named nested Sequence when they form a logical unit (setup, compute, validate, execute, cleanup).
- Always set the `name` attribute on nested Sequences — this makes them self-documenting and readable in the MoveIt Pro UI tree view.
- Name the top-level Sequence too (e.g., `name="root"` or a descriptive name like `name="PickAndPlace"`).
- Don't over-nest — if a group only has one child, don't wrap it in a Sequence. Two children is the minimum for a group worth creating.

#### When to Extract SubTrees

Extract a group of steps into a separate SubTree XML when any of these apply:

1. **Reused pattern** — the same sequence of steps (with different parameters) appears 2+ times in one tree, or across multiple Objectives.
2. **Logically self-contained** — the group has clear inputs, does one coherent thing, and produces clear outputs (e.g., "capture and send a point cloud", "plan and execute a motion").
3. **Parameterizable** — the group differs only in a few values between uses (waypoint name, planning group, scale factor). These become SubTree ports.
4. **Complexity management** — a nested Sequence is growing beyond 5–6 children and would benefit from being its own named, collapsible tree.

**Example — repeated pattern becomes a SubTree:**

```xml
<!-- Anti-pattern: duplicated steps -->
<Control ID="Sequence" name="CaptureAllLocations">
  <SubTree ID="Move to Waypoint" waypoint_name="right_side" />
  <Action ID="GetPointCloud" point_cloud="{pc_right}" />
  <Action ID="SendPointCloudToUI" point_cloud="{pc_right}" />
  <SubTree ID="Move to Waypoint" waypoint_name="middle" />
  <Action ID="GetPointCloud" point_cloud="{pc_middle}" />
  <Action ID="SendPointCloudToUI" point_cloud="{pc_middle}" />
  <SubTree ID="Move to Waypoint" waypoint_name="left_side" />
  <Action ID="GetPointCloud" point_cloud="{pc_left}" />
  <Action ID="SendPointCloudToUI" point_cloud="{pc_left}" />
</Control>
```

Extract the repeated pattern into its own SubTree XML file:

```xml
<!-- File: capture_point_cloud_at_waypoint.xml -->
<root BTCPP_format="4" main_tree_to_execute="Capture Point Cloud At Waypoint">
  <BehaviorTree ID="Capture Point Cloud At Waypoint">
    <Control ID="Sequence" name="CaptureAtWaypoint">
      <SubTree ID="Move to Waypoint" waypoint_name="{waypoint_name}" />
      <Action ID="GetPointCloud" point_cloud="{point_cloud}" />
      <Action ID="SendPointCloudToUI" point_cloud="{point_cloud}" />
    </Control>
  </BehaviorTree>
  <TreeNodesModel>
    <SubTree ID="Capture Point Cloud At Waypoint">
      <input_port name="waypoint_name" default="" />
      <output_port name="point_cloud" default="{point_cloud}" />
    </SubTree>
  </TreeNodesModel>
</root>
```

Then the parent tree becomes clean and scannable:

```xml
<Control ID="Sequence" name="CaptureAllLocations">
  <SubTree ID="Capture Point Cloud At Waypoint" _collapsed="true"
           waypoint_name="right_side" point_cloud="{pc_right}" />
  <SubTree ID="Capture Point Cloud At Waypoint" _collapsed="true"
           waypoint_name="middle" point_cloud="{pc_middle}" />
  <SubTree ID="Capture Point Cloud At Waypoint" _collapsed="true"
           waypoint_name="left_side" point_cloud="{pc_left}" />
</Control>
```

#### Readability Checklist

Before finalizing an Objective XML, check:

- [ ] Does the top-level Sequence have a `name` attribute?
- [ ] Are there more than 5 direct children under any Sequence? If so, can they be grouped into named nested Sequences?
- [ ] Are there duplicated step patterns? If so, extract them into SubTrees.
- [ ] Does every nested Sequence have a descriptive `name`?
- [ ] Can a reader understand the tree's high-level flow by reading only the Sequence names and SubTree IDs (without expanding every node)?

## Step 3: Write a Custom Behavior Node (only after user confirmation)

**Do not proceed unless the user has explicitly confirmed that a custom C++ Behavior Node is needed.** See "CRITICAL: Confirm Before Creating Custom C++ Behavior Nodes" above.

For all C++ authoring conventions — base class selection, header/source layout, port constants and naming, metadata, registration, and CMake — follow the guidance in the workspace's `src/behavior/CLAUDE.md` (creating Behavior plugins) and the official documentation:

- https://docs.picknik.ai/concepts/creating_behaviors/creating_behaviors/ — concepts and base class overview.
- https://docs.picknik.ai/how_to/custom_behaviors/additional_behavior_classes/additional_behavior_classes/ — how to choose between `SyncActionNode`, `AsyncBehaviorBase`, `ServiceClientBehaviorBase`, `ActionClientBehaviorBase`, `GetMessageFromTopicBehaviorBase`, etc.
- https://docs.picknik.ai/how_to/custom_behaviors/adding_ports/adding_ports/ — port declarations and validation.

Read those references first; they are the source of truth and stay in sync with the codebase. After implementing the node:

1. Add the include to the package's `register_behaviors.cpp` and call `registerBehavior<YourNodeName>(factory, "YourNodeName", shared_resources);`.
2. Add the source file to the `add_library()` call in `CMakeLists.txt`.
3. Add any new dependencies to both `CMakeLists.txt` (`THIS_PACKAGE_INCLUDE_DEPENDS`) and `package.xml`.
4. Add the new node to the `kXxxBehaviorsScenario` list in `test/test_load_behavior_loader_plugin.cpp` so plugin loading is verified.

## Workspace Reference Examples

- Example Behaviors: `src/example_behaviors/`
- Real Objective examples: `src/factory_sim/objectives/` and `src/lab_sim/objectives/`
- Registration pattern: `src/example_behaviors/src/register_behaviors.cpp`
- Plugin descriptor: `src/example_behaviors/example_behaviors_plugin_description.xml`

## Example SOP (Standard Operating Procedure) Document

`example-sop-driver-reliability-testing.md` (in this skill's directory) demonstrates a markdown SOP: setup/loop/teardown phases, a repeated capture-at-waypoint pattern that becomes a SubTree, and retries with `RetryUntilSuccessful` + `ForceSuccess`. Use it as a template for the kind of structure a markdown SOP should have.
