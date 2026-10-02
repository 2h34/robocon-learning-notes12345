# 2026-10-02 ROS2 + C++ 学习记录  
## Week 1 · Day 4：ROS2 传感器消息 —— 从 `std_msgs` 进入 `sensor_msgs`

> 学习主题：  
> **Message Structure / `sensor_msgs` / `Header` / `stamp` / `frame_id` / `LaserScan`**
>
> 今日目标不是深入 TF2、SLAM、IMU 数学或图像处理，而是建立一个统一认识：
>
> **机器人传感器数据不仅包含“测到了什么”，还必须包含“什么时候测到”和“在哪个坐标系下测到”。**

---

# 1. 今日学习背景与主线

当前环境：

```text
Windows
→ WSL2
→ Ubuntu 22.04
→ ROS2 Humble
```

开发方式：

```text
VS Code + WSL
```

Workspace：

```text
~/ros2_ws
```

Package：

```text
hello_ros2
```

Day 1～Day 3 已经掌握：

- Workspace / Package / Executable / Node
- `rclcpp::init()`
- 创建 Node
- 构造函数
- `rclcpp::spin()`
- Executor
- callback
- Publisher / Subscriber
- Topic
- `create_publisher()`
- `publish()`
- `create_subscription()`
- Timer
- ROS2 Graph
- `ros2 node list`
- `ros2 node info`
- `ros2 topic list`
- `ros2 topic info`
- `ros2 topic echo`
- `ros2 topic hz`

Day 4 的核心变化是：

```text
Day 3：
Publisher
→ Topic
→ Subscriber

Day 4：
Publisher
→ Message
→ Topic
→ Message
→ Subscriber
```

Day 3 主要关注：

> 数据怎么传？

Day 4 开始关注：

> **传的到底是什么数据？**

---

# 2. 开始前自测

## 2.1 Message type

问题：

如果 `/scan` Topic 传输二维激光雷达数据，它的完整 Message type 是什么？

我的回答：

```text
sensor_msgs/msg/LaserScan
```

正确。

---

## 2.2 Publisher / Subscriber 数据链

我的回答：

```text
Publisher
→ publish()
→ Topic
→ 消息到达 Subscriber
→ callback ready
→ Executor 调度
→ callback 执行
```

正确。

需要继续保持 Day 3 建立的严格顺序：

```text
事件发生
→ callback ready
→ Executor 调度
→ callback 执行
```

---

# 3. Part A：为什么真实传感器不能只使用 `std_msgs/msg/String`

## 3.1 `String` 的结构

```text
std_msgs/msg/String
```

本质上只有：

```text
string data
```

例如：

```text
data: "Hello"
```

它适合表达：

- 文本
- 简单状态
- 调试信息

但不适合表达真实传感器的一整套结构化数据。

---

## 3.2 一帧 2D LiDAR 数据需要表达什么？

至少需要：

```text
一帧扫描
│
├── 从哪个角度开始？
├── 扫到哪个角度结束？
├── 相邻两束之间差多少角度？
├── 每个方向测到了多远？
├── 扫描是什么时候发生的？
└── 数据在哪个坐标系下表达？
```

因此，仅仅发送：

```text
"1.2, 1.3, 1.5, 2.0"
```

是不够的。

我当时的理解：

> 不能，因为并不知道这些数据对应什么含义，只知道一串数字。

进一步准确地说：

> 仅有字符串中的数字，并不能知道每个数字对应的角度、单位、测量时间和参考坐标系。

---

## 3.3 Message 的核心理解

今天建立的一个重要认识：

> **ROS2 Message 是 Node 之间约定的数据结构 / 数据接口。**

它不仅可以装一个值，也可以是一套有明确：

- 字段名
- 字段类型
- 数组
- 嵌套 Message

的数据结构。

---

# 4. Part B：学会使用 `ros2 interface show`

以后遇到陌生 Message，第一反应不应只是上网搜索，而应该先：

```bash
ros2 interface show <message_type>
```

例如：

```bash
ros2 interface show std_msgs/msg/String
```

```bash
ros2 interface show sensor_msgs/msg/LaserScan
```

```bash
ros2 interface show std_msgs/msg/Header
```

它可以帮助判断：

- 字段名
- 字段类型
- 是否为数组
- 是否嵌套其他 Message

---

## 4.1 三类常见字段

### 普通字段

```text
float32 angle_min
```

表示：

```text
类型      字段名
float32   angle_min
```

---

### 数组字段

```text
float32[] ranges
```

