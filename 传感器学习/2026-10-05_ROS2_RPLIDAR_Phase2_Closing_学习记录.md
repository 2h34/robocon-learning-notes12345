# ROS2 RPLIDAR Phase 2 Closing 学习记录

> 日期：2026-10-05  
> 主题：`LaserScan → 有效性过滤 → 极坐标 → 笛卡尔坐标 → Point2D`  
> 阶段结论：**P2 Closing PASS**

---

## 1. 本阶段为什么要做

Phase 1 已经完成了真实 RPLIDAR 的接入、`/scan` 发布、QoS 检查、rosbag2 录制与回放、RViz 可视化等工作。

Phase 2 的核心目标不是继续学习更多 ROS2 工具，而是把激光雷达真正转换成后续几何算法可以使用的数据：

```text
真实世界
↓
RPLIDAR
↓
sllidar_ros2
↓
sensor_msgs/msg/LaserScan
↓
有效性过滤
↓
极坐标 (r, θ)
↓
笛卡尔坐标 (x, y)
↓
二维点集合
```

此前已经用 Python `scan_to_points` 验证过 `LaserScan → PointCloud2` 的总体思路，但 P2 Closing 要求进一步完成一个自己能够读懂、解释、修改和验证的 C++ 几何节点。

因此本阶段重点不是“再做一个能运行的程序”，而是把以下能力真正掌握：

- C++ 中接收 `LaserScan`
- 判断距离数据是否有效
- 保留 LaserScan 原始采样下标
- 根据原始下标计算每束激光的角度
- 完成 `(r, θ) → (x, y)`
- 使用 `Point2D` 和 `std::vector<Point2D>` 保存一帧二维点
- 同一个节点同时接受 rosbag 和真实雷达数据
- 建立可重复使用的受控测试 bag
- 理解 360° 扫描的首尾环绕问题

---

# 2. 新建独立几何处理包

为了不继续把感知算法塞进 ROS2 基础练习包 `hello_ros2`，新建：

```text
lidar_geometry
```

创建命令：

```bash
cd ~/ros2_ws/src

ros2 pkg create lidar_geometry \
  --build-type ament_cmake \
  --dependencies rclcpp sensor_msgs
```

生成结构：

```text
lidar_geometry/
├── CMakeLists.txt
├── package.xml
├── include/
│   └── lidar_geometry/
└── src/
```

当前阶段没有为了“工程化”而提前拆分 `.hpp/.cpp`、工具类或复杂模块。

原因是当前任务很小，只需要：

```text
lidar_geometry/
├── CMakeLists.txt
├── package.xml
└── src/
    └── scan_geometry_node.cpp
```

这符合最小闭环原则。

---

## 2.1 为什么先编译一个空 package

创建后首先执行：

```bash
cd ~/ros2_ws

colcon build --packages-select lidar_geometry

source install/setup.bash

ros2 pkg prefix lidar_geometry
```

同时：

```bash
colcon list
```

能够识别：

```text
lidar_geometry  src/lidar_geometry  (ros.ament_cmake)
```

这一操作建立了一个“已知正常的构建基线”。

意义：

```text
package 可以被发现
↓
package.xml / CMakeLists.txt 基础配置正常
↓
package 可以独立构建
↓
安装结果可以被 ROS2 找到
```

这样以后每次增加代码后出现错误，就可以优先怀疑“刚刚增加的内容”，而不是重新检查整个 package。

这是控制变量式调试。

---

# 3. 第一个最小闭环：C++ 接收 LaserScan

首先只实现：

```text
/scan
↓
LaserScan Subscriber
↓
callback
↓
ranges.size()
```

暂时不加入任何几何算法。

核心节点结构：

