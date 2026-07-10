# Autonomous Navigation with Nav2 and TurtleBot3

In the previous tutorial, we used **AMCL** to localize TurtleBot3 inside a saved map.

Now we will continue from the same workspace and add **Navigation2 (Nav2)**.

Nav2 will allow the robot to:

```text
Plan a path
Follow the path
Avoid obstacles
Recover if something goes wrong
Reach a goal automatically
```

---

# What Will We Do?

In this tutorial, we will:

* continue using the same `nav_ws` workspace
* continue using the same `robot_navigation` package
* add Nav2 configuration files
* configure the Planner Server
* configure the Global Costmap
* configure the Controller Server
* configure the Local Costmap
* configure the Behavior Server
* configure the BT Navigator
* create one launch file for the full Nav2 stack
* build and run the workspace
* send a navigation goal from RViz

---

# What Is Nav2?

**Navigation2**, usually called **Nav2**, is the ROS 2 navigation framework.

It allows a robot to move from its current position to a selected goal while using:

```text
Map
Localization
Path planning
Path following
Obstacle avoidance
Recovery behaviors
```

Nav2 is not one single node.

It is a group of servers that work together.

---

# Main Nav2 Servers

Nav2 contains several important servers.

```text
Planner Server
Controller Server
Behavior Server
BT Navigator
Map Server
AMCL
Lifecycle Manager
```

Each one has a specific role.

---

## Planner Server

The Planner Server creates the global path from the robot position to the goal.

In this tutorial, we will use:

```text
NavFn Planner
```

NavFn can use:

```text
Dijkstra
A*
```

In this tutorial, we will keep the default option:

```text
Dijkstra
```

---

## Controller Server

The Controller Server follows the path created by the Planner Server.

It publishes velocity commands to move the robot.

The output is:

```text
/cmd_vel
```

In this tutorial, we will use:

```text
DWB Local Planner
```

DWB means:

```text
Dynamic Window Approach
```

It checks multiple possible robot movements and chooses the best safe movement.

---

## Costmaps

Nav2 uses costmaps to understand free space and obstacles.

There are two main costmaps:

```text
Global Costmap
Local Costmap
```

---

### Global Costmap

The Global Costmap is used by the Planner Server.

It uses the saved map to plan the full path from start to goal.

---

### Local Costmap

The Local Costmap is used by the Controller Server.

It moves with the robot and helps it avoid nearby obstacles.

---

## Behavior Server

The Behavior Server runs recovery behaviors.

For example:

```text
Spin
Back up
Wait
Drive on heading
Assisted teleop
```

These behaviors help the robot recover if it gets stuck.

---

## BT Navigator

BT means:

```text
Behavior Tree
```

The BT Navigator manages the navigation logic.

It decides when to:

```text
Plan
Follow the path
Retry
Recover
Stop
```

---

# Before You Start

Make sure the TurtleBot3 simulation is running.

If it is not running:

1. Open the **Workspaces** section.
2. Start `turtlebot3_ws`.
3. Open the **Worlds** panel.
4. Launch:

```text
turtlebot3_world.launch
```

5. Wait until the robot and world are fully loaded.

---

# Continue from the AMCL Workspace

We will continue using the same workspace:

```bash
cd ~/workspaces/nav_ws/src/robot_navigation
```

Your package should already contain:

```text
robot_navigation
├── config
├── launch
└── map
```

The map folder should contain:

```text
turtlebot3_world_map.yaml
turtlebot3_world_map.pgm
```

---

# Install Nav2

If Nav2 is not installed, run:

```bash
sudo apt update
sudo apt install ros-jazzy-navigation2 ros-jazzy-nav2-bringup -y
```

---

# Create the Planner Server Config File

Go to the config folder:

```bash
cd ~/workspaces/nav_ws/src/robot_navigation/config
```

Create the planner file:

```bash
nano planner_server.yaml
```

Paste:

```yaml
planner_server:
  ros__parameters:
    use_sim_time: true

    expected_planner_frequency: 10.0

    planner_plugins: ["GridBased"]

    GridBased:
      plugin: "nav2_navfn_planner::NavfnPlanner"
      tolerance: 0.5
      use_astar: false
      allow_unknown: true
```

