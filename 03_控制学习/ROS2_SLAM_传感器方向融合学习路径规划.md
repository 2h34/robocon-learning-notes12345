# ROS2 + SLAM + 传感器方向融合学习路径规划

> 用途：用于后续开启新对话时快速恢复学习上下文，也作为未来可能进入 Robocon 传感器组后的长期学习路线参考。  
> 当前背景：已经具备 STM32、CAN、FreeRTOS、电机控制、Mechanism / Task / Operator、基础坐标变换、IMU / odom / LiDAR 概念等基础；正在考虑未来加入传感器组，并已拿到学长推荐的一套 SLAM B站课程。  
> 学习原则：**不是“先学完 ROS2，再学 SLAM”，也不是两门课交替刷视频，而是围绕“机器人怎样从真实传感器数据得到可靠状态与地图”这一主线，把 ROS2 工程能力与 SLAM / 状态估计理论同步学习。**

---

# 一、当前已有基础

## 1. STM32 / 嵌入式

已经掌握或基本掌握：

- STM32 GPIO / EXTI / Timer / PWM
- Clock
- IRQ / NVIC / ISR / HAL Callback
- UART / UART Interrupt
- DMA / ReceiveToIdle DMA
- CAN
- FreeRTOS Task / Queue
- 状态机
- 非阻塞程序
- `.c / .h` 模块化
- Git / GitHub

## 2. 电机与机器人控制

已经学习：

- 电机 / 驱动器 / 编码器 / 减速器
- 位置 / 速度 / 电流控制
- PID / 串级 PID
- DJI / ZDrive
- Homing / Zero
- 到位判断
- Fault 基本思想
- 双机械臂状态机
- 主从板 CAN 通信

已经形成软件架构：

```text
Operator
↓
Task
↓
Mechanism
↓
Motor
↓
Adapter
↓
Driver
↓
Hardware
```

因此 ROS2 中出现 Node、Topic、Callback、Action、Command / Feedback 时，可以与已有机器人软件经验建立联系，但不能把两者机制简单等同。

---

# 二、已有的传感器 / 数学基础

已经接触或基本学过：

- Frame
- Pose
- Position / Orientation
- Rotation Matrix
- Direction Cosine Matrix
- Homogeneous Transformation
- 坐标映射
- 固定轴 / 当前轴旋转
- K4A 外参
- IMU 基础
- Encoder / Odometry 基础
- LiDAR 基础
- Point Cloud 基础
- `map / odom / base_link / sensor_link`
- TF Tree 基本概念
- FAST-LIO2 基本系统链路

因此以后学习 TF2 时，不重新从线性代数和齐次变换开始，只做必要唤醒，然后直接进入 ROS2 工程实现。

---

# 三、目前明确缺少的新知识

## 近期核心新知识

```text
ROS2工程环境
Linux基础
C++够用知识
Node / Topic / Message
TF2
rosbag2
RViz2

Quaternion
Sensor Noise / Bias / Drift
Timestamp
Calibration
Synchronization

Bayesian Estimation
Kalman Filter
EKF

LiDAR Odometry
ICP
LIO
FAST-LIO2
```

## 中后期再学

```text
Graph Optimization
Loop Closure 深入
Lie Group / Lie Algebra
Jacobian
Factor Graph
LIO-SAM
VIO / LIVO
Nav2
MoveIt2
Deep Learning + SLAM
```

---

# 四、总体学习目标

最终不是做到：

```text
会 ros2 launch
会跑别人 SLAM
RViz 能出图
```

而是做到：

```text
Sensor
↓
Driver
↓
ROS2 Message / Topic
↓
Timestamp / Frame / Calibration
↓
Preprocessing
↓
Odometry / State Estimation
↓
Sensor Fusion
↓
SLAM
↓
Pose / Map
↓
Controller
↓
STM32
↓
Actuator
```

并能够解释：

- 数据来自哪里；
- 谁发布、谁订阅；
- 坐标系怎么变换；
- 时间戳为什么重要；
- 传感器误差怎么进入估计；
- 状态估计和 SLAM 分别解决什么；
- ROS2 与 STM32 如何分工；
- 为什么系统跑歪时不一定是算法本身错误。

