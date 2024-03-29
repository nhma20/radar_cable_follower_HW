# Radar Cable Follower Hardware setup

<img src="https://user-images.githubusercontent.com/76950970/208680969-866df8b5-8aec-4c6a-a195-0101bb107cd2.jpg" width="750">

Hardware setup for the Radar Cable Follower drone

Includes the mounting mechanism, setting up the Raspberry Pi (3B+) and Pixhawk Mini (4), and validating connections. 

## QAV Mount

See `/CAD/` directory for files.

<img src="https://user-images.githubusercontent.com/76950970/212426757-107a93c1-cf74-460e-98ba-200d014164a9.png" width="750">

<img src="https://user-images.githubusercontent.com/76950970/205514743-2433076a-09ab-4db5-b3d3-5ae6974ff65a.jpg" width="750">

<img src="https://user-images.githubusercontent.com/76950970/205514749-465f3f85-07b2-422c-9d46-62dabee551ee.jpg" width="750">


## Raspberry Pi setup

- Flash SD card with `Ubuntu 20.04 server` image.
- Increase swap to 5GB
  - `dd if=/dev/zero of=<path_to_swapfile.img> bs=1024 count=5M`
  - `mkswap <path_to_swapfile.img>`
  - Add to `/etc/fstab`: `<path_to_swapfile.img> swap swap sw 0 0`
  - `swapon <path_to_swapfile.img>`

Install ROS2:
  - `sudo apt update`
  - `sudo apt upgrade`
  - `sudo apt update && sudo apt install curl gnupg2 lsb-release`
  - `sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key  -o /usr/share/keyrings/ros-archive-keyring.gpg`
  - `echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null`
  - `pip3 install pyros-genmsg`
  - `sudo apt-get install build-essential`
  - `sudo apt install ros-foxy-desktop`
  - `sudo apt install python3-colcon-common-extensions`
  - `sudo apt-get install ros-foxy-cv-bridge`
  - `sudo apt install libpcl-dev`
  - ```sh
    cd ~
    mkdir -p ~/ros2_ws/src
    ```

Install PX4:
  - Foonathan memory:
    - `cd ~`
    - `git clone https://github.com/eProsima/foonathan_memory_vendor.git`
    - `cd foonathan_memory_vendor`
    - `mkdir build && cd build`
    - `cmake ..`
    - `sudo cmake --build . --target install`
   
  - Fast-RTPS-Gen
    - `sudo apt-get install openjdk-8-jre`
    - `export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-arm64`
    - `cd ~`
    - `git clone --recursive https://github.com/eProsima/Fast-DDS-Gen.git -b v1.0.4 ~/Fast-RTPS-Gen \
      && cd ~/Fast-RTPS-Gen \
      && ./gradlew assemble \
      && sudo ./gradlew install`
  
  - PX4:
    - ```sh
      cd ~
      git clone -n https://github.com/PX4/PX4-Autopilot.git
      cd PX4-Autopilot/
      git checkout 6823cbc4140e29568f00e1211ae60e057adb1a1f
      (not needed unless for simulation) git submodule update --init --recursive
      bash ./Tools/setup/ubuntu.sh
      ```
      
      To listen to radio channels, edit `~/PX4-Autopilot/msg/tools/urtps_bridge_topics.yaml` to include:
      ```sh
        - msg:     rc_channels
          send:    true
      ```
      
      Then run:
      ```sh
      cd ~/PX4-Autopilot/msg/tools/
      python3 uorb_to_ros_urtps_topics.py -i ~/PX4-Autopilot/msg/tools/urtps_bridge_topics.yaml -o ~/ros2_ws/src/px4_ros_com/templates/urtps_bridge_topics.yaml
      ```
      
      The PX4 firmware for the flight controller (.px4 file) must be built with the same topics in the above file. A known-to-work version has the following content:
      ```sh
      rtps:
        - msg:     offboard_control_mode
          receive: true
        - msg:     telemetry_status
          receive: true
        - msg:     timesync
          receive: true
          send:    true
        - msg:     vehicle_command
          receive: true
        - msg:     vehicle_control_mode
          send:    true
        - msg:     vehicle_local_position_setpoint
          receive: true
        - msg:     trajectory_setpoint # multi-topic / alias of vehicle_local_position_setpoint
          base:    vehicle_local_position_setpoint
          receive: true
        - msg:     vehicle_odometry
          send:    true
        - msg:     vehicle_status
          send:    true
        - msg:     timesync_status
          send:    true
        - msg:     rc_channels
          send:    true
      ```
      
    - ```sh
      cd ~/ros2_ws/src
      git clone -n https://github.com/PX4/px4_ros_com.git
      cd px4_ros_com/
      git checkout 1562ff30d56b7ba26e4d2436724490f900cc2375
      cd ..
      git clone -n https://github.com/PX4/px4_msgs.git
      cd px4_msgs/
      git checkout b64ef0475c1d44605688f4770899fe453d532be4
      ```
    - ```sh
      cd ~/PX4-Autopilot/msg/tools/
      ./uorb_to_ros_msgs.py ~/PX4-Autopilot/msg/ ~/ros2_ws/src/px4_msgs/msg/
      ```      
    - This step may take more than an hour:
      ```sh
      cd ~/ros2_ws/src/px4_ros_com/scripts/
      ./build_ros2_workspace.bash --verbose
      ```   
      