```cpp
#include <functional>
#include <memory>

#include "rclcpp/rclcpp.hpp"
#include "sensor_msgs/msg/laser_scan.hpp"

class ScanGeometryNode : public rclcpp::Node
{
public:
    ScanGeometryNode()
    : Node("scan_geometry_node")
    {
        auto qos = rclcpp::QoS(rclcpp::KeepLast(10));
        qos.best_effort();

        subscription_ =
            this->create_subscription<sensor_msgs::msg::LaserScan>(
                "/scan",
                qos,
                std::bind(
                    &ScanGeometryNode::scan_callback,
                    this,
                    std::placeholders::_1));
    }

private:
    void scan_callback(
        const sensor_msgs::msg::LaserScan::SharedPtr msg)
    {
        RCLCPP_INFO(
            this->get_logger(),
            "raw_count = %zu",
            msg->ranges.size());
    }

    rclcpp::Subscription<sensor_msgs::msg::LaserScan>::SharedPtr
        subscription_;
};

int main(int argc, char * argv[])
{
    rclcpp::init(argc, argv);

    auto node = std::make_shared<ScanGeometryNode>();
    rclcpp::spin(node);

    rclcpp::shutdown();
    return 0;
}
```

---

## 3.1 raw_count 的含义

```cpp
msg->ranges.size()
```

表示这一帧 `LaserScan` 中原始 `ranges[]` 数组的元素数量。

定义：

```text
raw_count = ranges.size()
```

它不是“有效点数量”。

例如：

```text
ranges = [
    0.32,
    inf,
    0.41,
    nan,
    12.5,
    ...
]
```

所有元素都计入 `raw_count`。

实际 rosbag 测试中：

```text
raw=1080
```

说明一帧中有 1080 个原始扫描位置。

---

# 4. CMakeLists.txt：让 C++ 节点真正成为 ROS2 executable

在 `CMakeLists.txt` 中加入：

```cmake
add_executable(
  scan_geometry_node
  src/scan_geometry_node.cpp
)

ament_target_dependencies(
  scan_geometry_node
  rclcpp
  sensor_msgs
)

install(
  TARGETS scan_geometry_node
  DESTINATION lib/${PROJECT_NAME}
)
```

三个命令分别回答：

```text
add_executable
→ 哪个 .cpp 编译成程序

ament_target_dependencies
→ 这个程序依赖哪些 ROS2 package

install
→ 编译后的程序安装到哪里
```

编译：

```bash
cd ~/ros2_ws

colcon build --packages-select lidar_geometry
source install/setup.bash
```

检查：

```bash
ros2 pkg executables lidar_geometry
```

实际结果：

```text
lidar_geometry scan_geometry_node
```

说明 executable 已经被正确安装并能由 `ros2 run` 找到。

---

# 5. rosbag → C++ callback 验证

先运行：

```bash
ros2 run lidar_geometry scan_geometry_node
```

如果此时没有 `/scan` 消息，节点虽然已经存在，但 callback 不会执行，因此没有 `raw_count` 输出。

这让我进一步理解：

```text
Node 已经运行
≠
callback 正在执行
```

callback 需要消息到达，并由 Executor 调度后才执行。

随后播放已有真实雷达 bag：

```bash
ros2 bag play bags/rplidar_front_static
```

此时节点开始持续输出：

```text
raw_count = ...
```

说明运行时数据流已经成立：

```text
磁盘中的 rosbag
↓
rosbag2_player
↓
/scan
↓
C++ Subscriber
↓
Executor
↓
scan_callback()
```

---

# 6. 有效性过滤：raw / valid / rejected

下一步只加入过滤，不加入二维坐标计算。

增加：

```cpp
#include <cmath>
```

使用：

```cpp
std::isfinite(r)
```

判断数值是否为有限数。

例如：

```text
0.35   → finite
2.10   → finite
inf    → not finite
nan    → not finite
```

过滤逻辑：

```cpp
const std::size_t raw_count = msg->ranges.size();

std::size_t valid_count = 0;
std::size_t rejected_count = 0;

for (std::size_t i = 0; i < msg->ranges.size(); ++i)
{
    const float r = msg->ranges[i];

    if (!std::isfinite(r))
    {
        ++rejected_count;
        continue;
    }

    if (r < msg->range_min || r > msg->range_max)
    {
        ++rejected_count;
        continue;
    }

    ++valid_count;
}
```

---

## 6.1 rejected 的两种情况

一个 `range` 被拒绝主要有两类原因。

第一类：

```cpp
!std::isfinite(r)
```

例如：

```text
inf
-inf
nan
```

第二类：

```cpp
r < msg->range_min || r > msg->range_max
```

即距离不在传感器声明的有效量程内。

有效条件为：

\[
range_{min}\le r\le range_{max}
\]

因此：

```text
r == range_min  → valid
r == range_max  → valid
```

---

