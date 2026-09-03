# ROS2 学习路线规划｜结合当前 Robocon / STM32 学习进度

> 用途：后续开启新的 ChatGPT 对话时，作为 ROS2 学习主线的长期参考文档。  
> 当前背景：已基本完成 2027 WHUROBOCON 校内赛上层机构程序设计，具备 STM32、CAN、FreeRTOS、电机控制、Mechanism / Task / Operator 等基础，希望开始系统接触 ROS2。  
> 学习方式：沿用当前 RC 学习项目的“理论 + 工程实践 + 视频 + GitHub 源码 + 用户先思考 + 验证闭环”流程。  
> 说明：ROS2、Ubuntu 与视频中的安装命令可能随版本变化。正式环境搭建前，应以**校队统一环境 + ROS2 官方文档 + 当前目标版本**为准，不机械照搬旧视频命令。

---

# 一、当前学习基础

当前不是从零开始学习机器人软件。

## 已掌握 / 基本掌握

### STM32 与嵌入式
- GPIO / EXTI / Timer / PWM / Clock
- IRQ / NVIC / ISR / HAL Callback
- UART / UART Interrupt / DMA / ReceiveToIdle DMA
- `.c/.h` 模块化、`static`
- 非阻塞程序、状态机
- Git / GitHub 基础

### CAN 与实时任务
- CAN 基本原理、ID、Filter、FIFO、IRQ
- HAL CAN 收发
- 多设备 CAN 通信
- FreeRTOS Task / Queue
- ISR → Queue → Task
- 周期任务、`osDelayUntil()`

### 电机与控制
- 电机 / 驱动器 / 编码器 / 减速器 / 负载
- 位置 / 速度 / 电流控制
- 闭环、多环伺服
- PID、离散 PID
- Speed PI、Position P、串级 PID
- Simulink 调参
- DJI、ZDrive
- Homing / Zero / 到位判断 / Fault 基本思想

### 机器人软件架构

已建立：

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

并已学习或实践：

- Mechanism / Task / Operator
- Motor 抽象
- Adapter / Ops Table
- 函数指针
- Dependency Injection
- 命令 ≠ 物理任务完成
- Accepted / Finish / Reject
- 双机械臂状态机
- 主从板 CAN 通信
- 气动机构、电磁阀控制

---

# 二、与 ROS2 直接相关的已有知识

已经学过：

- Vector / Matrix 基础
- Pose / Position / Orientation / Frame
- Rotation Matrix / Direction Cosine Matrix
- Homogeneous Transformation
- 固定轴 / 当前轴旋转
- 坐标映射、变换链
- K4A 外参
- IMU 基础
- Encoder / Odometry 基础
- LiDAR / Point Cloud 基础
- `map / odom / base_link / sensor_link`
- TF Tree 基本概念

因此以后学习：

```text
TF2
URDF
Robot State
Odometry
Sensor Fusion
```

时，不应重新从线性代数和齐次变换开始。

---

# 三、当前真正缺少的 ROS2 前置知识

## 1. Linux

只补机器人开发真正需要的部分：

```text
Linux 文件系统
终端
cd / ls / pwd
mkdir / rm
apt
sudo
环境变量
.bashrc
source
进程基础
权限基础
```

暂时不需要：

```text
Linux 内核
复杂网络管理
服务器运维
Shell 高级编程
```

## 2. C++

当前核心语言是嵌入式 C。

ROS2 C++ 开发需要补：

```text
struct → class
函数 → member function
初始化函数 → constructor
作用域 → namespace
裸指针 → smart pointer
函数指针 / Callback → lambda / std::function
引用
const
基础 STL
```

目标：

> 能够读懂和编写基础 `rclcpp` Node，而不是系统学完整 C++ 语言。

暂时不需要深入：

```text
模板元编程
复杂移动语义
STL 底层实现
高级泛型编程
```

---

# 四、ROS2 学习定位

ROS2 当前定位为第二学习主线。

近期推荐：

```text
70%  当前 Robocon 实机 / 程序可靠性 / 联调
30%  ROS2
```

