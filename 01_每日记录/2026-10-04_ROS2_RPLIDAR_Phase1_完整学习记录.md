# 2026-10-04 ROS2 + RPLIDAR Phase 1 学习记录

> 主题：真实 RPLIDAR 数据链建立  
> 环境：Windows → WSL2 → Ubuntu 22.04 → ROS2 Humble  
> Workspace：`~/ros2_ws`  
> 当前用户：`glance`  
> Phase 1 目标：让真实 RPLIDAR 成为稳定、可验证、可录制、可回放、可视化的 ROS2 `LaserScan` 数据源。

---

## 0. 今日结论

今天完成了 Phase 1 的完整闭环：

```text
真实 RPLIDAR
↓
USB / CP2102
↓
Windows
↓
usbipd
↓
WSL2
↓
Ubuntu /dev/ttyUSB0
↓
SLAMTEC SDK
↓
sllidar_node
↓
sensor_msgs/msg/LaserScan
↓
/scan
↓
CLI 验证
↓
真实物理实验
↓
rosbag record / play
↓
RViz
```

最终可以确认：

- Windows、WSL2、Ubuntu 三层设备链均已打通；
- `sllidar_ros2` Driver 成功编译、启动并连接真实雷达；
- `/scan` 正常发布，类型为 `sensor_msgs/msg/LaserScan`；
- 实际发布频率约 `7.26 Hz`；
- QoS 实测为 `RELIABLE + VOLATILE`；
- `frame_id = laser`；
- Service 可真实控制电机启停；
- 完成真实距离、角度、位置变化实验；
- 完成三组 rosbag 录制；
- 成功在无真实雷达连接的情况下 rosbag 回放 `/scan`；
- RViz 可正确显示真实扫描与 bag 回放扫描；
- Phase 1 可以判定完成，可以在后续进入 P2：
  `LaserScan → 极坐标 → 笛卡尔坐标 → 二维点集`。

---

# 1. Phase 1 的核心问题

Phase 1 不是“把一个 Driver 跑起来”这么简单，而是要真正回答：

> 一台真实 RPLIDAR，怎样从 USB 设备最终变成 ROS2 中稳定的 `/scan`？

因此今天严格按照分层思路推进：

```text
物理设备
↓
Windows USB
↓
WSL2 USB
↓
Linux device
↓
Driver / SDK
↓
ROS2 Node
↓
Publisher
↓
Topic
↓
LaserScan
↓
CLI / rosbag / RViz
```

整个过程中始终坚持：

```text
真实终端输出
>
预设教程答案
```

以及：

```text
Node 存在 ≠ 数据正常
Topic 存在 ≠ 数据正常
Publisher 存在 ≠ 通信一定正常
```

---

# 2. Windows → WSL2 → Linux USB 设备链

## 2.1 Windows 侧识别

设备最终识别为：

```text
Silicon Labs CP210x USB to UART Bridge (COM3)
VID:PID = 10c4:ea60
```

USB-UART 芯片为：

```text
CP2102 / CP210x
```

最初 Windows 中设备状态异常，Driver 未安装，后安装 Silicon Labs CP210x VCP Driver 后恢复正常。

### 关键理解

雷达“会转”并不能证明通信正常：

```text
雷达旋转
≠
Windows 正确识别 USB
≠
WSL2 已获得设备
≠
Linux 已创建串口设备
≠
ROS2 Driver 已经能通信
```

---

## 2.2 usbipd 转发

Windows 侧：

```powershell
usbipd list
```

识别到：

```text
2-9    10c4:ea60    Silicon Labs CP210x USB to UART Bridge (COM3)
```

设备曾处于：

```text
Shared
```

通过：

```powershell
usbipd attach --wsl --busid 2-9
```

变为：

```text
Attached
```

### 重要经验

`bind` 通常可以保留，但 `attach` 在以下情况后经常需要重新执行：