---

# Planner Parameters Explained

## `expected_planner_frequency`

```yaml
expected_planner_frequency: 10.0
```

This is how often the planner is expected to create or update a path.

---

## `planner_plugins`

```yaml
planner_plugins: ["GridBased"]
```

This tells Nav2 which planner plugin to use.

---

## `plugin`

```yaml
plugin: "nav2_navfn_planner::NavfnPlanner"
```

This selects the NavFn planner.

---

## `tolerance`

```yaml
tolerance: 0.5
```

This means the planner can accept a goal near the requested goal if the exact point is not reachable.

---

## `use_astar`

```yaml
use_astar: false
```

This means NavFn will use Dijkstra.

If you change it to `true`, it will use A*.

---

# Create the Global Costmap Config File

Create the file:

```bash
nano global_costmap.yaml
```

Paste:

```yaml
global_costmap:
  global_costmap:
    ros__parameters:
      use_sim_time: true

      update_frequency: 1.0
      publish_frequency: 1.0

      global_frame: map
      robot_base_frame: base_footprint
      transform_tolerance: 0.5

      robot_radius: 0.1
      resolution: 0.05
      track_unknown_space: true

      plugins: ["static_layer", "obstacle_layer", "voxel_layer", "inflation_layer"]

      static_layer:
        plugin: "nav2_costmap_2d::StaticLayer"
        map_subscribe_transient_local: true
        transform_tolerance: 0.1

      obstacle_layer:
        plugin: "nav2_costmap_2d::ObstacleLayer"
        enabled: true
        observation_sources: scan

        scan:
          topic: /scan
          max_obstacle_height: 2.0
          clearing: true
          marking: true
          data_type: "LaserScan"
          raytrace_max_range: 3.0
          raytrace_min_range: 0.0
          obstacle_max_range: 2.5
          obstacle_min_range: 0.0

      voxel_layer:
        plugin: "nav2_costmap_2d::VoxelLayer"
        enabled: true
        publish_voxel_map: true
        origin_z: 0.0
        z_resolution: 0.05
        z_voxels: 16
        max_obstacle_height: 2.0
        mark_threshold: 0
        observation_sources: scan

        scan:
          topic: /scan
          max_obstacle_height: 2.0
          clearing: true
          marking: true
          data_type: "LaserScan"
          raytrace_max_range: 3.0
          raytrace_min_range: 0.0
          obstacle_max_range: 2.5
          obstacle_min_range: 0.0

      inflation_layer:
        plugin: "nav2_costmap_2d::InflationLayer"
        inflation_radius: 0.5
        cost_scaling_factor: 5.0

      always_send_full_costmap: true
```

---

# Global Costmap Parameters Explained

## `global_frame`

```yaml
global_frame: map
```

The Global Costmap works in the map frame.

---

## `robot_base_frame`

```yaml
robot_base_frame: base_footprint
```

This is the main robot base frame used with TurtleBot3.

We keep it consistent with the AMCL tutorial.

---

## `robot_radius`

```yaml
robot_radius: 0.1
```

This is used instead of a rectangular footprint.

TurtleBot3 Burger is small and close to a circular shape, so `robot_radius` is simple and suitable.

---

## `static_layer`

The static layer uses the saved map.

---

## `obstacle_layer`

The obstacle layer uses LiDAR data from:

```text
/scan
```

It marks detected obstacles and clears free space.

---

## `voxel_layer`

The voxel layer helps process obstacle information in 3D space.

Even though TurtleBot3 uses a 2D LiDAR, this configuration is commonly used in Nav2 TurtleBot3 setups.

---

## `inflation_layer`

The inflation layer adds a safety buffer around obstacles.

```yaml
inflation_radius: 0.5
cost_scaling_factor: 5.0
```

This helps the robot avoid driving too close to walls.

---

# Create the Controller Server Config File

Create the file:

```bash
nano controller_server.yaml
```

Paste:

```yaml
controller_server:
  ros__parameters:
    use_sim_time: true

    controller_frequency: 10.0

    min_x_velocity_threshold: 0.001
    min_y_velocity_threshold: 0.001
    min_theta_velocity_threshold: 0.001

    failure_tolerance: 0.3

    progress_checker_plugins: ["progress_checker"]
    goal_checker_plugins: ["goal_checker"]
    controller_plugins: ["FollowPath"]

    progress_checker:
      plugin: "nav2_controller::SimpleProgressChecker"
      required_movement_radius: 0.1
      movement_time_allowance: 10.0

    goal_checker:
      stateful: true
      plugin: "nav2_controller::SimpleGoalChecker"
      xy_goal_tolerance: 0.25
      yaw_goal_tolerance: 0.25

    FollowPath:
      plugin: "dwb_core::DWBLocalPlanner"
      debug_trajectory_details: true

      min_vel_x: 0.0
      min_vel_y: 0.0
      max_vel_x: 0.3
      max_vel_y: 0.0
      max_vel_theta: 1.0

      min_speed_xy: 0.0
      max_speed_xy: 0.3
      min_speed_theta: 0.0

      acc_lim_x: 3.0
      acc_lim_y: 0.0
      acc_lim_theta: 3.2

      decel_lim_x: -2.5
      decel_lim_y: 0.0
      decel_lim_theta: -3.2

      vx_samples: 20
      vy_samples: 0
      vtheta_samples: 40

      sim_time: 1.5
      linear_granularity: 0.05
      angular_granularity: 0.025

      transform_tolerance: 0.2

      xy_goal_tolerance: 0.05
      trans_stopped_velocity: 0.25

      short_circuit_trajectory_evaluation: true
      stateful: true

      critics:
        [
          "RotateToGoal",
          "Oscillation",
          "BaseObstacle",
          "GoalAlign",
          "PathAlign",
          "PathDist",
          "GoalDist",
        ]

      BaseObstacle.scale: 0.02

      PathAlign.scale: 32.0
      PathAlign.forward_point_distance: 0.1

      GoalAlign.scale: 24.0
      GoalAlign.forward_point_distance: 0.1

      PathDist.scale: 32.0
      GoalDist.scale: 24.0

      RotateToGoal.scale: 32.0
      RotateToGoal.slowing_factor: 5.0
      RotateToGoal.lookahead_time: -1.0

    enable_stamped_cmd_vel: true
```

---

# Controller Server Parameters Explained

## `controller_frequency`

```yaml
controller_frequency: 10.0
```

The controller runs at 10 Hz.

This means it calculates movement commands 10 times per second.

---

## `progress_checker`

```yaml
required_movement_radius: 0.1
movement_time_allowance: 10.0
```

This checks if the robot is actually making progress.

If the robot does not move enough within the allowed time, Nav2 can trigger recovery behavior.

---

## `goal_checker`

```yaml
xy_goal_tolerance: 0.25
yaw_goal_tolerance: 0.25
```

This defines how close the robot must be to the goal position and direction.

---

## `FollowPath`

This is the DWB local planner.

It chooses the best velocity command for the robot.

---

## TurtleBot3 Velocity Limits

```yaml
max_vel_x: 0.3
max_vel_theta: 1.0
max_speed_xy: 0.3
```

These values keep TurtleBot3 movement safe and smooth.

---

## DWB Critics

The critics are scoring functions.

They help DWB choose the best trajectory.

```text
RotateToGoal
Oscillation
BaseObstacle
GoalAlign
PathAlign
PathDist
GoalDist
```

Each critic checks something different, such as obstacle distance, path alignment, and goal direction.

---

# Create the Local Costmap Config File

Create the file:

```bash
nano local_costmap.yaml
```

Paste:

```yaml
local_costmap:
  local_costmap:
    ros__parameters:
      use_sim_time: true

      update_frequency: 5.0
      publish_frequency: 2.0

      global_frame: odom
      robot_base_frame: base_footprint

      rolling_window: true
      width: 3
      height: 3
      resolution: 0.05

      robot_radius: 0.1

      plugins: ["obstacle_layer", "voxel_layer", "inflation_layer"]

      obstacle_layer:
        plugin: "nav2_costmap_2d::ObstacleLayer"
        enabled: true
        observation_sources: scan

        scan:
          topic: /scan
          max_obstacle_height: 2.0
          clearing: true
          marking: true
          data_type: "LaserScan"

      voxel_layer:
        plugin: "nav2_costmap_2d::VoxelLayer"
        enabled: true
        publish_voxel_map: true
        origin_z: 0.0
        z_resolution: 0.05
        z_voxels: 16
        max_obstacle_height: 2.0
        mark_threshold: 0
        observation_sources: scan

        scan:
          topic: /scan
          max_obstacle_height: 2.0
          clearing: true
          marking: true
          data_type: "LaserScan"
          raytrace_max_range: 3.0
          raytrace_min_range: 0.0
          obstacle_max_range: 2.5
          obstacle_min_range: 0.0

      inflation_layer:
        plugin: "nav2_costmap_2d::InflationLayer"
        inflation_radius: 0.5
        cost_scaling_factor: 5.0

      always_send_full_costmap: true
```

---

# Local Costmap Parameters Explained

## `global_frame`

```yaml
global_frame: odom
```

The Local Costmap uses the odom frame.

This keeps the local costmap smooth while the robot moves.

---

## `rolling_window`

```yaml
rolling_window: true
```

This means the Local Costmap moves with the robot.

---

## `width` and `height`

```yaml
width: 3
height: 3
```

The local area around the robot is 3 meters by 3 meters.

---

# Create the Behavior Server Config File

Create the file:

```bash
nano behavior_server.yaml
```

Paste:

```yaml
behavior_server:
  ros__parameters:
    use_sim_time: true

    local_costmap_topic: local_costmap/costmap_raw
    local_footprint_topic: local_costmap/published_footprint

    global_costmap_topic: global_costmap/costmap_raw
    global_footprint_topic: global_costmap/published_footprint

    cycle_frequency: 10.0

    behavior_plugins:
      ["spin", "backup", "drive_on_heading", "wait", "assisted_teleop"]

    spin:
      plugin: "nav2_behaviors::Spin"

    backup:
      plugin: "nav2_behaviors::BackUp"

    drive_on_heading:
      plugin: "nav2_behaviors::DriveOnHeading"

    wait:
      plugin: "nav2_behaviors::Wait"

    assisted_teleop:
      plugin: "nav2_behaviors::AssistedTeleop"

    local_frame: odom
    global_frame: map
    robot_base_frame: base_footprint

    transform_timeout: 0.1

    simulate_ahead_time: 2.0

    max_rotational_vel: 1.0
    min_rotational_vel: 0.4
    rotational_acc_lim: 3.2

    enable_stamped_cmd_vel: true
```

---

# Behavior Server Parameters Explained

## `behavior_plugins`

```yaml
behavior_plugins:
  ["spin", "backup", "drive_on_heading", "wait", "assisted_teleop"]
```

These are the recovery behaviors available to Nav2.

---

## `spin`

The robot rotates in place.

This can help when the robot needs to recover or find a better direction.

---

## `backup`

The robot moves backward.

This helps if the robot gets too close to an obstacle.

---

## `wait`

The robot stays still for a short time.

This is useful if an obstacle may move away.

---

# Create the BT Navigator Config File

Create the file:

```bash
nano bt_navigator.yaml
```

Paste:

```yaml
bt_navigator:
  ros__parameters:
    use_sim_time: true

    global_frame: map
    robot_base_frame: base_footprint

    transform_tolerance: 0.5
    filter_duration: 0.3

    default_nav_to_pose_bt_xml: "$(find-pkg-share nav2_bt_navigator)/behavior_trees/navigate_to_pose_w_replanning_and_recovery.xml"
    default_nav_through_poses_bt_xml: "$(find-pkg-share nav2_bt_navigator)/behavior_trees/navigate_to_pose_w_replanning_and_recovery.xml"

    always_reload_bt_xml: false

    goal_blackboard_id: goal
    goals_blackboard_id: goals
    path_blackboard_id: path

    navigators:
      ["navigate_to_pose", "navigate_through_poses"]

    navigate_to_pose:
      plugin: "nav2_bt_navigator::NavigateToPoseNavigator"

    navigate_through_poses:
      plugin: "nav2_bt_navigator::NavigateThroughPosesNavigator"

    error_code_name_prefixes:
      - assisted_teleop
      - backup
      - compute_path
      - dock_robot
      - drive_on_heading
      - follow_path
      - nav_thru_poses
      - nav_to_pose
      - spin
      - route
      - undock_robot
      - wait
```