等校内赛程序稳定以后，再逐渐提高 ROS2 比例。

---

# 五、ROS2 总学习路线

```text
阶段 0
ROS2 系统认知
+
Linux / C++ 必要前置
        ↓
阶段 1
Workspace / Package / Node
        ↓
阶段 2
Topic / Message
        ↓
Service
        ↓
Action
        ↓
Parameter
        ↓
阶段 3
Launch / CLI / Log / rosbag2 / QoS
        ↓
阶段 4
TF2 / URDF / RViz2
        ↓
阶段 5
ros2_control
        ↓
阶段 6
ROS2 ↔ STM32
        ↓
后续长期方向
状态估计 / 底盘 / LiDAR / SLAM / Nav2 / MoveIt2
```

---

# 六、阶段 0：建立 ROS2 系统心智模型

## 学习目标

第一阶段先不追求写大量代码。

必须先能回答：

```text
ROS2 为什么存在？
ROS2 在机器人系统中处于哪一层？

Node 是什么？
Topic 是什么？
Message 是什么？
Publisher / Subscriber 是什么？

Service 为什么存在？
Action 为什么存在？

ROS2 和 STM32 分别负责什么？
```

## 与已有知识建立联系

重点进行以下对比：

```text
FreeRTOS Task
vs
ROS2 Node
```

```text
FreeRTOS Queue
vs
ROS2 Topic
```

```text
CAN Command
→ Task
→ Mechanism
→ Finish

vs

ROS2 Action
```

注意：

> 这些只是用于建立第一版心智模型的类比，底层机制并不相同。

第一版系统模型：

```text
传感器
↓
ROS2 Node
↓
Topic
↓
定位 / 感知 / 决策
↓
ROS2 Node
↓
控制命令
↓
STM32
↓
Task
↓
Mechanism
↓
Motor / Driver
↓
Hardware
```

---

# 七、阶段 0.5：Linux + C++ 必要前置

## Linux

目标是能够理解和使用：

```bash
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
```

并理解：

```bash
source /opt/ros/<distro>/setup.bash
```

是在当前 shell 中加载 ROS2 环境。

## C++

建议从已有 C 迁移：

```text
C struct
↓
C++ class

模块函数
↓
成员函数

Init()
↓
Constructor

函数指针
↓
std::function / lambda
```

完成标准：

- 能读懂一个基础 `rclcpp::Node`
- 能看懂 Constructor
- 能看懂 `create_publisher`
- 能看懂 `create_subscription`
- 能理解 Callback
- 能理解 `std::shared_ptr`

---

# 八、阶段 1：Workspace / Package / Node

## Workspace

典型结构：

```text
robocon_ws/
├── src/
├── build/
├── install/
└── log/
```

## Package

逐渐理解：

```text
package.xml
CMakeLists.txt
src/
include/
launch/
config/
```

分别负责什么。

## Node

例如：

```text
imu_node
operator_node
localization_node
hardware_bridge_node
```

第一份最小工程：

```text
robocon_ws
↓
创建 package
↓
Node A
↓
Node B
↓
运行
↓
ros2 node list
↓
rqt_graph
```

暂时不连接真实机器人。

---

# 九、GitHub 源码学习节点 1：ros2/examples

重点仓库：

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

暂时不优先：

```text
executors
composition
wait_set
```

GitHub 阅读顺序：

```text
README
↓
package.xml
↓
CMakeLists.txt
↓
main()
↓
Node 类
↓
输入
↓
输出
↓
Callback
↓
运行
↓
修改一个功能
↓
预测
↓
验证
```

不能：

```text
clone
↓
build 成功
↓
结束
```

---

# 十、阶段 2：ROS2 通信模型

顺序：

```text
Topic
↓
Message
↓
Service
↓
Action
↓
Parameter
```

## Topic / Message

模型：

```text
Publisher
↓
Topic
↓
Subscriber
↓
Callback
```

例如：

```text
imu_node
↓ publish
/imu/data
↓ subscribe
state_estimator
```

最小实践：

```text
Publisher Node
↓
/robot_state
↓
Subscriber Node
```