- USB 拔插；
- WSL 重启；
- Windows 重启；
- BUSID 改变。

因此以后不要硬编码旧 BUSID。

推荐流程：

```text
插入雷达
↓
usbipd list
↓
找 VID:PID = 10c4:ea60
↓
确认当前 BUSID
↓
attach
```

---

## 2.3 WSL2 中识别

进入正确的发行版：

```powershell
wsl -d Ubuntu-22.04 -u glance
```

避免误进入：

```text
docker-desktop
```

Ubuntu 中：

```bash
lsusb
```

得到：

```text
Bus 001 Device 003: ID 10c4:ea60 Silicon Labs CP210x UART Bridge
```

进一步确认：

```bash
ls -l /dev/ttyUSB0
```

得到：

```text
crw-rw---- 1 root dialout ... /dev/ttyUSB0
```

因此 Linux device node 确认是：

```text
/dev/ttyUSB0
```

---

# 3. Linux 权限与 dialout

最开始用户 `glance` 不属于：

```text
dialout
```

执行：

```bash
sudo usermod -aG dialout glance
```

后，旧终端仍未立即刷新组权限。

临时通过：

```bash
newgrp dialout
```

验证可以打开：

```text
/dev/ttyUSB0
```

之后彻底重启对应 WSL 发行版：

```powershell
wsl --terminate Ubuntu-22.04
```

重新进入后：

```bash
id
```

确认：

```text
groups=...,20(dialout),...
```

### 关键理解

设备权限链：

```text
/dev/ttyUSB0
owner: root
group: dialout
mode: rw-rw----
```

因此普通用户访问串口的关键是：

```text
用户属于 dialout
```

而不是用 `sudo ros2 ...` 去绕过权限。

---

# 4. Driver 选择与源码获取

采用 SLAMTEC 官方 ROS2 Driver：

```text
sllidar_ros2
```

放入：

```text
~/ros2_ws/src/sllidar_ros2
```

源码结构中重点关注：

```text
CMakeLists.txt
package.xml
launch/
src/sllidar_node.cpp
src/sllidar_client.cpp
sdk/
rviz/
```

---

# 5. 从 CMake 理解 Executable

`CMakeLists.txt` 中：

```cpp
add_executable(
    sllidar_node
    src/sllidar_node.cpp
    ${SLLIDAR_SDK_SRC}
)
```

说明：

```text
src/sllidar_node.cpp
```

是源码文件，而：

```text
sllidar_node
```

才是最终 executable。

并安装到：

```text
lib/sllidar_ros2/
```

### 工程关系

```text
Package
sllidar_ros2
↓
Source
src/sllidar_node.cpp
↓
CMake add_executable
↓
Executable
sllidar_node
↓
ros2 launch / ros2 run
```

---

# 6. Launch 文件理解

使用：

```text
sllidar_a1_launch.py
```

关键参数：

```text
channel_type      = serial
serial_port       = /dev/ttyUSB0
serial_baudrate   = 115200
frame_id          = laser
inverted          = false
angle_compensate  = true
scan_mode         = Sensitivity
```

其中实际传入 Node 参数字典的包括：

```text
channel_type
serial_port
serial_baudrate
frame_id
inverted
angle_compensate
```

注意：

```text
scan_mode
```

虽然在该 launch 文件中被声明，但在看到的参数字典里没有传入 Node。

### 关键理解

Launch 的职责：

```text
选择 Package
↓
选择 Executable
↓
创建 Node
↓
传递 Parameters
```

Launch 本身不是 LiDAR Driver 核心数据处理逻辑。

---

# 7. Driver 五层结构

今天形成了一个比较清晰的五层认识：

```text
1. Linux Device
   /dev/ttyUSB0

2. SLAMTEC SDK
   处理设备协议、串口通信、扫描数据

3. ROS2 Node
   sllidar_node

4. ROS2 Interface
   /scan
   /start_motor
   /stop_motor

5. ROS2 Consumers
   CLI / rosbag / RViz / 后续算法
```

