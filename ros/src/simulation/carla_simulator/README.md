# Autoware in Carla

Integration of AutoWare AV software with the [CARLA simulator](http://carla.org/).

## Requirements

- ROS kinetic
- Autoware (tested with 1.11.0)
- PointCloud Map


## Setup

    # Download pointcloud maps for Carla Towns
    cd ~
    git clone https://bitbucket.org/carla-simulator/autoware-contents
    mkdir carla-python

    #extract the Carla Python API from the image
    docker run --rm --entrypoint tar carlasim/carla:0.9.5 cC /home/carla/PythonAPI . | tar xvC carla-python


## Run

The Carla Simulator is splitted into two parts, a server- and a client-part.

### Server

The Server needs to be running in the background. Multiple clients might connect to it.

    nvidia-docker run -p 2000-2001:2000-2001 -it --rm carlasim/carla:0.9.5 ./CarlaUE4.sh /Game/Carla/Maps/Town01 -benchmark -carla-server -fps=20 -world-port=2000 -windowed -ResX=100 -ResY=100 -carla-no-hud

This will start a docker container and map the ports 2000-2001 to the local machine.

### Client/Ego Vehicle

The Carla client to spawn the ego vehicle can either be started via the runtime manager (`CARLA Simulator` on page Simulation) or with `roslaunch`.

*Caution*: Before executing the runtime manager, set the following environment variables:

    export PYTHONPATH=$PYTHONPATH:~/carla/dist/carla-0.9.5-py2.7-linux-x86_64.egg:~/carla-python/carla/
    export CARLA_MAPS_PATH=~/autoware-contents/maps


To execute it via commandline:

    roslaunch carla_autoware_bridge carla_autoware_bridge_with_manual_control.launch


After startup the ego vehicle is controlled by the Autoware vehicle control commands. If you want to manually steer the car, press `B`within the manual_control window to toggle manual-mode.


# Troubleshooting

## The pygame window is black

This means, that no images were received via the carla-autoware-bridge.
Please check:
- is the Carla Server running
- is the environment set up correctly





1. Carla Server 

### Carla

Download version of Carla from here: https://github.com/carla-simulator/carla


### Carla Autoware Bridge

The Carla Autoware Bridge is a ROS package. Therefore we create a catkin workspace (containing all relevant packages).
The generic Carla ROS bridge (https://github.com/carla-simulator/ros-bridge.git) is included as GIT submodule and 
has to be initialized ("git submodule update --init") and updated ("git submodule update").

    cd ~
    git lfs clone https://github.com/carla-simulator/carla-autoware.git
    cd carla-autoware
    git submodule update --init
    cd catkin_ws
    catkin_init_workspace src/
    cd src
    ln -s <path-to-autoware>/ros/src/msgs/autoware_msgs
    cd ..
    catkin_make

## Run

To run Autoware within Carla please use the following execution order:

1. Carla Server
2. Autoware Runtime Manager
3. Carla Autoware Bridge
4. Autoware Stack

You need two terminals:

    #Terminal 1

    #execute Carla
    SDL_VIDEODRIVER=offscreen <path-to-carla>/CarlaUE4.sh /Game/Carla/Maps/Town01 -benchmark -fps=20


    #Terminal 2

    export CARLA_AUTOWARE_ROOT=~/carla-autoware
    
    #execute Autoware (forks into background)
    <path-to-autoware>/ros/run

    #execute carla-autoware-bridge and carla-autoware-bridge
    export PYTHONPATH=<path-to-carla>/PythonAPI/carla-<version_and_arch>.egg:<path-to-carla>/PythonAPI/
    source $CARLA_AUTOWARE_ROOT/catkin_ws/devel/setup.bash
    #either execute a headless Carla client
    roslaunch carla_autoware_bridge carla_autoware_bridge.launch
    #or
    roslaunch carla_autoware_bridge carla_autoware_bridge_with_manual_control.launch

In Autoware Runtime Manager, select the customized launch files:

![Autoware Runtime Manager Settings](docs/images/autoware-runtime-manager-settings.png)

In Autoware Runtime Manager, start rviz and open the configuration <autoware-dir>/ros/src/.config/rviz/default.rviz

Now you can start the Autoware Stack by starting all launch files from top to bottom. The car should start moving.

![Autoware Runtime Manager Settings](docs/images/autoware-rviz-carla-town01-running.png)

A special camera is positioned behind the car to see the car and its environment.
You can subscribe to it via ```/carla/ego_vehicle/camera/rgb/viewFromBehind/image_color```.

### Multi machine setup

You can run Autoware and Carla on different machines. 
To let the carla autoware bridge connect to a remote Carla Server, execute roslaunch with the following parameters

    roslaunch host:=<hostname> port:=<port number> carla_autoware_bridge carla_autoware_bridge.launch


## Development support

### Carla Autoware Ego Vehicle

When starting the carla_autoware_bridge a random spawn point and a fixed goal is used to calculate the route.

To override this, you can use RVIZ.

![Autoware Runtime Manager Settings](docs/images/rviz_set_start_goal.png)

- selecting a Pose with '2D Pose Estimate' will delete the current ego_vehicle and respawn it at the specified position.
- selecting a Pose with '2D Nav Goal' will set a new goal within `carla_waypoint_publisher`.

#### Manual steering

Press `B` to be able to steer the ego vehicle within ROS manual control.

Internally, this is done by stopping the conversion from the Autoware control message to AckermannDrive within the node `vehiclecmd_to_ackermanndrive`. The relevant ros-topic is `/vehicle_control_manual_override`.

## Design

The bridge contains three Carla Clients.

1. ROS Bridge - Monitors existing actors in Carla, publishes changes on ROS Topics (e.g. new sensor data)
2. Ego Vehicle - Instantiation of the ego vehicle with its sensor setup.
3. Waypoint Calculation - Uses the Carla Python API to calculate a route.

![Design Overview](docs/images/design.png)