并使用：

```bash
ros2 topic list
ros2 topic echo
ros2 topic info
```

观察系统。

## Service

模型：

```text
Client
↓ Request
Server
↓ Response
Client
```

适合：

```text
Reset
Calibrate
Query
Set Configuration
```

需要理解：

> Service 适合短时间请求—响应，不适合长时间物理动作。

## Action

模型：

```text
Goal
↓
Accepted / Rejected
↓
Executing
↓
Feedback
↓
Succeeded / Failed / Cancelled
```

与当前校内赛程序建立联系：

```text
CAN Command
↓
Accepted
↓
PickBlockTask
↓
BlockArm / Vacuum
↓
Finish
```

重点理解：

> 为什么长时间任务不能简单设计成 Service。

## Parameter

理解：

```text
代码逻辑
≠
运行参数
```

例如：

```text
max_speed
arm_tolerance
sensor_topic
device_port
```

并与当前 STM32 的：

```c
#define
const
config struct
```

进行比较。

---

# 十一、阶段 3：ROS2 系统工具

学习：

```text
ros2 node
ros2 topic
ros2 service
ros2 action
ros2 interface
rqt_graph
Launch
Parameter File
Logging
rosbag2
QoS
```

## Launch

用于一次启动完整机器人系统：

```text
robot.launch.py
│
├─ imu_node
├─ localization_node
├─ operator_node
└─ hardware_bridge_node
```

需要理解：

> Launch 是系统编排，不是机器人业务状态机。

## Logging

学习：

```text
DEBUG
INFO
WARN
ERROR
FATAL
```

未来目标：

```text
ROS2 上层日志
+
STM32 下层状态
```

一起定位问题。

## rosbag2

用于记录和回放 ROS2 Topic 数据。

未来特别适用于：

```text
IMU
LiDAR
Camera
Odometry
Robot State
```

调试。

## QoS

**QoS = Quality of Service，服务质量策略。**

第一阶段只理解：

```text
Reliability
History
Depth
Durability
```

并回答：

```text
IMU 高频数据丢一帧是否可以接受？
控制命令是否需要可靠传输？
新订阅者是否需要得到旧状态？
```

暂时不深入 DDS 内部实现。

---

# 十二、阶段 4：TF2 / URDF / RViz2

这是与已有知识衔接非常紧密的一阶段。

## TF2

**TF2 = ROS2 坐标变换库。**

已有理论：

```text
Frame
Pose
Rotation Matrix
Homogeneous Transform
TF Tree
```

因此这里直接学习：

```text
static transform
dynamic transform
transform broadcaster
transform listener
```

建立：

```text
map
↓
odom
↓
base_link
↓
arm_link
↓
camera_link
```

等关系。

## URDF

**URDF = Unified Robot Description Format，统一机器人描述格式。**

重点：

```text
link
joint
origin
axis
visual
collision
inertial
```

目标：把一个简化机器人模型描述出来。

## RViz2

学习可视化：

```text
Robot Model
TF
PointCloud
IMU
Path
Marker
```

目标不是“会点 RViz”，而是：

> 能判断显示异常来自 TF、Topic、Message 还是模型配置。

---

# 十三、阶段 5：ros2_control

这是当前 ROS2 路线中最值得深入的一阶段之一。

因为已有：

```text
Mechanism
↓
Motor
↓
Adapter
↓
Driver
```

而 ros2_control 会引入：

```text
Controller
↓
Command Interface
↓
Hardware Interface
↓
Hardware
```

需要重点比较两套架构。

核心循环：

```text
read
↓
update
↓
write
```

其中：

```text
read
→ 获取硬件状态

update
→ Controller 计算新命令

write
→ 把控制命令写到硬件
```

---

# 十四、GitHub 源码学习节点 2：ros2_control_demos

重点仓库：

```text
https://github.com/ros-controls/ros2_control_demos
```

学习顺序：

```text
简单位置控制 Demo
↓
Hardware Interface
↓
Controller
↓
多执行器 Demo
```

不要一开始读完整 ros2_control 框架源码。

重点比较：

