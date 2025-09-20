# IsaacSim_ROS2_HelloWorld

This repository is a personal log and guide for setting up NVIDIA Isaac Sim on Windows 11 to communicate with ROS 2 (Jazzy) running in WSL2 (Ubuntu 24.02). It documents the steps, configurations, and workarounds encountered during the process.

## Step 1: Install Isaac Sim on Windows 11 (Host)

- Installed the Isaac Sim binary on the Windows 11 host machine via the Omniverse Launcher.

## Step 2: Install WSL2, Ubuntu, and ROS 2

- Installed WSL2 on Windows 11.
- Installed the Ubuntu 24.02 distribution from the Microsoft Store.
- Followed the official documentation to install ROS 2 Jazzy on the Ubuntu 24.02 instance.

## Step 3: Configure the Network Bridge (Windows & WSL2)

To allow Isaac Sim on the Windows host to communicate with ROS 2 in the WSL2 guest, we need to set up port forwarding.

> **Note:** This guide assumes your WSL2 instance is using the default `NAT` networking mode.

1.  **Find IP Addresses**

    In a **PowerShell** terminal, find the necessary IP addresses:

    ```powershell
    # Get the IP address of your WSL2 instance
    # This will be your <WSL2_IP>
    wsl hostname -I

    # Find your Windows host IP (IPv4) from your active network adapter (e.g., Wi-Fi or Ethernet)
    # This will be your <WINDOWS_IP>
    ipconfig /all
    ```

2.  **Set Up Port Forwarding**

    Define environment variables in the same PowerShell session for convenience and then run the port forwarding commands.

    ```powershell
    # Replace <WSL2_IP> and <WINDOWS_IP> with the actual addresses you found above
    $WSL2_IP = "<WSL2_IP>"
    $Windows_IP = "<WINDOWS_IP>"

    # Execute the port forwarding commands for DDS discovery
    netsh interface portproxy add v4tov4 listenport=7400 listenaddress=$Windows_IP connectport=7400 connectaddress=$WSL2_IP
    netsh interface portproxy add v4tov4 listenport=7410 listenaddress=$Windows_IP connectport=7410 connectaddress=$WSL2_IP
    netsh interface portproxy add v4tov4 listenport=9387 listenaddress=$Windows_IP connectport=9387 connectaddress=$WSL2_IP
    ```

    > **Tip:** You can verify that no other applications are using these ports with the command:
    >
    > ```powershell
    > netstat -ano | findstr ":7400 :7410 :9387"
    > ```

## Step 4: Configure Fast DDS Middleware in WSL2

For robust discovery across the network bridge, we need to configure ROS 2 to use a specific Fast DDS profile that forces UDPv4 transport.

1.  **Create the Fast DDS Configuration File**

    In your Ubuntu (WSL2) terminal, create the `fastdds.xml` file.

    ```bash
    # Create the directory if it doesn't exist and open the file for editing
    mkdir -p ~/.ros
    nano ~/.ros/fastdds.xml
    ```

    Paste the following XML content into the file:

    ```xml
    <?xml version="1.0" encoding="UTF-8" ?>
    <!--
    Copyright (c) 2022-2024, NVIDIA CORPORATION.  All rights reserved.

    NVIDIA CORPORATION and its licensors retain all intellectual property
    and proprietary rights in and to this software, related documentation
    and any modifications thereto.  Any use, reproduction, disclosure or
    distribution of this software and related documentation without an express
    license agreement from NVIDIA CORPORATION is strictly prohibited.
    -->
    <profiles xmlns="http://www.eprosima.com/XMLSchemas/fastRTPS_Profiles" >
        <transport_descriptors>
            <transport_descriptor>
                <transport_id>UdpTransport</transport_id>
                <type>UDPv4</type>
            </transport_descriptor>
        </transport_descriptors>

        <participant profile_name="udp_transport_profile" is_default_profile="true">
            <rtps>
                <userTransports>
                    <transport_id>UdpTransport</transport_id>
                </userTransports>
                <useBuiltinTransports>false</useBuiltinTransports>
            </rtps>
        </participant>
    </profiles>
    ```

2.  **Export the Configuration and Source ROS 2**

    For the changes to take effect, export the path to this file as an environment variable and then source your ROS 2 installation.

    ```bash
    export FASTRTPS_DEFAULT_PROFILES_FILE=~/.ros/fastdds.xml
    source /opt/ros/jazzy/setup.bash
    ```

    > **Note:** To make this change persistent across terminal sessions, add the `export` command to your `~/.bashrc` file.

## Step 5: Launching Isaac Sim with the ROS 2 Bridge

This section describes the documented method and the workaround that was ultimately successful for me.

### The Official Method

As per the Isaac Sim documentation, the following **PowerShell** commands are used to set environment variables and launch the application with the ROS 2 Bridge enabled:

```powershell
# Set environment variables
$env:isaac_sim_package_path = "C:\isaacsim" # Or your Isaac Sim install path
$env:ROS_DISTRO = "jazzy"
$env:RMW_IMPLEMENTATION = "rmw_fastrtps_cpp"

# Only set this once per session to avoid path conflicts
$env:PATH = "$env:PATH;$env:isaac_sim_package_path\exts\isaacsim.ros2.bridge\jazzy\lib"

# Run Isaac Sim with ROS 2 Bridge Enabled
& "$env:isaac_sim_package_path\isaac-sim.bat" --/isaac/startup/ros_bridge_extension=isaacsim.ros2.bridge
```

### My Workaround