## 6.2 第一条重要不变量

程序要求：

\[
\boxed{
raw = valid + rejected
}
\]

实际 rosbag 输出：

```text
raw=1080 valid=879 rejected=201
raw=1080 valid=885 rejected=195
raw=1080 valid=876 rejected=204
raw=1080 valid=887 rejected=193
```

逐帧满足：

```text
879 + 201 = 1080
885 + 195 = 1080
876 + 204 = 1080
887 + 193 = 1080
```

说明过滤计数逻辑闭环成立。

---

# 7. 为什么一定保留原始 index

循环使用：

```cpp
for (std::size_t i = 0; i < msg->ranges.size(); ++i)
```

这里的 `i` 是 LaserScan 中该距离值的原始采样下标。

每一束激光的方向由：

\[
\boxed{
\theta_i
=
angle_{min}
+
i\cdot angle_{increment}
}
\]

决定。

因此：

> 一个距离值和它原始的 `i` 必须保持对应关系。

错误做法是先过滤得到：

```text
valid_ranges = [ ... ]
```

然后重新从：

```text
0, 1, 2, ...
```

编号。

例如：

```text
原始 index    range
0             1.0
1             inf
2             2.0
3             3.0
```

过滤以后：

```text
valid_ranges = [1.0, 2.0, 3.0]
```

此时：

```text
2.0
```

仍然来自原始：

```text
i = 2
```

如果重新编号，它会错误地使用：

```text
i = 1
```

于是本该计算：

\[
\theta_2
\]

却错误使用：

\[
\theta_1
\]

最终 `(x,y)` 方向也会错。

因此正确的数据流是：

```text
ranges[i]
↓
判断 r 是否有效
↓
如果有效，仍然使用原始 i
↓
计算 theta_i
```

而不是先把 `r` 与 `i` 分开。

---

# 8. 新知识：Point2D

定义：

```cpp
struct Point2D
{
    double x;
    double y;
};
```

`struct` 可以理解为定义一种新的数据结构。

这里：

```text
Point2D
├── x
└── y
```

表示一个二维点。

例如：

```cpp
Point2D point;
point.x = 1.2;
point.y = -0.4;
```

---

# 9. 新知识：std::vector<Point2D>

定义：

```cpp
std::vector<Point2D> points;
```

它表示一个能够动态增长、专门存储 `Point2D` 元素的顺序容器。

可以直观理解成：

```text
points = [
  Point2D{x0, y0},
  Point2D{x1, y1},
  Point2D{x2, y2},
  ...
]
```

它类似“动态数组”，但不是普通固定长度数组。

普通数组：

```cpp
Point2D points[100];
```

长度通常在创建时已经固定。

而：

```cpp
std::vector<Point2D> points;
```

初始可以为空：

```text
points.size() == 0
```

然后不断加入新元素。

---

## 9.1 push_back()

```cpp
points.push_back(point);
```

`push_back()` 是 `std::vector` 的成员函数。

作用：

> 把一个新元素加入 vector 的末尾。

例如：

```cpp
Point2D point;
point.x = 1.0;
point.y = 2.0;

points.push_back(point);
```

执行前：

```text
points = []
```

执行后：

```text
points = [
    {x=1.0, y=2.0}
]
```

因此：

```cpp
points.size()
```

表示当前保存了多少个二维点。

---

# 10. 极坐标 → 笛卡尔坐标

对于通过过滤的有效距离：

```cpp
const double theta =
    msg->angle_min + i * msg->angle_increment;

Point2D point;

point.x = r * std::cos(theta);
point.y = r * std::sin(theta);

points.push_back(point);

++valid_count;
```

数学关系：

\[
\boxed{x=r\cos\theta}
\]

\[
\boxed{y=r\sin\theta}
\]

所以完整数据流：

```text
ranges[i]
↓
r 是否有效
↓
保留原始 i
↓
theta = angle_min + i * angle_increment
↓
x = r cos(theta)
y = r sin(theta)
↓
Point2D
↓
points.push_back(point)
↓
vector<Point2D>
```

---

# 11. 第二条重要不变量

每一个有效 range 都应该转换成一个 `Point2D`。

因此：

\[
\boxed{
points.size() = valid\_count
}
\]

日志改成：