表示：

> `ranges` 是一组 `float32`，不是一个单独的距离值。

---

### 嵌套 Message

```text
std_msgs/Header header
```

表示：

> `header` 本身又是另一个 Message。

继续展开：

```text
Header
├── stamp
└── frame_id
```

---

# 5. Part C：`sensor_msgs/msg/LaserScan`

我实际运行：

```bash
ros2 interface show sensor_msgs/msg/LaserScan
```

得到的核心结构为：

```text
std_msgs/Header header

float32 angle_min
float32 angle_max
float32 angle_increment

float32 time_increment
float32 scan_time

float32 range_min
float32 range_max

float32[] ranges
float32[] intensities
```

---

# 6. 一帧二维 LiDAR 扫描的物理图像

二维 LiDAR 可以粗略理解为：

```text
                 第 2 束
                   ↑
                   │
          第 1 束  │  第 3 束
               \   │   /
                \  │  /
                 \ │ /
                [ LiDAR ]
```

雷达从某个起始角开始，以一定角度间隔进行测量，每个方向得到一个距离。

---

# 7. `angle_min / angle_max / angle_increment`

三个字段分别表示：

```text
angle_min
→ 扫描起始角

angle_max
→ 扫描结束角

angle_increment
→ 相邻两次测量之间的角度间隔
```

单位都是：

```text
rad
```

也就是弧度。

---

## 7.1 `ranges[i]` 对应哪个方向？

核心关系：

\[
\theta_i
=
\text{angle\_min}
+
i\cdot\text{angle\_increment}
\]

例如：

```text
angle_min       = -1.0 rad
angle_increment =  0.5 rad
```

那么：

```text
i = 0 → -1.0 rad
i = 1 → -0.5 rad
i = 2 →  0.0 rad
i = 3 →  0.5 rad
i = 4 →  1.0 rad
```

如果：

```text
ranges = [1.2, 0.8, 2.0, 1.5, 3.1]
```

则对应：

```text
-1.0 rad → 1.2 m
-0.5 rad → 0.8 m
 0.0 rad → 2.0 m
 0.5 rad → 1.5 m
 1.0 rad → 3.1 m
```

所以：

> **`ranges[i]` 本身只是距离；它的方向要结合 `angle_min` 和 `angle_increment` 才能确定。**

---

## 7.2 自测

已知：

```text
angle_min       = -1.2 rad
angle_increment =  0.2 rad
ranges[4]       =  2.6 m
```

我计算：

\[
\theta_4=-1.2+4\times0.2=-0.4\text{ rad}
\]

所以：

```text
角度 = -0.4 rad
距离 = 2.6 m
```

正确。

---

# 8. LaserScan 坐标方向

接口注释指出：

- 角度绕 `+Z` 轴测量
- 当 `+Z` 朝上时，正方向为逆时针
- `0 rad` 沿 `+X`

俯视可理解为：

```text
                 +Y
                  ↑
          +90°    │
                  │
180°  ←──────── [LiDAR] ───────→ +X
                  │              0°
                  │
          -90°    ↓
                 -Y
```

因此：

```text
0 rad      ≈ +X
+π/2 rad   ≈ +Y
-π/2 rad   ≈ -Y
```

但这里的 `+X / +Y / +Z` 属于哪个坐标系，要由：

```text
header.frame_id
```

决定。

---

# 9. `range_min / range_max`

这两个字段表示有效测距范围：

```text
range_min
→ 最小有效距离

range_max
→ 最大有效距离
```

它们不是：

> 当前这一帧中实际测到的最近和最远障碍物。

例如：

```text
range_min = 0.1 m
range_max = 12 m
ranges[20] = 15 m
```

我的判断：

> 不能直接把 `15 m` 当作有效障碍物距离，因为已经超出了 `range_max`。

正确。

---

# 10. `time_increment / scan_time`

两个字段虽然都与时间有关，但含义不同。

## 10.1 `time_increment`

表示：

> 一帧内部，相邻测量之间的时间间隔。

例如：

```text
第0束   第1束   第2束   第3束
  │       │       │       │
  ●───────●───────●───────●──→ 时间
      Δt      Δt      Δt
```

---

## 10.2 `scan_time`

表示：

> 相邻两帧扫描之间的时间间隔。

例如：

```text
scan_time = 0.5 s
```

则大约：

```text
2 Hz
```

---

## 10.3 当前阶段的理解边界

知道即可：

