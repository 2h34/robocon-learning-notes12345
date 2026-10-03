# 2026-10-03 ROS2 + C++ 系统学习记录
## Week 1 · Day 5：QoS + ROS2 数据流调试 + rosbag2 + RViz2

## 1. 今日主线

围绕 Day 4 的 Fake LiDAR：

```text
FakeLidarNode
→ /scan
→ sensor_msgs/msg/LaserScan
```

完成：

```text
发布 → 故障 → 诊断 → 修复 → 记录 → 回放 → 可视化
```

今天的重点不是背 QoS 枚举，而是建立真实 ROS2 数据流调试能力。

---

## 2. Day 4 快速回顾

### `create_publisher(..., 10)` 中的 `10`

```cpp
publisher_ =
    this->create_publisher<sensor_msgs::msg::LaserScan>(
        "/scan", 10);
```

这里的 `10` 表示 QoS 的队列深度（Depth），不是 10 Hz。

Timer 周期为：

```text
500 ms = 0.5 s
```

所以：

\[
f = \frac{1}{0.5}=2\ \mathrm{Hz}
\]

必须明确：

```text
Depth = 10
≠ 10 Hz
≠ 保存 10 秒
```

### 当前 Fake LiDAR 的运行时数据流

```text
Timer 到期
→ callback 进入 ready 状态
→ Executor 调度
→ 执行 callback
→ 构造 / 填充 LaserScan message
→ publisher_->publish(message)
→ 消息发布到 /scan
```

分析数据流时，只描述代码真实存在的组件。

---

## 3. QoS 的核心模型

QoS = Quality of Service。

可以理解为：

> Publisher / Subscriber 对消息传输行为的通信约定。

### Message type 与 QoS

```text
Message type
→ 数据长什么样

QoS
→ 数据希望怎样传
```

例如 `sensor_msgs/msg/LaserScan` 规定 `header`、`ranges[]`、`angle_min` 等字段；QoS 则规定可靠性、历史缓存等通信行为。

---

## 4. Reliability

### BEST_EFFORT

```text
尽力传输
允许部分消息丢失
不要求可靠传输保证
```

它常见于连续、高频的传感器数据，但不能简单理解成“所有传感器必须 BEST_EFFORT”，也不能理解成“强制最低延迟”。

ROS2 提供：

```cpp
rclcpp::SensorDataQoS()
```

作为常见传感器 QoS Profile。

### RELIABLE

```text
要求可靠传输能力
```

系统会尽量保证消息可靠到达。

两者不是“差 / 好”的关系，而是针对不同通信需求的策略。

---

## 5. Offered QoS 与 Requested QoS

Publisher：

```text
Offered QoS
→ 能提供什么通信能力
```

Subscriber：

```text
Requested QoS
→ 要求什么通信能力
```

通信不仅要求：

```text
Topic 一样
Message type 一样
```

还要求：

```text
QoS compatible
```

### Reliability 兼容关系

```text
Publisher = BEST_EFFORT
Subscriber = RELIABLE
→ 不兼容
```

因为 Subscriber 要求的能力高于 Publisher 能提供的能力。

```text
Publisher = RELIABLE
Subscriber = BEST_EFFORT
→ 可以兼容
```

因为 Publisher 能满足 Subscriber 的要求。

重要修正：

```text
Offered QoS
不必完全等于
Requested QoS
```

要求的是“兼容”，不是“所有参数完全相同”。

---

## 6. Durability

### VOLATILE

对于后来加入的 Subscriber，不会把之前已经发布的历史消息作为持久数据提供给它。

### TRANSIENT_LOCAL

Publisher 可以为后来加入的 Subscriber 保留一定历史消息。

不能简单理解成：

```text
“保存若干秒”
```

也不能理解成：

```text
“永久保存所有历史消息”
```

能保留多少历史消息还会受到 History / Depth 等设置影响。

---

## 7. History / Depth

```text
History = KEEP_LAST
Depth = N
```

例如：

```text
KEEP_LAST(10)
```

可先理解为：

> 最多保留最近 10 条待处理历史消息。

若：

```text
Publisher = 100 Hz
Subscriber callback = 10 Hz
Depth = 10
```

Publisher 持续快于 Subscriber 时，队列会积压；由于容量有限，旧消息可能被丢弃 / 覆盖。

---

## 8. 第一次真实 QoS 故障实验

Fake LiDAR Publisher 修改为：

```cpp
publisher_ =
    this->create_publisher<sensor_msgs::msg::LaserScan>(
        "/scan",
        rclcpp::SensorDataQoS());
```

执行：

```bash
ros2 topic info /scan --verbose
```

实际确认：

```text
Publisher count: 1
Node name: fake_lidar
Endpoint type: PUBLISHER

Reliability: BEST_EFFORT
Durability: VOLATILE
```

