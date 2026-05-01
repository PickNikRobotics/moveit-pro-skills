---
name: create-behavior
description: Author a new MoveIt Pro Behavior plugin (C++) — header, source, registration, CMake, and test wiring. Use when the user asks to add a custom Behavior, when an Objective needs a leaf Action that doesn't exist in the catalog, or when porting external robot logic into MoveIt Pro as a registered Behavior. Pairs with the create-objective skill, which handles the XML side of Behavior Trees.
argument-hint: [optional behavior description, e.g. "compute pose offset between two frames"]
---

# Create a Behavior Plugin

Author a new MoveIt Pro Behavior — a C++ plugin that implements a single leaf-node action (motion step, sensing, computation) callable from an Objective XML. This skill covers the conventions, file layout, registration, and test wiring; the official documentation linked at the bottom is the source of truth and stays in sync with the runtime.

## When NOT to Author a New Behavior

Before writing C++, exhaust the alternatives:

1. **Search the catalog.** https://picknik.ai/behaviors lists every shipped Behavior with port definitions. Many tasks (waypoints, gripper actions, motion planning, point-cloud capture) already have a Behavior.
2. **Compose existing Behaviors with control flow.** A longer XML using `Repeat` / `ForEach` / `Script` / `IfThenElse` plus existing Actions is preferable to new C++.
3. **Reuse an existing Objective as a SubTree.** If similar logic exists as an Objective XML in the workspace, reference it via `<SubTree ID="...">` rather than re-implementing.

Only proceed to authoring C++ when none of the above fit. If you're unsure, present the gap to the user and ask before writing C++ — see the **CRITICAL: Confirm Before Creating Custom C++ Behavior Nodes** section in the `create-objective` skill.

## Pick a Base Class

| Base class | When to use | Override |
|---|---|---|
| `SharedResourcesNode<BT::SyncActionNode>` | Synchronous, completes in a single tick (< 1 ms). | `tick()` |
| `SharedResourcesNode<BT::StatefulActionNode>` | Multi-tick work with explicit state. | `onStart()`, `onRunning()`, `onHalted()` |
| `AsyncBehaviorBase` | Long-running tasks (planning, vision inference). Returns `RUNNING` until complete. | `doWork()` returning `tl::expected<bool, std::string>` |
| `GetMessageFromTopicBehaviorBase<MsgT>` | Subscribe to a ROS topic and output the latest message. | (inherits async topic listening) |
| `ServiceClientBehaviorBase<SrvT>` | Call a ROS service. | `getServiceName()`, `createRequest()`, `processResponse()` |
| `ActionClientBehaviorBase<ActionT>` | Call a ROS action server. | `getActionName()`, `createGoal()`, `getResultTimeout()` |
| `BT::SyncActionNode` / `BT::StatefulActionNode` | No ROS resources needed at all. | `tick()` (or stateful equivalents) |

Authoritative reference: https://docs.picknik.ai/how_to/custom_behaviors/additional_behavior_classes/

## Class and File Naming

- **Class:** `PascalCase`, descriptive and specific. Prefer `GetCenterMostAprilTag` over `GetTag`. Include the method/sensor when relevant (`UsingAprilTags`, `FromPointCloud`).
- **File:** `snake_case` matching the class — e.g., `compute_tray_place_positions_using_apriltags.hpp` and `.cpp`.

## Header File

Place at `src/<package>/include/<package>/<file>.hpp`.

- Include the project copyright header (current year).
- Use `#pragma once`.
- Mark the class `final` unless designed for inheritance.
- Add a `@brief` one-liner and a `@details` block with a markdown port table listing every port.
- Document constructor parameters with `@param` tags.

```cpp
#pragma once

#include <behaviortree_cpp/action_node.h>
#include <moveit_pro_behavior_interface/shared_resources_node.hpp>

namespace your_namespace
{

/**
 * @brief Brief description of what this Behavior does.
 *
 * @details Longer explanation of the algorithm, inputs, and outputs.
 *
 * | Data Port Name | Port Type | Object Type                       |
 * | -------------- | --------- | --------------------------------- |
 * | input_pose     | input     | geometry_msgs::msg::PoseStamped   |
 * | output_pose    | output    | geometry_msgs::msg::PoseStamped   |
 */
class YourBehaviorName final
  : public moveit_pro::behaviors::SharedResourcesNode<BT::SyncActionNode>
{
public:
  /**
   * @param name Name assigned to this Behavior in the Objective XML.
   * @param config Behavior tree config provided at construction.
   * @param shared_resources Shared MoveIt Pro context (logger, TF buffer, ROS node).
   */
  YourBehaviorName(const std::string& name, const BT::NodeConfiguration& config,
                   const std::shared_ptr<moveit_pro::behaviors::BehaviorContext>& shared_resources);

  static BT::PortsList providedPorts();
  static BT::KeyValueVector metadata();
  BT::NodeStatus tick() override;
};

}  // namespace your_namespace
```

