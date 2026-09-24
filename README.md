# Level 1: ROS2 Navigation Assignment - Mallela Sai Abhinay Narayan

1. SETUP AND WORKSPACE

I started by setting up ROS 2 Humble on Ubuntu and created the assignment workspace.

Workspace:
~/assignment_ws

I sourced ROS 2 and created the required packages:

- testbed_description
- testbed_bringup
- testbed_gazebo
- testbed_navigation

The workspace was built using:

source /opt/ros/humble/setup.bash
cd ~/assignment_ws
colcon build
source install/setup.bash


2. UNDERSTANDING THE STARTER ROBOT

I first ran the provided robot simulation and checked that:

- Gazebo starts correctly.
- The Testbed robot is spawned.
- The differential-drive controller works.
- /cmd_vel is available.
- TF frames are being published.
- Robot description is available through /robot_description.

The robot simulation was launched through the existing bringup/gazebo launch files.


3. CREATING THE testbed_navigation PACKAGE

The main task was to create a new ROS 2 package:

testbed_navigation

This package was responsible for managing the complete Nav2 navigation workflow.

The package was organized approximately as:

testbed_navigation/
├── launch/
│   ├── map_loader.launch.py
│   ├── localization.launch.py
│   └── navigation.launch.py
│
├── config/
│   ├── amcl_params.yaml
│   └── nav2_params.yaml
│
├── behavior_trees/
│   └── navigate_to_pose_w_replanning.xml
│
├── rviz/
│   └── navigation.rviz
│
├── package.xml
└── setup.py


4. MAP LOADER

The first navigation component I implemented was the map server.

I created:

launch/map_loader.launch.py

The map server was configured to load the provided occupancy-grid map.

The map was successfully loaded and the terminal showed information similar to:

Received map
405 X 400 map
0.05 m/pix

I verified the map topic using:

ros2 topic list | grep map

This showed:

/map
/map_server/transition_event
/map_updates

I also verified the map contents using:

ros2 topic echo /map --once

The map had:

width: 405
height: 400
resolution: 0.05
frame_id: map

Therefore the map server was working.


5. AMCL LOCALIZATION

After the map server, I implemented localization using AMCL.

I created:

config/amcl_params.yaml

and:

launch/localization.launch.py

The localization launch file started:

- AMCL
- lifecycle_manager_localization

I launched it using:

source ~/assignment_ws/install/setup.bash
ros2 launch testbed_navigation localization.launch.py

The terminal showed:

Subscribed to map topic.
Received a 405 X 400 map
Creating bond
Managed nodes are active

I verified AMCL using:

ros2 lifecycle get /amcl

It returned:

active [3]

Therefore AMCL was successfully running.


6. CHECKING TF

I then checked the TF tree between the robot and odometry.

I used:

ros2 run tf2_ros tf2_echo odom base_footprint

This eventually produced a valid transform containing translation and rotation values.

I also checked:

ros2 run tf2_ros tf2_echo map odom

After publishing the initial pose, AMCL started producing the map → odom transform.

This confirmed that the localization system was functioning.


7. INITIAL ROBOT POSE

AMCL requires an initial estimate of the robot's position on the map.

Normally this can be provided through RViz using:

2D Pose Estimate

I initially had an RViz visualization problem where the robot and map were not displayed correctly.

The map topic itself was working, because:

ros2 topic info /map --verbose

showed:

Publisher count: 1
Node name: map_server

and RViz was subscribed to /map.

The map also became visible after configuring RViz correctly with:

Fixed Frame = map

and:

Map Topic = /map

The robot visualization required the RobotModel display and the correct TF tree.


8. TESTING INITIAL POSE FROM THE TERMINAL

Because RViz visualization was problematic during debugging, I also tested the initial pose directly through the ROS 2 topic.

The topic used was:

/initialpose

with message type:

geometry_msgs/msg/PoseWithCovarianceStamped

For example:

ros2 topic pub --once /initialpose geometry_msgs/msg/PoseWithCovarianceStamped "{
  header: {
    frame_id: map
  },
  pose: {
    pose: {
      position: {
        x: -0.197,
        y: 4.824,
        z: 0.0
      },
      orientation: {
        x: 0.0,
        y: 0.0,
        z: -0.342,
        w: 0.940
      }
    },
    covariance: [
      0.25, 0.0, 0.0, 0.0, 0.0, 0.0,
      0.0, 0.25, 0.0, 0.0, 0.0, 0.0,
      0.0, 0.0, 0.0, 0.0, 0.0, 0.0,
      0.0, 0.0, 0.0, 0.0, 0.0, 0.0,
      0.0, 0.0, 0.0, 0.0, 0.0, 0.0,
      0.0, 0.0, 0.0, 0.0, 0.0, 0.25
    ]
  }
}"

AMCL successfully received the initial pose.

The terminal showed:

initialPoseReceived
Setting pose


9. NAV2 CONFIGURATION

After completing map loading and localization, I moved to the navigation stack.

I created:

config/nav2_params.yaml