数据流：

```text
RPLIDAR
↓
CP2102
↓
/dev/ttyUSB0
↓
SLAMTEC SDK
↓
sllidar_node
↓
LaserScan
↓
/scan
```

控制流：

```text
ros2 launch
↓
launch file
↓
Node + Parameters
```

---

# 8. `SLlidarNode` 构造函数

源码中：

```cpp
SLlidarNode()
: Node("sllidar_node")
{
    scan_pub = this->create_publisher<
        sensor_msgs::msg::LaserScan
    >(
        "scan",
        rclcpp::QoS(rclcpp::KeepLast(10))
    );
}
```

这里完成：

```text
创建 Node
↓
创建 /scan Publisher
```

注意：

构造函数中看到的是 Publisher 创建。

参数初始化由：

```cpp
init_param()
```

单独完成。

---

# 9. Parameter 初始化

`init_param()` 中主要通过：

```cpp
declare_parameter(...)
```

注册参数；

通过：

```cpp
get_parameter_or(...)
```

把参数值读取到 C++ 成员变量中。

例如：

```text
frame_id
serial_port
serial_baudrate
scan_mode
scan_frequency
```

### 理解

```text
declare_parameter
```

解决的是：

> ROS2 参数系统里“这个参数存在”。

而：

```text
get_parameter_or
```

解决的是：

> C++ 代码真正拿到参数值并保存到变量。

---

# 10. work_loop：真实硬件 Driver 的主循环

核心工作函数：

```cpp
int work_loop()
```

大致流程：

```text
init_param()
↓
createLidarDriver()
↓
createSerialPortChannel()
↓
drv->connect()
↓
getDeviceInfo()
↓
checkHealth()
↓
创建 start/stop motor Service
↓
setMotorSpeed()
↓
startScan()
↓
while(...)
    grabScanDataHq()
    ascendScanData()
    publish_scan()
    spin_some()
```

---

# 11. `main()` 与 `work_loop()`

源码：

```cpp
int main(int argc, char * argv[])
{
    rclcpp::init(argc, argv);

    auto sllidar_node =
        std::make_shared<SLlidarNode>();

    signal(SIGINT, ExitHandler);

    int ret = sllidar_node->work_loop();

    rclcpp::shutdown();

    return ret;
}
```

这里没有：

```cpp
rclcpp::spin(node);
```

而是：

```text
main
↓
work_loop
↓
while
```

---

# 12. `spin_some()` 与 Executor

源码中：

```cpp
rclcpp::spin_some(shared_from_this());
```

说明这个 Driver 采用的是：

```text
硬件主动工作循环
+
ROS2 callback 调度
```

模式。

一次循环可以理解为：

```text
grabScanDataHq()
↓
处理数据
↓
publish_scan()
↓
spin_some()
↓
下一轮
```

### 重要理解

`publish()` 本身不要求 Executor。

Executor 主要负责：

```text
Timer callback
Subscription callback
Service callback
Action callback
```

这里：

```text
硬件数据读取
```

由 `work_loop()` 自己控制；

而：

```text
/start_motor
/stop_motor
```

这类 Service callback 则靠：

```cpp
spin_some()
```

调度。

---

# 13. `publish_scan()`：SDK 数据 → ROS2 LaserScan

函数定义：

```cpp
void publish_scan(
    Publisher::SharedPtr& pub,
    sl_lidar_response_measurement_node_hq_t *nodes,
    size_t node_count,
    rclcpp::Time start,
    double scan_time,
    bool inverted,
    float angle_min,
    float angle_max,
    float max_distance,
    std::string frame_id
)
```

首先明确：

```text
括号里的变量
```

是函数调用时传入的实参对应的形参。

它们不是“全部由函数自己生成”。

---

## 13.1 创建消息

