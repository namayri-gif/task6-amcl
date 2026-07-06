# Assignment 5: SLAM Mapping and Localization with SLAM Toolbox

## Overview
This assignment builds on the differential-drive robot from Assignment 4 and adds **SLAM Toolbox** for mapping and localization in ROS 2.
Using the robot's `/scan` (LiDAR) and `/odom` data, `slam_toolbox` builds a 2D occupancy grid map of the `turtlebot3_world` environment in **mapping mode**, saves that map + pose graph, and then reuses them in **localization mode** to localize the robot against the pre-built map.

This assignment covers: running SLAM Toolbox in online async mapping mode, saving a map and serialized pose graph, and running SLAM Toolbox in localization mode against that saved map.

---

## Objective
By the end of this task, you should have:

* The robot from Assignment 4 driving around `turtlebot3_world` via teleop
* `slam_toolbox` (async mode) building a live occupancy grid map from `/scan` + `/odom` + `/tf`
* A saved map (`.pgm` + `.yaml`) and serialized pose graph (`.posegraph` + `.data`)
* `slam_toolbox` in **localization mode**, loading the saved map/pose graph and localizing the robot without rebuilding the map
* Map and robot pose visualized in RViz

---

## Project Structure
```
src/task5-robot_files/
├── my_robot_description/
│   ├── config/
│   │   └── gz_bridge.yaml
│   ├── launch/
│   │   ├── display.launch.py
│   │   └── gazebo.launch.py
│   ├── meshes/
│   │   └── lidar.STL
│   ├── rviz/
│   │   ├── rviz.rviz
│   │   └── slamScan.rviz
│   ├── urdf/
│   │   ├── robot.gazebo.xacro
│   │   └── robot.urdf.xacro
│   ├── CMakeLists.txt
│   └── package.xml
│
├── slam_toolbox_demo/
│   ├── config/
│   │   ├── slam_toolbox_localization.yaml
│   │   └── slam_toolbox_online_async.yaml
│   ├── launch/
│   │   ├── localization.launch.py
│   │   └── slam_toolbox_online_async.launch.py
│   ├── map/
│   │   ├── turtlebot3_world_map.pgm
│   │   └── turtlebot3_world_map.yaml
│   ├── posegraph/
│   │   ├── turtlebot3_world.data
│   │   └── turtlebot3_world.posegraph
│   ├── CMakeLists.txt
│   └── package.xml

```

---

## Packages Used

| Package | Purpose |
|---|---|
| `my_robot_description` | Robot URDF, Gazebo plugins, LiDAR mesh (from Assignment 4) |
| `slam_toolbox_demo` | SLAM Toolbox configs, launch files, saved map + pose graph |
| `slam_toolbox` | Provides `async_slam_toolbox_node`, does the actual SLAM |

---

## Nodes / Modes

| Mode | Node | Config | Purpose |
|---|---|---|---|
| Mapping | `async_slam_toolbox_node` | `slam_toolbox_online_async.yaml` | Builds a new occupancy grid map live from `/scan` + `/odom` |
| Localization | `async_slam_toolbox_node` | `slam_toolbox_localization.yaml` | Loads saved map + pose graph, localizes robot against it (no new mapping) |

---

## Topics Used

| Topic | Type | Direction | Purpose |
|---|---|---|---|
| `/scan` | `sensor_msgs/msg/LaserScan` | Robot → slam_toolbox | LiDAR input for scan matching |
| `/odom` | `nav_msgs/msg/Odometry` | Robot → slam_toolbox | Odometry input |
| `/tf`, `/tf_static` | `tf2_msgs/msg/TFMessage` | Robot ↔ slam_toolbox | `odom → base_footprint` from robot, `map → odom` published by slam_toolbox |
| `/map` | `nav_msgs/msg/OccupancyGrid` | slam_toolbox → RViz | Live/loaded occupancy grid |
| `/cmd_vel` | `geometry_msgs/msg/Twist` | Teleop → Robot | Drive robot to explore the map |

---

## Requirements
- ROS 2 Jazzy
- Gazebo Sim (`gz sim`, Harmonic)
- `slam_toolbox`
- `teleop_twist_keyboard`
- `rviz2`

---

## Build
```bash
cd ~/ros2_ws
colcon build 
source install/setup.bash
```

---

## How to Run — Mapping Mode

**Step 1 — Launch Gazebo with the robot**
```bash
ros2 launch my_robot_description gazebo.launch.py
```

**Step 2 — Launch SLAM Toolbox (online async, mapping mode)**
```bash
ros2 launch slam_toolbox_demo slam_toolbox_online_async.launch.py
```

**Step 3 — Drive the robot around to build the map**
```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

**Step 4 — Visualize in RViz**
```bash
rviz2 -d src/task5-robot_files/my_robot_description/rviz/slamScan.rviz
```
Set Fixed Frame = `map`, add `Map` and `LaserScan` displays.

**Step 5 — Save the map**
```bash
ros2 run nav2_map_server map_saver_cli -f src/task5-robot_files/slam_toolbox_demo/map/turtlebot3_world_map
```

**Step 6 — Serialize the pose graph (for localization mode later)**
```bash
ros2 service call /slam_toolbox/serialize_map slam_toolbox/srv/SerializePoseGraph \
"{filename: '/absolute/path/to/slam_toolbox_demo/posegraph/turtlebot3_world'}"
```

---

## How to Run — Localization Mode

**Step 1 — Launch Gazebo with the robot**
```bash
ros2 launch my_robot_description gazebo.launch.py
```

**Step 2 — Launch SLAM Toolbox in localization mode**
```bash
ros2 launch slam_toolbox_demo localization.launch.py
```
This loads the saved map (`turtlebot3_world_map.yaml`) and pose graph (`turtlebot3_world.posegraph` / `.data`) instead of building a new map.

**Step 3 — Drive the robot and confirm it localizes on the existing map**
```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

**Step 4 — Verify in RViz**
```bash
rviz2 -d src/task5-robot_files/my_robot_description/rviz/slamScan.rviz
```
Confirm the robot's pose tracks correctly on the pre-built map without the map being rebuilt.

---

## Expected TF Tree
```
map
 └── odom
      └── base_footprint
           └── base_link
                ├── lidar_link
                ├── left_front_wheel_link
                ├── right_front_wheel_link
                └── caster_wheel_link
```

---

## Verification
- [ ] `async_slam_toolbox_node` starts with no parameter/config errors
- [ ] Map builds correctly in RViz while driving via teleop in mapping mode
- [ ] Map and pose graph save successfully to `slam_toolbox_demo/map` and `slam_toolbox_demo/posegraph`
- [ ] Localization mode loads the saved map without rebuilding it
- [ ] Robot pose updates correctly on the loaded map while driving
- [ ] TF tree is fully connected: `map → odom → base_footprint → base_link → {lidar_link, wheel links}`

---

## Screenshots / Demo

Mapping in progress (RViz):



Final saved map:

`<screenshot>`

Localization mode — robot on pre-built map:

`<screenshot>`

Demo Video Link:

`<link>`

---