```text
当前：

Mechanism
↓
Motor API
↓
Adapter
↓
DJI / ZDrive
```

和：

```text
ROS2：

Controller
↓
Command Interface
↓
Hardware Interface
↓
Hardware
```

目标：理解相似问题与机制差异。

---

# 十五、阶段 6：ROS2 ↔ STM32

前面基础完成后，再研究 ROS2 和现有 STM32 系统怎么连接。

## 方案 A：Bridge Node

优先理解：

```text
ROS2
↓
hardware_bridge_node
↓ CAN / UART
STM32
↓
Task
↓
Mechanism
↓
Motor
↓
Driver
```

优点：

- 现有 STM32 架构变化小
- ROS2 与硬实时控制边界清楚
- 适合 Robocon 当前工程

## 方案 B：micro-ROS

后续了解：

```text
https://github.com/micro-ROS/micro_ros_stm32cubemx_utils
```

但第一轮不直接使用。

先回答：

```text
为什么需要 micro-ROS？

现有：
ROS2
↓ CAN
STM32

是否已经足够？
```

只有出现明确工程需求后，再考虑 micro-ROS。

---

# 十六、B站视频学习方案

视频只作为教学节点，不作为唯一主线。

原则：

```text
理论理解
↓
看指定视频的小范围内容
↓
回来讨论
↓
最小实践
↓
GitHub 源码
↓
自己实现
```

不要一次刷完整课程。

## 主课 1：鱼香ROS

**鱼香ROS《ROS2机器人开发从入门到实践》**

```text
https://www.bilibili.com/video/BV1GW42197Ck/
```

作用：

```text
Linux
C++
Workspace
Package
Node
Topic
Service
Action
URDF
RViz
ros2_control
```

使用原则：

> 选择与当前知识节点对应的视频，不顺序刷完全部课程。

## 主课 2：古月居 ROS2 入门

**古月居《ROS2入门21讲》**

```text
https://www.bilibili.com/video/BV16B4y1Q7jQ/
```

适合：

```text
ROS2总体概念
Node
Topic
Service
Action
TF
URDF
RViz
```

作用：

> 快速建立概念模型，不直接把旧版本安装命令作为当前环境依据。

---

# 十七、视频使用规则

每个视频节点都要先明确：

```text
这个视频是为了解决哪个知识问题？

我看之前已经知道什么？

需要重点关注哪些内容？

看完后我必须能解释什么？

有没有实际代码需要复现？

有没有版本差异？
```

禁止：

```text
“先把这20集看完再说”
```

---

# 十八、GitHub 开源项目学习方法

每次进入一个新项目时，先系统建模：

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
Topic / Service / Action
↓
Package
↓
关键类
↓
核心 Callback
↓
运行入口
```

然后再读代码。

## 第一层：官方最小示例

```text
ros2/examples
```

目标：理解基础 API。

## 第二层：官方功能 Demo

```text
ros2_control_demos
```

目标：理解机器人控制架构。

## 第三层：真实机器人项目

第一轮基础完成以后，再选择一个真实 Robocon / RoboMaster / 移动机器人 ROS2 项目。

学习重点：

```text
真实项目如何划分 package？
如何组织 launch？
如何管理 TF？
如何桥接硬件？
如何设计 controller？
```

而不是复制代码。

---

# 十九、每一小阶段固定使用当前学习 Skill

学习流程：

```text
1. 读取当前进度
↓
2. 判断：
   已掌握
   需要唤醒
   新知识