> 当雷达在运动时，一帧内部不同激光束实际采样时间不同，`time_increment` 后续可用于处理这种时间差。

Day 4 暂不深入：

- LiDAR 运动畸变
- 去畸变
- 时间同步算法

---

# 11. `intensities[]`

```text
float32[] intensities
```

表示每束回波的强度信息。

接口说明强调：

```text
device-specific units
```

即：

> 不同设备对强度的定义可能不同。

如果设备不提供强度：

```text
intensities = []
```

可以为空。

---

# 12. Part D：`Header`

LaserScan 顶部：

```text
std_msgs/Header header
```

其结构：

```text
Header
│
├── stamp
└── frame_id
```

今天最重要的统一认识：

> **真实机器人传感器数据 = 数值 + 时间 + 空间**

也就是说，一条传感器数据不能只回答：

> 测到了什么？

还要回答：

> 什么时候测到？

和：

> 在哪个坐标系下测到？

---

# 13. Part E：`stamp`

`header.stamp` 表示：

> **该条 / 该帧传感器数据所对应的测量 / 采集时间。**

不能简单理解为：

> Subscriber callback 执行的时间。

---

## 13.1 测量时间和处理时间的区别

```text
t1
LiDAR 完成测量 / 扫描
        ↓
Driver 创建 LaserScan
        ↓
USB / 网络 / ROS2 传输
        ↓
消息进入 Subscriber
        ↓
callback ready
        ↓
Executor 调度
        ↓
t2
callback 执行
```

其中：

```text
t1
→ measurement / acquisition time
→ 测量 / 采集时间

t2
→ receive / processing time
→ 接收 / 处理时间
```

而：

```text
header.stamp
```

通常应该尽可能描述：

```text
t1
```

---

## 13.2 为什么时间戳重要？

例如：

```text
LiDAR.header.stamp = 20.000 s
IMU.header.stamp   = 20.002 s
```

若判断两条数据在时间上是否接近，应比较：

```text
两条消息的 header.stamp
```

而不是比较两个 callback 什么时候执行。

我的回答：

> B，因为是要判断是不是同一时刻进行的测量。

正确。

---

## 13.3 工程上不能说死的一点

真实 Driver 中：

```text
header.stamp
```

具体是什么时间点，需要看：

- 设备本身是否提供硬件时间戳
- Driver 如何赋值
- 是扫描开始、结束还是其他采集时间定义

因此：

> **真实传感器的时间戳语义需要查具体 Driver 文档或源码。**

不能仅凭 Message type 猜测。

---

# 14. `builtin_interfaces/msg/Time`

`stamp` 内部：

```text
builtin_interfaces/Time stamp
    int32 sec
    uint32 nanosec
```

可以理解为：

```text
sec
→ 整秒

nanosec
→ 不足一秒的纳秒部分
```

例如：

```text
sec     = 100
nanosec = 250000000
```

对应：

\[
100.25\text{ s}
\]

Day 4 不深入：

- ROS Time
- System Time
- Steady Time

---

# 15. Part F：`frame_id`

如果：

```text
header.frame_id = "laser"
```

表示：

> **这一帧数据是在 `laser` 坐标系下表达的。**

例如 `LaserScan` 中：

```text
angle = 0
```

表示的是：

> `laser` 坐标系的 `+X` 方向。

---

## 15.1 `frame_id` 与 Node / Topic 的区别

例如：

```text
Node：
lidar_driver

Topic：
/scan

Message type：
sensor_msgs/msg/LaserScan

frame_id：
laser
```

四者分别表示：

```text
lidar_driver
→ 哪个 Node 在运行

/scan
→ 数据通过哪条 Topic 传输

sensor_msgs/msg/LaserScan
→ 数据结构是什么

laser
→ 数据在哪个坐标系下表达
```

因此必须记住：

```text
frame_id ≠ Topic 名
frame_id ≠ Node 名
```

---

## 15.2 我的自测回答

问题：

`ranges[100]` 的方向相对于哪个参考系定义？

我的回答：

> 相对于 `laser` 坐标系，因为 `header.frame_id = laser`。

正确。

---

# 16. 为什么后面需要 TF2

当前只建立问题，不深入。

例如：

```text
LaserScan
frame_id = laser
        ↓
如何转换？
        ↓
base_link
        ↓
以后还可能到 odom / map
```

因此：

> 当不同数据处于不同坐标系时，需要知道坐标系之间的变换关系。

这就是后续学习 TF2 的直接动机。

Day 4 不深入：

- TF tree
- `map / odom / base_link`
- 坐标变换数学

