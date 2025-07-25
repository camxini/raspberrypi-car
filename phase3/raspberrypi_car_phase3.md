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
v = msg.linear.x
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

Through **serial**, the command will be sent to Arduino.

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

### Emergency stop

The *measure_distance* function is used for calculating the distance between the obstacle and the ultrasonic sensor:

```
pulse_start = time.time()
pulse_end = time.time()
while not self.echo.is_active:
    pulse_end = time.time()
while self.echo.is_active:
    pulse_end = time.time()
pulse_duration = pulse_end - pulse_start
distance = (pulse_duration * 340) / 2
```

The *while* sentences are used for recording the end time. For example, if echo=0, the first loop begins and *pulse_end* adds until echo=1.

If the car is close to the obstacle, a "V0,0" command will be sent.

## Arduino 

*Receive velocity commands, drive the motor and send the encoder pulse data.*

### Receive velocity commands

Arduino has received a string like "V0.50,0.50" through serial, and now process this string:

```
input = input.substring(1); // remove 'V'
int commaIndex = input.indexOf(',');
if (commaIndex != -1) {
    float left = input.substring(0, commaIndex).toFloat();
    float right = input.substring(commaIndex + 1).toFloat();
}
```

The code above removes "V" first, and then split the rest string in two numbers.

Use *digitalWrite* and *analogWrite* functions to drive motors with PWM control.
*Change the code here if you want to use PID.*

Send serial info:

```
Serial.print("Received: L");
Serial.print(left);
Serial.print(",R");
Serial.println(right);
```

You can see the output in the serial monitor.

### Send encoder data

The function *attachInterrupt()* runs when the monitored variable changes. It is fast and uses less calculation.

Let's take motor1 for example.

**Initialize:**

```
attachInterrupt(digitalPinToInterrupt(encoder1A), encoder1ISR, RISING);
```

**Count when attachInterrupt is triggered:**

```
void encoder1ISR() {
  if (digitalRead(encoder1B) == HIGH)
    encoder1++;
  else
    encoder1--;
}
```

**Send serial info:**

```
noInterrupts();
long left = encoder1;
long right = encoder2;
interrupts();
Serial.print(left);
Serial.print(",");
Serial.println(right);
```

Now the encoder pulse data of both motors are output in a form like "500,1000". Raspberry pi receives this info through **serial**.

# Part 5: TF tree building

## encoder_reader/encoder_odom

*Receive and display data from Arduino, calculate odometry (displacement, rotation and velocity) and send tf messages between base_link and odom.*

### Receive Arduino data

There are two types of output of Arduino:

- Received: Lx.xx,Rx.xx
- Lxxx,Rxxx

Filter the first one, since we need encoder data only:

```
if ':' in msg.data or msg.data.startswith("Received"):
    return
if ',' not in msg.data:
    return
```

*Note: Some messages will be sent like 'Received: Lx.xx' or 'Lxxx', which are not completed.*

### Calculate odometry

Messages odom needs: displacement ($x,y,z$), angle quaternion ($q$), linear and angular velocity of the whole car ($v_x, v_\theta$).

Calculation:

$$\Delta l = \pi D (n_l - n_{l0}) / P$$
$$\Delta r = \pi D (n_r - n_{r0}) / P$$
$$\Delta c = (\Delta l + \Delta r) / 2$$
$$\Delta \theta = (\Delta r - \Delta l) / B$$
$$x = \Delta c + cos(\theta + \Delta \theta / 2)$$
$$y = \Delta c + sin(\theta + \Delta \theta / 2)$$
$$z = 0$$
$$\theta += \Delta \theta$$
$$q_x = 0$$
$$q_y = 0$$
$$q_z = sin(\theta / 2)$$
$$q_w = cos(\theta / 2)$$
$$v_x = \Delta c / \Delta t$$
$$v_\theta = \Delta \theta / \Delta t$$

Symbol explanation:

| Symbol | Explanation |
| --- | --- |
| $D$ | wheel diameter |
| $P$ | number of encoder pulses per revolution |
| $B$ | wheel distance |
| $n_l, n_r$ | number of encoder pulses of left (right) wheel |
| $n_{l0}, n_{r0}$ | number of pulses of left (right) wheel of last moment |
| $d_l, d_r$ | moving distance of left (right) wheel |
| $d_c$ | moving distance of car center |
| $d_\theta$ | angular variation of car center |
| $(x,y,z)$ | displacement of the car |
| $\theta$ | angular increment of the car |
| $d_t$ | time difference |
| $(q_x,q_y,q_z,q_w)$ | quaternion (rotation) |
| $v_x$ | linear velocity |
| $v_\theta$ | angular velocity |

### Send tf messages

The tf relationship of base_link and odom is clear after odometry calculation. Odom is a fixed frame, and base_link moves with the car.

Send displacement and quaternion info. This creates a TF branch odom->base_link, which is useful in TF tree in the following chapters.

After coding, compiling and *source* command, run:

```
ros2 run encoder_reader encoder_odom
```

You will see the encoder pulse data in the terminal.

## Laser

```
cd ros2_ws/src
git clone https://github.com/Slamtec/rplidar_ros.git
cd ..
rosdep install --from-paths src --ignore-src -r -y
colcon build
source install/setup.bash
```
Every time after compiling and *source* command, you can run:

```
ros2 launch rplidar_ros rplidar_a1_launch.py
```

to start broadcasting info of node /scan.

## Completion of other TF messages

*The whole TF is map->odom->base_link->laser. We have merely a branch odom->base_link.*

### base_link to laser

Force a static TF transformation:

```
ros2 run tf2_ros static_transform_publisher 0.1 0 0 0 0 0 1 base_link laser
```

The seven numbers above are given in the form $x,y,z,q_x,q_y,q_z,q_w$. Edit the numbers if the laser is in another place.

### map to odom

Use slam_toolbox to link /map with /odom. Gmapping is not recommended in ros2 jazzy.

```
ros2 launch slam_toolbox online_async_launch.py scan_topic:=/scan odom_topic:=\odom use_sim_time:=false
```

Ensure topics /scan and /odom have been launched. You can check your current topic with this command:

```
ros2 topic list
```

Now the tf tree has been set. You can examine your tf relation by:

```
ros2 run tf2_tools view_frames
```

It will generate a pdf file of your tf tree. Make sure all your 'ros2 run' and 'ros2 launch' commands are running.

### Summary

Here are all the commands running currently:

```
ros2 run car_serial_control cmdvel_to_serial
ros2 run teleop_twist_keyboard teleop_twist_keyboard
ros2 run encoder_reader encoder_odom
ros2 launch rplidar_ros rplidar_a1_launch.py
ros2 run tf2_ros static_transform_publisher 0.1 0 0 0 0 0 1 base_link laser
ros2 launch slam_toolbox online_async_launch.py scan_topic:=/scan odom_topic:=\odom use_sim_time:=false
```

Search in the topic list and be sure you can see the following topics:

- /scan
- /odom
- /base_link
- /map

Make sure your tf tree (map->odom->base_link->laser) is complete.

# Part6: SLAM mapping

## ssh

Enable your ssh service:

```
sudo apt install openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
```

Get your IP address:

```
hostname -I
```

Your PC should connect the same WLAN or Ethernet with the raspberry pi. Try ssh connection in your windows cmd terminal:

```
ssh <hostname>@<ip address>
```

where <hostname> is your ubuntu hostname, <ip address> is the ip address you have just got.

## Graphic interface forwarding

SSH allows non-graphic commands. If commands including graphics like *rviz* is needed, we should configure x server.

In Windows, download MobaXterm Xserver at: [https://mobaxterm.mobatek.net/download-home-edition.html](https://mobaxterm.mobatek.net/download-home-edition.html)

Run MobaXterm Xserver, choose *Session* in the upper left corner, then choose *SSH*.

Enter the IP address of raspberry pi, then the terminal will appear. Enter *rviz2* ro check if the graphic interface can be displayed in Windows.