↓
3. 先讲新概念
↓
4. 与已有 STM32 / Robocon 知识建立联系
↓
5. 给对应 B站视频节点
↓
6. 用户观看
↓
7. 回来讨论
↓
8. 用户先复述 / 建模
↓
9. 最小代码实践
↓
10. GitHub 源码阅读
↓
11. 用户自己分析
↓
12. 用户自己迁移实现
↓
13. 预测运行结果
↓
14. 实际运行
↓
15. 对比预测与结果
↓
16. 进入下一节点
```

---

# 二十、代码学习层级

遇到代码时始终区分：

```text
ROS2 middleware
↓
rclcpp / rclpy
↓
Node
↓
Callback
↓
业务逻辑
↓
硬件桥接
↓
STM32
```

不要把：

```text
DDS机制
ROS2 API
用户业务
STM32底层
```

混成一个层级解释。

---

# 二十一、第一轮明确不进入

第一轮 ROS2 基础阶段不深入：

```text
Nav2
SLAM
FAST-LIO2源码
MoveIt2
MoveIt Servo
DDS内部
RMW内部
复杂多机网络
复杂生命周期节点
实时 Linux 深入
复杂 Gazebo 仿真
高级视觉
```

这些属于第二阶段以后的方向。

---

# 二十二、第一轮 ROS2 学习完成标准

完成后应该能够独立解释：

```text
Workspace
Package
Node
Topic
Message
Publisher
Subscriber
Service
Action
Parameter
Launch
QoS基础
TF2
URDF
RViz2
ros2_control基础
```

并且能够独立完成：

```text
创建 Workspace
↓
创建 C++ Package
↓
写 Publisher
↓
写 Subscriber
↓
写 Service
↓
写 Action
↓
用 Launch 启动
↓
用 CLI 检查
↓
用 rqt_graph 看节点关系
↓
用 rosbag2 记录
↓
建立 TF
↓
加载 URDF
↓
RViz 显示机器人
↓
理解 ros2_control 的 read-update-write
```

---

# 二十三、第一轮最终系统建模目标

能够自己设计：

```text
ROS2 PC
│
├─ operator_node
│
├─ perception_node
│
├─ state_estimator_node
│
├─ robot_state_node
│
└─ hardware_bridge_node
       ↓
     CAN / UART
       ↓
     STM32
       ↓
      Task
       ↓
   Mechanism
       ↓
      Motor
       ↓
     Driver
       ↓
    Hardware
```

并解释：

```text
哪些功能应该在 ROS2？
哪些功能应该在 STM32？
哪些信息通过通信传输？
为什么这样划分？
```

---

# 二十四、后续长期路线

ROS2 基础完成后再进入：

```text
ROS2基础
↓
传感器驱动
↓
四元数
↓
IMU
↓
Odometry
↓
Sensor Fusion
↓
EKF
↓
底盘运动学
↓
ros2_control 实机
↓
LiDAR
↓
SLAM / LIO
↓
Nav2
↓
机械臂 MoveIt2
```

具体进入哪一条，根据当时 Robocon 队内任务决定，不机械全部学习。

---

# 二十五、推荐学习节奏

ROS2 当前作为副线：

```text
每周 2～3 次
每次 1～1.5 小时
```

第一轮预计约：

```text
5～7 周
```

不以时间为硬性目标，以实际掌握为准。

| 阶段 | 内容 | 建议次数 |
|---|---|---:|
| 0 | ROS2 系统认知 | 2 |
| 0.5 | Linux + C++ 必要知识 | 3～4 |
| 1 | Workspace / Package / Node | 2～3 |
| 2 | Topic / Service / Action | 4～5 |
| 3 | Launch / Tools / QoS | 2～3 |
| 4 | TF2 / URDF / RViz | 3～4 |
| 5 | ros2_control | 4～6 |
| 6 | ROS2 ↔ STM32 | 2～3 |

---

# 二十六、第一天建议内容

第一天不要先安装大量软件。

先学习：

# ROS2 为什么需要 Node 和 Topic？

重点对比：

```text
ROS2 Node
vs
FreeRTOS Task
```

```text
ROS2 Topic
vs
FreeRTOS Queue
```

```text
ROS2 Message
vs
CAN Message / 自定义 struct
```

第一天完成标准：

```text
✓ 知道 ROS2 处于机器人哪一层
✓ 知道 Node 的职责
✓ 知道 Topic 的职责
✓ 知道 Publisher / Subscriber
✓ 知道 Message
✓ 能画出两个 Node 通过 Topic 通信
✓ 能说明它和 FreeRTOS Queue 的本质区别
```

完成这一小块以后，再进入：

```text
环境选择
↓
Ubuntu
↓
ROS2版本
↓
安装
```

---

# 二十七、新对话启动方式

以后开启新的 ROS2 学习对话时，可以直接使用：

```markdown
我现在开始 ROS2 学习。