---

# 五、总体融合路线

```text
阶段 0
SLAM系统总览
+
ROS2系统总览
        ↓
阶段 1
ROS2 Node / Topic / Message
+
Linux / C++最低前置
        ↓
阶段 2
刚体变换快速复习
+
TF2 / RViz2
        ↓
阶段 3
IMU / LiDAR / Camera / Encoder
+
ROS2 Sensor Message
+
rosbag2
        ↓
阶段 4
Noise / Bias / Drift
+
Bayes / KF / EKF
+
Quaternion
        ↓
阶段 5
IMU + Encoder
状态估计实践
        ↓
阶段 6
Visual Odometry（了解）
+
LiDAR Odometry（重点）
+
PointCloud / ICP
        ↓
阶段 7
Loop Closure
+
Graph Optimization
        ↓
阶段 8
LiDAR + IMU
→ LIO
        ↓
阶段 9
FAST-LIO2
        ↓
阶段 10
地图 / 视觉 / Deep Learning 扩展
        ↓
后续
Nav2 / VIO / LIVO / LIO-SAM
+
ros2_control / ROS2 ↔ STM32
```

---

# 六、阶段 0：建立机器人感知与 SLAM 全景

## 核心问题

先回答：

```text
机器人为什么需要定位？
Odometry 为什么会漂？
Localization 和 Mapping 是什么？
SLAM 为什么存在？
Sensor → Estimator → Pose 是什么链路？
Front End / Back End 分别负责什么？
Loop Closure 为什么重要？
```

## SLAM视频

使用学长推荐合集：

### 第 1 课：SLAM 概述和架构

第一次观看只要求理解：

```text
Localization
Mapping
SLAM
Odometry
Drift
Prediction
Observation
Front End
Back End
Loop Closure
Map
```

遇到 EKF、Bayes、Graph Optimization、Essential Matrix、Fundamental Matrix，只记录“它是什么、解决什么问题、后面在哪个阶段正式学习”，暂时不推导。

## ROS2同步学习

同时看 ROS2 概念视频：

- ROS / ROS2 是什么
- Node
- Topic
- Message
- Publisher
- Subscriber

推荐：

- 鱼香ROS：`https://www.bilibili.com/video/BV1GW42197Ck/`
- 古月居 ROS2 入门21讲：`https://www.bilibili.com/video/BV16B4y1Q7jQ/`

阶段成果：

```text
IMU Node ───── /imu/data ─────┐
                              │
LiDAR Node ─── /points ───────┼→ SLAM Node
                              │
Encoder Node ─ /odom ─────────┘
                               ↓
                         Pose / Map
```

并能够解释：为什么 SLAM 系统天然适合被拆成多个 ROS2 Node。

---

# 七、阶段 1：把 ROS2 真正跑起来

## 环境

正式进入这一阶段后再配置。

优先级：

```text
校队统一环境
>
ROS2官方当前推荐环境
>
视频中的环境
```

建议准备：

```text
Ubuntu
ROS2
VS Code
Git
gcc / g++
cmake
colcon
rosdep
Python3
```

## Linux只补够用知识

```text
pwd
ls
cd
mkdir
rm
cp
mv
sudo
apt
source
环境变量
.bashrc
路径
```

## C++只补 ROS2 需要的内容

```text
struct → class
函数 → member function
Init() → constructor
函数指针 → std::function / lambda
模块作用域 → namespace
裸指针 → smart pointer
```

重点掌握：

```text
class
constructor
namespace
reference
const
std::shared_ptr
lambda
```

目标是能读懂基础 `rclcpp::Node`。

## ROS2最小实践

```text
Workspace
↓
Package
↓
Node A
↓
Topic
↓
Node B
```

会使用：

```bash
ros2 node list
ros2 topic list
ros2 topic echo
ros2 topic info
```

并用 `rqt_graph` 观察 Node 关系。

---

# 八、GitHub源码学习节点 1：ros2/examples

官方仓库：

