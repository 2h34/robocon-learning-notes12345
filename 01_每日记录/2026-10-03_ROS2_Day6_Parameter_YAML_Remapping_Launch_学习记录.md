# 2026-10-03 ROS2 + C++ Week 1 Day 6 学习记录
## 主题：Parameter + YAML + Remapping + Launch + Launch Argument

> 今日目标：把原本“参数和 Topic 名写死在 C++ 代码中、需要手动启动多个 Node”的小程序，逐步改造成一个**可配置、可重映射、可一键启动、可用 CLI 系统验证的 ROS2 小系统**。
>
> 今日说明：**今天不再进行额外自测。**  
> 本次学习已经包含多次阶段性判断与运行验证，后续在“当前 ROS2 学习进度总结 / 阶段复盘”时，再统一安排综合自测，用来检查知识是否真正内化，而不是今天继续增加认知负担。

---

# 一、今日学习主线

Day 6 的核心变化不是单独学习某一个 ROS2 API，而是第一次把多个机制串起来：

```text
C++ Node
  ↓
Parameter
  ↓
YAML
  ↓
Remapping
  ↓
Launch
  ↓
Launch Argument
  ↓
ROS2 Graph
  ↓
CLI 验证
```

最终实现的系统结构：

```text
                 day6_lidar.yaml
                       │
                       │ Parameters
                       ▼
                FakeLidarNode
                       │
                       │ 源码默认 /scan
                       │
                  Remapping
                       │
                       ▼
                 /front_scan
                       │
                       ▼
                ScanSubscriber

       ↑ 整个系统由 Python Launch 统一组织
```

并且可以在启动时通过 Launch Argument 把运行时 Topic 从：

```text
/front_scan
```

改成：

```text
/rear_scan
```

而不修改 C++ 源码。

---

# 二、Parameter：把写死在代码中的配置抽出来

## 2.1 Parameter 的作用

Parameter 是：

> **属于某个 ROS2 Node 实例的运行配置。**

本次给 `FakeLidarNode` 增加的主要 Parameter：

```text
frame_id
publish_rate_hz
range_min
range_max
```

例如：

```cpp
this->declare_parameter<double>("range_max", 10.0);
```

含义：

```text
Parameter 名字：range_max
类型：double
默认值：10.0
```

随后通过：

```cpp
range_max_ =
    this->get_parameter("range_max").as_double();
```

把 ROS2 Parameter 读入 C++ 成员变量。

---

## 2.2 Parameter 名与 C++ 成员变量不是同一个东西

必须区分：

```text
"range_max"
→ ROS2 Parameter 名

range_max_
→ C++ 成员变量
```

两者之间的关系：

```text
ROS2 Parameter
range_max
    │
    │ get_parameter()
    ▼
C++ 成员变量
range_max_
    │
    ▼
程序真正使用
```

类似地：

```text
"frame_id"
→ ROS2 Parameter 名

frame_id_
→ C++ 成员变量
```

因此，ROS2 Parameter 是“ROS2 配置层”的概念，而成员变量属于“C++ 程序内部状态”。

---

# 三、运行时修改 Parameter 的边界

这是今天一个非常重要的实验结论。

假设启动时：

```bash
ros2 run hello_ros2 fake_lidar_node \
  --ros-args \
  -p range_max:=12.0
```

构造函数中读取：

```cpp
range_max_ =
    this->get_parameter("range_max").as_double();
```

则：

```text
Parameter range_max = 12.0
range_max_          = 12.0
```

运行过程中再执行：

```bash
ros2 param set /fake_lidar range_max 15.0
```

此时：

```text
ROS2 Parameter = 15.0
```

但如果程序没有：

- 再次调用 `get_parameter()`
- 或注册 Parameter 更新回调

那么：

```text
C++ 成员变量 range_max_
仍然可能保持 12.0
```

因此：

```text
Parameter 改变
≠
程序内部成员变量一定自动同步
```

这是以后调试 ROS2 Parameter 时必须注意的边界。

---

# 四、把发布频率 Parameter 化

原先 Timer 周期写死：

```cpp
500ms
```

即：

\[
T=0.5s
\]

所以：

\[
f=\frac{1}{T}=2Hz
\]

后来增加：

