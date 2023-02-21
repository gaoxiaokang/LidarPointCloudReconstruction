# 实时激光雷达点云重建


### 环境配置
1. 在 Ubuntu 64-bit 18.04 系统下使用该项目.

2. 安装 ROS Melodic 环境
    ```
    cd install_shell/shell
    ./install_ros.sh
    ```

3. 安装 Ceres Solver
    ```
    wget http://ceres-solver.org/ceres-solver-2.1.0.tar.gz
    sudo apt-get install libgoogle-glog-dev libgflags-dev
    sudo apt-get install libatlas-base-dev
    sudo apt-get install libeigen3-dev
    sudo apt-get install libsuitesparse-dev
    tar zxf ceres-solver-2.1.0.tar.gz
    mkdir ceres-bin
    cd ceres-bin
    cmake ../ceres-solver-2.1.0
    make -j3
    make test
    make install
    ```

4. 安装 PCL, 参见 <a href="https://pointclouds.org/downloads/#linux">PCL 官方文档</a>.
    ```
    sudo apt-get update
    sudo apt-get install git build-essential linux-libc-dev
    sudo apt-get install cmake cmake-gui
    sudo apt-get install libusb-1.0-0-dev libusb-dev libudev-dev
    sudo apt-get install mpi-default-dev openmpi-bin openmpi-common 
    sudo apt-get install libflann1.8 libflann-dev
    sudo apt-get install libflann1.8 libflann-dev
    sudo apt-get install libboost-all-dev
    sudo apt-get install libeigen3-dev
    sudo apt-get install libvtk7-jni libvtk7-java libvtk7-dev
    sudo apt-get install libvtk7.1-qt libvtk7.1 libvtk7-qt-dev
    sudo apt-get install libqhull* libgtest-dev
    sudo apt-get install freeglut3-dev pkg-config
    sudo apt-get install libxmu-dev libxi-dev
    sudo apt-get install mono-complete
    sudo apt-get install openjdk-8-jdk openjdk-8-jre
    # 下载编译
    git clone https://github.com/PointCloudLibrary/pcl.git
    ## 编译
    cd pcl
    mkdir release 
    cd release
    cmake -DCMAKE_BUILD_TYPE=None -DCMAKE_INSTALL_PREFIX=/usr \ -DBUILD_GPU=ON-DBUILD_apps=ON -DBUILD_examples=ON \ -DCMAKE_INSTALL_PREFIX=/usr .. 
    make
    ## 安装
    sudo make install
    # 也可以直接安装
    sudo apt install libpcl-dev
    sudo apt-get install ros-melodic-pcl-conversions
    sudo apt-get install ros-melodic-pcl-ros
    ```

5. 安装 Anaconda
    ```
    wget https://mirrors.tuna.tsinghua.edu.cn/anaconda/archive/Anaconda3-2022.05-Linux-x86_64.sh
    sh Anaconda3-2022.05-Linux-x86_64.sh
    ```

6. 安装 CGAL
    ```
    sudo apt-get install libcgal-dev
    ```

7. 修改 ROS 脚本文件的超时时间, 使得在程序运行结束时可以保存点云结果.

   ```shell
   # 打开文件 nodeprocess.py
   sudo gedit /opt/ros/melodic/lib/python2.7/dist-packages/roslaunch/nodeprocess.py
   
   # 修改 nodeprocess.py 文件中的如下语句:
   DEFAULT_TIMEOUT_SIGINT = 15.0 
   # 将时间从 15.0s 改为 60.0 秒或更多
   DEFAULT_TIMEOUT_SIGINT = 60.0
   ```
8. 其他依赖
    ```
    sudo apt-get install libpcap-dev
    ```

### 构建项目

```shell
# 先进入项目目录
cd LidarPointCloudReconstruction

# 创建环境1：配置 anaconda 环境, 如果不需要泊松重建部分可以忽略此项(zc注：这条命令没成功，可以采用第二种方式创建环境)
conda env create -f environment.yml
# 创建环境2：配置 anaconda 环境
conda create -n denseRos python=3.8

# 进入环境
conda activate denseRos

# 创建工作目录
mkdir -p ~/catkin_ws/src

# 将项目文件夹拷贝到 ~/catkin_ws/src 文件夹中
cd .. 
cp -r LidarPointCloudReconstruction ~/catkin_ws/src

# 构建项目
cd ~/catkin_ws
catkin_make -DCMAKE_BUILD_TYPE=Release 
# 遇到vtkRenderingOpenGL2;vtkRenderingContextOpenGL2的错误,是因为PCL版本过高,要求VTK的版本也较高,因此在PCLConfig.cmake里面要求是vtkRenderingOpenGL2;vtkRenderingContextOpenGL2,而自己电脑里面装的VTK版本低于8.0,只能提供vtkRenderingOpenGL;vtkRenderingContextOpenGL,因此需要在PCLConfig.cmake里面将vtkRenderingOpenGL2;vtkRenderingContextOpenGL2更改为vtkRenderingOpenGL;vtkRenderingContextOpenGL #
# 如果需要泊松重建,则运行这一句(注意xxxx要改成自己的用户名，找到自己conda的路径)
catkin_make -DCMAKE_BUILD_TYPE=Release -DPYTHON_EXECUTABLE=/home/xxxx/anaconda3/envs/puma/bin/python

```