---

# 17. Part G：`sensor_msgs/msg/Imu`

我实际运行：

```bash
ros2 interface show sensor_msgs/msg/Imu
```

主要结构：

```text
std_msgs/Header header

geometry_msgs/Quaternion orientation
float64[9] orientation_covariance

geometry_msgs/Vector3 angular_velocity
float64[9] angular_velocity_covariance

geometry_msgs/Vector3 linear_acceleration
float64[9] linear_acceleration_covariance
```

---

# 18. IMU 的三组主要数据

```text
orientation
→ 姿态

angular_velocity
→ 角速度

linear_acceleration
→ 线加速度
```

我一开始回答：

> 角速度和加速度，因为之前 ESKF 项目中接触过。

这个回答对应的是 IMU 最基本的惯性观测。

ROS2 的标准 `Imu` Message 还提供：

```text
orientation
```

字段。

---

## 18.1 四元数 `orientation`

接口中：

```text
geometry_msgs/Quaternion orientation
    float64 x
    float64 y
    float64 z
    float64 w
```

所以姿态通过四元数表示：

\[
q=(x,y,z,w)
\]

默认：

```text
x = 0
y = 0
z = 0
w = 1
```

即单位四元数。

这与之前 15D ESKF 中 nominal state 的姿态表示可以对应起来。

---

## 18.2 一个重要区分

陀螺仪直接测的是：

```text
angular_velocity
→ 角速度 ω
```

不是：

```text
orientation
```

`orientation` 是姿态估计结果，可能来自：

- 设备内部算法
- 姿态融合算法
- 其他驱动或算法

因此：

> **陀螺仪测角速度，姿态通常是通过积分或融合得到。**

---

## 18.3 covariance

Message 中还有：

```text
orientation_covariance
angular_velocity_covariance
linear_acceleration_covariance
```

我之前在 ESKF 中已经学过 covariance。

Day 4 这里只建立：

> ROS2 Message 不只能传测量值，也可以传递测量不确定性。

今天不重新进入 ESKF 协方差数学。

---

# 19. Part G：`sensor_msgs/msg/Image`

我实际运行：

```bash
ros2 interface show sensor_msgs/msg/Image
```

主要结构：

```text
std_msgs/Header header

uint32 height
uint32 width

string encoding

uint8 is_bigendian
uint32 step
uint8[] data
```

---

# 20. Image 主要字段

## `height`

```text
图像行数
```

## `width`

```text
图像列数
```

例如：

```text
height = 480
width = 640
```

即：

```text
640 × 480
```

---

## `encoding`

告诉接收者：

> `data[]` 中的字节应该按什么像素格式解释。

例如未来可能遇到：

```text
rgb8
bgr8
mono8
```

今天不要求记具体格式。

我的理解：

> 如果只有 `data[]` 而没有 `encoding`，就只有原始数据，不知道应该按什么像素格式解释。

正确。

---

## `step`

```text
Full row length in bytes
```

表示：

> 图像一整行占多少字节。

---

## `data[]`

是真正的图像字节数据。

接口说明：

```text
data size = step * rows
```

可以理解为：

\[
\text{data.size}
=
\text{step}\times\text{height}
\]

---

## `is_bigendian`

表示底层多字节数据的字节序。

Day 4 仅认识，不深入大小端原理。

---

# 21. Image 中的 Header

接口说明：

```text
Header timestamp should be acquisition time of image
```

所以：

```text
stamp
→ 图像采集时间
```

不是：

```text
callback 执行时间
```

`frame_id` 应对应相机光学坐标系。

接口中还给出了典型 optical frame 方向：

```text
+x → 图像右方
+y → 图像下方
+z → 指向图像平面内部 / 观察方向
```

今天不进入相机模型。

---

# 22. 三种传感器 Message 的统一认识

```text
LaserScan
│
├── header
└── ranges / angles
       ↓
    距离观测
```

```text
Imu
│
├── header
├── orientation
├── angular_velocity
└── linear_acceleration
       ↓
    运动相关观测
```

```text
Image
│
├── header
├── width / height
├── encoding
└── data[]
       ↓
    图像像素数据
```

因此：

```text
不同传感器
      ↓
主体数据完全不同
      ↓
使用不同 Message type
      ↓
很多 sensor_msgs 都包含 Header
      ↓
stamp + frame_id
```

最终统一为：

> **传感器数据 = 数值 + 时间 + 空间**

---

# 23. Part H：C++ 中使用 `sensor_msgs`