```cpp
RCLCPP_INFO(
    this->get_logger(),
    "raw=%zu valid=%zu rejected=%zu points=%zu",
    raw_count,
    valid_count,
    rejected_count,
    points.size());
```

实际运行结果：

```text
raw=1080 valid=880 rejected=200 points=880
raw=1080 valid=873 rejected=207 points=873
```

同时满足：

\[
raw = valid + rejected
\]

以及：

\[
points.size() = valid
\]

---

# 12. 用真实数值验证几何转换

为了避免只验证“数量正确”，程序临时打印每帧第一个有效点：

```cpp
if (valid_count == 0)
{
    RCLCPP_INFO(
        this->get_logger(),
        "sample: i=%zu r=%.3f theta=%.3f x=%.3f y=%.3f",
        i,
        r,
        theta,
        point.x,
        point.y);
}
```

这里：

```text
%zu
```

用于输出 `std::size_t` 类型。

```text
%.3f
```

用于显示浮点数，并保留三位小数。

---

## 12.1 实际样本 1

```text
i=1
r=0.411
theta=-3.136
x=-0.411
y=-0.002
```

因为：

\[
-\pi \approx -3.142
\]

所以：

```text
theta ≈ -π
```

方向接近负 x 轴。

预期：

\[
x\approx-r
\]

\[
y\approx0
\]

实际：

```text
x=-0.411
y=-0.002
```

符合预期。

---

## 12.2 实际样本 2

```text
i=0
r=0.415
theta=-3.142
x=-0.415
y=0.000
```

因为：

\[
\cos(-\pi)=-1
\]

\[
\sin(-\pi)=0
\]

所以：

\[
x=-0.415
\]

\[
y=0
\]

与程序完全一致。

---

# 13. 用 LaserScan 原始字段验证 theta

实际 `LaserScan`：

```text
angle_min:       -3.1415927410125732
angle_max:        3.1415927410125732
angle_increment:  0.005823156330734491
time_increment:   0.00012479702127166092
scan_time:        0.13465598225593567
range_min:        0.05000000074505806
range_max:        12.0
```

对于：

```text
i = 1
```

计算：

\[
\theta_1
=
-3.141592741
+
1\times0.005823156
\]

得到：

\[
\theta_1
=
-3.135769585
\]

保留三位小数：

\[
\boxed{
\theta_1=-3.136
}
\]

程序实际输出：

```text
theta=-3.136
```

说明：

```text
原始 LaserScan 字段
↓
原始 index i
↓
theta
```

的计算正确。

---

# 14. LaserScan 中其他字段的认识

## angle_min / angle_max

当前：

```text
angle_min ≈ -π
angle_max ≈ +π
```

因此这一帧扫描大约覆盖：

```text
-180° → +180°
```

即完整 360°。

---

## angle_increment

```text
angle_increment ≈ 0.005823 rad
```

约等于：

```text
0.334°
```

也就是相邻两个 `ranges[i]` 的扫描方向相差约 `0.334°`。

---

## time_increment

表示相邻激光测量之间的时间间隔。

当前 P2 不使用它。

以后如果机器人运动较快，一帧激光扫描期间机器人自身发生明显运动，就可能需要考虑扫描运动畸变。

---

## scan_time

表示一帧完整扫描对应的时间尺度。

当前阶段只需要知道含义，不进入运动补偿。

---

# 15. rosbag 与真实雷达使用同一个 geometry node

先前已经验证：

```text
rosbag2_player
↓
/scan
↓
scan_geometry_node
```

之后切换成：

```text
真实 RPLIDAR
↓
sllidar_ros2
↓
/scan
↓
scan_geometry_node
```

无需修改 geometry node 的算法代码。

原因：

> `scan_geometry_node` 是 `/scan` 的 Subscriber，它面向的是 Topic 接口，而不是某一个具体 Publisher。

只要：

```text
Topic 名正确
消息类型兼容
QoS 兼容
```

Publisher 可以是：

```text
sllidar_ros2
rosbag2_player
FakeLidar
```

geometry node 不需要知道数据源是谁。

这体现了 ROS2 Publisher / Subscriber 的解耦。

---

# 16. 为什么 geometry subscriber 使用 BestEffort

代码：

```cpp
auto qos = rclcpp::QoS(rclcpp::KeepLast(10));
qos.best_effort();
```