```text
publish_rate_hz
```

作为 Parameter，并计算：

```cpp
auto publish_period =
    std::chrono::duration<double>(
        1.0 / publish_rate_hz_
    );
```

核心关系：

\[
T=\frac{1}{f}
\]

例如：

```text
publish_rate_hz = 5 Hz
```

则：

\[
T=\frac{1}{5}=0.2s
\]

因此 Timer 每约 `0.2 s` 触发一次。

实际运行日志也验证了：

```text
...106.408...
...106.608...
...106.808...
```

相邻时间约差：

```text
0.2 s
```

即：

```text
约 5 Hz
```

---

# 五、LaserScan 中时间字段的处理

本次还把消息中的：

```cpp
scan.scan_time
scan.time_increment
```

与发布频率关联起来。

若：

\[
publish\_rate\_hz=f
\]

则：

\[
scan\_time=\frac{1}{f}
\]

Fake LaserScan 当前包含 5 个采样值：

```text
●──●──●──●──●
```

5 个采样点之间有：

```text
4 个时间间隔
```

所以：

\[
time\_increment=\frac{scan\_time}{4}
\]

例如：

```text
f = 4 Hz
```

则：

\[
scan\_time=0.25s
\]

\[
time\_increment=0.0625s
\]

这一步把：

```text
Parameter
→ 数学关系
→ Timer
→ Message 字段
```

串起来了。

---

# 六、Parameter CLI

本次使用和理解了：

```bash
ros2 param list /fake_lidar
```

作用：

```text
查看某个 Node 当前有哪些 Parameter
```

---

```bash
ros2 param get /fake_lidar range_max
```

作用：

```text
查看某个 Parameter 当前值
```

---

```bash
ros2 param describe /fake_lidar range_max
```

作用：

```text
查看 Parameter 类型、描述等信息
```

---

```bash
ros2 param set /fake_lidar range_max 15.0
```

作用：

```text
运行时修改 Parameter
```

核心记忆：

```text
ros2 param ...
→ 观察 / 修改 Node 的配置层
```

而不是直接观察 Topic 中的数据。

---

# 七、YAML：集中保存一组 Parameter

如果每次启动都写：

```bash
-p frame_id:=front_laser
-p publish_rate_hz:=5.0
-p range_min:=0.2
-p range_max:=8.0
```

会比较繁琐。

因此建立：

```text
config/day6_lidar.yaml
```

示例结构：

```yaml
fake_lidar:
  ros__parameters:
    frame_id: front_laser
    publish_rate_hz: 5.0
    range_min: 0.2
    range_max: 8.0
```

结构含义：

```text
fake_lidar
│
│ Node 名
│
└── ros__parameters
      │
      ├── frame_id
      ├── publish_rate_hz
      ├── range_min
      └── range_max
```

YAML 的职责：

> 把一组 ROS2 Parameter 统一保存在配置文件中，方便重复使用、集中修改和工程管理。

---

# 八、YAML 不替代 `declare_parameter()`

C++ 中仍然需要：

```cpp
declare_parameter(...)
```

YAML 主要提供启动时的 Parameter override。

可以理解为：

```text
C++ 默认值
    ↓
YAML 覆盖
    ↓
命令行显式覆盖
```

例如：

```text
C++ 默认：range_max = 10
YAML：   range_max = 6
命令行： range_max = 7
```

最终：

```text
range_max = 7
```

因此 YAML 不是“定义 ROS2 Parameter 的唯一来源”，而是外部配置来源之一。

---

# 九、Remapping：改变 ROS Graph 中的名字

源码中 FakeLidarNode 的 Publisher 使用：

```text
/scan
```

但运行时希望使用：

```text
/front_scan
```

不修改 C++，可以通过：

```bash
ros2 run hello_ros2 fake_lidar_node \
  --ros-args \
  -r /scan:=/front_scan
```

完成 Remapping。

核心关系：

```text
源码中的 Topic 名
/scan
    │
    │ Remapping
    ▼
运行时 Graph 中的 Topic 名
/front_scan
```

---

# 十、Parameter 与 Remapping 的职责区别

这是 Day 6 最核心的概念边界之一。

```text
Parameter
→ Node 怎么工作

Remapping
→ Node 在 ROS Graph 中使用什么名字 / 连接到哪里
```

