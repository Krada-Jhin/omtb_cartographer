# 使用说明

该包为gmapping各种参数和地图更换测试使用

### launch文件夹

totalmove_gmapping为配置好的launch文件,直接roslaunch运行即可实现启动gazebo,gmapping,rviz,automove等初始配置

```
roslaunch omtb_test totalmove_gmapping_room1.launch 
```

#### 保存map格式文件

不同地图需修改map的信息名称

```
rosrun map_server map_saver map:=/robot1/map -f ~/catkin_slam/src/omtb/omtb_test/map/mapname
```

#### cartographer保存map文件

```
rosservice call /robot1/finish_trajectory 0
rosservice call /robot1/write_state "/home/sparta/Downloads/mymap.pbstream"
rosrun cartographer_ros cartographer_pbstream_to_ros_map -map_filestem=/home/sparta/Downloads/mymap -pbstream_filename=/home/sparta/Downloads/mymap.pbstream -resolution=0.05
```

注意不同电脑上的存储路径不一样

#### rosbag文件记录

在SLAM过程中记录文件

```
rosbag record -a   //目前如果在launch文件中引用那么会报错直接exit,
rosbag record ~/catkin_slam/src/omtb/omtb_test/bag/ /om_with_tb3/scan /robot1/map /robot1/scan /robot1/odom

```

记录文件后重放

```
roslaunch omtb_slam2d slam.launch  //已经将slam.launch修改为默认启动gmapping
rosbag play bagname.bag     //在omtb_test/bag内打开终端,或者填写完整地址
```

## 建立cartographer3D的安装配置(ongoing 12.3)不能加载robotmodel,建图drift严重

### 所需依赖

首先现有imu会出现以下的报错

```
[FATAL] [1575873053.434009550, 532.204000000]: F1209 14:30:53.000000 21726 imu_tracker.cc:66] Check failed: (orientation_ * gravity_vector_).z() > 0. (-nan vs. 0) 
```

目前的解决方案是使用一个node对输入的imu数据进行一个转换

```
git clone https://github.com/googlecartographer/cartographer_turtlebot.git
cd /workspacename  到工作区间
catkin_make -DCATKIN_WHITELIST_PACKAGES="cartographer_turtlebot" 编译后才可使用flat_world_imu_node
```

之后修改 omtb_slam2d 中launch文件.来启动3Dslam的lua:robot1_vlp16.lua

```
slam.launch  -- slam_mothod为 cartographer_vlp16
cartographer_vlp16.launch   --添加/替换以下内容

 <!-- cartographer_node -->
  <node pkg="cartographer_ros" type="cartographer_node" name="cartographer_node"
        args="-configuration_directory $(find omtb_slam2d)/config
              -configuration_basename $(arg configuration_basename)"
        output="screen">
   <remap from="/robot1/imu" to="/imu" />
  </node>
  <node name="flat_world_imu_node" pkg="cartographer_turtlebot"
      type="cartographer_flat_world_imu_node" output="screen">
    <remap from="imu_in" to="/robot1/imu" />
    <remap from="imu_out" to="/imu" />
  </node> <!--使imu输出的数据进行一个转化,改为定值 -->
```

其中cartographer_node添加了remap,新增一个node.

```
rqt_graph    --在程序运行时观测 flat_world_imu_node 是否输出到cartographer_ros
```

robot1_vlp16.lua主要参数配置

```
map_frame = "robot1/map",
tracking_frame = "robot1/imu_link",
published_frame = "robot1/base_link",
odom_frame = "robot1/odom",
TRAJECTORY_BUILDER_3D.num_accumulated_range_data = 1
```

统一运行测试,建立一个launch文件 ,slam.launch中已经设置为3D故可以测试,文件结构如下

```
<launch>
  <include file="$(find omtb_gazebo)/launch/1tb_room1.launch">
  </include>
  <include file="$(find omtb_slam2d)/launch/slam.launch">
  </include>
    <include file="$(find omtb_control)/launch/automove_1tb_room1.launch">
  </include>
</launch>
```