请先读取：

1. 我的学习笔记仓库：
   https://github.com/2h34/robocon-learning-notes12345

2. 本文件：
   《ROS2学习路线规划_结合当前Robocon学习进度.md》

请严格按照 RC 学习项目的学习 Skill 推进。

不要把我当成完全没有机器人基础的新手。

我已经掌握：

- STM32
- UART / CAN
- FreeRTOS
- PID
- DJI / ZDrive
- Mechanism
- Task
- Operator
- Motor Adapter
- 状态机
- 齐次变换
- Frame / Pose
- IMU / odom / LiDAR 基础
- TF Tree 基本概念

请先根据真实仓库判断：

- 已掌握
- 需要唤醒
- 新知识

然后从当前 ROS2 路线真正停止的位置继续。

学习时继续采用：

理论
→ 与已有知识建立联系
→ B站指定视频节点
→ 讨论
→ 最小实践
→ GitHub源码阅读
→ 我自己分析
→ 我自己实现
→ 预测
→ 验证

不要一次性推进太多。

如果这是第一次正式进入 ROS2，则从：

“ROS2为什么需要 Node 和 Topic，它们与 FreeRTOS Task / Queue 有什么相似和不同？”

开始。
```

---

# 二十八、核心学习原则

最终目标不是：

```text
会 ros2 launch
会跑别人的 SLAM
会在 RViz 看到机器人
```

而是：

> **真正理解一个 ROS2 机器人软件系统为什么这样拆、模块之间如何通信、ROS2 与 STM32 如何分工，并能够自己设计和调试完整机器人软件架构。**

最终主线：

```text
已有 STM32 / Robocon 工程能力
↓
ROS2 系统模型
↓
Linux + C++
↓
Node
↓
Topic / Service / Action
↓
ROS2 工程组织
↓
TF2 / URDF / RViz
↓
ros2_control
↓
ROS2 ↔ STM32
↓
状态估计 / 底盘 / 感知 / 导航 / 机械臂高级功能
```

---

# 参考资料

## 当前学习知识库

```text
https://github.com/2h34/robocon-learning-notes12345
```

重点承接：

```text
2026-08-21_ZDrive与Lift_Mechanism层学习记录
2026-08-22_Motor抽象_Adapter_函数指针与多Mechanism学习记录
2026-08-22_校内赛上层控制架构设计_学习记录
2026-08-24_蓝牙通信协议_Mapping与Operator上层控制学习记录
2026-08-25_空间描述_齐次变换_K4A外参与多传感器定位_完整学习记录
2026-09-01程序逻辑学习记录
```

## ROS2 官方示例

```text
https://github.com/ros2/examples
```

## ros2_control 示例

```text
https://github.com/ros-controls/ros2_control_demos
```

## micro-ROS STM32CubeMX

```text
https://github.com/micro-ROS/micro_ros_stm32cubemx_utils
```

## B站主课

鱼香ROS：

```text
https://www.bilibili.com/video/BV1GW42197Ck/
```

古月居 ROS2 入门：

```text
https://www.bilibili.com/video/BV16B4y1Q7jQ/
```

---

# 当前进度

```text
✓ STM32 / Embedded C
✓ UART / CAN
✓ FreeRTOS 基础
✓ PID / 电机控制
✓ DJI / ZDrive
✓ Mechanism / Task / Operator
✓ Motor Adapter
✓ 气动机构
✓ 坐标变换 / TF Tree 基础
✓ 校内赛上层程序架构

→ ROS2 阶段 0
   ROS2 系统模型
   Node / Topic
   Linux / C++ 必要前置

□ Workspace / Package
□ Publisher / Subscriber
□ Service
□ Action
□ Parameter
□ Launch / rosbag / QoS
□ TF2
□ URDF
□ RViz
□ ros2_control
□ ROS2 ↔ STM32
□ 后续状态估计 / 底盘 / 感知
```