```text
https://github.com/ros2/examples
```

优先阅读：

```text
rclcpp/topics
rclcpp/services
rclcpp/actions
rclcpp/timers
```

第一阶段主要先读 `rclcpp/topics`。

固定阅读顺序：

```text
README
↓
package.xml
↓
CMakeLists.txt
↓
main()
↓
Node class
↓
Publisher / Subscriber
↓
Message
↓
Callback
↓
运行
↓
修改一个行为
↓
预测
↓
验证
```

---

# 九、阶段 2：坐标系理论与 TF2 合流

## SLAM视频

### 第 2 课：SLAM基本理论一

重点是坐标系、刚体运动。

已有：

```text
Frame
Pose
R
T
齐次变换
坐标映射
外参
```

因此采用快速复习 + 只补新内容。

如果出现：

```text
Quaternion
SO(3)
SE(3)
```

则 Quaternion 后面正式学习；SO(3) / SE(3) 先建立概念，不深入李群数学。

## ROS2同步内容：TF2

学习：

```text
Transform
Static Transform
Dynamic Transform
Broadcaster
Listener
```

把已有数学变成：

```text
map
↓
odom
↓
base_link
├─ imu_link
├─ lidar_link
└─ camera_link
```

同时开始 RViz2，观察 TF 坐标轴、sensor frame、base_link、PointCloud、Pose、Path。

阶段目标：

> 不再只是会计算变换矩阵，而是能在 ROS2 系统中维护和检查真实坐标关系。

---

# 十、阶段 3：真实传感器数据工程

## SLAM视频

把原课程第 5 课提前：

### 第 5 课：SLAM 的传感器

重点学习：

```text
IMU
Encoder
LiDAR
Mono Camera
Stereo Camera
RGB-D
```

## 学习深度升级

不再只回答“这个传感器测什么”，还要理解：

```text
Sampling Rate
Timestamp
Latency
Noise
Bias
Drift
Intrinsic
Extrinsic
Synchronization
Unit
Frame
```

建立重要工程认知：

```text
SLAM结果异常
≠
算法一定错了
```

还可能是：

```text
frame错
timestamp错
extrinsic错
frequency错
unit错
driver错
同步错
```

## ROS2同步内容

学习：

```text
sensor_msgs/Imu
sensor_msgs/PointCloud2
nav_msgs/Odometry
```

重点理解：

```text
Header
stamp
frame_id
```

## rosbag2提前学习

学习：

```text
record
play
info
```

建立典型工作流：

```text
机器人采数据
↓
rosbag2 record
↓
回电脑
↓
rosbag2 play
↓
反复调算法 / 参数
↓
验证
```

---

# 十一、阶段 4：为什么传感器不能直接相信

## SLAM视频

### 第 3 课：从贝叶斯开始学滤波

第一块需要放慢的新理论。

顺序：

```text
Random Variable
↓
Probability
↓
Gaussian
↓
Mean
↓
Variance
↓
Covariance
↓
Prior
↓
Prediction
↓
Measurement
↓
Posterior
↓
Bayesian Estimation
↓
Kalman Filter
↓
EKF
```

第一阶段重点理解：

```text
Prediction：
根据运动模型预测当前状态

Observation：
传感器提供外界约束

Fusion：
综合两者的不确定性得到更可信状态
```

---

# 十二、Quaternion 插入点

在正式处理 IMU 时学习。

顺序：

```text
Euler Angle
↓
Quaternion
↓
Orientation
↓
IMU姿态
```

第一阶段目标：

- 知道四元数表示什么；
- 知道为什么机器人系统常使用四元数；
- 知道 x / y / z / w 是什么；
- 知道 ROS2 orientation 为什么使用 Quaternion。

暂时不深入复杂四元数微分推导。

---

# 十三、阶段 5：第一次真正做状态估计

不要直接进入 FAST-LIO2。

先完成：

```text
IMU
+
Encoder Odometry
↓
State Estimator
↓
Filtered Odometry
```

理论：

```text
Prediction
+
Measurement Update
↓
KF / EKF
```

ROS2实践：

