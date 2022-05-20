# Using multiple robots
This document will present how to launch the simulation and navigation files for multiple instances of the DeDe robot. The example files creates two robots and initiates a navigation stack for each of them.

**Note!** While it is possible to use multiple robots to execute the mapping process, this will result in each robot creating their own map. As such, it is recommended to use a single robot to create a map and load the map for multiple robot navigation.

## Multi Robot Simulation with Gazebo
To launch the multiple robots gazebo simulation, the following command can be used:
```bash
ros2 launch dede multi_simulation.launch.py 
```
The simulation creates two virtual robots: dede1 and dede2. Each virtual robot publishes odometry and tf data and listens for velocity commands.

**Note!** The virtual robots use the same topic names while having different parent namespaces.

## Understanding the Multi Robot Simulation file:
This subchapter will break down the launch file and address the differences between the single robot simulation and multi robot simulation. It is recommended to read and understand the single robot simulation before continuing with this chapter.

### 1) Launch file structure
The launch file contains code segments divided by comments, each code block executes a specific role:
- *File paths*: This code section loads the relative paths for launch resources. These resources are required for the nodes and included launch files.
- *Gazebo Client and Server*: This code section launches the gazebo server with the selected world and the gazebo client, for visualization and interaction with the world.
- *Define robots*: This section contains the a variable holding the short robot configurations. Each element of the array represents a single simulated robot configuration and it is a dictionary of the shape:
    - *name*: The robot name, the name needs to be a unique string and avoid using spaces or special symbols. The name is used as a namespace for the topics and services associated with the robot.
    - *pose_x*, *pose_y*, *pose_z*: The robot position in the world relative to world origin. These values are strings representing a floating point.
    - *roll*, *pitch*, *yaw*: The robot orientation in the world, around the robot center. These values are strings representing a floating point.
- *Spawn robots*: This code section has the role of reading the robot definitions, create the robots in the simulated world and set the proper topic namespaces.
- *Launch Description Setup*: This code section creates a launch description object and appends to it all the actions that are to be executed.

### 2) Differences between single robot and multi robot setup
While the overall structure of the launch file shares similarities between the single robot and multi robot simulation, there are some key differences:
- In the "File paths" section, the single robot setup code will retrieve the robot URDF, the gazebo package directory and the path to the world file. But in the multi robot setup, the code will also load a robot SDF file path. This is required as the robots are not integrated in the world file. By not having the robots in the simulated world, it becomes easier to define how many robots, what is their name and even what type of robot the simulation should use.
- In order to allow ease of modification, the launch file uses a dictionary to define the simulated robots and iterates over the dictionary to create them.
- TF needs to be remapped from absolute paths to relative paths in order to differentiate between the robots, as the simulation spawns multiple instances of the same robot, their link and joint names would be identical if they are published on global topic. This will cause errors.

## Multiple Robot Navigation
The package provides an example launch file to start the navigation stack for two simulated robots. To start the launch file, the following command is used with the appropriate parameter values:
```bash
 ros2 launch dede multi_nav2_slam.launch.py map:=/path/to/map/file
```
The launch file will start the navigation stack for two virtual robots and namespace them under: dede1 and dede2. The launch file can be changed to start the navigation stack under different namespaces or with different configuration files.

**Note!** When issuing a goal, ensure that the appropriate namespace is used.

To visualize a specific robot, the package provides a Rviz2 launch file with the required remaps and namespace setup. To launch it, execute the command and set the parameter to match the desired robot name/namespace:
```bash
ros2 launch dede rviz2.launch.py namespace:='robot_name'
```
## Understanding the Multi Robot Navigation launch file
The subchapter will break down the launch file found under *"launch/multi_nav2_slam.launch.py* and address the differences between using the navigation stack for a single robot and multiple robots. It is important to read the documentation for the single robot navigation before continuing.

### 1) Launch file structure
The launch file contains code segments divided by comments, each code block executes a specific role:
- *File paths*: This code section loads the relative paths for launch resources. This resources are required for the nodes and included launch files. In the case of navigation the nodes require the configuration file locations and the behavior tree description.
- *Node Lifecycle Order*: This block contains a single variable that holds a list of node names. The list is required in order to activate the nodes in a predetermined order.
- *Parameters Setup*: This section sets the launch file parameters and their defaults. For this launch file two parameters are set:
    - *map:* A string, which represents the path for the map file without the file extension. Is recommended to use absolute paths.
    - *use_sim_time:* A boolean, defaulting to *"true"*, the parameter sets the nodes to either use the gazebo time stamp or the system time stamp. Set it to false when using a real robot.
- *Remap*: This section contains list of tuples representing changes to topic and services names. The lists are passed to nodes when they are started.
- *Define robots*: This section contains the variable holding robot definitions. For each robot a dictionary object is created and is structured as follows:
    - *name*: The robot name and the namespace that the navigation nodes should be run under. It is important to match the name with the one present in either the simulated or real robots
    - *robot_pose*: An array holding the X Position, Y Position and Theta values describing the robot's initial pose in the world. This values can be obtained from odometry or from the simulation startup pose.
    - *slam*: Name of the configuration file that is loaded by the slam_toolbox node, the file is located under the directory *"config/slam"*
    - *controller*: Name of the configuration file that is loaded by the controller_server node, the file is located under the directory *"config/nav2"*
    - *planer*: Name of the configuration file that is loaded by the planer_server node, the file is located under the directory *"config/nav2"*
    - *recoveries*: Name of the configuration file that is loaded by the recoveries_server node, the file is located under the directory *"config/nav2"*
    - *bt_navigator_config*: Name of the configuration file that is loaded by the bt_navigator node, the file is located under the directory *"config/nav2"*
    *bt_navigator_tree*: Name of the description file that is used by the bt_navigator node, the file is found under the *"nav2_bt_navigator"* package, in the *"behavior_trees"* directory.
    - *waypoint_follower*: Name of the configuration file that is loaded by the waypoint_follower node, the file is located under the directory *"config/nav2"*

- *Spawn Navigation*: This section has the role of iterating over the robot definitions and for each entry, activate the navigation nodes, set the remapping, set the node namespace and pass the configuration files. For each node group that is created, this section adds it to a list in order to be used by the launch description later in the file. Additionally this code sections builds a list of namespace node names for the lifecycle manager.
- *Lifecycle manager*: This section start the lifecycle manager and passes to it the list of namespaced nodes. The list is ordered to activate all the nodes for a robot navigation stack before moving to the next robot.
- *Launch Description Setup*: This code section creates a launch description object and appends to it all the actions that are to be executed.

### 2) Difference between single and multi robot navigation
The overall structure of the launch file is similar with the one present in the single robot navigation launch file. It uses the same nodes and packages. However, there are some key differences:
- The lifecycle node manager array is composed in two parts. First there is an array holding the node names that is further translated into another array which contains the node names prefixed by the namespace of each robot. This allows the usage of a single lifecycle manager for multiple navigation stacks.
- Remapping tuples are used because of present limitations within the way the TF tree is interpreted in ROS2. As each of the robot publishes an identical TF tree, this causes conflicts as multiple robots publish different transforms.
- Remapping tuples are used for the slam toolbox to remap some of the absolute topics to namespaced ones.
- In order to keep it modular, an array holding the parameters for each robot is used. To create a new navigation stack, a new element is added to the array.
- As the nodes are namespaced, the parameter files need to have their paramaters also namespaced. This is done on the fly by passing the parameter files through a "RewrittenYaml" object and setting the "root_key" to the robot namespace.
- The same map file is used for all the "slam_toolbox" nodes, the configuration used the map in read only mode.