### 下载数据包(需要翻墙，有需要@我)
```
下载 [NSH indoor outdoor](https://drive.google.com/file/d/1s05tBQOLNEDDurlg48KiUWxCp-YqYyGH/view) 数据包，并保存到`data`文件夹中。
```

### 项目运行方式一（注意bag包的路径）
```
# 这种方式是直接加载bag包，与我们实时加载点云topic的要求有些许出入
source ~/catkin_ws/bin/setup.zsh # 如果你安装了zsh
source ~/catkin_ws/bin/setup.bash # 如果你没有安装zsh,采用的原始的bash
roslaunch simple_frame mapping.launch rosbag:=/home/xxxx/catkin_ws/src/LidarPointCloudReconstruction/data/nsh_indoor_outdoor.bag
```

### 项目运行方式二（目前该方法只是暂时，后续需要改进为实时读取激光雷达传感器数据流）
```
# 这种方式是通过播放bag包，产生点云topic的方式，来加载点云数据
# 需要修改launch文件中的bag包路径指定
vim ~/catkin_ws/src/LidarPointCloudReconstruction/simple_frame/launch/mapping.launch
# 删除或注释掉第52行代码
<node pkg="rosbag" type="play" name="rosbag" args="--clock -r 1 $(arg rosbag) --topics /velodyne_points"/>
# 打开终端一输入命令：
source ~/catkin_ws/bin/setup.zsh # 如果你安装了zsh
source ~/catkin_ws/bin/setup.bash # 如果你没有安装zsh,采用的原始的bash
roslaunch simple_frame mapping.launch
# 打开终端二输入命令：
rosbag play -l nsh_indoor_outdoor.bag --clock
```
### rviz界面出现可视化界面，说明运行成功
```
# 查看节点的关系
rqt_graph
```

## 其它问题：
1. 该数据流是基于velodyne激光雷达的数据开发，所接收的点云topic是/velodyne_points,目前我的测试是，直接更改其他激光雷达驱动的launch文件的topic信息，镭神科技为例，将其点云/topic修改为/velodyne_points。
2. 源码中的bag文件中的点云topic是/velodyne_points，rviz可视化中的Fixed Frame为velodyne。
3. 很遗憾，目前我录制的镭神雷达的包不能正常运行在他的原代码里，他默认的参数设置是针对64线velodyne，我测试的镭神激光雷达是16线，可以从这个方面去尝试解决。


## 附加镭神C16激光雷达的安装与启动
### 安装驱动
实验室的镭神C16是18年款，比较老，网上开源驱动不适配，编译会报错。
```
# 创建编译空间
mkdir -p ~/leishen_ws/src
cd ~/leishen_ws
cp -r lslidar_c16 ~/leishen_ws/src # lslidar_c16也在该项目other文件夹内，复制到新的工作空间内
catkin_make
```
### 配置雷达网口
镭神激光雷达的默认ip是192.168.1.200，需要修改本地电脑ip。
```
打开‘设置’, 'network', 点击wired的设置，修改ipv4如下图,然后点击Apply.
```
<image src="./other/net.png">

```
# 修改完后，检查本电脑ip是否跟改为192.168.1.102
ifconfig
# 检查激光雷达是否正常连接
ping 192.168.1.200
# 再次检查雷达数据是否正常（可以忽略）
rostopic hz /lslidar_point_cloud
```
### 启动镭神激光雷达
```
cd ~/leishen_ws
source ~/leishen_ws/devel/setup.zsh # 如果你安装的zsh
source ~/leishen_ws/devel/setup.bash # 如果你安装的bash
roslaunch lslidar_c16_decoder lslidar_c16.launch --screen
```
### 可视化镭神点云
```
# 镭神激光雷达的默认Fixed Frame是/laser_link，所以可视化需要修改rviz的Fixed Frame信息。（或者在驱动的launch文件内修改参数，使其Fixed Frame更改为velodyne）
rviz
```

## 目前首要的是实现镭神激光雷达的实时建图，具体部分操作如下（尚未完成测试）
### 修改镭神激光雷达驱动的launch文件
```
vim ~/catkin_ws2/src/lslidar_c16/lslidar_c16_decoder/launch/lslidar_c16.launch
# 第11行内容修改 原内容：<param name="frame_id" value="laser_link"/>  修改后：<param name="frame_id" value="velodyne"/>
# 第11后添加： <remap from="lslidar_point_cloud" to="$velodyne_points"/>
```
### 运行建图程序
```
source ~/catkin_ws/bin/setup.zsh # 如果你安装了zsh
source ~/catkin_ws/bin/setup.bash # 如果你没有安装zsh,采用的原始的bash
roslaunch simple_frame mapping.launch
```
### 运行镭神激光雷达
```
source ~/leishen_ws/devel/setup.zsh
roslaunch lslidar_c16_decoder lslidar_c16.launch --screen --clock
```

### 目前点云建图及显示存在问题，可以参考“其它问题”中的第三条或者其他什么的。

---
# 兄弟们，我尽力了，加油！！！