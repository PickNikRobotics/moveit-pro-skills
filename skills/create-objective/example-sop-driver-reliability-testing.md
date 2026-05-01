# Driver Reliability Testing

## Purpose

Stress-test robot motion drivers by repeatedly commanding the robot to known waypoints, capturing point clouds at each location to verify positional consistency over many cycles. This validates that the driver does not accumulate drift, drop commands, or produce degraded trajectories under sustained operation.

## Parameters

- **test_cycles**: 1000
- **scan_waypoints**: right_scan, center_scan, left_scan
- **marker_count**: 100 (number of pose markers to visualize between endpoints)

## Procedure

### 1. Setup

1. Load the current planning scene.
2. Move the robot to the "home" waypoint.

### 2. Test Loop (repeat `test_cycles` times)

For each cycle, clear the previous visualization, capture a point cloud at each scan location, then merge and visualize the results.

1. Clear all point cloud visualizations in the UI.
2. Start the cycle timer.

#### 2a. Capture at each scan location

For each waypoint in `scan_waypoints`, in order:

1. Move to the waypoint.
   - If the motion fails, retry up to 3 times.
   - If still failing after retries, log the failure and skip to the next waypoint.
2. Capture a wrist-mounted point cloud.
3. Send the point cloud to the UI for live monitoring.

#### 2b. Merge and visualize

1. Merge all captured point clouds from this cycle into a single combined cloud.
2. Send the merged point cloud to the UI.
3. Publish `marker_count` pose markers evenly spaced between the first and last scan waypoints.
4. Stop the cycle timer and log the elapsed time.

### 3. Teardown

1. Move the robot to the "home" waypoint.