---

## 9. 故意创建 RELIABLE Subscriber

Subscriber 保持：

```text
Topic = /scan
Type = sensor_msgs/msg/LaserScan
```

只把 QoS 设为：

```cpp
auto qos = rclcpp::QoS(rclcpp::KeepLast(10));
qos.reliable();
```

于是：

```text
Publisher = BEST_EFFORT
Subscriber = RELIABLE
```

### 实际现象

`ros2 topic info /scan --verbose` 同时能看到：

```text
Publisher endpoint
Subscriber endpoint
```

但 Subscriber 报警：

```text
offering incompatible QoS.
No messages will be sent to it.
Last incompatible policy: RELIABILITY_QOS_POLICY
```

说明：

```text
Graph 中能发现 endpoint
≠
两端一定可以正常传数据
```

---

## 10. 修复 QoS

只把 Subscriber：

```cpp
qos.reliable();
```

改成：

```cpp
qos.best_effort();
```

Topic、Message type、Publisher 主体逻辑、Subscriber callback 均不修改。

重新检查：

```text
Publisher Reliability = BEST_EFFORT
Subscriber Reliability = BEST_EFFORT
```

通信恢复。

核心认识：

> 只修改 QoS，其他接口与代码主体不变，也可能让通信从失败恢复为成功。

---

## 11. “Topic 存在但没数据”的排查流程

形成以下顺序：

```text
Node 是否存在
↓
Topic 是否存在 / 是否一致
↓
Message type 是否一致
↓
Publisher / Subscriber endpoint 是否存在
↓
QoS 是否兼容
↓
数据是否真的流动
```

常用工具：

```bash
ros2 node list
ros2 node info
ros2 topic list
ros2 topic info /scan --verbose
ros2 topic echo /scan
ros2 topic hz /scan
```

必须记住：

```text
Topic 存在
≠ 数据一定在流动
```

```text
Publisher / Subscriber 都存在
≠ 一定能通信
```

```text
QoS 参数不同
≠ 一定不兼容
```

---

## 12. rosbag2：保存 ROS2 Topic 数据

rosbag2 的核心用途：

```text
ROS2 Topic
↓
ros2 bag record
↓
Bag
↓
ros2 bag play
↓
重新发布 Topic
```

记录的是：

```text
Topic 消息数据
+
相关元信息
```

不是：

```text
原来的 Node
+
整个程序执行过程
```

所以：

```text
rosbag play
= 重新发布以前保存的 Message
```

而不是：

```text
重新运行原来的 FakeLidarNode
```

---

## 13. 实际录制 `/scan`

执行：

```bash
ros2 bag record /scan -o day5_scan_bag
```

随后：

```bash
ros2 bag info day5_scan_bag
```

实际结果：

```text
Files:       day5_scan_bag_0.db3
Bag size:    25.0 KiB
Storage id:  sqlite3
Duration:    12.413548722s
Messages:    26
Topic:       /scan
Type:        sensor_msgs/msg/LaserScan
Count:       26
```

Fake LiDAR 约 2 Hz，12.4 s 记录 26 条消息，与预期基本一致。

---

## 14. rosbag 路径问题

第一次：

```bash
ros2 bag play day5_scan_bag
```

报：

```text
Bag path 'day5_scan_bag' does not exist!
```

原因是当前终端所在目录与 bag 实际位置不一致。

使用：

```bash
ros2 bag play ~/day5_scan_bag
```

成功。

复习：

```text
day5_scan_bag
→ 相对路径

~/day5_scan_bag
→ 从 Home 目录开始
```

---

## 15. 停止 FakeLidarNode 后回放

停止原 Fake LiDAR 后：

```bash
ros2 bag play ~/day5_scan_bag -l
```

再检查：

```bash
ros2 topic info /scan --verbose
```

实际看到：

```text
Publisher count: 1
Node name: rosbag2_player
Endpoint type: PUBLISHER
Reliability: BEST_EFFORT
Durability: VOLATILE
```

说明当前数据流已经变成：

```text
day5_scan_bag
↓
rosbag2_player
↓
/scan
↓
Subscriber / RViz
```

而不是 FakeLidarNode 再次运行。

---

## 16. RViz2 显示 `/scan`

设置：

```text
Fixed Frame = laser
```

添加：

```text
LaserScan Display
```

Topic：

```text
/scan
```

### 第一次故障：RViz QoS 不兼容

RViz 报：

```text
New publisher discovered on topic '/scan',
offering incompatible QoS.
No messages will be sent to it.
Last incompatible policy: RELIABILITY_QOS_POLICY
```

原因：

```text
rosbag2_player = BEST_EFFORT
RViz LaserScan = RELIABLE
```

