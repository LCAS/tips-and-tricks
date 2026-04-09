# Setting up Unitree L2 LiDAR

## Overview
The **Unitree L2 LiDAR** is a compact **4D LiDAR** sensor designed for robotic perception, mapping, and obstacle avoidance. It provides **360° × 96° ultra-wide-angle scanning**, a **very short 0.05 m blind zone**, up to **30 m ranging** at high reflectivity, and around **64,000 points per second**. Unitree also highlights its built-in **IMU**, compact size, and resistance to strong ambient light, making it suitable for both indoor and outdoor robotic platforms.

What makes the L2 stand out from many other LiDARs is its focus on **ultra-wide coverage in a small, lightweight package**. Compared with more conventional 3D LiDARs, it emphasises **non-repetitive scanning**, a **photograph-like dense scan effect**, and an unusually **large vertical field of view**, which is useful for near-range obstacle perception and mobile robot navigation. It also differs from many depth cameras because it offers true LiDAR ranging with **360° coverage**, longer range, and better robustness to outdoor lighting. 

![Unitree L2 LiDAR](https://www.unitree.com/images/ab6e0f38e57246418dd030d555f061a3_214x202.png)

*Figure: Unitree L2 LiDAR*

## 1. Installation
1. Clone the repository.
```bash 
git clone https://github.com/unitreerobotics/unilidar_sdk2.git
```

2. Compile the SDK.
```bash
cd unitree_lidar_sdk

mkdir build

cd build

cmake .. && make -j2
```
3. Compile ROS2 workspace

```bash
cd unilidar_sdk/unitree_lidar_ros2

colcon build

source install/setup.bash
```

## 2. Configuring the LiDAR
Configuring the Unitree L2 lidar is important before you use it as the LiDAR stores its configuration onboard and any configuration stored in the LiDAR must match with the respective ROS/ROS2 launch file configuration. There are 2 ways of updating the configuration: 1. with SDK; 2. with the windows tool "Unilidar".

### 2.1 Updating the configuration with SDK

**Prerequisite for both methods:** Your LiDAR must currently be powered on and connected to your computer via **Ethernet**, as it cannot receive the "switch to serial" command over a serial connection.

1. Open `unitree_lidar_sdk/examples/example_lidar_udp.cpp`.
2. Locate the working mode configuration line and change it to `9`:
   ```cpp
   uint32_t workMode = 9; // Wide FOV, 3D, IMU on, Serial
   ```
3. Compile the SDK:
   ```bash
   cd unitree_lidar_sdk
   mkdir build && cd build
   cmake .. && make -j2
   ```
4. Run the UDP example to send the command over Ethernet:
   ```bash
   ../bin/example_lidar_udp
   ```

### 2.2 Updating the configuration with the windows tool "Unilidar"

Easiest way to update the configuration is with the Unilidar software.

1. **Connect to the device**  
   Ensure your LiDAR is connected via the correct serial port (for example, `COM3`), then click **Open serial port**.

2. **Synchronise settings**  
   This is the most important step. After connecting, the software may display default values. Click **Synchronous** to read the actual current configuration from the LiDAR into the software. If you skip this step and make changes straight away, the software may overwrite your existing custom settings with the displayed defaults.

3. **Adjust the required modes**  
   Modify the settings as needed in the tabbed interface. For example, you may enable the IMU, switch between **Normal** and **Negative angle** mode, or change the **Start mode**.

4. **Apply changes**  
   Some settings take effect immediately. Others require you to click **Set mode** and then restart the unit for the changes to be applied.

[Click here for a video tutorial](https://www.youtube.com/watch?v=qj5su41qjRs)

## 3. ROS2 Setup
Now that the configuration is done, you can setup your ros2 launch file to match the configuration. Locate the launch file: `unilidar_sdk/unitree_lidar_ros2/src/unitree_lidar_ros2/launch/launch.py` and update the following configuration. Most importantly, match the `initialize_type` and `work_mode`. 

```python
parameters= [
                
                {'initialize_type': 1},#2: Ethernet, 1: Serial
                {'work_mode': 9},
                {'use_system_timestamp': True},
                {'range_min': 0.0},
                {'range_max': 100.0},
                {'cloud_scan_num': 18},

                {'serial_port': '/dev/ttyACM0'},
                {'baudrate': 4000000},

                {'lidar_port': 6101},
                {'lidar_ip': '192.168.1.62'},
                {'local_port': 6201},
                {'local_ip': '192.168.1.2'},
                
                {'cloud_frame': "unilidar_lidar"},
                {'cloud_topic': "unilidar/cloud"},
                {'imu_frame': "unilidar_imu"},
                {'imu_topic': "unilidar/imu"},
                ]
```

Follow the below table to identify the uint32 equivalent integer value for setting `work_mode` to the desired configuration.

| Bit Position | Function | Value 0 | Value 1 |
|---|---|---|---|
| 0 | Switch between standard FOV and wide-angle FOV | Standard FOV (180°) | Wide-angle FOV (192°) |
| 1 | Switch between 3D and 2D measurement modes | 3D measurement mode | 2D measurement mode |
| 2 | Enable or disable IMU | Enable IMU | Disable IMU |
| 3 | Switch between Ethernet mode and serial mode | Ethernet mode | Serial mode |
| 4 | Switch between lidar power-on default start mode | Power on and start automatically | Power on and wait for start command without rotation |
| 5-31 | Reserved | Reserved | Reserved |

**Now run the ros2 launch file.**
```bash
ros2 launch unitree_lidar_ros2 launch.py
```