例如：

| 需求 | 机制 |
|---|---|
| 最大测距 8 m | Parameter |
| 发布频率 5 Hz | Parameter |
| `frame_id = front_laser` | Parameter |
| `/scan → /front_scan` | Remapping |
| IMU 噪声标准差 | Parameter |
| `/odom → /robot_odom` | Remapping |

---

# 十一、为什么 Publisher 和 Subscriber 都需要 Remapping

源码中：

```text
FakeLidarNode
发布 /scan

ScanSubscriber
订阅 /scan
```

如果只给 Publisher 做：

```text
/scan → /front_scan
```

运行时会变成：

```text
Publisher  → /front_scan
Subscriber → /scan
```

此时它们处于两个不同的 Topic，不能通信。

因此两边都需要：

```text
/scan → /front_scan
```

最终运行时：

```text
FakeLidarNode
      │
      ▼
 /front_scan
      │
      ▼
ScanSubscriber
```

本次实际运行已经验证：

```text
Publisher count: 1
Subscription count: 1
```

两个 Node 最终确实连接到了同一个 Topic。

---

# 十二、Launch：从“手动启动 Node”到“组织 ROS2 系统”

之前需要：

```bash
ros2 run ...
```

分别启动多个 Node。

Day 6 开始使用：

```bash
ros2 launch ...
```

Launch 的主要职责不是单纯“启动一个程序”，而是：

> **统一描述和组织一个 ROS2 系统。**

当前 Launch 管理：

```text
两个 Node
+
YAML
+
Remapping
+
Launch Argument
```

结构：

```text
Launch
├── FakeLidarNode
│   ├── 加载 YAML
│   └── Remapping
│
└── ScanSubscriber
    └── Remapping
```

---

# 十三、Python Launch 的最小结构

基本形式：

```python
from launch import LaunchDescription
from launch_ros.actions import Node


def generate_launch_description():
    return LaunchDescription([
        Node(...),
        Node(...)
    ])
```

当前学习要求：

> Python Launch 只需做到“能读懂、能修改”，暂不系统学习 Python，也暂不深入复杂 Launch 机制。

---

# 十四、Launch 中 `Node(...)` 的常用字段

示例：

```python
Node(
    package='hello_ros2',
    executable='fake_lidar_node',
    name='fake_lidar',
    parameters=[lidar_config],
    remappings=[
        ('/scan', scan_topic)
    ],
    output='screen'
)
```

字段含义：

| 字段 | 含义 |
|---|---|
| `package` | ROS2 package |
| `executable` | 要启动的可执行程序 |
| `name` | 运行时 Node 名 |
| `parameters` | 加载 Parameter / YAML |
| `remappings` | 名称重映射 |
| `output='screen'` | 日志输出到 Launch 终端 |

特别注意：

```text
output='screen'
```

不是：

```text
/screen Topic
```

它表示：

> Node 的日志显示在启动 Launch 的终端中。

本次运行时已经看到：

```text
[fake_lidar_node-1] ...
[scan_subscriber-2] ...
```

两个 Node 的日志同时出现。

---

# 十五、为什么使用 `get_package_share_directory()`

Launch 文件中没有硬编码：

```text
~/ros2_ws/src/hello_ros2/config/day6_lidar.yaml
```

而是：

```python
get_package_share_directory('hello_ros2')
```

再结合：

```python
os.path.join(...)
```

得到：

```text
install/hello_ros2/share/hello_ros2/config/day6_lidar.yaml
```

这样做的意义：

```text
不依赖某一台电脑的源码目录绝对路径
```

而是通过 ROS2 package 的安装位置寻找资源，更符合 ROS2 工程结构。

---

# 十六、CMake 中安装 `launch/` 与 `config/`

仅仅在：

```text
src/hello_ros2/
```

下存在：

```text
launch/
config/
```

还不够。

为了让 ROS2 从安装空间中找到它们，需要在 `CMakeLists.txt` 中加入：

```cmake
install(DIRECTORY
  launch
  config
  DESTINATION share/${PROJECT_NAME}
)
```

构建后：

```text
src/hello_ros2/launch
        ↓
install
        ↓
install/hello_ros2/share/hello_ros2/launch
```