从 Day 3 的：

```cpp
rclcpp::Publisher<std_msgs::msg::String>::SharedPtr publisher_;
```

迁移到：

```cpp
rclcpp::Publisher<sensor_msgs::msg::LaserScan>::SharedPtr publisher_;
```

核心变化只是：

```text
std_msgs::msg::String
        ↓
sensor_msgs::msg::LaserScan
```

Publisher 的通信框架没有改变。

---

# 24. Message 定义与 C++ 字段的对应

通过：

```bash
ros2 interface show sensor_msgs/msg/LaserScan
```

可以知道 Message 有：

```text
angle_min
angle_max
ranges
header
...
```

C++ 中：

```cpp
sensor_msgs::msg::LaserScan message;

message.angle_min = ...;
message.angle_max = ...;
message.ranges = ...;
message.header.frame_id = ...;
```

因此：

```text
Message interface
        ↓
C++ Message object
        ↓
message.xxx
```

这是今天很重要的工程阅读能力。

---

# 25. Part I：Fake LaserScan 实验

创建：

```text
~/ros2_ws/src/hello_ros2/src/fake_lidar_node.cpp
```

最终核心代码：

```cpp
#include <chrono>
#include <functional>
#include <memory>

#include "rclcpp/rclcpp.hpp"
#include "sensor_msgs/msg/laser_scan.hpp"

using namespace std::chrono_literals;

class FakeLidarNode : public rclcpp::Node
{
public:
    FakeLidarNode()
    : Node("fake_lidar")
    {
        publisher_ =
            this->create_publisher<sensor_msgs::msg::LaserScan>(
                "/scan", 10);

        timer_ =
            this->create_wall_timer(
                500ms,
                std::bind(&FakeLidarNode::timer_callback, this));
    }

private:
    void timer_callback()
    {
        sensor_msgs::msg::LaserScan message;

        message.header.stamp = this->now();
        message.header.frame_id = "laser";

        message.angle_min = -1.0;
        message.angle_max = 1.0;
        message.angle_increment = 0.5;

        message.time_increment = 0.125;
        message.scan_time = 0.5;

        message.range_min = 0.1;
        message.range_max = 10.0;

        message.ranges = {
            1.2,
            0.8,
            2.0,
            1.5,
            3.1
        };

        publisher_->publish(message);

        RCLCPP_INFO(
            this->get_logger(),
            "Published fake LaserScan with %zu ranges",
            message.ranges.size());
    }

    rclcpp::Publisher<sensor_msgs::msg::LaserScan>::SharedPtr publisher_;
    rclcpp::TimerBase::SharedPtr timer_;
};

int main(int argc, char * argv[])
{
    rclcpp::init(argc, argv);

    auto node = std::make_shared<FakeLidarNode>();

    rclcpp::spin(node);

    rclcpp::shutdown();

    return 0;
}
```

注意：

> 初版代码里使用了 `std::bind()`，之后补充了：

```cpp
#include <functional>
```

这是一次很小但有价值的工程检查。

---

# 26. Fake LaserScan 数据设计

```text
angle_min       = -1.0
angle_max       =  1.0
angle_increment =  0.5
```

```text
ranges =
[1.2, 0.8, 2.0, 1.5, 3.1]
```

对应：

```text
ranges[0] → -1.0 rad → 1.2 m
ranges[1] → -0.5 rad → 0.8 m
ranges[2] →  0.0 rad → 2.0 m
ranges[3] →  0.5 rad → 1.5 m
ranges[4] →  1.0 rad → 3.1 m
```

Timer：

```text
500 ms
```

所以约：

```text
2 Hz
```

同时：

```text
scan_time = 0.5 s
```

整体基本自洽。

---

# 27. `this->now()` 在 Fake Node 中的作用

代码：

```cpp
message.header.stamp = this->now();
```

自测答案：

> B：给这帧模拟 `LaserScan` 填入时间戳。

这里再次强调：

> Fake Node 中用当前 ROS 时间模拟测量时间；真实 Driver 如何打时间戳必须查看具体实现。

---

# 28. CMakeLists.txt 修改

由于使用：

```cpp
#include "sensor_msgs/msg/laser_scan.hpp"
```

需要新增：

```cmake
find_package(sensor_msgs REQUIRED)
```

以及：

```cmake
add_executable(fake_lidar_node src/fake_lidar_node.cpp)

ament_target_dependencies(
  fake_lidar_node
  rclcpp
  sensor_msgs
)
```

