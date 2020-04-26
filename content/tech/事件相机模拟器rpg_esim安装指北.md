---
title: "事件相机模拟器rpg_esim安装指北"
subtitle: ""
description: ""
tags:
    - "dvs"
    - "tutorial"
date: 2020-03-26T23:38:12+08:00
draft: false
author: EvernessW

toc: true
katex: false
mermaid: false
---

## 系统环境

VirtualBox 6.1.4 & Ubuntu 16.04 LTS

## 配置网络环境

为加速github clone，先配置虚拟机网络使之使用主机代理

VirtualBox选用NAT模式后，若虚拟机ip地址为10.0.x.15，则宿主机ip为10.0.x.2

配置git代理

```bash
git config --global http.proxy http://10.0.x.2:10809
git config --global https.proxy http://10.0.x.2:10809
```

由于apt源可使用国内镜像，因此可不配置系统代理

## 安装ROS

参考[官方文档](http://wiki.ros.org/kinetic/Installation/Ubuntu)

使用tuna镜像

```bash
vim /etc/apt/sources.list.d/ros-latest.list
> deb https://mirrors.tuna.tsinghua.edu.cn/ros/ubuntu/ xenial main
sudo apt-key adv --keyserver 'hkp://keyserver.ubuntu.com:80' --recv-key C1CF6E31E6BADE8868B172B4F42ED6FBAB17C654
sudo apt update
sudo apt install ros-kinetic-desktop-full
```

## 配置ROS

初始化

```bash
sudo rosdep init
rosdep update
```

环境配置

（需要注意这里使用的是zsh，如果使用其他命令行工具需要进行相应更改）

```bash
echo "source /opt/ros/kinetic/setup.zsh" >> ~/.zshrc
source ~/.zshrc
```

安装编译依赖

```bash
sudo apt install python-rosinstall python-rosinstall-generator python-wstool build-essential
```

## 安装rpg_esim

* 安装编译工具

  ```bash
  sudo apt-get install ros-kinetic-catkin python-catkin-tools python-vcstool
  ```

* 创建workspace

  ```bash
  mkdir -p ~/sim_ws/src && cd ~/sim_ws
  catkin init
  catkin config --extend /opt/ros/kinetic --cmake-args -DCMAKE_BUILD_TYPE=Release
  ```

* 下载源码

  ```bash
  cd src/
  git clone https://github.com/uzh-rpg/rpg_esim.git
  ```

* 处理依赖

  ```bash
  sudo apt install ros-kinetic-pcl-ros libglfw3 libglfw3-dev libglm-dev ros-kinetic-hector-trajectory-server
  vcs-import < rpg_esim/dependencies.yaml
  cd ze_oss
  touch imp_3rdparty_cuda_toolkit/CATKIN_IGNORE \
        imp_app_pangolin_example/CATKIN_IGNORE \
        imp_benchmark_aligned_allocator/CATKIN_IGNORE \
        imp_bridge_pangolin/CATKIN_IGNORE \
        imp_cu_core/CATKIN_IGNORE \
        imp_cu_correspondence/CATKIN_IGNORE \
        imp_cu_imgproc/CATKIN_IGNORE \
        imp_ros_rof_denoising/CATKIN_IGNORE \
        imp_tools_cmd/CATKIN_IGNORE \
        ze_data_provider/CATKIN_IGNORE \
        ze_geometry/CATKIN_IGNORE \
        ze_imu/CATKIN_IGNORE \
        ze_trajectory_analysis/CATKIN_IGNORE
  ```

  如果在这一步出现无法git clone的错误，请检查是否[正确配置git私钥](#配置ssh私钥)

* 开始编译

  ```bash
  catkin build esim_ros
  ```

* 编译成功后为模拟器配置环境变量

  ```bash
  echo "alias ssim='source ~/sim_ws/devel/setup.zsh'" >> ~/.zshrc
  source ~/.zshrc
  ```


## 测试模拟器

* 获取视频源

  这里给出一些常用的命令以帮助使用

  * 下载视频

    ```bash
    you-get -i 'https://youtube.com/watch?v=xxx'
    you-get -o /path/to/dir -O save_name --itag=18 'https://youtube.com/watch?v=xxx'
    ```

  * 剪辑视频

    ```bash
    ffmpeg -ss 00:00:00 -t 00:00:00 -i input_file output_file
    ```

  * 调整分辨率

    ```bash
    ffmpeg -i input_file -vf scale=640:-1 output_file
    ```

  * 调整帧率

    ```bash
    ffmpeg -i input_file -filter:v fps=fps=30 output_file
    ```

  * 拆分帧图像

    ```bash
    ffmpeg -i input_file out_dir/frames_%10d.jpg
    ```

* 运行模拟器

  ```bash
  #terminal 1
  roscore
  
  #terminal 2
  ssim
  roscd esim_ros
  python scripts/generate_stamps_file.py -i /path/to/frames -r 60
  
  rosrun esim_ros esim_node \
   --data_source=2 \
   --path_to_output_bag=/path/to/out.bag \
   --path_to_data_folder=/path/to/frames \
   --ros_publisher_frame_rate=60 \
   --exposure_time_ms=10.0 \
   --use_log_image=1 \
   --log_eps=0.1 \
   --contrast_threshold_pos=0.15 \
   --contrast_threshold_neg=0.15
  ```

  这里对一些用到的参数进行说明

  * `-r 60`：源视频帧率
  * `--ros_publisher_frame_rate=60`：输出DVS数据帧率
  * `--exposure_time_ms=10.0`：DVS曝光时间
  * `--contrast_threshold_*=0.15`：对比度阈值，阈值高对应灵敏度低事件数少

  注意这里输入的路径必须为**完整的绝对路径**，不可使用环境变量（如`~/`、`./`）

  运行示例

  ![](https://awesome-image.oss-cn-beijing.aliyuncs.com/20200414124624.png)

* 可视化

  ```bash
  #terminal 1
  roscore
  
  #terminal 2
  ssim
  rosrun dvs_renderer dvs_renderer events:=/cam0/events
  
  #terminal 3
  rosbag play /path/to/out.bag -l -r 0.1
  
  #terminal 4
  rqt_image_view /dvs_rendering
  ```

  `-r 0.1`为播放速率

  效果如下

  ![](https://awesome-image.oss-cn-beijing.aliyuncs.com/20200414124524.png)

## 数据解包

由于模拟器输出的文件格式为`rosbag`格式，需要特殊方式读出，这里使用`rospkg`自带的Python API解包

首先查看包中的信息

```
topics:      /cam0/camera_info        1 msg            : sensor_msgs/CameraInfo
             /cam0/events            49 msgs @ 20.0 Hz : dvs_msgs/EventArray   
             /cam0/image_corrupted   49 msgs @ 20.0 Hz : sensor_msgs/Image     
             /cam0/image_raw         50 msgs @ 20.0 Hz : sensor_msgs/Image
```

我们这里主要关注`/cam0/events`，查看其对应类型`dvs_msgs/EventArray`

```
std_msgs/Header header
  uint32 seq
  time stamp
  string frame_id
uint32 height
uint32 width
dvs_msgs/Event[] events
  uint16 x
  uint16 y
  time ts
  bool polarity
```

接下来将单msg信息整理至文本文件中，这里使用csv格式

注意rosbag包仅支持python2版本

```python
import os
import rosbag

def bag2txt(bag_path, out_path):
    b = rosbag.Bag(bag_path)
    for i, (topic, msgs, t) in enumerate(b.read_messages(topics=['/cam0/events'])):
        with open(os.path.join(out_path, 'dvs_%04d' % (i+1)), 'w') as f:
            f.write('%d,%d\n' % (msgs.width, msgs.height))
            for e in msgs.events:
                f.write('%d,%d,%d,%s\n' % (e.ts.nsecs, e.x, e.y, e.polarity))
    b.close()
```

其文件格式为

```
width, height
event1_t, event1_x, event1_y, event1_polarity
event2_t, event2_x, event2_y, event2_polarity
```

## 注意事项

* 尽量选择**高帧率**的视频，当采样频率大于源视频帧率时，模拟效果很差
* 尽量选择**无转场**的视频，当视频出现转场时，会出现全屏事件点影响数据

## 脚本

（由于本人的sh语法处于现学现用的状态，以下内容仅供参考

### run_sim.sh

```bash
#!/bin/zsh

source /opt/ros/kinetic/setup.zsh
source ~/setupeventsim.sh

# Pre treatment
echo "->Start Pre Treatment"
gnome-terminal -x zsh -c "source /opt/ros/kinetic/setup.zsh; source ~/sim_ws/devel/setup.zsh; roscd esim_ros; python scripts/generate_stamps_file.py -i $1 -r $2; exit"
echo "<-End Pre Treatment"

# Start roscore
echo "->Check Roscore"
ps_out=`ps -ef | grep roscore | grep -v 'grep'`
if [[ "$ps_out" != "" ]];then
	echo "<-Roscore Already Running"
else
	gnome-terminal -x sh -c "roscore; exec bash"
	echo "<-Start Roscore"
fi


# Waiting roscore to start
sleep 2s

# Start simulation
echo "->Start Simulation"
gnome-terminal -x zsh -c "rosrun esim_ros esim_node \
--data_source=2 \
--path_to_output_bag=$3 \
--path_to_data_folder=$1 \
--ros_publisher_frame_rate=60 \
--exposure_time_ms=10.0 \
--use_log_image=1 \
--log_eps=0.1 \
--contrast_threshold_pos=0.15 \
--contrast_threshold_neg=0.15; exit"
echo "Succeed! Roscore can be shut down after simulation exits."
```

使用方法

```bash
bash run_sim.sh /path/to/frames/folder 60[fps] /path/to/out.bag  
```

### run_view.sh

```bash
#!/bin/zsh

source /opt/ros/kinetic/setup.zsh
source ~/setupeventsim.sh

# Start roscore
echo "->Check Roscore"
ps_out=`ps -ef | grep roscore | grep -v 'grep'`
if [[ "$ps_out" != "" ]];then
	echo "<-Roscore Already Running"
else
	gnome-terminal -x sh -c "roscore; exec bash"
	echo "<-Start Roscore"
fi

# Start renderer
echo "->Start Renderer"
gnome-terminal -x zsh -c "source /opt/ros/kinetic/setup.zsh; source ~/sim_ws/devel/setup.zsh; rosrun dvs_renderer dvs_renderer events:=/cam0/events; exit"

# Start play
echo "->Start Playing $1 in rate $2"
gnome-terminal -x zsh -c "rosbag play $1 -l -r $2"

# Start player
echo "->Start Player Window"
gnome-terminal -x zsh -c "rqt_image_view /dvs_rendering"

echo "Done!"
```

使用方法

```bash
bash run_view.sh /path/to/out.bag 0.1[rate]
```

## 配置ssh私钥

参考[Git生成SSH公钥](https://git-scm.com/book/zh/v2/服务器上的-Git-生成-SSH-公钥)

由于vcs默认以ssh方式访问github repo，因此需要私钥以获得git@github.com访问权

```bash
ssh-keygen -t rsa -C "email or username"
ssh-add ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub
```

将得到的公钥添加至github即可