Radar Cable Follower and IWR6843ISK ROS2 packages installation:
  - `cd ~/ros2_ws/src/`
  - `git clone https://github.com/nhma20/radar_cable_follower.git`
  - `git clone https://github.com/nhma20/radar_cable_follower_msgs.git`
  - `git clone https://github.com/nhma20/iwr6843isk_ros2.git`
  - `cd ~/ros2_ws/`
  - `colcon build`

Set up SSH:
  - `sudo apt update`
  - `sudo apt install openssh-server`
  - `sudo systemctl status ssh` (check if running)
  - `sudo ufw allow ssh`

Connect to network:
  - `nmcli radio wifi` (check if wifi on)
  - `nmcli dev wifi list` (list available networks)
  - `sudo nmcli dev wifi connect <wifi_ssid> password "<password>"`
  - Get host ip:
    - `ifconfig` (look for wlan* or eth or enp or similar)
  - `sudo nmap -sP <host_ip>/24 | awk '/^Nmap/{ip=$NF}/B8:27:EB/{print ip}'` (for Raspberry Pi 3)
  - `e.g.: sudo nmap -sP 192.168.1.0/24 | awk '/^Nmap/{ip=$NF}/B8:27:EB/{print ip}'`
  - if `dnet: Failed to open.. ` try `sudo snap connect nmap:network-control` or `nmap` with `--unprivileged` flag.
  - Try logging in on each output IP addres:
    - `ssh ubuntu@<ip_from_above>`


## Pixhawk Mini 4 setup
- Connect via USB to Host PC running QGroundControl
- On host PC:
  - Load Pixhawk with default firmware (1.12.3) through QGroundControl
  - Reset all parameters
  - Build specific version of firmware:
    - `git clone -n https://github.com/PX4/PX4-Autopilot.git`
    - `git checkout d7a962b4269d3ca3d2dcae44da7a37177af1d8cd`
    - `git submodule update --init --recursive`
    - Delete the micro_dds directory in the firmware (`PX4-Autopilot/src/modules/microdds_client/`)
    - Go to `PX4-Autopilot/boards/px4/fmu-v5/rtps.px4board` (create if does not exist) and set CONFIG_MODULES_MICRODDS_CLIENT=n
    - `cd ~/PX4-Autopilot/`
    - `make px4_fmu-v5_rtps`
  - Load Pixhawk with newly built firmware (`~/PX4-Autopilot/build/px4_fmu-v5_rtps/px4_fmu-v5_rtps.px4`) through QGroundControl
  - Set parameters (file provided in `/pixhawk_parameters/` directory):
    - `RTPS_CONFIG TELEM1`
    - `MAV_0_CONFIG TELEM/SERIAL4`
    - `RTPS_MAV_CONFIG Disabled`
    - `SER_TEL1_BAUD 460800 8N1`
    - `SER_TEL4_BAUD 57600 8N1`
    Optional speed parameters:
    - `MPC_Z_VEL_ALL 3`
    - `MPC_XY_VEL_ALL 5`
- Validate connection between Raspberry Pi and Pixhawk:
  - Connect FTDI-USB-to-Serial between Raspberry Pi USB port and Pixhawk TELEM1 port.
  - Power on Pixhawk (battery or debug micro-USB port).
  - On Raspberry Pi:
    - `source ~/ros2_ws/install/setup.sh`
    - `micrortps_agent -d /dev/ttyUSB0` (find correct port first)
    - `ros2 topic echo /fmu/vehicle_odometry/out` (should print the odometry messages from the Pixhawk)
    - If there is no connection, check that the micrortps_client is running on the Pixhawk with the MAVLink Console:
      - `micrortps_client status`
      - debug from there
  
  