以及：

```text
src/hello_ros2/config
        ↓
install
        ↓
install/hello_ros2/share/hello_ros2/config
```

同时将：

```cmake
ament_package()
```

放到 `CMakeLists.txt` 最后，完成 package 的收尾。

---

# 十七、package.xml 中增加 Launch 运行依赖

Launch 文件使用：

```python
ament_index_python
launch
launch_ros
```

因此增加：

```xml
<exec_depend>ament_index_python</exec_depend>
<exec_depend>launch</exec_depend>
<exec_depend>launch_ros</exec_depend>
```

当前理解：

```text
rclcpp / sensor_msgs
→ C++ Node 的依赖

launch / launch_ros / ament_index_python
→ Launch 运行时使用的依赖
```

本阶段不需要在 CMake 中额外对这些 Python Launch 包做 C++ `find_package()`。

---

# 十八、Launch Argument：让 Launch 本身也可以接受输入

原先 Launch 中写死：

```python
('/scan', '/front_scan')
```

如果想改为：

```text
/rear_scan
```

还需要修改 Launch 文件。

因此增加：

```python
DeclareLaunchArgument(
    'scan_topic',
    default_value='/front_scan'
)
```

启动时可以：

```bash
ros2 launch hello_ros2 day6_system.launch.py \
  scan_topic:=/rear_scan
```

于是：

```text
Launch 文件不改
↓
启动参数改变
↓
Remapping 目标改变
↓
运行时 Graph 改变
```

---

# 十九、Parameter、Remapping、Launch Argument、Launch 的最终职责划分

| 机制 | 核心问题 |
|---|---|
| Parameter | Node 怎么工作？ |
| Remapping | ROS Graph 中名字怎么映射？ |
| Launch Argument | 这一次 Launch 采用什么输入？ |
| Launch | 整个 ROS2 系统如何组织和启动？ |

典型例子：

```text
range_max = 8.0
→ Parameter

/scan → /front_scan
→ Remapping

启动时选择 /front_scan 或 /rear_scan
→ Launch Argument

同时启动 LiDAR + Subscriber
→ Launch
```

---

# 二十、`LaunchConfiguration()` 与 `DeclareLaunchArgument()`

本次专门讨论了这两段代码为什么处于不同位置。

```python
scan_topic =
    LaunchConfiguration('scan_topic')
```

这里：

```text
scan_topic
→ Python 局部变量

'scan_topic'
→ Launch Argument 的名字
```

`LaunchConfiguration('scan_topic')` 并不是立刻得到：

```text
/front_scan
```

它更像：

> 创建一个“Launch 真正执行时，再去读取 `scan_topic` 值”的引用 / 占位对象。

例如：

```python
topic_config =
    LaunchConfiguration('scan_topic')
```

同样可以工作。

因为：

```text
topic_config
```

只是 Python 变量名，而真正关联 Launch Argument 的是：

```text
'scan_topic'
```

---

# 二十一、为什么 `DeclareLaunchArgument()` 放在 `LaunchDescription` 中

```python
DeclareLaunchArgument(...)
```

属于：

```text
Launch Action
```

所以必须被放进：

```python
LaunchDescription([
    ...
])
```

才能真正成为 Launch 执行流程的一部分。

例如：

```python
LaunchDescription([
    DeclareLaunchArgument(...),
    Node(...),
    Node(...)
])
```

可以理解为：

```text
Launch 运行时：

1. 声明 scan_topic
2. 启动 FakeLidarNode
3. 启动 ScanSubscriber
```

---

# 二十二、Python 构造阶段 与 Launch 执行阶段

这是理解 Python Launch 的关键。

## 阶段 1：Python 构造 Launch 描述

```text
创建 LaunchConfiguration 对象
↓
创建 DeclareLaunchArgument Action
↓
创建两个 Node Action
↓
放入 LaunchDescription
```

此时还没有真正启动 ROS2 Node。

---

## 阶段 2：Launch 执行描述

```text
DeclareLaunchArgument
↓
Launch Context 中得到 scan_topic
↓
Node Action 被执行
↓
LaunchConfiguration 被解析
↓
得到 /front_scan 或 /rear_scan
```

因此看起来：