这样可以让 geometry node 对不同数据源保持更好的兼容性。

当前工程中：

```text
FakeLidar
→ BestEffort Publisher

真实 sllidar
→ Reliable Publisher
```

BestEffort Subscriber 可以接收相应兼容的数据。

本阶段没有重新学习 QoS 理论，而是把之前已经掌握的 QoS 知识用于新的工程节点。

---

# 17. 受控场景 bag：建立可复现输入

为了以后修改算法时不必每次重新接雷达、重新摆场景，建立新的受控测试 bag。

场景：

```text
静止盒子
↓
固定位置
↓
真实 RPLIDAR
```

录制命令：

```bash
ros2 bag record -o bags/p2_box_static_v2 /scan
```

其中：

```text
-o
```

是：

```text
--output
```

的缩写，用于指定输出目录。

---

# 18. 第一次录包失败：0 messages

第一次得到：

```text
Files:             p2_box_static_0.db3
Bag size:          24.5 KiB
Duration:          0.000000000s
Messages:          0
Topic information:
```

还出现异常时间：

```text
Apr 12 2262 ...
```

这并不表示雷达真的产生了 2262 年的数据。

真正的重要信息是：

```text
Messages: 0
```

说明 recorder 没有写入任何 `/scan` 消息。

当时没有直接修改算法，而是先分层排查：

```text
真实 LiDAR
↓
Driver
↓
/scan
↓
rosbag record
```

优先验证：

```bash
ros2 topic hz /scan
```

以及：

```bash
ros2 topic echo /scan --once
```

确认 `/scan` 是否真的在正常发布。

这是典型的分层调试：

```text
提出假设
↓
检查数据源
↓
只修改录包这一项
↓
重新验证
```

---

# 19. 第二次录包成功

重新录制：

```bash
ros2 bag record -o bags/p2_box_static_v2 /scan
```

结果：

```text
Files:             p2_box_static_v2_0.db3
Bag size:          637.0 KiB
Storage id:        sqlite3
Duration:          9.997350987s
Messages:          71
Topic:             /scan
Type:              sensor_msgs/msg/LaserScan
Count:             71
```

粗略频率：

\[
f
\approx
\frac{71}{9.997}
\approx
7.1\text{ Hz}
\]

与之前真实 RPLIDAR 约 7 Hz 的实际观察一致。

---

# 20. 关闭硬件后的回放验证

关闭真实雷达后：

```bash
ros2 bag play bags/p2_box_static_v2
```

然后：

```bash
ros2 run lidar_geometry scan_geometry_node
```

实际输出：

```text
sample: i=0 r=0.313 theta=-3.142 x=-0.313 y=0.000
raw=1080 valid=1043 rejected=37 points=1043

sample: i=0 r=0.314 theta=-3.142 x=-0.314 y=0.000
raw=1080 valid=1031 rejected=49 points=1031
```

验证：

```text
1043 + 37 = 1080
1031 + 49 = 1080
```

同时：

```text
points = valid
```

几何关系：

```text
theta ≈ -π
x ≈ -r
y ≈ 0
```

也合理。

最终建立：

```text
真实实验
↓
记录 bag
↓
断开真实硬件
↓
播放相同输入
↓
运行相同 C++ 算法
↓
得到可重复验证结果
```

这使得该 bag 可以作为后续算法修改时的回归测试输入。

---

# 21. 360° LaserScan 的首尾环绕问题

当前：

\[
angle_{min}\approx-\pi
\]

\[
angle_{max}\approx+\pi
\]

因此：

```text
ranges[0]
```

位于约：

```text
-180°
```

而：

```text
ranges[1079]
```

位于约：

```text
+180°
```

从数组角度看：

```text
[0] [1] [2] ... [1078] [1079]
 ↑                       ↑
首                        尾
```

但物理空间中：

\[
-\pi
\]

与：

\[
+\pi
\]

实际上指向几乎同一个方向。

所以一个连续物体可能出现在：

```text
1077 1078 1079 | 0 1 2
```

也就是说：

> 数组首尾虽然索引相距很远，但空间位置可能连续。

以后进行聚类时不能仅仅因为它们位于数组两端就直接认为是两个不同物体。

这称为：

```text
wrap-around
首尾环绕
```