```text
/imu/data
+
/wheel/odom
↓
Estimator
↓
/odometry/filtered
```

可引入 `robot_localization`，但流程必须是：

```text
理解简单一维 KF
↓
理解 Prediction / Update
↓
理解状态和协方差
↓
再使用现成 ROS2 estimator
```

---

# 十四、阶段 6：Visual Odometry 与 LiDAR Odometry

## 第 6 课：视觉里程计和回环检测

定位：了解为主。

认识：

```text
Feature
Feature Matching
Triangulation
Epipolar Geometry
Essential Matrix
PnP
Reprojection Error
```

暂时不完整推导。

## 第 7 课：激光里程计和回环检测

定位：重点学习。

路线：

```text
LiDAR
↓
PointCloud
↓
两帧点云
↓
Registration
↓
Relative Pose
↓
LiDAR Odometry
```

正式进入：

```text
ICP
Point-to-Point
Point-to-Plane
Scan Matching
```

ROS2同步实践：

```text
PointCloud2
TF2
RViz2
rosbag2
```

检查点云方向、frame、timestamp、frequency 和运动是否合理。

---

# 十五、阶段 7：Loop Closure 与 Graph Optimization

到这里再回到：

### 第 4 课：图优化

此时已经理解：

```text
Pose
Odometry
Drift
Observation
Loop Closure
```

因此再学习：

```text
Node
Edge
Residual
Constraint
Optimization
Pose Graph
Least Squares
```

第一轮只建立：

- 为什么多个局部约束之间会互相矛盾；
- 为什么需要整体优化；
- Loop Constraint 怎样修正长期漂移。

暂时只认识：

```text
Gauss-Newton
Levenberg-Marquardt
Jacobian
Hessian
```

不深入数学推导。

---

# 十六、阶段 8：LiDAR + IMU → LIO

正式进入：

**LIO = LiDAR-Inertial Odometry，激光雷达惯性里程计**

模型：

```text
IMU
↓
高频 Prediction
                           → State Estimator → Pose
         /
LiDAR
↓
Geometry Observation
```

重点理解：

- 为什么 LiDAR 和 IMU 互补；
- IMU 提供什么；
- LiDAR 提供什么；
- 为什么时间同步重要；
- 为什么外参重要；
- 为什么单独 LiDAR / 单独 IMU 都有局限。

---

# 十七、阶段 9：FAST-LIO2

到这里才正式进入 FAST-LIO2。

前置应该已经具备：

```text
ROS2
TF2
rosbag2
RViz2
LiDAR
IMU
Quaternion
Timestamp
Calibration
Noise / Bias
EKF思想
PointCloud
LiDAR Odometry
```

第一遍先看系统层：

```text
Input
├─ LiDAR
└─ IMU
    ↓
FAST-LIO2
    ↓
Output
├─ Pose
├─ Odometry
└─ Map
```

然后阅读：

```text
Node
Topic
Parameter
TF
Extrinsic
Synchronization
数据流
```

第二遍再进入算法层：

```text
Iterated EKF
Jacobian
Point Registration
Map Structure
```

---

# 十八、阶段 10：地图、应用和深度学习扩展

## 第 8 课：地图以及无人驾驶系统

理解：

```text
PointCloud Map
Occupancy Map
Semantic Map
HD Map
```

## 第 9 课：视觉 / 无人机 / 室内导航

定位：应用扩展，根据未来队内任务选择性学习。

## 第 10 课：Deep Learning + SLAM

最后再看。

前提是已经理解：

```text
Front End
Back End
Loop Closure
Map
State Estimation
```

---

# 十九、ROS2后续暂缓内容

并不是不学，而是暂时不作为传感器主线优先级：

```text
Service
Action
URDF
ros2_control
ROS2 ↔ STM32
```

以后按需求补。

最终形成：

```text
LiDAR / IMU
↓
ROS2
↓
Estimator / SLAM
↓
Robot Pose
↓
Planner / Controller
↓
CAN / UART
↓
STM32
↓
Task / Mechanism
↓
Actuator
```

---

# 二十、B站视频的使用方式