> I found that launching with the command line argument did not enable the ROS 2 Bridge extension correctly. The extension remained disabled and could not be re-enabled from the UI.
>
> **My solution was to launch Isaac Sim normally and enable the extension manually:**

1.  Run Isaac Sim using the standard selector: `C:/isaacsim/isaac-sim.selector.bat`
2.  In the startup pop-up, **do not** select the ROS 2 Bridge checkbox.
3.  Once the Isaac Sim application is open, navigate to the extensions menu (`Window > Extensions`).
4.  Search for `ROS 2 Bridge` and manually enable it by toggling the switch. This worked successfully.

## Result: Hello World!

Following these steps, I successfully set up a simple rover in Isaac Sim equipped with a Lidar and a Camera. The simulation now correctly publishes Lidar data to ROS 2 Jazzy topics, which can be visualized and consumed by ROS nodes running in WSL2 (e.g., `rviz2`). This serves as a successful "Hello World" proof-of-concept for the entire setup.

### The Simulation Scene

The test environment and robot were built with the following components:

-   **The Rover:**
    -   **Frame:** A simple chassis constructed from a rectangular cube.
    -   **Wheels:** Four cylindrical wheels.
    -   **Drivetrain:** A **differential drive** controller is applied to the two rear wheels, enabling realistic turning.
    -   **Control:** The rover is manually controlled using standard **WASD** keyboard input to drive forward, backward, and turn.

-   **The Environment:**
    -   The scene contains several primitive shapes (a cylinder, a cube, and a cone) that serve as simple obstacles to navigate around.

### ROS 2 Lidar Stream Action Graph
The following image shows the Action Graph in Isaac Sim responsible for streaming the Lidar data to ROS 2.
<img width="1344" height="770" alt="Lidar Action Graph" src="https://github.com/user-attachments/assets/7397ee3c-c51a-4980-a29d-18364de22f42" />

### Robot Setup in Isaac Sim
This is the initial setup of the simple rover in the Isaac Sim scene, equipped with the Lidar and Camera sensors.

<img width="1920" height="928" alt="Robot in Isaac Sim" src="https://github.com/user-attachments/assets/ba857c8d-40fe-4b30-9b7b-66bbdd32cb6f" />

### Demonstration Video
Here is a short clip demonstrating the Lidar data being published from Isaac Sim and visualized in RViz2 within WSL2.

If the GIF above doesn't load, you can view or download it directly from the project's releases:

![Demonstration Video](https://github.com/AydinLT00/IsaacSim_ROS2_HelloWorld/releases/download/release_demo/demo.gif)



> **Disclaimer:** This guide is heavily based on the official [NVIDIA Isaac Sim ROS 2 Bridge Documentation](https://docs.isaacsim.omniverse.nvidia.com/latest/installation/install_ros.html). This repository serves as a personal log of the steps I took and the specific workarounds I found while following their instructions. All credit for the core technology and original documentation goes to the NVIDIA team.



## Phase 1: Robot State Publishing (Odometry & TF)

After establishing basic manual control, the next critical step towards autonomy is enabling the robot to publish its state to the ROS 2 ecosystem. This involves broadcasting its position and orientation (Odometry) and the spatial relationship between all its components (TF).

### What I Did

1.  **Odometry Publishing:** I created a new Action Graph in Isaac Sim dedicated to calculating and publishing the robot's odometry. This graph sends messages to the `/odom` topic, providing real-time data about the robot's movement in its own starting frame.
2.  **TF Tree Publishing:** The same Action Graph was configured to publish the robot's transform tree to the `/tf` topic. This is essential for ROS 2 to understand the physical layout of the robot (e.g., where the wheels and sensors are in relation to the main body).
3.  **Lidar Frame Configuration:** The existing Lidar continues to publish its data to the `/scan_lidar` topic.

### Key Challenge & Solution: Bridging Simulation and ROS Frames

A common challenge when integrating simulators with ROS is ensuring the transform tree (TF) is complete and correct.

*   **The Problem:** The Lidar sensor in Isaac Sim was physically attached to a frame called `front_sensor` on the robot model. However, the ROS 2 Bridge published the Lidar data with its own `frame_id` called `sim_lidar`. Without a link between `front_sensor` and `sim_lidar`, RViz2 had no way to know where the Lidar scans were coming from relative to the robot's body.

*   **The Solution:** To fix this, I added a **`Raw Transform Tree`** node within the Odometry Action Graph. This node acts as a static transform publisher, explicitly defining the spatial relationship between the robot's `front_sensor` frame and the Lidar's `sim_lidar` frame. This crucial step completed the TF tree, allowing all data to be correctly visualized.

### Result & Visualization

With odometry and a complete TF tree being published, we can now visualize the robot's state properly in RViz2. By setting the "Fixed Frame" to `odom`, the world remains stationary while the robot model and its sensor data move together realistically.

**RViz2 Visualization:**
This video shows the robot being driven manually in Isaac Sim. In RViz2, you can see the `odom` frame fixed in place, the robot's TF frames moving correctly, and the Lidar scan data perfectly aligned with the robot's position.

![Odometry and TF Visualization](https://github.com/AydinLT00/IsaacSim_ROS2_HelloWorld/releases/download/odom_fixed/odom_demo.gif)

**Odometry & TF Action Graph:**
This image shows the Isaac Sim Action Graph responsible for publishing both the `/odom` and `/tf` topics, including the `Raw Transform Tree` node used to fix the Lidar's frame issue.
<img width="1609" height="779" alt="odom_tf_ActionGraph" src="https://github.com/user-attachments/assets/2a43092c-d332-4815-86d9-ed1d8d1fae02" />
