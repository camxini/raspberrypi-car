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

Install rosdep:

```
sudo apt update
sudo apt install python3-rosdep
sudo rosdep init
rosdep update
```

Apps recommended:

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

