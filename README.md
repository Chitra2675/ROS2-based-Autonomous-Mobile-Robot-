# articubot_one

A complete ROS 2 autonomous mobile robot (AMR) stack: a differential-drive robot with `ros2_control`, 2D LiDAR, and a camera, capable of teleoperation, SLAM mapping, autonomous navigation (Nav2), and vision-based ball tracking — in both Gazebo simulation and on real hardware.

> **Attribution:** This project is built on the **`articubot_one`** robot from [Articulated Robotics](https://articulatedrobotics.xyz/) (Josh Newans) — see [the "Building a mobile robot" series](https://articulatedrobotics.xyz/mobile-robot-1-intro/) and the [original repository](https://github.com/joshnewans/articubot_one). The base URDF, `ros2_control` configuration, `twist_mux` setup, and SLAM/Nav2 wiring follow that series. See [My Modifications](#my-modifications) for what has been changed or added in this fork.

---

## Features

- **Differential-drive base** described in modular xacro, driven through `ros2_control` (`diff_drive_controller` + `joint_state_broadcaster`)
- **Gazebo simulation** with custom worlds (empty + obstacle course)
- **2D LiDAR** (360 samples, 0.3–12 m range, 10 Hz)
- **RGB camera** (640×480, ~62° FOV); optional depth camera available in the description (currently disabled)
- **SLAM** via `slam_toolbox` (online asynchronous mapping)
- **Autonomous navigation** via the Nav2 stack (localization + planning + control)
- **Vision-based ball tracking** using OpenCV, publishing velocity commands to follow a coloured ball
- **Joystick teleoperation**
- **Command arbitration** via `twist_mux`, prioritising joystick > tracker > navigation
- **Separate launch paths** for simulation and real hardware

## System Architecture

```
                         ┌──────────────┐
  Joystick ───────────►  │              │
  Ball Tracker (CV) ──►  │  twist_mux   │ ──► /diff_cont/cmd_vel_unstamped
  Nav2 ───────────────►  │ (priority)   │
                         └──────────────┘
                                                  │
                                                  ▼
                                       ┌────────────────────┐
   LiDAR ──► /scan ───────────────►    │   ros2_control     │
   Camera ──► /camera/image_raw ──►    │  diff_drive_ctrl   │ ──► wheel commands
                                       │  joint_state_bcast │ ──► /joint_states → /tf
                                       └────────────────────┘
                                                  │
   /scan + /odom + /tf  ──►  slam_toolbox (map)  ──►  Nav2 (autonomous goals)
```

| Priority | Source | Topic |
|---------:|--------|-------|
| 100 | Joystick | `cmd_vel_joy` |
| 20 | Ball tracker | `cmd_vel_tracker` |
| 10 | Navigation (Nav2) | `cmd_vel` |

## Robot Specifications

| Property | Value |
|----------|-------|
| Drive type | Differential drive + caster |
| Chassis length | 0.335 m |
| Wheel radius | 0.034 m (URDF) |
| Wheel separation | ~0.33 m |
| LiDAR | 360 samples, 0.3–12 m, 10 Hz, 360° FOV |
| Camera | 640×480 RGB, ~62° horizontal FOV, 10 Hz |
| Control loop | 50 Hz (`controller_manager`) |

## Package Structure

```
articubot_one/
├── description/            # Robot model (xacro)
│   ├── robot.urdf.xacro    #   top-level assembly
│   ├── robot_core.xacro    #   chassis, wheels, caster
│   ├── ros2_control.xacro  #   hardware interface (real robot)
│   ├── gazebo_control.xacro #  diff-drive plugin (sim, non-ros2_control mode)
│   ├── lidar.xacro / camera.xacro / depth_camera.xacro
│   ├── inertial_macros.xacro
│   └── face.xacro
├── launch/
│   ├── launch_sim.launch.py     # full simulation bringup
│   ├── launch_robot.launch.py   # real robot bringup
│   ├── rsp.launch.py            # robot_state_publisher
│   ├── joystick.launch.py       # joystick teleop
│   ├── ball_tracker.launch.py   # CV ball tracking
│   ├── camera.launch.py         # v4l2_camera (real camera)
│   ├── rplidar.launch.py        # rplidar_ros (real LiDAR)
│   ├── online_async_launch.py   # slam_toolbox
│   ├── navigation_launch.py     # Nav2 navigation
│   └── localization_launch.py   # Nav2 localization (AMCL)
├── config/
│   ├── my_controllers.yaml      # ros2_control diff-drive params
│   ├── twist_mux.yaml           # command arbitration
│   ├── mapper_params_async.yaml # slam_toolbox params
│   ├── nav2_params.yaml         # Nav2 stack params
│   ├── gazebo_params.yaml
│   ├── ball_tracker_params_*.yaml
│   └── *.rviz                   # RViz configs
├── worlds/
│   ├── empty.world
│   └── obstacles.world
├── CMakeLists.txt
└── package.xml
```

## Dependencies

Built with `ament_cmake` on ROS 2 (Humble recommended). Beyond the core ROS 2 desktop install, the following packages are required:

```bash
sudo apt install \
  ros-$ROS_DISTRO-ros2-control \
  ros-$ROS_DISTRO-ros2-controllers \
  ros-$ROS_DISTRO-gazebo-ros-pkgs \
  ros-$ROS_DISTRO-gazebo-ros2-control \
  ros-$ROS_DISTRO-twist-mux \
  ros-$ROS_DISTRO-slam-toolbox \
  ros-$ROS_DISTRO-navigation2 \
  ros-$ROS_DISTRO-nav2-bringup \
  ros-$ROS_DISTRO-joy \
  ros-$ROS_DISTRO-teleop-twist-joy \
  ros-$ROS_DISTRO-v4l2-camera \
  ros-$ROS_DISTRO-rplidar-ros
```

The ball-tracking launch also depends on the external [`ball_tracker`](https://github.com/joshnewans/ball_tracker) package — clone it into your workspace `src/` if you want that feature.

## Build

```bash
cd ~/ros2_ws
colcon build --packages-select articubot_one --symlink-install
source install/setup.bash
```

## Usage

### Simulation

```bash
# Launch the robot in Gazebo (spawns robot, controllers, twist_mux, joystick)
ros2 launch articubot_one launch_sim.launch.py

# In a second terminal — SLAM mapping
ros2 launch articubot_one online_async_launch.py use_sim_time:=true

# Autonomous navigation (after a map exists / or with SLAM running)
ros2 launch articubot_one navigation_launch.py use_sim_time:=true

# Visualise
rviz2 -d src/articubot_one/config/main.rviz
```

To load the obstacle world, point the sim launch at `worlds/obstacles.world`.

### Real Robot

```bash
# Core bringup (robot_state_publisher + controllers + twist_mux)
ros2 launch articubot_one launch_robot.launch.py

# Sensors
ros2 launch articubot_one rplidar.launch.py
ros2 launch articubot_one camera.launch.py

# SLAM / Navigation (use_sim_time:=false)
ros2 launch articubot_one online_async_launch.py use_sim_time:=false
ros2 launch articubot_one navigation_launch.py use_sim_time:=false
```

### Ball Tracking

```bash
ros2 launch articubot_one ball_tracker.launch.py
```

Tune the HSV thresholds in `config/ball_tracker_params_sim.yaml` (sim) or `config/ball_tracker_params_robot.yaml` (real camera).

## Key Topics

| Topic | Type | Notes |
|-------|------|-------|
| `/scan` | `sensor_msgs/LaserScan` | LiDAR |
| `/camera/image_raw` | `sensor_msgs/Image` | Camera |
| `/cmd_vel`, `/cmd_vel_joy`, `/cmd_vel_tracker` | `geometry_msgs/Twist` | twist_mux inputs |
| `/diff_cont/cmd_vel_unstamped` | `geometry_msgs/Twist` | controller input (mux output) |
| `/odom` | `nav_msgs/Odometry` | wheel odometry |
| `/joint_states` → `/tf` | — | robot state |

## My Modifications

<!-- Fill this in with what YOU changed/added on top of the base tutorial. Examples: -->
<!--
- Added the `obstacles.world` environment for navigation testing
- Retuned Nav2 controller / costmap parameters for <X>
- Reconciled wheel-radius calibration between URDF and controller config
- Integrated <additional sensor / behaviour>
- Ran the stack on <real hardware platform>
-->

_Describe your contributions here._

## Acknowledgements

Based on the **articubot_one** project and tutorial series by **Josh Newans / Articulated Robotics**. The ball-tracking component uses the [`ball_tracker`](https://github.com/joshnewans/ball_tracker) package. Built with [ROS 2](https://docs.ros.org/), [Gazebo](https://gazebosim.org/), [Nav2](https://nav2.org/), and [slam_toolbox](https://github.com/SteveMacenski/slam_toolbox).

## License

TODO — note that the upstream `articubot_one` is MIT licensed; if you redistribute, retain that license and add your own as appropriate.