```cpp
auto scan_msg =
    std::make_shared<sensor_msgs::msg::LaserScan>();
```

---

## 13.2 Header

```cpp
scan_msg->header.stamp = start;
scan_msg->header.frame_id = frame_id;
```

---

## 13.3 角度

```cpp
scan_msg->angle_min
scan_msg->angle_max
scan_msg->angle_increment
```

其中：

```cpp
scan_msg->angle_increment =
    (scan_msg->angle_max - scan_msg->angle_min)
    / (double)(node_count - 1);
```

---

## 13.4 时间

```cpp
scan_msg->scan_time = scan_time;

scan_msg->time_increment =
    scan_time / (double)(node_count - 1);
```

---

## 13.5 量程

```cpp
scan_msg->range_min = 0.05;
scan_msg->range_max = max_distance;
```

---

## 13.6 距离数据

核心转换：

```cpp
float read_value =
    (float)nodes[i].dist_mm_q2 / 4.0f / 1000;
```

含义：

```text
dist_mm_q2
÷ 4
→ mm
÷ 1000
→ m
```

最终：

```cpp
scan_msg->ranges[i] = read_value;
```

如果：

```text
read_value == 0
```

则：

```cpp
ranges[i] = infinity
```

---

## 13.7 intensities

```cpp
scan_msg->intensities[i] =
    (float)(nodes[i].quality >> 2);
```

理解为：

```text
ranges[i]
= 第 i 个方向测了多远

intensities[i]
= 第 i 个测距点的回波质量 / 强度指标
```

它不是严格统一的物理单位。

---

## 13.8 发布

最终：

```cpp
pub->publish(*scan_msg);
```

完整数据转换：

```text
SLAMTEC SDK nodes[]
+
时间 / frame_id / 配置
↓
publish_scan()
↓
sensor_msgs/msg/LaserScan
↓
/scan
```

---

# 14. 编译 Driver

执行：

```bash
cd ~/ros2_ws
colcon build --packages-select sllidar_ros2
```

结果：

```text
Finished <<< sllidar_ros2
Summary: 1 package finished
```

期间出现不少 SDK warning，例如：

```text
zero-size array
unused parameter
cast warning
```

但无 error，因此：

```text
warning ≠ build failure
```

编译结果 PASS。

之后：

```bash
source install/setup.bash
```

并通过：

```bash
ros2 pkg list | grep sllidar
```

确认 package 已进入当前 ROS2 环境。

---

# 15. 真实 Driver 首次启动

真实连接设备后：

```bash
ros2 launch sllidar_ros2 sllidar_a1_launch.py
```

得到：

```text
SLLidar.ROS2 SDK Version: 1.0.1
SLLIDAR SDK Version: 2.1.0

SLLidar S/N: ...
Firmware Ver: 1.29
Hardware Rev: 7

SLLidar health status : 0
SLLidar health status : OK.

current scan mode: Sensitivity
sample rate: 8 Khz
max_distance: 12.0 m
scan frequency: 10.0 Hz
```

说明：

```text
connect()
PASS

getDeviceInfo()
PASS

checkHealth()
PASS

startScan()
PASS
```

---

# 16. 真实 `/scan` 验证

```bash
ros2 topic list
```

得到：

```text
/scan
```

进一步：

```bash
ros2 topic info /scan
```

得到：

```text
Type: sensor_msgs/msg/LaserScan
Publisher count: 1
Subscription count: 0
```

与源码：

```cpp
create_publisher<sensor_msgs::msg::LaserScan>("scan", ...)
```

完全对应。

---

# 17. 真实 LaserScan 字段

通过：

```bash
ros2 topic echo /scan --once
```

实际观察到：

```text
header:
  frame_id: laser

angle_min: ≈ -π
angle_max: ≈ +π

angle_increment: ≈ 0.00582 rad

range_min: 0.05
range_max: 12.0

scan_time: ≈ 0.1346 s

ranges:
  ...
  .inf
  ...

intensities:
  ...
```