ROS2 和 SLAM 都配合视频学习，但视频不是主线。

固定流程：

```text
① 明确当前要解决的问题
↓
② AI先结合已有知识讲核心机制
↓
③ 指定 ROS2 / SLAM 视频中的对应内容
↓
④ 用户观看
↓
⑤ 必要时导出 .srt
↓
⑥ AI基于真实字幕核对课程内容
↓
⑦ 用户复述系统模型
↓
⑧ 最小实践
↓
⑨ GitHub源码阅读
↓
⑩ 用户自己分析
↓
⑪ 预测
↓
⑫ 运行
↓
⑬ 验证
```

---

# 二十一、B站视频资料分工

## ROS2实践主课

鱼香ROS：

```text
https://www.bilibili.com/video/BV1GW42197Ck/
```

主要承担：

```text
Linux
C++
Workspace
Package
Node
Topic
TF
RViz
rosbag
ros2_control
```

## ROS2概念辅助

古月居 ROS2 入门21讲：

```text
https://www.bilibili.com/video/BV16B4y1Q7jQ/
```

主要用于：

```text
Node
Topic
Service
Action
TF
URDF
RViz
```

快速建立第一版概念。

## SLAM主课

学长推荐的 B站 SLAM 合集。

当前已知课程主体：

```text
1. SLAM概述和架构
2. SLAM基本理论一：坐标系、刚体运动
3. SLAM基本理论二：从贝叶斯开始学滤波
4. SLAM基本理论三：图优化
5. SLAM的传感器
6. 视觉里程计和回路检测
7. 激光里程计和回路检测
8. 地图以及无人驾驶系统
9. 视觉和无人机、室内辅助导航等
10. 深度学习和SLAM
```

推荐实际顺序：

```text
1 → 2 → 5 → 3 → 6 → 7 → 4 → 8 → 9 → 10
```

中间穿插：

```text
ROS2
TF2
rosbag2
RViz2
Quaternion
EKF实践
LIO
FAST-LIO2
```

---

# 二十二、GitHub学习路线

## 第一层：ROS2官方最小示例

```text
https://github.com/ros2/examples
```

目标：学会 ROS2 API 和 Package 组织。

## 第二层：状态估计

后续进入：

```text
robot_localization
```

目标：

```text
IMU
+
Odometry
↓
State Estimation
```

## 第三层：FAST-LIO2

目标：

```text
LiDAR
+
IMU
↓
LIO
```

先系统，后源码，最后数学。

## 第四层：后续扩展

根据需求再看：

```text
LIO-SAM
VINS-Fusion
FAST-LIVO2
Nav2
MoveIt2
```

不是当前必修。

---

# 二十三、每个 GitHub 项目的固定阅读顺序

```text
项目目标
↓
输入
↓
输出
↓
数据流
↓
Node
↓
Topic / Message
↓
Parameter
↓
TF
↓
Package
↓
关键类
↓
Callback
↓
运行入口
↓
核心算法
```

必须先理解整个系统在做什么，再看具体函数怎么写。

---

# 二十四、未来传感器组能力树

```text
                    传感器组
                       │
        ┌──────────────┼─────────────┐
        ↓              ↓             ↓
      工程能力        数学能力       算法能力
        │              │             │
      ROS2           线代           Filtering
      Linux          概率           KF / EKF
      C++            旋转           Optimization
      TF2            Quaternion     Odometry
      rosbag2        Covariance      SLAM
      RViz2          Least Squares  LIO
      Driver         Jacobian       Loop Closure
      Calibration
      Synchronization
```

再叠加已有：

```text
STM32
CAN
FreeRTOS
Motor
Control
```

最终形成优势方向：

```text
Sensor
↓
ROS2
↓
State Estimation / SLAM
↓
Control
↓
STM32
↓
Actuator
```

---

# 二十五、当前阶段优先级

当前不要同时大规模推进所有内容。

正式主线：

```text
→ 当前：
SLAM 第1课总览
+
ROS2 系统总览

下一步：
ROS2 Node / Topic / Message

再下一步：
SLAM 第2课
+
TF2

之后：
SLAM 第5课 Sensors
+
ROS2 Sensor Message / rosbag2
```