并安装 executable：

```cmake
install(TARGETS
  fake_lidar_node
  DESTINATION lib/${PROJECT_NAME}
)
```

如果原来已有其他 executable，则把：

```text
fake_lidar_node
```

加入已有 `install(TARGETS ...)`。

---

# 29. `package.xml` 修改

新增：

```xml
<depend>sensor_msgs</depend>
```

我的理解：

> 因为这里使用了一种新的 Message type，它属于新的 ROS2 package，所以构建系统需要新增依赖。

准确表述：

> `sensor_msgs/msg/LaserScan` 属于 `sensor_msgs` package，因此编译和运行该 Node 时需要声明对 `sensor_msgs` 的依赖。

---

# 30. 编译与运行

执行：

```bash
cd ~/ros2_ws
colcon build --packages-select hello_ros2
```

然后：

```bash
source install/setup.bash
```

运行：

```bash
ros2 run hello_ros2 fake_lidar_node
```

实际运行结果：

> 正确，符合预期。

---

# 31. Fake LaserScan Node 的运行时流程

```text
Timer 到期
    ↓
timer_callback ready
    ↓
Executor 调度
    ↓
timer_callback()
    ↓
创建 sensor_msgs/msg/LaserScan
    ↓
填写 header
填写 angle
填写 range
填写 ranges[]
    ↓
publisher_->publish(message)
    ↓
消息进入 /scan Topic
```

---

# 32. Part J：CLI 验证 `/scan`

使用：

```bash
ros2 topic list
```

```bash
ros2 topic info /scan
```

```bash
ros2 topic echo /scan
```

```bash
ros2 topic hz /scan
```

观察内容包括：

- Topic
- Message type
- Publisher count
- Subscription count
- Message 字段
- frame_id
- ranges
- 发布频率

---

# 33. `Subscription count: 0`

假设：

```text
Type: sensor_msgs/msg/LaserScan
Publisher count: 1
Subscription count: 0
```

我的判断：

> 不是 Publisher 出问题，因为我们目前没有创建订阅 `/scan` 的 Subscriber。

正确。

---

# 34. `ros2 topic echo /scan`

可以直接看到运行中的 Message：

```text
header:
  stamp:
    sec: ...
    nanosec: ...
  frame_id: laser

angle_min: -1.0
angle_max: 1.0
angle_increment: 0.5

time_increment: 0.125
scan_time: 0.5

range_min: 0.1
range_max: 10.0

ranges:
- 1.2
- 0.8
- 2.0
- 1.5
- 3.1

intensities: []
```

这建立了完整链路：

```text
ros2 interface show
        ↓
Message 定义
        ↓
C++ Message 对象
        ↓
message.xxx
        ↓
publish()
        ↓
Topic
        ↓
ros2 topic echo
        ↓
看到运行时真实消息
```

---

# 35. `ros2 topic hz /scan`

由于：

```text
Timer = 500 ms
```

理论上：

```text
frequency ≈ 2 Hz
```

实际观察：

> 符合预期。

---

# 36. Part K：陌生 `sensor_msgs` Node 阅读任务

阅读的 Node：

```text
SimpleImuPublisher
```

关键代码包括：

```cpp
Node("simple_imu_publisher")
```

```cpp
create_publisher<sensor_msgs::msg::Imu>(
    "/imu/data", 10);
```

```cpp
create_wall_timer(100ms, ...)
```

以及：

```cpp
msg.header.stamp = this->now();
msg.header.frame_id = "imu_link";

msg.angular_velocity.z = 0.1;

msg.linear_acceleration.z = 9.8;
```

---

# 37. 我的分析

## 1. Node 名

```text
simple_imu_publisher
```

正确。

## 2. Topic

```text
/imu/data
```

正确。

## 3. Message type

```text
sensor_msgs/msg/Imu
```

正确。

## 4. Timer

```text
100 ms
→ 约 10 Hz
```

正确。

## 5. callback

```text
timer_callback()
```

正确。

## 6. 主要字段

我回答：

```text
stamp
frame_id
angular_velocity
linear_acceleration
```

进一步应分类为：

```text
Header 元数据：
- stamp
- frame_id

IMU 主体数据：
- angular_velocity
- linear_acceleration
```

所以如果问：

> 主要发布了哪些 IMU 数据？

更准确答：

```text
angular_velocity
linear_acceleration
```

---

## 7. `stamp`

我的回答：

> 测量时间。

正确。

---

## 8. `frame_id`

我的回答：