当前 P2 只要求理解概念，不实现首尾聚类算法。

---

# 22. 本阶段涉及的新 C++ 知识

## std::size_t

用于表示数组大小、容器大小、索引等非负整数。

例如：

```cpp
std::size_t raw_count;
```

以及：

```cpp
for (std::size_t i = 0; ...)
```

---

## struct

用于定义简单的数据结构。

例如：

```cpp
struct Point2D
{
    double x;
    double y;
};
```

---

## std::vector<T>

动态顺序容器。

例如：

```cpp
std::vector<Point2D> points;
```

表示一组 `Point2D`。

---

## push_back()

在 vector 末尾加入元素：

```cpp
points.push_back(point);
```

---

## size()

得到 vector 当前元素数量：

```cpp
points.size()
```

---

## std::isfinite()

判断浮点数是否为有限值：

```cpp
std::isfinite(r)
```

用于排除：

```text
inf
-inf
nan
```

---

## continue

例如：

```cpp
if (!std::isfinite(r))
{
    ++rejected_count;
    continue;
}
```

`continue` 表示：

> 当前这一次循环不再继续执行下面的代码，直接进入下一次循环。

因此无效 `r` 不会进入几何转换。

---

# 23. 最终掌握的数据流

到 P2 Closing 结束，我现在能够解释以下完整执行链：

```text
真实 RPLIDAR / rosbag
        ↓
/scan
        ↓
sensor_msgs/msg/LaserScan
        ↓
msg->ranges[i]
        ↓
判断 std::isfinite(r)
        ↓
判断 range_min <= r <= range_max
        ↓
保留原始 index i
        ↓
theta_i = angle_min + i * angle_increment
        ↓
x = r cos(theta_i)
y = r sin(theta_i)
        ↓
Point2D
        ↓
points.push_back(point)
        ↓
std::vector<Point2D>
```

同时建立两个重要检查关系：

\[
\boxed{
raw = valid + rejected
}
\]

\[
\boxed{
points.size() = valid
}
\]

---

# 24. 最终知识验收

## 问题 1：raw_count 是什么？

回答：

> 原始 `ranges[]` 中的数据数量，还没有经过有效性筛选。

---

## 问题 2：什么时候 rejected？

回答：

> 距离数据为非有限数值，或者超出传感器声明的有效测量范围。

更精确：

```text
!isfinite(r)
```

或：

```text
r < range_min
r > range_max
```

---

## 问题 3：为什么不能在过滤后重新编号？

回答：

> `r` 和原始 index `i` 是对应的，角度由原始 `i` 决定。如果过滤后重新编号，距离会对应到错误的扫描方向。

---

## 问题 4：theta 怎么算？

\[
\boxed{
\theta_i
=
angle_{min}
+
i\cdot angle_{increment}
}
\]

---

## 问题 5：二维坐标怎么计算？

\[
\boxed{x=r\cos\theta}
\]

\[
\boxed{y=r\sin\theta}
\]

---

## 问题 6：push_back() 做什么？

回答：

> 把当前转换得到的 `Point2D` 加入 `points` 容器末尾，从而逐渐形成这一帧的二维有效点集合。

---

## 问题 7：为什么 rosbag 和真实雷达可以使用同一个 geometry node？

回答：

> geometry node 作为 Subscriber 从 `/scan` 接收 `LaserScan`，它不关心 Publisher 是 rosbag2_player 还是真实雷达 Driver。

---

## 问题 8：Topic 名和消息类型都正确但收不到消息，还要检查什么？

回答：

> QoS 是否兼容。

---

## 问题 9：为什么 360° 扫描数组首尾可能属于同一个物体？

回答：

> 因为 `-π` 和 `+π` 在物理空间中是同一个方向附近，数组只是人为把圆形扫描从某个位置“切开”后展开。

---

# 25. 本阶段遇到的问题与经验

## 问题一：只写代码但没有解释新的 C++ 知识

在第一次加入：

```cpp
std::vector<Point2D> points;
points.push_back(point);
```

时，没有立即解释：

```text
vector 是什么
Point2D 是什么
push_back 为什么可以调用
```

这导致需要额外追问。

后续教学规则明确调整为：

> 只要代码里出现此前没有接触过的新知识，第一次出现时必须解释“它是什么、为什么现在需要、这一行做什么”。