Port Type column values: `input`, `output`, or `bidirectional`.

## Source File

Place at `src/<package>/src/<file>.cpp`.

### Port Constants

Define port-name constants as `constexpr auto` in an anonymous namespace, with the `kPortID` prefix:

```cpp
namespace
{
constexpr auto kPortIDInputPose = "input_pose";
constexpr auto kPortIDOutputPose = "output_pose";
constexpr auto kPortIDMarkerSize = "marker_size";
}  // namespace
```

### Port Definitions

In `providedPorts()`:

- `BT::InputPort<T>` — data flowing in (read-only). Most common.
  - Required: `BT::InputPort<T>("name", "{placeholder}", "Description.")`
  - Optional with default: `BT::InputPort<T>("name", default_value, "Description.")`
- `BT::OutputPort<T>` — data the Behavior produces. Always uses a blackboard placeholder.
- `BT::BidirectionalPort<T>` — reads and writes the same port (e.g., accumulating into a vector). Use sparingly.

Most ports should be `InputPort`. Only use `OutputPort` for data the Behavior produces. Only use `BidirectionalPort` when the Behavior must modify an existing blackboard value in-place.

Reference: https://docs.picknik.ai/how_to/custom_behaviors/adding_ports/

### Port Descriptions

Every port **must** have a description string. Descriptions should:

- Be a complete sentence ending with a period.
- Explain the purpose, not just restate the type — "Distance in meters between adjacent place positions." not "A double."
- Mention units where applicable (meters, seconds, radians).
- Note constraints or special values ("0 means forever.", "Negative indexes start backwards from the last element.").
- For optional ports, state what happens when not provided ("If not provided, visualization is skipped.").

### Port Naming

- `snake_case` for port names: `input_pose`, `camera_info`, `tray_apriltag_id`.
- Prefix input ports with context when ambiguous: `input_image`, `input_pose`, `target_frame_id`.
- Suffix output ports descriptively: `place_positions`, `detection_pose`, `annotated_image`.
- For topic-name ports, use `_topic` suffix: `visualization_topic`.
- Match existing conventions — check similar Behaviors before inventing new names.

### Metadata

Provide both `subcategory` and `description` metadata. The description is HTML wrapped in `<p>` tags and is rendered in the Behavior Hub UI.

```cpp
inline constexpr auto kDescriptionYourBehaviorName = R"(
                <p>
                    Clear, concise description. Explain inputs, outputs, and side effects.
                </p>
            )";

BT::KeyValueVector YourBehaviorName::metadata()
{
  return { { kSubcategoryMetadataKey, "Pose Handling" },
           { kDescriptionMetadataKey, kDescriptionYourBehaviorName } };
}
```

Subcategory examples: `"Vision"`, `"Pose Handling"`, `"Vector Handling"`, `"Motion - Planning"`, `"Perception - 2D Image"`.

### Input Validation

For required ports, use `getRequiredInputs()` and short-circuit on failure:

```cpp
BT::NodeStatus YourBehaviorName::tick()
{
  const auto ports = getRequiredInputs(
      getInput<SomeMsg>(kPortIDDetections),
      getInput<CameraInfo>(kPortIDCameraInfo));
  if (!ports.has_value())
  {
    shared_resources_->logger->publishFailureMessage(
        name(), "Failed to get required values from input data ports: " + ports.error());
    return BT::NodeStatus::FAILURE;
  }
  const auto& [detections, camera_info] = ports.value();

  // ... do the work ...

  setOutput(kPortIDOutputPose, result);
  return BT::NodeStatus::SUCCESS;
}
```

For optional ports with defaults, use `.value()` or `.value_or()`:

```cpp
const double marker_size = getInput<double>(kPortIDMarkerSize).value();  // has default
const bool visualize = getInput<bool>(kPortIDVisualize).value_or(false);
```

`.value()` is correct when the port has a default in `providedPorts()` — `getInput()` is guaranteed to succeed in that case. Reserve `.value_or()` for truly optional ports that may not be wired at all.