```python
scan_topic = LaunchConfiguration('scan_topic')
```

写在：

```python
DeclareLaunchArgument(...)
```

之前，并不意味着运行时先使用再声明。

必须区分：

```text
Python 代码构造顺序
≠
Launch Action 执行顺序
```

---

# 二十三、最终实际运行结果

执行：

```bash
ros2 launch hello_ros2 day6_system.launch.py
```

Launch 终端中看到：

```text
[fake_lidar_node-1] Published fake LaserScan with 5 ranges
[scan_subscriber-2] Received scan: 5 ranges
```

证明：

```text
两个 Node 都成功启动
+
Publisher 正常发布
+
Subscriber 正常接收
```

运行时结构：

```text
/fake_lidar
      │
      │ sensor_msgs/msg/LaserScan
      │ BEST_EFFORT
      ▼
 /front_scan
      │
      │ BEST_EFFORT
      ▼
/scan_subscriber
```

---

# 二十四、ROS2 Graph 验证结果

执行：

```bash
ros2 node list
```

实际得到：

```text
/fake_lidar
/scan_subscriber
```

说明：

```text
两个 Node 均存在
```

---

执行：

```bash
ros2 topic list
```

实际包含：

```text
/front_scan
/parameter_events
/rosout
```

说明：

```text
运行时真正存在的业务 Topic 是 /front_scan
```

源码中虽然仍然使用：

```text
/scan
```

但运行时 Graph 已经通过 Remapping 变成：

```text
/front_scan
```

---

执行：

```bash
ros2 topic info /front_scan --verbose
```

得到：

```text
Type:
sensor_msgs/msg/LaserScan

Publisher count: 1
Subscription count: 1
```

并且两端 QoS：

```text
Reliability: BEST_EFFORT
Durability: VOLATILE
```

因此：

```text
Publisher 与 Subscriber 的 QoS 兼容
```

通信正常。

---

# 二十五、Launch Argument 实际实验

默认启动：

```bash
ros2 launch hello_ros2 day6_system.launch.py
```

默认：

```text
scan_topic = /front_scan
```

运行时：

```text
/front_scan
```

---

覆盖 Launch Argument：

```bash
ros2 launch hello_ros2 day6_system.launch.py \
  scan_topic:=/rear_scan
```

运行时 Topic 变为：

```text
/rear_scan
```

两个 Node 仍然能够正常通信。

这证明：

```text
C++ 源码没有改变
YAML 没有改变
Launch 文件没有再次修改
↓
只修改 Launch Argument
↓
运行时 ROS Graph 改变
```

---

# 二十六、CLI 验证体系

Day 5～Day 6 已经形成一套基本调试思路。

| 想验证什么 | CLI |
|---|---|
| 当前有哪些 Node | `ros2 node list` |
| 当前有哪些 Topic | `ros2 topic list` |
| Topic 类型和端点数量 | `ros2 topic info /front_scan` |
| Publisher / Subscriber / QoS 详细信息 | `ros2 topic info /front_scan --verbose` |
| 实际消息 | `ros2 topic echo /front_scan` |
| 实际发布频率 | `ros2 topic hz /front_scan` |
| 当前 Parameter | `ros2 param get /fake_lidar ...` |

不需要机械死记，而可以按照：

```text
我要验证什么？
    ↓
这个信息属于哪一层？
    ↓
选择对应 ROS2 CLI
```

---

# 二十七、推荐的 ROS2 分层调试顺序

今天形成的一个重要工程习惯：

```text
先看“有没有”
↓
再看“连没连上”
↓
再看“配置对不对”
↓
再看“实际消息对不对”
↓
再看“实际行为对不对”
```

例如：

## 1. Node 是否存在

```bash
ros2 node list
```

## 2. Topic 是否存在

```bash
ros2 topic list
```

## 3. Graph 是否正确

```bash
ros2 topic info /front_scan --verbose
```

## 4. Parameter 是否正确

```bash
ros2 param get /fake_lidar publish_rate_hz
```

## 5. 实际消息是否正确

```bash
ros2 topic echo /front_scan
```

## 6. 实际行为是否符合配置

```bash
ros2 topic hz /front_scan
```

例如：

```text
publish_rate_hz = 5.0
```