---

# 18. `/scan` 实际频率

执行：

```bash
ros2 topic hz /scan
```

得到稳定结果：

```text
average rate ≈ 7.26 Hz
```

周期：

```text
≈ 0.138 s
```

---

## 18.1 10 Hz 与 7.26 Hz 的区别

启动日志：

```text
scan frequency: 10.0 Hz
```

不是实测的真实 ROS2 发布频率。

源码中的：

```text
scan_frequency = 10
```

主要参与：

```cpp
points_per_circle =
    1000000 /
    current_scan_mode.us_per_sample /
    scan_frequency;
```

因此它属于：

```text
程序配置 / 估计参数
```

而：

```bash
ros2 topic hz /scan
```

测到的：

```text
7.26 Hz
```

才是实际 ROS2 消息发布频率。

---

# 19. QoS 实测

```bash
ros2 topic info /scan --verbose
```

得到：

```text
Reliability: RELIABLE
Durability: VOLATILE
History (Depth): UNKNOWN
```

源码：

```cpp
rclcpp::QoS(rclcpp::KeepLast(10))
```

因此可以确认：

```text
KeepLast(10)
```

在源码中存在，但 CLI introspection 没有返回具体 depth，所以显示：

```text
UNKNOWN
```

不能把 UNKNOWN 理解为没有 queue depth。

---

# 20. Service：真实电机控制

Driver 创建：

```text
/start_motor
/stop_motor
```

类型：

```text
std_srvs/srv/Empty
```

---

## 20.1 回调函数

```cpp
bool stop_motor(
    const std::shared_ptr<std_srvs::srv::Empty::Request> req,
    std::shared_ptr<std_srvs::srv::Empty::Response> res)
```

以及：

```cpp
bool start_motor(...)
```

就是 Service callback 的定义。

由于 `Empty` 请求和响应都没有字段：

```cpp
(void)req;
(void)res;
```

只是告诉编译器：

```text
这两个参数存在，但这里故意不使用
```

---

## 20.2 stop_motor

实际调用：

```bash
ros2 service call \
/stop_motor \
std_srvs/srv/Empty "{}"
```

电机真实停转。

完整链：

```text
Service Request
↓
spin_some()
↓
stop_motor()
↓
drv->setMotorSpeed(0)
↓
真实电机停转
```

---

## 20.3 start_motor

执行：

```bash
ros2 service call \
/start_motor \
std_srvs/srv/Empty "{}"
```

电机真实重新启动。

源码中：

```cpp
drv->setMotorSpeed();
drv->startScan(0,1);
```

---

# 21. 电机自动旋转现象

今天确认了一个比较特殊的真实硬件行为：

```text
USB 插入
→ 雷达自动旋转
```

Driver 运行时：

```text
/stop_motor
→ 可以停转
```

但：

```text
Driver 退出
→ 电机又重新开始旋转
```

目前只能确认：

```text
Driver 可通过 SDK 控制电机
```

但还不能确定：

```text
为什么无 Driver 时硬件会自动旋转
```

可能与：

```text
转接板
硬件默认状态
控制线逻辑
```

有关。

该问题不影响 Phase 1，因此暂不深入 SDK / 电机底层控制。

---

# 22. 真实物理实验

Phase 1 不直接进入算法，而是先建立：

```text
物理世界
↕
LaserScan
```

的直觉。

---

## 22.1 实验 A：真实物体 ↔ ranges[]

使用一个保温杯作为目标。

第一次直接根据某个 `0.405 m` 数据猜测目标位置，被发现证据不足。

之后改用：

```text
有杯子
↓
拿走杯子
↓
重新放回杯子
```

三次对照。

最终确认：

```text
ranges[1077..1079]
+
ranges[0..约15]
```

附近存在稳定目标点簇。

第一次有杯子：

```text
≈ 0.35 m
```

拿走后：