---

# BT Navigator Parameters Explained

## `default_nav_to_pose_bt_xml`

This defines the behavior tree used when sending one navigation goal.

```text
Navigate to Pose
```

---

## `default_nav_through_poses_bt_xml`

This defines the behavior tree used when sending multiple navigation goals.

```text
Navigate Through Poses
```

---

## `navigators`

```yaml
navigators:
  ["navigate_to_pose", "navigate_through_poses"]
```

This enables the navigation action servers.

---

# Create the Full Nav2 Launch File

Go to the launch folder:

```bash
cd ~/workspaces/nav_ws/src/robot_navigation/launch
```

Create the file:

```bash
nano nav2_bringup.launch.py
```

Paste:

```python
import os

from ament_index_python.packages import get_package_share_directory

from launch import LaunchDescription

from launch_ros.actions import Node


def generate_launch_description():

    pkg_share = get_package_share_directory('robot_navigation')

    map_file = os.path.join(
        pkg_share,
        'map',
        'turtlebot3_world_map.yaml'
    )

    amcl_config = os.path.join(
        pkg_share,
        'config',
        'amcl.yaml'
    )

    planner_config = os.path.join(
        pkg_share,
        'config',
        'planner_server.yaml'
    )

    global_costmap_config = os.path.join(
        pkg_share,
        'config',
        'global_costmap.yaml'
    )

    controller_config = os.path.join(
        pkg_share,
        'config',
        'controller_server.yaml'
    )

    local_costmap_config = os.path.join(
        pkg_share,
        'config',
        'local_costmap.yaml'
    )

    behavior_config = os.path.join(
        pkg_share,
        'config',
        'behavior_server.yaml'
    )

    bt_navigator_config = os.path.join(
        pkg_share,
        'config',
        'bt_navigator.yaml'
    )

    lifecycle_nodes = [
        'map_server',
        'amcl',
        'planner_server',
        'controller_server',
        'behavior_server',
        'bt_navigator'
    ]

    map_server = Node(
        package='nav2_map_server',
        executable='map_server',
        name='map_server',
        output='screen',
        parameters=[
            {'use_sim_time': True},
            {'yaml_filename': map_file}
        ]
    )

    amcl = Node(
        package='nav2_amcl',
        executable='amcl',
        name='amcl',
        output='screen',
        parameters=[
            amcl_config
        ]
    )

    planner_server = Node(
        package='nav2_planner',
        executable='planner_server',
        name='planner_server',
        output='screen',
        parameters=[
            planner_config,
            global_costmap_config
        ]
    )

    controller_server = Node(
        package='nav2_controller',
        executable='controller_server',
        name='controller_server',
        output='screen',
        parameters=[
            controller_config,
            local_costmap_config
        ]
    )

    behavior_server = Node(
        package='nav2_behaviors',
        executable='behavior_server',
        name='behavior_server',
        output='screen',
        parameters=[
            behavior_config
        ]
    )

    bt_navigator = Node(
        package='nav2_bt_navigator',
        executable='bt_navigator',
        name='bt_navigator',
        output='screen',
        parameters=[
            bt_navigator_config
        ]
    )

    lifecycle_manager = Node(
        package='nav2_lifecycle_manager',
        executable='lifecycle_manager',
        name='lifecycle_manager_navigation',
        output='screen',
        parameters=[
            {'use_sim_time': True},
            {'autostart': True},
            {'node_names': lifecycle_nodes}
        ]
    )

    return LaunchDescription([
        map_server,
        amcl,
        planner_server,
        controller_server,
        behavior_server,
        bt_navigator,
        lifecycle_manager
    ])
```

---

# Why Do We Use Lifecycle Manager?

Nav2 servers are lifecycle nodes.

This means they have states:

```text
Unconfigured
Inactive
Active
Finalized
```

The lifecycle manager automatically configures and activates all Nav2 nodes in the correct order.

This is important because:

```text
Map Server must load the map.
AMCL must localize the robot.
Planner and Controller must start after the required data is available.
```

---

# Update CMakeLists.txt

Open:

```bash
cd ~/workspaces/nav_ws/src/robot_navigation
nano CMakeLists.txt
```

Make sure this install rule exists:

```cmake
install(
  DIRECTORY launch config map
  DESTINATION share/${PROJECT_NAME}
)
```

---

# Build the Workspace

Go to the workspace root:

```bash
cd ~/workspaces/nav_ws
colcon build
source install/setup.bash
```

---

# Run TurtleBot3 Simulation

Make sure TurtleBot3 simulation is running.

If it is not running, launch it from the simulator panel:

```text
turtlebot3_world.launch
```

Wait until the robot appears in Gazebo.

---

# Run Nav2

Open a new terminal and run:

```bash
source ~/workspaces/nav_ws/install/setup.bash
ros2 launch robot_navigation nav2_bringup.launch.py
```

This will start:

```text
map_server
amcl
planner_server
controller_server
behavior_server
bt_navigator
lifecycle_manager
```

---

# Open RViz

Open RViz:

```bash
rviz2
```

Set:

```text
Fixed Frame = map
```

Add these displays:

```text
Map
TF
RobotModel
LaserScan
ParticleCloud
Global Costmap
Local Costmap
Path
```

---

# RViz Topics

Use these topics:

```text
Map Topic: /map
LaserScan Topic: /scan
ParticleCloud Topic: /particlecloud
Global Costmap Topic: /global_costmap/costmap
Local Costmap Topic: /local_costmap/costmap
Global Plan Topic: /plan
Local Plan Topic: /local_plan
```

---

# Set the Initial Pose

Before sending a navigation goal, AMCL needs the robot initial pose.

In RViz:

1. Click **2D Pose Estimate**.
2. Click on the map where the robot is located.
3. Drag in the direction the robot is facing.

The particle cloud should appear around the robot.

---

# Send a Navigation Goal

In RViz:

1. Click **Nav2 Goal**.
2. Click on a free place in the map.
3. Drag to set the final direction.
4. Release the mouse.

The robot should:

```text
Create a global path
Follow the path
Avoid obstacles
Reach the goal
Stop
```

---

# Check Nav2 Topics

You can check the running topics:

```bash
ros2 topic list
```

Useful topics include:

```text
/map
/scan
/amcl_pose
/particlecloud
/plan
/local_plan
/cmd_vel
/global_costmap/costmap
/local_costmap/costmap
```

---

# Check Nav2 Actions

Nav2 uses actions for navigation.

Run:

```bash
ros2 action list
```

You should see:

```text
/navigate_to_pose
/navigate_through_poses
```

---

# Troubleshooting

## Robot Does Not Move

Check that `/cmd_vel` is being published:

```bash
ros2 topic echo /cmd_vel
```

If messages appear but the robot does not move, make sure the simulator is running correctly.

---

## Robot Cannot Plan

Make sure:

```text
The map is visible.
The robot is localized.
The goal is inside free space.
The global costmap is visible.
```

---

## Robot Spins or Gets Lost

Set the initial pose again using:

```text
2D Pose Estimate
```

Then move the robot slowly or send a closer goal.

---

## Costmaps Do Not Appear

Check if Nav2 nodes are active:

```bash
ros2 lifecycle get /map_server
ros2 lifecycle get /amcl
ros2 lifecycle get /planner_server
ros2 lifecycle get /controller_server
```

They should be:

```text
active
```

---

## TF Error

Check the TF tree:

```bash
ros2 run tf2_tools view_frames
```

The important frames are:

```text
map
odom
base_footprint
base_link
```

The expected TF chain is:

```text
map → odom → base_footprint → base_link
```

---

# What We Learned

In this tutorial, you learned:

* what Nav2 is
* how Nav2 servers work together
* how to configure the Planner Server
* how to configure the Global Costmap
* how to configure the Controller Server
* how to configure the Local Costmap
* how to configure the Behavior Server
* how to configure the BT Navigator
* how to launch the full Nav2 stack
* how to localize the robot before navigation
* how to send a navigation goal from RViz
* how to check Nav2 topics and actions

---

# Next Tutorial

In the next tutorial, we can tune Nav2 to improve:

```text
Path smoothness
Obstacle clearance
Goal accuracy
Robot speed
Recovery behavior
```