> 当前测量的参考坐标系。

概念正确。

代码中的具体值是：

```text
imu_link
```

所以更完整：

> 这组 IMU 数据在 `imu_link` 坐标系下表达。

---

## 9. 运行时数据流

我当时回答：

```text
publisher → topic → subscriber
```

这个回答太宏观，而且该代码实际上没有 Subscriber。

更准确应该是：

```text
Timer 到期
    ↓
timer_callback ready
    ↓
Executor 调度
    ↓
timer_callback()
    ↓
创建 sensor_msgs/msg/Imu
    ↓
填写 header
填写 angular_velocity
填写 linear_acceleration
    ↓
publisher_->publish(msg)
    ↓
消息发布到 /imu/data
```

如果以后存在 Subscriber，才继续：

```text
消息到达 Subscription
    ↓
subscriber callback ready
    ↓
Executor 调度
    ↓
subscriber callback 执行
```

这是今天最后一个需要继续保持严格的点。

---

# 38. 今日最重要的知识主线

可以把 Day 4 压缩成下面一条链：

```text
机器人传感器
      ↓
产生某种物理观测
      ↓
ROS2 使用明确的 Message type 表达
      ↓
Message 包含结构化字段
      ↓
很多传感器 Message 带 Header
      ↓
stamp + frame_id
      ↓
数值 + 时间 + 空间
      ↓
Publisher 发布到 Topic
      ↓
Subscriber / CLI / 后续算法读取
```

---

# 39. 今天形成的几个关键认识

## 39.1 Message 不只是“一个值”

Message 可以是一套复杂数据接口。

例如：

```text
LaserScan
```

包含：

- 角度
- 时间
- 有效测距范围
- 距离数组
- 强度
- Header

---

## 39.2 `ranges[]` 不是孤立数组

必须结合：

```text
angle_min
angle_increment
```

才能知道每个元素对应哪个方向。

---

## 39.3 Header 不是附加信息

它解决：

```text
stamp
→ 时间问题

frame_id
→ 空间问题
```

所以：

> **传感器数据 = 数值 + 时间 + 空间**

---

## 39.4 `stamp` ≠ callback 时间

`stamp` 关注：

> 数据何时被测量 / 采集。

callback 时间关注：

> 程序何时处理。

两者不能混淆。

---

## 39.5 `frame_id` ≠ Topic ≠ Node

```text
Node
→ 谁在运行

Topic
→ 数据走哪条通信通道

Message type
→ 数据结构是什么

frame_id
→ 数据属于哪个空间参考系
```

---

## 39.6 ROS2 通信框架没有因为 Message 复杂而改变

```text
Publisher<String>
```

变为：

```text
Publisher<LaserScan>
```

核心只是：

> Message type 变复杂了。

Publisher / Topic / callback / Executor 的整体框架没有变化。

---

# 40. 今天遇到的问题与修正

## 问题 1：Header 与主体传感器数据容易混在一起

在陌生 IMU Node 阅读任务中，我把：

```text
stamp
frame_id
angular_velocity
linear_acceleration
```

全部作为“主要 IMU 字段”一起回答。

更准确应区分：

```text
Header 元数据：
- stamp
- frame_id

主体观测：
- angular_velocity
- linear_acceleration
```

---

## 问题 2：描述数据流时过于简化

我写：

```text
publisher → topic → subscriber
```

问题：

- 当前代码未必存在 Subscriber
- 忽略了 Timer / callback / Executor
- 不够具体

以后应先按真实代码分析：

```text
事件
→ callback ready
→ Executor
→ callback
→ Message
→ publish
→ Topic
```

如果存在 Subscriber，再继续分析接收链。

---

## 问题 3：Fake LaserScan 初版缺少 `<functional>`

由于使用：

```cpp
std::bind(...)
```

应该包含：

```cpp
#include <functional>
```

后续已经补充。

这也提醒我：

> 即使整体架构正确，仍要检查具体 C++ 依赖是否完整。

---

# 41. 今日完成标准检查

## Message 基础

- [x] 理解 Message 是 Node 之间的数据结构 / 数据接口
- [x] 会使用 `ros2 interface show`
- [x] 知道如何查看陌生 Message

## LaserScan

- [x] 理解一帧二维 LiDAR 扫描的物理意义
- [x] 理解 `angle_min`
- [x] 理解 `angle_max`
- [x] 理解 `angle_increment`
- [x] 理解 `ranges[]`
- [x] 会使用