修改 RViz：

```text
Reliability Policy = Best Effort
```

之后：

```text
LaserScan Status: Ok
```

这说明 QoS 故障不仅会出现在自己写的 Subscriber 中，实际工具 RViz 也会遇到完全相同的问题。

---

## 17. RViz 扫描点太小

QoS 修复后，一开始仍不明显。

当时：

```text
LaserScan Size = 0.01 m
View Distance ≈ 27 m
```

把：

```text
Size (m) = 0.05
```

并拉近视角后，成功看到白色扫描点。

---

## 18. frame_id 与 Fixed Frame

当前每条 LaserScan：

```text
frame_id = "laser"
```

RViz：

```text
Fixed Frame = "laser"
```

两者处于同一个参考坐标系，因此当前显示 `/scan` 不需要跨坐标系转换。

最终数据流：

```text
ros2 bag play
↓
启动 rosbag2_player
↓
读取 bag 中保存的 LaserScan
↓
发布到 /scan
↓
RViz 的 LaserScan Display 订阅 /scan
↓
读取 LaserScan
↓
把扫描点绘制在界面中
```

当前虽然 RViz 全局仍可能看到 `No tf data`，但：

```text
LaserScan Status = Ok
```

且：

```text
LaserScan.frame_id = laser
Fixed Frame = laser
```

所以当前 LaserScan 可以正常直接显示。

以后如果：

```text
LaserScan.frame_id = laser
Fixed Frame = base_link
```

就必须知道：

```text
laser → base_link
```

之间的坐标变换，这属于后续 TF2 内容。

---

## 19. 今天遇到并修正的理解问题

### 1）Reliability 兼容方向一开始判断反了

修正为从：

```text
Publisher 能提供什么
Subscriber 要求什么
```

来判断，而不是死背表。

### 2）误以为 QoS 不兼容时 Subscriber endpoint 不会出现

实际 Graph 仍然可以发现两个 endpoint，只是消息无法正常传输。

### 3）把 TRANSIENT_LOCAL 误理解成“按时间保存”

修正为：

```text
是否为后来加入的 Subscriber 提供历史数据
```

具体历史量由其他 QoS 参数共同决定。

### 4）rosbag play 路径错误

学会区分相对路径与 `~/...` 路径。

### 5）RViz 第一次不显示 `/scan`

不是 rosbag 没数据，而是 RViz 的 Reliability 与 rosbag2_player 不兼容。

### 6）RViz Status OK 但点不明显

原因是视角过远、点太小；调整 Size 和视角后解决。

---

## 20. 当前真正掌握的能力

已经能够：

```text
✓ 区分 Message type 与 QoS
✓ 区分 Depth 与发布频率
✓ 理解 BEST_EFFORT / RELIABLE
✓ 从 Offered / Requested 解释 Reliability compatibility
✓ 初步理解 VOLATILE / TRANSIENT_LOCAL
✓ 理解 KEEP_LAST / Depth
✓ 使用 ros2 topic info --verbose 检查 endpoint QoS
✓ 制造一次真实 QoS 不兼容
✓ 定位并修复该故障
✓ 建立“Topic 存在但无数据”的排查流程
✓ 使用 ros2 bag record / info / play
✓ 理解 rosbag play 是重新发布消息，不是重启原 Node
✓ 停止 FakeLidarNode 后仍用 bag 恢复 /scan
✓ 使用 RViz 显示 LaserScan
✓ 处理 RViz 中真实 QoS incompatibility
✓ 理解 frame_id 与 Fixed Frame 的当前关系
```

---

## 21. 暂时后置的内容

今天不深入：

```text
DDS 内部实现
RMW 内部
Deadline / Lifespan / Liveliness
MultiThreadedExecutor
Callback Group
Lifecycle
TF2 完整体系
TF tree
map / odom / base_link
URDF
Gazebo
SLAM
真实 LiDAR Driver
复杂 rosbag storage
RViz 插件开发
```

---

## 22. Day 5 结论

**Day 5：通过。**

今天真正完成的是：

```text
Fake LiDAR 发布
→ 故意制造 QoS 故障
→ 用 ROS2 Graph 诊断
→ 修复 QoS
→ rosbag2 记录 /scan
→ 停止 FakeLidarNode
→ rosbag2_player 回放
→ RViz 订阅并显示 /scan
```

最大的收获不是记住几个 QoS 枚举，而是建立了一条真实的 ROS2 数据流调试主线：

```text
Publisher
↓
Topic
↓
QoS
↓
Subscriber
↓
rosbag2
↓
RViz
```

按照既定路线，Day 6 可以进入：

```text
Parameter
+
Launch
+
小系统整合
```

但 Day 5 到这里结束，不自动开始 Day 6。