This file contained the parameters for the main Nav2 components:

- planner_server
- controller_server
- bt_navigator
- behavior_server
- global_costmap
- local_costmap

The planner was configured to generate a path from the robot's current position to the goal.

The controller was configured to follow the generated path.

The BT Navigator was configured to manage the navigation behavior using a Behavior Tree.


10. NAVIGATION LAUNCH FILE

I created:

launch/navigation.launch.py

This launch file starts:

- planner_server
- controller_server
- bt_navigator
- behavior_server
- lifecycle_manager

I tested it using:

cd ~/assignment_ws
source /opt/ros/humble/setup.bash
source install/setup.bash

ros2 launch testbed_navigation navigation.launch.py


11. DEBUGGING NAV2 PARAMETERS

During the first navigation launch, the controller server crashed because of a parameter type mismatch.

The error was:

parameter 'height' has invalid type:
Wrong parameter type, parameter {height} is of type {integer},
setting it to {double} is not allowed.

This indicated that a parameter in nav2_params.yaml had the wrong YAML data type.

I corrected the parameter so that the value matched the type expected by Nav2.

After changing the parameter, I rebuilt the package:

cd ~/assignment_ws

source /opt/ros/humble/setup.bash

colcon build --packages-select testbed_navigation --symlink-install

source install/setup.bash


12. STARTING NAVIGATION

After correcting the parameters, I launched Nav2 using:

ros2 launch testbed_navigation navigation.launch.py

The navigation stack consists of:

MAP
 ↓
AMCL
 ↓
map → odom
 ↓
Global Costmap
 ↓
Planner Server
 ↓
Global Path
 ↓
Local Costmap
 ↓
Controller Server
 ↓
/cmd_vel
 ↓
Robot


13. HOW THE GOAL IS GIVEN

The navigation goal is sent after Nav2 has been launched and the navigation nodes are active.

When using RViz, the goal can be given using:

2D Goal Pose

The user selects a position on the map and orientation.

The goal is then sent to the Nav2 BT Navigator.

The planner generates a path to the goal.

The controller follows that path by publishing velocity commands.

The robot receives the velocity commands through:

/cmd_vel


14. BEHAVIOR TREE

Nav2 uses a Behavior Tree (BT) to coordinate the navigation process.

The Behavior Tree is executed by the BT Navigator. It coordinates the
different stages of navigation, including:

- receiving a navigation goal
- requesting a path from the planner
- following the planned path
- monitoring for navigation failures
- replanning when necessary
- performing recovery behaviors

I did not create a custom Behavior Tree XML file because the assignment
only required a basic Navigate-to-Pose navigation workflow.

My testbed_navigation/behavior_trees/navigate_to_pose_w_replanning.xml file was therefore left empty.

Instead, I configured the BT Navigator to use Nav2's default installed
Behavior Tree:

navigate_to_pose_w_replanning_and_recovery.xml

This file is provided by the Nav2 installation and is referenced through
the BT Navigator parameters in nav2_params.yaml.

Therefore, I used the existing Nav2 Behavior Tree rather than writing a
complete Behavior Tree from scratch.


15. TESTING DIFFERENT GOALS

I tested navigation by sending different goal positions.

The goal should be sent only after:

1. Gazebo is running.
2. Robot is spawned.
3. Map server is active.
4. AMCL is active.
5. Initial robot pose has been provided.
6. Nav2 navigation nodes are active.

The navigation stack then calculates a path between the current localized robot position and the requested goal.


16. Current Status and Limitations

The complete Nav2 navigation pipeline has been implemented and tested,
including map loading, AMCL localization, global planning, local control,
and the Nav2 Behavior Tree.

The robot is able to receive navigation goals and attempt to navigate
towards them. However, the navigation is **not perfectly reliable yet**.
In some cases, the robot may deviate from the planned path, take a
suboptimal path, or exhibit circling/oscillatory behavior while trying
to reach the goal.

The main navigation components are functional, but further tuning of
the planner, controller, costmaps, and velocity-related parameters would
be required to achieve smoother and more consistent navigation.

The default Nav2 Behavior Tree was used rather than creating a custom
Behavior Tree, as the assignment's basic navigation workflow could be
handled by the default `navigate_to_pose_w_replanning_and_recovery.xml`.


17. VERIFICATION COMMANDS

Important commands used during debugging were:

Check nodes:

ros2 node list

Check map:

ros2 topic list | grep map

Check map information:

ros2 topic echo /map --once

Check map publisher:

ros2 topic info /map --verbose

Check AMCL state:

ros2 lifecycle get /amcl

Check TF:

ros2 run tf2_ros tf2_echo odom base_footprint

Check map to odom:

ros2 run tf2_ros tf2_echo map odom

Check controllers:

ros2 control list_controllers

Build workspace:

cd ~/assignment_ws
colcon build

Source workspace:

source ~/assignment_ws/install/setup.bash

## Contact Info

- Name: M Sai Abhinay Narayan
- Contact Number:6302898343
- Email Address: saiabhinay0607@gmail.com