\[
\theta_i
=
\text{angle\_min}
+
i\cdot\text{angle\_increment}
\]

- [x] 理解 `range_min / range_max`
- [x] 初步理解 `time_increment / scan_time`
- [x] 知道 `intensities[]` 的基本作用

## Header

- [x] 理解 Header 的作用
- [x] 理解 `header.stamp`
- [x] 能区分测量时间与 callback 处理时间
- [x] 理解 `header.frame_id`
- [x] 知道 `frame_id ≠ Topic`
- [x] 知道 `frame_id ≠ Node`
- [x] 知道后续为什么需要 TF2

## 其他 sensor_msgs

- [x] 查看 `sensor_msgs/msg/Imu`
- [x] 查看 `sensor_msgs/msg/Image`
- [x] 理解不同传感器主体数据不同
- [x] 理解很多传感器 Message 都带 Header

## C++ 实践

- [x] 从 `Publisher<String>` 迁移理解到 `Publisher<LaserScan>`
- [x] 创建 Fake LaserScan Publisher
- [x] 增加 `sensor_msgs` 依赖
- [x] 编译成功
- [x] 运行成功

## CLI

- [x] `ros2 topic list`
- [x] `ros2 topic info`
- [x] `ros2 topic echo`
- [x] `ros2 topic hz`
- [x] 验证 `/scan`
- [x] 验证 Message type
- [x] 验证频率
- [x] 验证 `frame_id`
- [x] 验证 `ranges`

## 代码阅读

- [x] 能阅读陌生 `sensor_msgs/msg/Imu` Publisher
- [x] 能判断 Node / Topic / Message type
- [x] 能判断 Timer / callback
- [x] 能识别主要 Message 字段
- [x] 能解释 `stamp`
- [x] 能解释 `frame_id`
- [x] 基本能描述运行时数据流
- [ ] 运行时数据流描述还需要继续保持完整和严格

---

# 42. Day 4 最终掌握情况

## 已经掌握

目前已经能够比较稳定地理解：

```text
Message
sensor_msgs
LaserScan
Header
stamp
frame_id
Imu
Image
```

并能把它们连接到：

```text
C++ Message object
→ Publisher
→ Topic
→ ROS2 Graph
→ CLI
```

---

## 仍需保持的两个细节

### 1. 区分元数据与主体观测

```text
Header
→ 时间 / 空间元数据

ranges / angular_velocity / image data
→ 传感器主体观测
```

### 2. 运行时数据流必须基于真实代码

不要机械套：

```text
publisher → topic → subscriber
```

而应看当前代码实际有哪些：

```text
Timer
Publisher
Subscriber
callback
```

再描述完整流程。

---

# 43. Day 4 结论

**Day 4 主线完成。**

本日最核心的知识跃迁是：

```text
Day 3：
知道 ROS2 数据怎么传

        ↓

Day 4：
开始理解 ROS2 到底在传什么
```

并形成：

> **Message = Node 之间明确的数据接口**

以及：

> **机器人传感器数据 = 数值 + 时间 + 空间**

这两条主线。

---

# 44. 下一阶段位置

按照当前学习路线，Day 5 可以进入：

```text
QoS
+
ROS2 数据流调试工具
+
RViz / rosbag 初步
```

但在进入 Day 5 前，不需要重新复习 Day 4 全部内容。

下一次开始时只需要快速确认：

1. `Message type` 是什么  
2. `Header` 中 `stamp / frame_id` 的作用  
3. `LaserScan` 中 `ranges[i]` 如何对应角度  
4. 当前通信链是否能从代码与 Graph 两边互相反推

即可继续推进。

---

# 45. 一页速记版

```text
ROS2 Message
= Node 之间的数据接口

LaserScan
├── header
│   ├── stamp      → 什么时候测的
│   └── frame_id   → 在哪个坐标系下
├── angle_min
├── angle_max
├── angle_increment
├── time_increment
├── scan_time
├── range_min
├── range_max
├── ranges[]
└── intensities[]

θ_i = angle_min + i × angle_increment

ranges[i]
= θ_i 方向上的距离

Header
= 时间 + 空间

Sensor data
= 数值 + 时间 + 空间

Imu
├── header
├── orientation
├── angular_velocity
└── linear_acceleration

Image
├── header
├── width / height
├── encoding
├── step
└── data[]

C++：
Message object
→ 填字段
→ publish()
→ Topic
→ CLI / Subscriber

运行时事件：
事件发生
→ callback ready
→ Executor 调度
→ callback 执行
```