## Registration

After implementing the Behavior, wire it into the package:

1. Add `#include "<package>/<your_behavior>.hpp"` to the package's `register_*_behaviors.cpp`, keeping alphabetical order.
2. Inside the registration function, call:

   ```cpp
   moveit_pro::behaviors::registerBehavior<YourBehaviorName>(
       factory, "YourBehaviorName", shared_resources);
   ```

3. Add `"YourBehaviorName"` to the corresponding scenario list in `test/test_load_behavior_loader_plugin.cpp` (alphabetically) so plugin loading is verified.

## CMake

Add a static library block in the package's `CMakeLists.txt`:

```cmake
add_library(your_behavior_name STATIC src/your_behavior_name.cpp)
target_link_libraries(your_behavior_name
  PUBLIC rsl::rsl <other_public_deps>)
target_link_libraries(your_behavior_name PRIVATE fmt::fmt)  # if using fmt
target_include_directories(your_behavior_name
  PUBLIC $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/include> $<INSTALL_INTERFACE:include>)
```

Then add `your_behavior_name` to the PRIVATE link list of the package's shared library target (e.g., the `core_behaviors` / `core_behaviors_vision` aggregation).

If the Behavior pulls in new dependencies, add them to both `CMakeLists.txt` (`THIS_PACKAGE_INCLUDE_DEPENDS`) and `package.xml`.

## Testing

- **Minimum:** Every Behavior must be listed in `test_load_behavior_loader_plugin.cpp` — this verifies it can be loaded and instantiated.
- **Recommended:** Use the `WithBehavior<T>` test fixture from `moveit_pro_behavior_interface`:
  - Define a port setter map with `BEGIN_BEHAVIOR_PORT_SETTER_MAP` / `DEFINE_BEHAVIOR_PORT_SETTER` / `DEFINE_OPTIONAL_BEHAVIOR_PORT_SETTER`.
  - Use `INSTANTIATE_SYNC_BEHAVIOR_PORT_NOT_SET_TESTS` (or the `ASYNC` variant) to auto-generate "port not set" tests for every required port.
- Always set `tf2_spin_thread=false` in `BehaviorContext` for tests.
- For deterministic unit tests, extract pure logic out of `tick()` into free functions and test those directly. ROS / TF / topic interactions should stay in `tick()`.

## Common Patterns

- **TF lookups:** call `shared_resources_->transform_buffer_ptr->canTransform()` before `lookupTransform()`. Include `<tf2_eigen/tf2_eigen.hpp>` for `tf2::transformToEigen()`.
- **Image annotation:** use `cv_bridge::toCvCopy()` + OpenCV drawing + `ROSPublisherHandle` to publish annotated images.
- **Camera intrinsics:** extract from `CameraInfo.k[]` — `fx=k[0]`, `fy=k[4]`, `cx=k[2]`, `cy=k[5]`.
- **Pinhole projection:** `u = fx * (x/z) + cx`, `v = fy * (y/z) + cy`.

## Design Tips

- **Keep `tick()` focused.** If it grows beyond ~40 lines, extract private helper methods (e.g., `publishVisualizationMarkers()`). `tick()` should read like an outline.
- **Name utilities by what they do, not by the first use case.** `marker_utils` (lines, poses, text, delete-all) is better than `pose_marker_utils`.
- **Use the Behavior in an Objective.** Once registered and built, exercise the Behavior from an Objective XML — the `create-objective` skill covers the XML side, including the **Node and Port Validation** rules that prevent the most common Objective-side mistakes.

## Build and Verify

After authoring, registering, and updating CMake:

```bash
moveit_pro build user_workspace
moveit_pro run
```

Expect to see your Behavior appear in the Behavior catalog dropdown in the UI, and to be loadable from any Objective XML by `<Action ID="YourBehaviorName" ... />`.

If load fails with `Missing manifest for element_ID: YourBehaviorName`, see the `debug-objective` skill — it usually means registration is wrong, the package wasn't rebuilt, or the active config's `behavior_loader_plugins` doesn't include the right loader.

## Authoritative References

- **Concepts — Creating Behaviors:** https://docs.picknik.ai/concepts/creating_behaviors/
- **How-To — Additional Behavior Classes:** https://docs.picknik.ai/how_to/custom_behaviors/additional_behavior_classes/
- **How-To — Adding Ports:** https://docs.picknik.ai/how_to/custom_behaviors/adding_ports/
- **Behavior catalog (with port definitions):** https://picknik.ai/behaviors