```text
≈ 0.7 ~ 0.94 m
或 inf
```

重新放回：

```text
≈ 0.40 m
```

说明该连续点簇确实对应真实杯子。

### 重要理解

不能看到某个距离值就直接说：

```text
“这就是杯子”
```

必须通过真实对照实验确认。

---

# 23. 实验 B：方向变化

把杯子从原位置向右移动。

原位置主要位于：

```text
ranges[1067..1079]
+
ranges[0..15]
```

右移后，新的一簇近距离点出现在：

```text
ranges[982..990]
```

大约对应：

```text
约 +149°
```

说明：

```text
真实物体方向变化
↓
LaserScan 中对应下标区域变化
```

同时，因为实际移动不是沿等半径圆弧进行，杯子与雷达之间的径向距离也发生变化。

---

# 24. 实验 C：距离变化

最后重新把杯子放到数组首尾区域。

近距离：

```text
区域中位数 ≈ 0.216 m
```

把杯子沿大致同一方向远离雷达约 15~20 cm。

远距离：

```text
区域中位数 ≈ 0.397 m
```

变化：

```text
0.397 - 0.216
= 0.181 m
≈ 18 cm
```

与真实移动距离高度一致。

实验结论：

```text
真实物体离雷达更远
↓
同一方向附近 ranges[i] 增大
```

实验 C PASS。

---

# 25. rosbag 数据集

今天录制了三组真实 `/scan` 数据。

目录：

```text
~/ros2_ws/bags
```

---

## 25.1 杯子正前方

```text
rplidar_front_static
```

信息：

```text
Duration: 9.1009 s
Messages: 66
Topic: /scan
Type: sensor_msgs/msg/LaserScan
```

平均约：

```text
7.25 Hz
```

---

## 25.2 无杯子背景

```text
rplidar_background_static
```

信息：

```text
Duration: 10.3538 s
Messages: 75
```

平均约：

```text
7.24 Hz
```

---

## 25.3 杯子偏右

```text
rplidar_cup_right
```

信息：

```text
Duration: 10.9185 s
Messages: 79
```

平均约：

```text
7.24 Hz
```

三组 bag 频率与：

```bash
ros2 topic hz /scan
```

得到的：

```text
≈ 7.26 Hz
```

高度一致。

---

# 26. rosbag 回放

真实 Driver 退出并拔掉雷达后：

```bash
ros2 bag play rplidar_front_static
```

仍然可以重新发布：

```text
/scan
```

RViz 中重新出现扫描点。

因此验证：

```text
rosbag
↓
/scan
↓
LaserScan
↓
RViz
```

完整闭环。

这意味着后续 P2 / P3 可以：

```text
ros2 bag play
↓
/scan
↓
算法 Node
```

不需要始终连接真实雷达。

---

# 27. RViz

RViz 设置：

```text
Fixed Frame = laser
Topic = /scan
View = TopDownOrtho
```

LaserScan：

```text
Status: Ok
```

成功看到真实环境中的二维扫描点。

---

## 27.1 TF Warning

RViz 左侧存在：

```text
Global Status: Warn
No tf data
```

当前原因：

```text
只有 laser frame
没有完整 TF tree
```

Phase 1 不正式展开：

```text
laser
↓
base_link
↓
odom
↓
map
```

因此只记录该需求，不在当前阶段处理。

---

# 28. FakeLidarNode 与真实 Driver 对比

## 共同点

ROS2 接口层可以完全一致：

```text
Node
Publisher
Topic
Message Type
QoS
```

都可以最终：

```text
publish sensor_msgs/msg/LaserScan
→ /scan
```

后面的算法 Node 不需要知道：

```text
数据来自真实雷达
还是 Fake Node
```

---

## 区别

### FakeLidarNode

数据来源：

```text
程序虚构 / 模拟
```

执行方式通常：

```text
Timer
↓
callback
↓
构造 LaserScan
↓
publish
```

