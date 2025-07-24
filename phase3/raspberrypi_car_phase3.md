# Part1: Hardware

- Raspberry pi 5 8GB
- Arduino UNO
- Rplidar A1
- HC-SR04
- L298N
- Two encoding motors
- Other hardwares like a display, a chessis, two wheels, a universal wheel, a breadboard, wires, a U disk (or SD card) and etc.

# Part2: Ubuntu system burning and configuration

## Burning

Get a ubuntu mirror for raspberry pi at [https://ubuntu.com/download/raspberry-pi](https://ubuntu.com/download/raspberry-pi).

I recommend an LTS system because it is stable.

You will find a mirror named like *ubuntu-24.04.2-preinstalled-desktop-arm64+raspi.img.xz*. That's good. Make sure its system architecture is arm64, not amd64.

Get a raspberry pi imager at [https://www.raspberrypi.com/software/](https://www.raspberrypi.com/software/).

In the raspberry pi imager, choose:

- Device: Raspberry pi 5
- System: Click *Use Custom*, then choose the mirror you have just downloaded.
- Storage: Choose your SD card or U disk.

**You should be clear that the following steps will clear your SD card or U disk.**

Press *Next*, and you don't need to configure anything. Wait until the burning is finished.

Insert the SD Card in the card slot, or insert your U disk in a blue USB socket.

## Configuration

Connect your power and display, and configure the system like any system you have once configured. 

Connect to WLAN or Ethernet, but an enterprise or school WLAN is not recommended, since ubuntu needs some strange information you might not know.

If you are in a place which does not support Internatonal Internet, you should find and download a .deb **arm64** package of clash or other proxy tools.

# Part3: ROS2 jazzy install

Follow this: [https://docs.ros.org/en/jazzy/Installation/Ubuntu-Install-Debs.html](https://docs.ros.org/en/jazzy/Installation/Ubuntu-Install-Debs.html)

Choose the full version of ros when installing, i.e. *sudo apt install ros-jazzy-desktop*.

**Install rosdep:**

```
sudo apt update
sudo apt install python3-rosdep
sudo rosdep init
rosdep update
```

**Create ROS2 workspace:**

- Create the workspace:

```
mkdir -p ~/ros2_ws/src
```

- Activate and compile:

```
cd ~/ros2_ws
colcon build
source install/setup.bash
```

Always activate and compile your ROS workspace under *ros2_ws* directory, not under *src*.

Remember to do this **every time you changed your code in the ROS workspace**.

**Apps recommended:**

- Vscode:

Download the .deb **arm64** package at [https://code.visualstudio.com/Download](https://code.visualstudio.com/Download).

Install:

```
sudo apt install ./vscode.deb
```

Change *vscode.deb* to your package name. Keep ./ in your command, or the command will fail.

Open Vscode:

```
code
```

- Terminator:

```
sudo apt install terminator
terminator
```

- Gnome screenshot: It is good if your keyboard doesn't have a printscreen button.

```
sudo apt install gnome-screenshot
```

Then find *Screenshot* in your app list.

# Part4: Motor driving

## teleop_twist_keyboard

This is a program which enables you to send linear and angular velocity commands. Now install and run:

```
cd ros2_ws
sudo apt install ros-jazzy-teleop-twist-keyboard
colcon build
source install/setup.bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

Press *i, j, k, l and ,* to send velocity commands, and *u, o, m and .* to increase or decrease the velocity.

**You should know the commands are sent through the node /cmd_vel.** 

## car_serial_control/cmdvel_to_serial:

*Receive info from node teleop_twist_keyboard, send velocity commands to arduino and stop when in emergency.*

**Create a python package:**

```
ros2 pkg create car_serial_control --build-type ament_python --dependencies rclpy
```

This will generate:

```
car_serial_control/
├── car_serial_control/
│   └── __init__.py
├── resource/
│   └── car_serial_control
├── setup.py
├── package.xml
└── setup.cfg
```

**Create a python file, which is the main code:**

```
cd ros2_ws/src/car_serial_control/car_serial_control
```

Details of the code are in the folder. You can edit your code with Vscode, gedit, vim, nano, etc.

### Read from the keyboard and send command

**Store the message of node /cmd_vel in msg:**

```
from geometry_msgs.msg import Twist
self.subscription = self.create_subscription(Twist, '/cmd_vel', self.cmd_callback, 10)
```

Since teleop_twist_keyboard sends info in node /cmd_vel, info of the keyboard can be received now.

Set two variables storing the properties of msg:

```
v = -msg.linear.x
w = msg.angular.z
```

*Note: The positive or negative of the variables depends on the installation direction of the wheel. It doesn't matter.*

**Calculate the speeds of both wheels:**

```
left = max(min(left, MAX_SPEED), -MAX_SPEED)
right = max(min(right, MAX_SPEED), -MAX_SPEED)
```

**Send velocity command:**

```
command = f"V{left:.2f},{right:.2f}\n"
self.serial_port.write(command.encode('utf-8'))
self.get_logger().info(f"Sent: {command.strip()}")
```

**Revise *setup.py*:**

Find this in *setup.py*:

```
entry_points={
        'console_scripts': [
        ],
    },
```

Insert this in the brackets[]:

```
'cmdvel_to_serial = car_serial_control.cmdvel_to_serial:main',
```

After coding, compiling and *source* command, run:

```
ros2 run car_serial_control cmdvel_ro_serial
```

*Note: The two 'ros2 run' command should be run at the same time.*

Now if pressing the keyboard, the command is sent in the **string** format like "V0.50,0.50", and you will receive info like "Sent: V0.50,0.50" in the terminal.