只能证明配置值是：

```text
5.0
```

如果：

```bash
ros2 topic hz /front_scan
```

也是：

```text
≈ 5 Hz
```

才能进一步证明：

```text
Parameter
→ C++ 程序逻辑
→ Timer
→ Topic 实际行为
```

这一整条链是正确的。

---

# 二十八、今天最容易混淆的概念

| 容易混淆 | 正确区分 |
|---|---|
| Parameter vs C++ 成员变量 | Parameter 是 ROS2 配置；成员变量是程序内部状态 |
| Parameter vs Remapping | 前者控制行为，后者控制 Graph 名称 |
| Remapping vs Launch Argument | Remapping 是映射关系；Launch Argument 是 Launch 输入 |
| Launch Argument vs Parameter | 一个属于 Launch，一个属于 Node |
| `topic list` vs `topic info` | 前者看“有哪些”，后者看“这个是什么” |
| `param get` vs `topic echo` | 前者看配置，后者看实际消息 |
| 源码 `/scan` vs 运行时 `/front_scan` | Remapping 后运行时名称可以不同 |
| Python 变量 vs Launch Argument 名 | `topic_config` 与 `'scan_topic'` 属于不同层 |
| `LaunchConfiguration` vs `DeclareLaunchArgument` | 前者引用输入，后者声明输入 |
| Python 构造阶段 vs Launch 执行阶段 | 一个生成描述，一个执行描述 |

---

# 二十九、最终系统架构图

```text
                        ┌──────────────────┐
                        │ Launch Argument  │
                        │ scan_topic       │
                        └────────┬─────────┘
                                 │
                                 ▼
                           day6_system
                              Launch
                      ┌──────────┴──────────┐
                      │                     │
                      ▼                     ▼
               FakeLidarNode         ScanSubscriber
                      ▲                     │
                      │                     │
            Parameters│                     │
                      │                     │
           day6_lidar.yaml                  │
      ┌───────────────┴───────────────┐     │
      │ frame_id                      │     │
      │ publish_rate_hz               │     │
      │ range_min                     │     │
      │ range_max                     │     │
      └───────────────────────────────┘     │
                      │                     │
                source: /scan          source: /scan
                      │                     │
                      └──── Remapping ──────┘
                                 │
                                 ▼
                           /front_scan
                                 │
                       sensor_msgs/LaserScan
                                 │
                        BEST_EFFORT QoS
```

---

# 三十、Day 6 核心知识压缩

以后快速复习时，可以先记住：

> **Parameter**：配置一个 Node 怎么工作。  
> **YAML**：把一组 Parameter 集中保存在配置文件中。  
> **Remapping**：不修改源码，改变 ROS Graph 中的名字和连接关系。  
> **Launch**：统一组织多个 Node、YAML、Remapping 等系统关系。  
> **Launch Argument**：给 Launch 文件本身提供启动时输入。  
> **CLI**：用于从运行时 Graph、配置、消息和行为多个层次验证系统是否符合设计。

Day 6 的本质是：

```text
从“一个写死配置的单独 Node”
            ↓
走向
            ↓
“一个可配置、可组合、可部署、可验证的 ROS2 小系统”
```

---

# 三十一、今日个人理解与关键收获

本次学习过程中已经表现出以下理解：

1. 能区分 ROS2 Parameter 与 C++ 成员变量，不再把 `range_max_` 和 `"range_max"` 当作同一层概念。
2. 能理解 `ros2 param set` 改变 Parameter 后，若程序没有重新读取或注册回调，成员变量可能仍然保持旧值。
3. 能根据频率自行推导 Timer 周期，并理解 `T = 1/f`。
4. 能区分 Parameter 与 Remapping 的职责：
   - 行为配置 → Parameter
   - Graph 名称映射 → Remapping
5. 能理解为什么 Publisher 和 Subscriber 两边都需要映射到同一个运行时 Topic。
6. 能理解 Launch 不只是“启动程序”，而是组织一个 ROS2 系统。
7. 能区分：
   - Python 变量
   - Launch Argument 名
   - LaunchConfiguration 对象
8. 能理解：
   - Python 构造 Launch 描述
   - Launch 执行 Action
   是两个阶段。