不依赖真实硬件。

---

### sllidar_node

数据来源：

```text
真实 RPLIDAR
```

执行方式：

```text
work_loop
↓
SDK 读取
↓
nodes[]
↓
publish_scan()
↓
LaserScan
↓
publish
```

同时通过：

```cpp
spin_some()
```

处理 ROS2 Service callback。

依赖：

```text
USB
/dev/ttyUSB0
权限
SDK
真实雷达
```

---

# 29. 今天形成的工程认知

## 29.1 不根据名字猜代码

以后阅读真实工程必须：

```text
谁触发
↓
执行了什么
↓
修改了什么
↓
数据来自哪里
↓
哪里真正 publish
↓
ROS graph 发生什么
```

---

## 29.2 配置值 ≠ 实测值

今天典型例子：

```text
scan_frequency = 10 Hz
```

不等于：

```text
实际 /scan 发布频率
```

实测：

```text
≈ 7.26 Hz
```

以后遇到：

```text
Parameter
Default
Config
Expected
```

都不能自动认为：

```text
Actual
```

---

## 29.3 单次观察不能直接认定物理目标

第一次看到：

```text
0.405 m
```

不能直接认定：

```text
“这就是杯子”
```

必须使用：

```text
控制变量
+
前后对照
+
重复实验
```

来验证。

---

## 29.4 ROS2 的接口解耦真正落地

今天第一次真实体验到：

```text
FakeLidarNode ─┐
               ├→ /scan → Algorithm Node
Real RPLIDAR ──┘
```

只要 ROS2 Interface 保持一致，后续算法可以复用。

---

# 30. 今日遇到的问题与解决方法

## 问题 1：Windows 不识别 CP2102

原因：

```text
CP210x VCP Driver 未安装
```

解决：

```text
安装 Silicon Labs CP210x Driver
```

---

## 问题 2：WSL 进入错误发行版

进入了：

```text
docker-desktop
```

正确：

```powershell
wsl -d Ubuntu-22.04 -u glance
```

---

## 问题 3：没有 `/dev/ttyUSB0` 权限

原因：

```text
用户未进入 dialout
```

解决：

```bash
sudo usermod -aG dialout glance
```

重新进入 WSL 后生效。

---

## 问题 4：`sed -n 536,546p` 没有输出

原因：

```bash
ros2 topic echo /scan --field ranges
```

在当前 ROS2 Humble 中把整个数组输出成：

```text
array('f', [...])
```

一整行。

因此不能按“第 536 行”等方式截取。

后改用 Python 解析数组。

### 经验

不要假设 CLI 输出格式。

先看真实输出，再设计解析方法。

---

## 问题 5：直接猜 0.405 m 是杯子

原因：

```text
单帧数据不足以确认物体身份
```

解决：

```text
有杯子
↓
无杯子
↓
重新放回
```

三帧对照。

---

## 问题 6：电机退出 Driver 后重新旋转

现象：

```text
stop_motor
→ 停转

Ctrl+C 退出 Driver
→ 又开始旋转
```

结论：

```text
SDK 控制有效
```

但硬件默认旋转机制未确定。

当前先记录，不深入底层。

---

# 31. 当前明确知道的信息

## 雷达相关

```text
系列：高概率 RPLIDAR A1 / A1M8 系列
具体 R5 / R6：未严谨确认
```

设备返回：

```text
Firmware Ver: 1.29
Hardware Rev: 7
Health: OK
```

不能仅凭这些信息直接说死具体小版本。

---

## USB

```text
VID:PID = 10c4:ea60
CP210x / CP2102 USB-UART
Windows COM3
Linux /dev/ttyUSB0
```

---

## ROS2

```text
Package: sllidar_ros2
Executable: sllidar_node
Node: /sllidar_node
Topic: /scan
Message: sensor_msgs/msg/LaserScan
frame_id: laser
```

---

## 实际扫描

