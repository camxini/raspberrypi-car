**Note: 这部分基本没怎么写，因为树莓派4B中途损坏了，被迫更换了新的树莓派。**

# Phase 2 自由探索部分

Time: 2025.5-2025.6

Progress: 

---

看到有同学做了一个用树莓派5烧录ubuntu 24.04 desktop的机器人，我感觉ubuntu24的界面非常流畅，想要体验一下，另外也确实是想看一下ROS2现在发展到了什么样子，所以我又买了一张TF卡，烧录了一下ubuntu24系统，准备开工。

另外，再次从零开始也可以检验一下前面phase1的步骤是否有误。

那我们开始吧。

## 1. 系统烧录

这次使用的是从ubuntu官方提供的ubuntu 24.04 desktop的树莓派arm64镜像。

由于使用的是桌面端的镜像，因此我直接给树莓派接上了显示屏。其实步骤都差不多，就省略了一个安装桌面环境的步骤。但是，我发现了安装桌面版（或者配桌面环境）的一个很显著的优点：

在最开始在raspberry pi imager里烧录系统的时候，可以自定义配置，配置树莓派的用户名，wifi以及ssh. 但是，这里的wifi是不可以有用户名的那种，而校网恰好有用户名，这也就意味着，如果你安装的不是桌面版，然后又没装桌面环境，那么在校网环境中的你只能用手机热点连接树莓派了。

但是，如果你有了桌面环境，不管你在最开始烧录的时候wifi的初始设置是怎么样的，你都可以顺利连接校网，从而避免了挂着手机流量的麻烦（我可是在半个月内为了做树莓派耗费了50G流量的人）。

烧录过程一切照常，烧录完以后启动树莓派，你完全可以把它当作一个普通的PC，该怎么设置ubuntu系统就怎么设置ubuntu系统。

## 2. ROS配置

由于ubuntu24用的是ros2，版本是jazzy, 因此有很多操作都和ros1有非常大的区别。我们一步一步来：

### 2.1 配置镜像源

和之前的一样，先导入镜像源，导入密钥，这里以清华的镜像为例：

```text
sudo apt install curl
sudo curl -sSL https://mirrors.tuna.tsinghua.edu.cn/rosdistro/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] https://mirrors.tuna.tsinghua.edu.cn/ros2/ubuntu jammy main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
```

然后更新软件：

```text
sudo apt update
```

### 2.2 安装ros2

先安装ROS完整桌面版，再配置环境变量：

```text
sudo apt install ros-jazzy-desktop
source /opt/ros/jazzy/setup.bash
```

### 2.3 安装rosdep

接下来，按照原来的步骤应该是安装rosdep，或者在国内安装rosdepc. 在这之前，需要先安装python3:

```text
sudo apt install python3-pip
```

然后问题就来了：rosdepc毕竟是个人来更新的，还没更新到jazzy的ros2版本。所以，我们不得不用rosdep，这可能会涉及到国内连接不上的问题，但好像清华源作为镜像源的情况下能顺利安装上（也可能是我运气好一次就成功了）。

这个网络连接的问题我们稍后再聊。在ubuntu20里面，安装rosdep需要用到pip3, 而pip3是python3的指令。但是在ubuntu24中，jazzy禁止了通过全局pip来安装python包，所以我们需要先创建一个python的虚拟环境，这需要用到venv:

```text
sudo apt install python3.12-venv
python3 -m venv ~/rosdep_venv
source ~/rosdep_venv/bin/activate
```

这三句指令分别是安装venv，创建虚拟环境，激活虚拟环境。现在你的命令行前面应该出现了一个(rosdep_venv)，代表已经进入到了python的虚拟环境。

现在可以来安装rosdep了：

```text
pip install rosdep
```

接下来需要初始化rosdep，这个命令需要全局权限，因此需要先退出虚拟环境：

```text
deactivate
```

然后初始化rosdep:

```text
sudo rosdep init
```

如果init报错not found, 可以用绝对路径来初始化（先查找rosdep的位置，再进行初始化）：

```text
which rosdep
sudo /path init
```

记得把/path替换为刚才查找到的rosdep的路径。

再进入虚拟环境：

```text
source ~/rosdep_venv/bin/activate
```

然后对rosdep检查更新:

```text
rosdep update
```

这样rosdep就成功安装了。

另外，**什么时候需要开启虚拟环境？** 其实从刚才的操作中可以看出，和python有关的都需要开启虚拟环境，你可以具体理解为自己写的python脚本，涉及到rosdep的指令等等。常规的ros2命令不需要开启虚拟环境。

### 2.4 其他安装

由于ros1变成了ros2，原来需要安装的包和现在的名字已经不一样了，他们的替换关系如下：

```text
rosinstall -> colcon
wstool -> vcstool
roslaunch -> ros2launch
```

这几个包的安装需要rosdep，所以需要在虚拟环境中运行。

```text
sudo apt install python3-colcon-common-extensions python3-vcstool ros-jazzy-ros2launch
```

然后再检查一次更新：

```text
sudo apt update
```

正常情况下，ros2就已经安装好了。你可以通过下面的步骤来检查ros2是否已经成功安装。

### 2.5 确认ros2已经成功安装

在原来ROS1的环境里，想要启动ros的所有服务需要先roscore. 在ROS2里修改了所有节点的通讯方式，这就意味着你不需要再用roscore来启动了，可以直接输入ros2的指令来启动。

ros2里面有示例的talker和listener文件，也就是让talker发送信息，然后让listener接收信息。如果listener成功接收了talker的信息，就证明ros2的安装没有问题。（这里插一嘴，原来我用商用的AGV的时候，当时是ubuntu16，那个时候还要自己写talker和listener的文件来验证功能，现在已经有例程了，看来ROS还是知道我们想要什么的）

说多了，在一个终端里启动talker（这是常规的ros2指令，你可以先退出虚拟环境再执行命令）：

```text
ros2 run demo_nodes_cpp talker
```

应该会显示一堆的[talker]: Publishing: 'Hello World: xxx', 然后再新开一个终端，启动listener:

```text
ros2 run demo_nodes_cpp listener
```

如果能接收到信息，显示[listener]: I heard: [Hello world: xxx]，那么恭喜你，你的ros2已经成功安装了。

记得把一些常用的包，比如gedit或者nano或者vim安装好，也还是用sudo apt install指令就可以。

## 3. 关于编码器电机

电机的接线和原理在phase1当中都有提到，由于电机并没有换，因此在phase2里这部分都是一样的，包括所有的oython代码都是一样的。

但是，ros2的python代码是需要在虚拟环境里面运行的。而在虚拟环境里运行调用的是虚拟环境里面的库，因此你需要在虚拟环境里面安装库：

```text
pip install RPi.GPIO
pip install numpy
```

然后在虚拟环境中运行所有的python代码，但是需要注意，由于GPIO需要有sudo权限，而sudo权限默认调用的是系统自带的python库，因此在运行文件的时候要把python库重定向到虚拟环境的python库：

```text
sudo ~/rosdep_venv/bin/python3 test.py
```