9. 在最终架构分析中，能够独立判断：
   - YAML 放 Parameter
   - Remapping 处理 `/scan → /front_scan`
   - Launch 负责统一启动与组织
   - 两个 Node 因为最终进入同一个运行时 Topic 而通信
10. CLI 的具体拼写还不是完全熟练，但已经能根据“要验证的系统层次”判断应该看 Node、Topic、QoS、Parameter、频率还是具体消息。这说明当前问题主要是命令熟练度，而不是概念理解错误。

---

# 三十二、今天遇到的典型问题与纠正

## 1. 把占位符 `<scan_subscriber>` 直接复制到 Bash

曾经出现类似：

```bash
ros2 run hello_ros2 <scan_subscriber> ...
```

Bash 会把：

```text
<
```

解释成输入重定向，而不是普通文字。

因此文档和教程中的：

```text
<xxx>
```

如果表示“这里替换成真实内容”，不能原样复制。

---

## 2. 一开始对 `output='screen'` 的理解不准确

曾将其联想到：

```text
/screen
```

后来明确：

```text
output='screen'
```

只是：

> 把 Node 的日志输出到运行 Launch 的终端。

---

## 3. 对 LaunchConfiguration 与 DeclareLaunchArgument 的位置产生疑问

疑问本质：

```text
为什么看起来先写 LaunchConfiguration，
后写 DeclareLaunchArgument？
```

最终理解：

```text
LaunchConfiguration
→ Python 构造阶段创建运行时引用

DeclareLaunchArgument
→ LaunchDescription 中的 Launch Action
```

真正运行时仍然是：

```text
先声明 Argument
↓
再在 Node 配置中解析使用
```

这个问题帮助进一步理解了 Launch 的执行模型，而不仅是记语法。

---

# 三十三、Day 6 当前完成度

今日已经实际完成：

```text
Parameter 基础                    ✅
Parameter CLI                     ✅
frame_id 参数化                   ✅
range_min / range_max 参数化      ✅
publish_rate_hz 参数化            ✅
Timer 周期与频率关系              ✅
YAML 参数文件                     ✅
启动时加载 YAML                   ✅
启动时 Parameter override         ✅
运行时 param set 边界             ✅
Remapping                         ✅
Publisher / Subscriber 双边映射   ✅
Python Launch                     ✅
Launch 同时启动两个 Node          ✅
Launch 加载 YAML                  ✅
Launch 配置 Remapping             ✅
CMake 安装 launch/config          ✅
package.xml Launch 运行依赖       ✅
Launch Argument                   ✅
/front_scan → /rear_scan 实验     ✅
ROS2 Graph 验证                   ✅
QoS 验证                          ✅
实际发布频率验证                  ✅
系统架构分析                      ✅
```

Day 6 主线已经完成。

---

# 三十四、后续安排

## 今天不再进行额外自测

原因：

- Day 6 内容较多；
- 今日已经在学习过程中完成了多次局部判断、预测和运行验证；
- 继续立即增加综合自测，容易把“复习巩固”和“新知识输入”混在一起。

因此本日结束时：

```text
不再追加 Day 6 综合自测。
```

## 后续统一进行阶段性自测

下一次对当前 ROS2 学习进度进行：

```text
阶段总结 / 当前进度回顾
```

时，再统一加入综合自测。

届时自测应重点覆盖：

```text
Topic / Publisher / Subscriber
QoS
Parameter
YAML
Remapping
Launch
Launch Argument
ROS2 CLI
运行时 Graph 分析
```

重点不是死记命令，而是检查能否独立完成：

```text
需求分析
↓
选择 ROS2 机制
↓
设计系统结构
↓
运行
↓
使用 CLI 分层验证
```

---

# 三十五、Day 6 结束时的能力状态

经过本日学习，目前已经不再只是：

```text
“会写一个 ROS2 Publisher / Subscriber”
```

而是开始具备：

```text
“把多个 Node 组织成一个可配置 ROS2 系统”
```

的初步能力。

从工程视角看，今天完成的是从：

```text
单节点编程
```

向：

```text
ROS2 系统配置与集成
```

迈出的第一步。

下一阶段继续学习时，应在此基础上推进，而不需要重新从 Publisher / Subscriber 基础开始。