等这些完成后再进入：

```text
Bayes / EKF
Quaternion
State Estimation
LiDAR Odometry
LIO
FAST-LIO2
```

---

# 二十六、当前阶段不学

暂时不要深入：

```text
Nav2
MoveIt2
复杂 Gazebo
FAST-LIO2核心数学源码
LIO-SAM因子图源码
VINS-Fusion
GTSAM
Ceres深入
Sophus深入
完整李群理论
复杂视觉SLAM
Deep Learning SLAM
```

---

# 二十七、新对话启动提示词

以后新开对话可以直接使用：

```markdown
我现在继续进行 ROS2 + SLAM + 传感器方向融合学习。

请优先读取：

1. 我的学习笔记仓库：
   https://github.com/2h34/robocon-learning-notes12345

2. 《ROS2_SLAM_传感器方向融合学习路径规划.md》

3. 如果当前阶段涉及学长推荐的 SLAM 视频，我会提供对应 B站链接或 `.srt` 字幕。

请严格按照 RC学习 项目的学习 Skill 推进。

不要把我当成机器人零基础。

我已经掌握或基本掌握：

- STM32
- UART / CAN
- FreeRTOS
- PID
- DJI / ZDrive
- Motor Adapter
- Mechanism
- Task
- Operator
- 状态机
- Frame / Pose
- Rotation Matrix
- Homogeneous Transformation
- IMU / odom / LiDAR 基础
- TF Tree 基本概念

当前学习目标不是分别学完 ROS2 和 SLAM 两门课，而是围绕：

“机器人怎样从真实传感器数据，最终得到可靠的 Pose / Odometry / Map？”

把 ROS2 工程知识和 SLAM / 状态估计理论同步学习。

每次开始前请先判断：

- 已掌握
- 需要唤醒
- 新知识

学习流程：

核心机制
→ 与已有知识建立联系
→ ROS2 / SLAM B站视频对应节点
→ 必要时读取 `.srt`
→ 讨论
→ 用户复述系统模型
→ 最小实践
→ GitHub源码
→ 用户自己分析
→ 预测
→ 运行
→ 验证

不要一次推进过多。

如果没有其他进度说明，则从当前真正停止的位置继续。
```

---

# 二十八、当前进度

```text
✓ STM32 / Embedded C
✓ UART / CAN
✓ FreeRTOS
✓ PID / DJI / ZDrive
✓ Motor / Adapter
✓ Mechanism / Task / Operator
✓ 坐标变换
✓ IMU / odom / LiDAR 基础
✓ TF Tree 基本概念
✓ SLAM第1课字幕已能通过 .srt 读取

→ 当前阶段：
SLAM 系统总览
+
ROS2 系统总览

□ ROS2 Node / Topic / Message
□ Linux / C++最低前置
□ Workspace / Package
□ TF2
□ RViz2
□ Sensor Message
□ rosbag2
□ Sensor Noise / Bias / Drift
□ Quaternion
□ Bayes / KF / EKF
□ IMU + Encoder State Estimation
□ Visual Odometry
□ LiDAR Odometry
□ ICP
□ Loop Closure
□ Graph Optimization
□ LIO
□ FAST-LIO2
□ 地图 / 应用 / Deep Learning扩展
```

---

# 最终原则

整个路线始终围绕：

```text
真实世界
↓
Sensor
↓
ROS2
↓
Data
↓
Frame / Time
↓
State Estimation
↓
Odometry
↓
SLAM
↓
Pose / Map
↓
Control
↓
STM32
↓
Actuator
```

其中：

```text
ROS2
=
数据怎样进入、流动、组织、记录、观察和调试

SLAM / State Estimation
=
怎样把这些有噪声的传感器数据变成可信的机器人状态和地图

STM32 / Control
=
怎样利用这些状态真正控制机器人执行动作
```

目标是最终成为：

> **既理解底层 STM32 / 控制，又能理解 ROS2 / 传感器 / 状态估计 / SLAM 的机器人系统型成员。**