```text
scan mode: Sensitivity
sample rate: 8 kHz
max_distance: 12 m
实际 /scan: ≈ 7.26 Hz
```

QoS：

```text
RELIABLE
VOLATILE
```

---

# 32. 当前仍未解决的问题

1. RPLIDAR A1/A1M8 的具体硬件小版本尚未严格确认；
2. USB 一插入电机就自动旋转的底层硬件原因尚未确认；
3. Driver 退出后硬件恢复旋转的底层控制线/转接板行为尚未确认；
4. 尚未正式建立 TF2：
   ```text
   laser → base_link → odom → map
   ```
5. 尚未进入点云转换、聚类、RANSAC、ICP、SLAM 等算法阶段。

这些都不阻碍 Phase 1 完成。

---

# 33. Phase 1 能力验收

当前已经能够独立解释或操作：

- [x] Windows USB 设备识别
- [x] WSL2 USB 转发
- [x] `usbipd` attach
- [x] Linux `lsusb`
- [x] `/dev/ttyUSB0`
- [x] `dialout`
- [x] ROS2 Driver 选择
- [x] CMake / executable 对应关系
- [x] Launch 参数
- [x] Driver / SDK / Node 分层
- [x] `work_loop()`
- [x] `spin_some()`
- [x] `publish_scan()`
- [x] SDK nodes → LaserScan
- [x] `/scan`
- [x] LaserScan 字段
- [x] QoS
- [x] `ros2 topic hz`
- [x] Service callback
- [x] `/start_motor`
- [x] `/stop_motor`
- [x] 真实物理实验
- [x] rosbag record
- [x] rosbag info
- [x] rosbag play
- [x] RViz LaserScan 可视化
- [x] Fake LiDAR 与真实 Driver 的 ROS2 接口对比

---

# 34. 自测题

## Q1

为什么：

```text
lsusb 能看到雷达
```

仍然不能证明 ROS2 Driver 可以正常使用？

> 提示：从 USB → Linux device → 权限 → Driver 参数 → 串口通信逐层思考。

---

## Q2

`sllidar_node` 为什么既有：

```cpp
while(...)
```

又有：

```cpp
spin_some()
```

？

> 提示：一个负责持续硬件数据采集，一个负责 ROS2 callback 调度。

---

## Q3

`publish_scan()` 的本质是什么？

应能回答：

```text
输入：
SLAMTEC SDK nodes[]
+
运行参数

输出：
sensor_msgs/msg/LaserScan

最后：
Publisher.publish()
```

---

## Q4

为什么：

```text
scan_frequency = 10
```

不能直接说：

```text
真实雷达就是 10 Hz
```

？

---

## Q5

为什么第一次看到：

```text
ranges[i] = 0.405
```

不能直接断言：

```text
“这个点就是杯子”
```

？

---

## Q6

FakeLidarNode 和真实 RPLIDAR Driver 最大的共同点是什么？

> 答题关键词：
> ROS2 Interface、`/scan`、`LaserScan`、解耦。

---

# 35. 下一阶段入口

Phase 1 已完成。

下一阶段建议进入：

```text
P2
LaserScan
↓
极坐标
↓
(x, y)
↓
二维点集
```

其中最基础的数学关系将是：

\[
\theta_i
=
\text{angle\_min}
+
i\cdot\text{angle\_increment}
\]

以及：

\[
x_i=r_i\cos\theta_i
\]

\[
y_i=r_i\sin\theta_i
\]

但当前学习记录到此结束，不自动进入 P2。

---

# 36. 一句话总结

今天真正完成的不是“把雷达插上并看到几个红点”，而是已经能够沿着：

```text
真实硬件
→ 操作系统设备
→ SDK
→ ROS2 Driver
→ LaserScan
→ /scan
→ rosbag
→ RViz
```

解释一条完整的真实机器人传感器数据链，并通过终端、源码和物理实验对它进行了逐层验证。