如果属于当前主线的重要内容则详细讲解；如果只是辅助语法，也至少做简要介绍。

目标：

> 不出现“代码能运行，但里面存在自己无法解释的语句”。

---

## 问题二：第一次 rosbag 录制得到 0 messages

没有直接反复尝试，也没有修改算法。

而是先检查：

```text
真实雷达
↓
Driver
↓
Topic
↓
record
```

最终通过重新验证 `/scan` 并重新录制，成功得到有效 bag。

经验：

> 工程问题优先分层定位，不要看到结果异常就直接修改算法。

---

## 问题三：对答案格式的误读

在一次自测中：

```text
1. 2
2. 0 rad
```

被误解成第一题答案是 `1.2`。

实际用户表达的是：

```text
第 1 题：2
第 2 题：0 rad
```

这提醒后续在判断用户答案错误前，要先确认上下文和编号格式，避免把表达格式误判成知识错误。

---

# 26. P2 Closing 最终结论

## 工程能力验证

```text
C++ package 创建与构建              PASS
LaserScan C++ Subscriber           PASS
rosbag → callback                  PASS
真实 RPLIDAR → callback            PASS
finite 过滤                        PASS
range_min/range_max 过滤           PASS
raw = valid + rejected             PASS
保留原始 index                     PASS
theta_i 计算                       PASS
极坐标 → Cartesian                 PASS
Point2D                            PASS
std::vector<Point2D>               PASS
points.size() = valid              PASS
真实数值手工核验                   PASS
受控场景 rosbag                    PASS
脱离真实硬件回放验证               PASS
Publisher / Subscriber 解耦理解    PASS
QoS 排查意识                       PASS
360° wrap-around 概念              PASS
```

因此：

# Phase 2 Closing：PASS

---

# 27. 当前还没有做的事情

以下内容刻意没有在 P2 提前实现：

```text
聚类
直线拟合
盒子边缘提取
350 mm 目标识别
姿态估计
PCL
Marker 可视化
复杂 PointCloud2 C++ 发布
动态 TF
odom
ICP
scan matching
SLAM
wrap-around 聚类实现
```

原因不是这些内容“不重要”，而是：

> 当前 P2 的任务是把 `LaserScan → 2D geometry` 这个最小闭环做正确、做可验证、做可理解。

只有进入下一阶段并且真实任务需要时，再引入新的算法复杂度。

---

# 28. 下一阶段进入条件

现在已经具备进入更高层二维几何算法的基础：

```text
LaserScan
↓
有效二维点集合
↓
后续才可能进行
聚类 / 线段 / 目标几何 / 位姿估计
```

但下一阶段不应该直接跳到复杂 SLAM 或 ICP。

应继续遵守：

```text
真实任务
→ 分析所需能力
→ 最小算法
→ 建立验证场景
→ 验证正确
→ 再增加复杂度
```

---

# 29. 本阶段最重要的几句话

### LaserScan 的角度不是保存在每个 range 旁边，而是通过 index 计算出来的

\[
\boxed{
\theta_i=angle_{min}+i\cdot angle_{increment}
}
\]

所以：

> **过滤距离时不能丢失原始 index。**

---

### 有效点数量不等于原始采样数量

\[
\boxed{
raw=valid+rejected
}
\]

---

### 每个有效距离最终应该对应一个二维点

\[
\boxed{
points.size()=valid
}
\]

---

### 极坐标转换

\[
\boxed{x=r\cos\theta}
\]

\[
\boxed{y=r\sin\theta}
\]

---

### ROS2 节点面向 Topic，而不是具体 Publisher

```text
不同数据源
↓
同一个 /scan 接口
↓
同一个 geometry node
```

---

### rosbag 的真正价值

rosbag 不只是“保存数据”。

更重要的是：

> **把一次真实物理实验变成可以反复重放的固定输入，从而让算法开发具备可重复验证能力。**

---

## 最终状态

P2 已完成：

```text
真实激光数据
↓
ROS2 LaserScan
↓
C++ 有效性过滤
↓
正确角度索引
↓
二维坐标
↓
Point2D 集合
↓
实时验证
↓
rosbag 可复现验证
```

下一阶段应在这一稳定基础上继续，而不是重新回到 ROS2 基础或直接跳入复杂 SLAM。
