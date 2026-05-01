# IndoorRobotSimulation — ROS Noetic 迁移版启动指南

**适用环境：** Ubuntu 20.04 LTS + ROS Noetic + Python 3.8 + Turtlebot3 Waffle  
**项目定位：** 室内移动机器人仿真，支持 RRT 探索建图、Hector SLAM 手动建图、定点导航、固定线路巡航等多种功能。

> ⚠️ **说明：** 本文件是迁移后的新版本启动指南。原 `README.md` 为 Kinetic + Turtlebot2 版本，命令路径不适用于当前环境。请以本文件为准。

---

## 目录

- [1. 环境准备](#1-环境准备)
- [2. 仿真环境启动](#2-仿真环境启动)
- [3. SLAM 建图](#3-slam-建图)
  - [3.1 RRT 自动探索建图](#31-rrt-自动探索建图)
  - [3.2 Hector SLAM 手动建图](#32-hector-slam-手动建图)
- [4. 导航与巡航](#4-导航与巡航)
  - [4.1 定点导航（单一目标点）](#41-定点导航单一目标点)
  - [4.2 固定线路巡航（多目标点联动）](#42-固定线路巡航多目标点联动)
- [5. 全局路径规划算法验证](#5-全局路径规划算法验证)
- [6. 从零开始完整流程](#6-从零开始完整流程)
- [常见问题](#常见问题)

---

## 1. 环境准备

### 1.1 设置 Turtlebot3 型号和环境变量

```bash
# 必须设置：Turtlebot3 型号为 Waffle（同时拥有激光雷达和摄像头）
export TURTLEBOT3_MODEL=waffle

# 可选：设置默认世界文件路径
export TURTLEBOT3_GAZEBO_WORLD_FILE=/home/hamker/catkin_ws1/IndoorRobotSimulation/gazebo_worlds/square_hall.world

# 建议将以上两行添加到 ~/.bashrc，避免每次打开终端时重复设置
echo 'export TURTLEBOT3_MODEL=waffle' >> ~/.bashrc
echo 'export TURTLEBOT3_GAZEBO_WORLD_FILE=/home/hamker/catkin_ws1/IndoorRobotSimulation/gazebo_worlds/square_hall.world' >> ~/.bashrc
source ~/.bashrc
```

### 1.2 刷新 ROS 工作空间环境

```bash
source /home/hamker/catkin_ws1/IndoorRobotSimulation/devel/setup.bash

# 建议添加到 ~/.bashrc
echo 'source /home/hamker/catkin_ws1/IndoorRobotSimulation/devel/setup.bash' >> ~/.bashrc
source ~/.bashrc
```

### 1.3 启动 roscore（所有后续操作都需要 roscore 在运行）

```bash
# 在单独的终端中启动，保持运行
roscore
```

---

## 2. 仿真环境启动

### 2.1 启动 Gazebo 仿真 + Turtlebot3 Waffle 机器人

**目标：** 启动 Gazebo 仿真环境，加载室内走廊地图 `square_hall.world`，并在其中生成 Turtlebot3 Waffle 机器人。

```bash
# 终端 1：确保环境变量已设置
export TURTLEBOT3_MODEL=waffle

# 使用项目自定义的启动文件（自动加载 square_hall.world + 生成机器人）
roslaunch robot_remould turtlebot_world.launch
```

> **验证方法：** Gazebo 窗口应显示室内走廊场景，Turtlebot3 Waffle 机器人出现在走廊中。

---

## 3. SLAM 建图

### 3.1 RRT 自动探索建图

**目标：** 使用基于快速扩展随机树（RRT）的自主探索算法，让机器人在未知环境中自动移动并完成地图构建。

**说明：** 此功能由 `rrt_exploration` 和 `rrt_exploration_tutorials` 两个包共同完成。

#### 步骤 1：启动 Gazebo 仿真（单机器人模式）

```bash
# 终端 2：启动 RRT 探索专用的单机器人仿真
roslaunch rrt_exploration_tutorials single_simulated_square_hall.launch
```

> 此命令将：
> 1. 启动 Gazebo（加载 square_hall.world）
> 2. 生成 Turtlebot3 Waffle 机器人
> 3. 启动 gmapping SLAM 节点
> 4. 启动 move_base 导航节点
> 5. 打开 RViz 可视化界面

#### 步骤 2：启动探索控制节点

```bash
# 终端 3：启动 RRT 探索节点
roslaunch rrt_exploration single.launch
```

#### 步骤 3：启动键盘遥控（可选）

```bash
# 终端 4：Turtlebot3 键盘遥控
roslaunch turtlebot3_teleop turtlebot3_teleop_key.launch
```

---

### 3.2 Hector SLAM 手动建图

**目标：** 使用 Hector SLAM 算法（不需要里程计信息），通过键盘遥控机器人手动完成地图构建。

#### 步骤 1：启动 Gazebo 仿真

```bash
roslaunch robot_remould turtlebot_world.launch
```

#### 步骤 2：启动 Hector SLAM 建图

```bash
roslaunch rplidar_ros hector_mapping_demo.launch
```

#### 步骤 3：键盘遥控建图

```bash
roslaunch turtlebot3_teleop turtlebot3_teleop_key.launch
```

#### 步骤 4：保存建图结果

```bash
rosrun map_server map_saver -f /home/hamker/catkin_ws1/IndoorRobotSimulation/ros_maps/hector_slam_map
```

---

## 4. 导航与巡航

### 4.1 定点导航（单一目标点）

```bash
# 终端 1：Gazebo 仿真
roslaunch robot_remould turtlebot_world.launch

# 终端 2：AMCL + move_base 导航
roslaunch turtlebot3_navigation turtlebot3_navigation.launch map_file:=/home/hamker/catkin_ws1/IndoorRobotSimulation/ros_maps/square_hall.yaml

# 终端 3：运行定点导航脚本
rosrun scripts go_to_specific_point_on_map.py
```

### 4.2 固定线路巡航（多目标点联动）

```bash
# 终端 1：Gazebo 仿真
roslaunch robot_remould turtlebot_world.launch

# 终端 2：AMCL + move_base 导航
roslaunch turtlebot3_navigation turtlebot3_navigation.launch map_file:=/home/hamker/catkin_ws1/IndoorRobotSimulation/ros_maps/square_hall.yaml

# 终端 3：运行线路巡航脚本
rosrun scripts follow_the_route.py
```

---

## 5. 全局路径规划算法验证

### 算法列表

| 算法 | 包名 | 特性 |
|------|------|------|
| NavfnROS | `navfn` | 默认算法，基于 Dijkstra/NavFn |
| GlobalPlanner | `global_planner` | 替代 Navfn，支持 A* |
| CarrotPlanner | `carrot_planner` | 简单直线规划 |
| DStarPlannerROS | `dstar` | D* Lite 算法 |
| RAstarPlannerROS | `relaxed_astar` | 改进的 A* |
| rrt_planner | `rrt_planner` | RRT 随机采样规划 |

```bash
# 切换算法示例
rosparam set /move_base_node/GlobalPlanner/planner_type "NavfnROS"
rosparam set /move_base_node/base_global_planner "navfn/NavfnROS"
```

---

## 6. 从零开始完整流程

```bash
# 首次使用：重新编译工作空间
cd /home/hamker/catkin_ws1/IndoorRobotSimulation
catkin_make

# 步骤 1：设置环境变量
export TURTLEBOT3_MODEL=waffle
source /home/hamker/catkin_ws1/IndoorRobotSimulation/devel/setup.bash

# 步骤 2：启动 roscore
roscore

# 步骤 3（新终端）：启动 Gazebo 仿真
source /home/hamker/catkin_ws1/IndoorRobotSimulation/devel/setup.bash
export TURTLEBOT3_MODEL=waffle
roslaunch robot_remould turtlebot_world.launch

# 可选：验证话题
rostopic list
# 应看到 /scan, /odom, /cmd_vel, /camera/rgb/image_raw 等话题
```

---

## 常见问题

### Q1: Gazebo 启动后黑屏或加载缓慢
- **原因：** 首次启动需要下载模型资源
- **解决：** 等待自动下载完成，或使用 `GAZEBO_MODEL_PATH` 指定本地模型路径

### Q2: RViz 中看不到激光雷达数据
- **原因：** Turtlebot3 Waffle 的话题是 `/scan`
- **解决：** RViz 中 LaserScan 的 Topic 设置为 `/scan`

### Q3: AMCL 定位不准确
- **原因：** 机器人初始位姿未知
- **解决：** 在 RViz 中使用 "2D Pose Estimate" 工具手动标注

### Q4: `roslaunch robot_remould turtlebot_world.launch` → RLException 或场景加载失败（只有空世界）
- **原因（RLException）：** `robot_remould` 原本不在 `src/` 目录下，catkin 找不到该包
- **解决：** 已将其移动到 `src/robot_remould/` 并添加了 `CMakeLists.txt` 和 `package.xml`，重新 `catkin_make` 后即可
- **原因（空世界、无走廊场景）：** 包移到 `src/robot_remould/` 后，launch 文件中的 `$(find robot_remould)/../gazebo_worlds/` 只上了一层到 `src/gazebo_worlds/`（不存在），实际路径在 `IndoorRobotSimulation/gazebo_worlds/`
- **解决：** 已将路径改为 `../../gazebo_worlds/`（上两层），指向正确的 `IndoorRobotSimulation/gazebo_worlds/square_hall.world`

### Q5: xacro 报错 `No module named 'rospkg'`
- **原因：** Python virtualenv（`.venv`）隔离了系统 Python 包，导致 xacro 子进程找不到 rospkg、defusedxml、numpy 等
- **解决：** 修改 `.venv/pyvenv.cfg`，将 `include-system-site-packages` 改为 `true`；或直接使用系统 Python（deactivate 退出 venv）

### Q6: 编译失败 — `pgm.h: No such file or directory`
- **原因：** map_server 需要 netpbm 库的头文件
- **解决：** `sudo apt install -y libnetpbm10-dev` 后重新 `catkin_make`

### Q7: 编译失败 — robot_pose_ekf 或 hector_geotiff
- **原因：** 旧版 Turtlebot2/hector_slam 的可选组件，在 Noetic 中因缺少 Qt4/bfl 依赖而无法编译
- **解决：** 已在源码中标记 `CATKIN_IGNORE`，不影响核心功能

### Q8: roslaunch 执行后显示 "Terminated"
- **原因：** 命令在 timeout 结束时自动终止，并非错误。Gazebo 已成功启动并一直在运行
- **解决：** 将 GUI 模式打开（不传 `gui:=false`），在 Gazebo 窗口中验证机器人是否出现

### Q9: Python 脚本找不到 rospkg/catkin_pkg 等模块
- **原因：** `pip install` 安装到 `~/.local` 而非 venv 内
- **解决：** 已修改 `pyvenv.cfg` 包含系统包，或运行 `deactivate` 退出 venv 环境后执行
