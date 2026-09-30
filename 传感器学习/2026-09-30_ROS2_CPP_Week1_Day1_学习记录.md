# Week 1 · Day 1 学习记录：ROS2 知识体检 + C++ 起点确认 + 编译链入门

> 日期：2026-09-30  
> 环境：Windows → WSL2 → Ubuntu 22.04 → ROS2 Humble  
> 开发方式：VS Code + WSL 插件  
> 主要语言：C++  
> 本日定位：不是从零学习 ROS2，而是重新确认已有知识边界，并建立后续 ROS2 C++ 工程学习所需的基础框架。

---

## 1. 今日学习目标

Day 1 的核心目标不是大量学习新 API，而是完成以下几件事：

1. 检查已有 ROS2 知识的真实掌握程度。
2. 检查 C++ 的实际起点。
3. 把此前零散掌握的 ROS2 知识重新组织成统一框架。
4. 明确 Windows、WSL2、Ubuntu、ROS2 之间的关系。
5. 建立普通 C++ 程序从源码到可执行程序的基本编译链。
6. 为 Day 2 的 Workspace / Package / CMake / colcon 学习做好准备。

---

# 2. Part A：ROS2 知识体检

## 2.1 Publisher、Timer、Callback、Executor

### 已有理解

对于下面这类 ROS2 C++ 代码：

```cpp
publisher_ = this->create_publisher<std_msgs::msg::String>("chatter", 10);

timer_ = this->create_wall_timer(
    500ms,
    std::bind(&MinimalPublisher::timer_callback, this)
);
```

以及：

```cpp
void timer_callback()
{
    std_msgs::msg::String message;
    message.data = "Hello ROS2";
    publisher_->publish(message);
}
```

我已经能正确区分：

- `create_publisher()`：创建 Publisher 通信实体。
- Timer 到期：并不代表 callback 立刻执行，而是 callback 进入 ready / 可执行状态。
- Executor：负责发现并调度 ready 的 callback。
- `publisher_->publish(message)`：真正将消息发布到 Topic。

### 需要特别注意

`publish()` 不是“打印到终端”。

```text
publish()       → 向 ROS2 Topic 发布消息
RCLCPP_INFO()   → 输出日志
```

因此，Publisher 的创建与真正发送消息是两个不同阶段。

### 当前结论

```text
Publisher 创建 / publish        已掌握
Timer ready 机制               已掌握
Callback 基本概念              已掌握
Executor 基本职责              基本掌握
```

---

## 2.2 Service、Future 与 spin_until_future_complete

典型代码：

```cpp
auto future = client_->async_send_request(request);

rclcpp::spin_until_future_complete(node, future);
```

### 我的理解

Service Client 发送请求后，结果不会立即返回，因此使用 Future 表示“未来会得到的结果”。

主线：

```text
Client 发送 Request
        ↓
获得 Future
        ↓
Server 处理请求
        ↓
Response 返回
        ↓
Future ready
        ↓
Client 取得结果
```

### Future 的正确理解

Future 不是“不断变化的结果值”，而是：

> 一个代表“未来结果”的对象 / 句柄。

调用：

```cpp
auto future = client_->async_send_request(request);
```

时 Future 已经存在，只是结果暂时没有 ready。

### spin_until_future_complete 的理解

`spin_until_future_complete()` 会阻塞当前调用流程，直到 Future 完成。

但它并不是简单的忙等待。

可以理解为：

```text
等待 Future
    │
    ├─ 继续处理 ROS2 事件
    ├─ 调度相关 callback
    ├─ 处理 Service Response
    └─ Future ready
          ↓
        返回
```

### 与 FreeRTOS 的类比

我提出了类比：

> `spin_until_future_complete()` 有点像 FreeRTOS 中某些“当前任务等待，但系统整体仍然继续运行”的机制。

这个类比在直觉上是有帮助的，但要注意：

- 当前调用流程本身是阻塞的；
- ROS2 执行系统并没有完全停住；
- 与 FreeRTOS 的任务调度机制并不是完全相同的实现。

可记为：

```text
当前代码等待
≠
整个系统停摆
```

---

## 2.3 Topic 与 Service 的本质区别

### Topic

Topic 是 Publish–Subscribe 模型。

```text
Publisher
   │
   ├──> Subscriber A
   ├──> Subscriber B
   └──> Subscriber C
```

特点：

- Publisher 只负责发布数据；
- 通常不关心具体是谁接收；
- 一个 Topic 可以被多个 Subscriber 订阅。

适用于：

```text
/scan
/odom
/cmd_vel
传感器持续数据
状态广播
```

### Service

Service 是 Request–Response 模型。

```text
Client
   │ Request
   ▼
Server
   │ Response
   ▼
Client
```

特点：

- Client 明确发起一次请求；
- Server 对请求进行处理；
- Client 明确等待这次请求对应的 Response。

### 核心区别

```text
Topic
→ 数据分发

Service
→ 请求某个功能，并等待一次响应
```

---

## 2.4 Action

Action 适合较长时间任务，但“时间长”并不是唯一原因。

典型过程：

```text
Goal
  ↓
执行任务
  ├─ Feedback
  ├─ Cancel / Goal 管理
  └─ Result
```

例如机器人导航：

```text
Goal：
前往 302 房间

Feedback：
当前进度 / 当前位置 / 剩余距离

Result：
成功 / 失败
```

### 与 Service 的区别

Service：

```text
Request
   ↓
等待
   ↓
Response
```

Action：

```text
Goal
   ↓
长期执行
   ├─ 过程反馈
   ├─ 中途取消 / 管理
   └─ 最终结果
```

因此导航、机械臂长动作等任务通常更适合 Action。

---

# 3. Part A 中发现的工程结构知识缺口

ROS2 通信模型掌握较好，但工程结构层存在明显缺口。

---

## 3.1 source install/setup.bash

原先误以为：

```text
source install/setup.bash
→ 下载依赖
```

这是错误的。

正确理解：

```text
colcon build
    ↓
生成 build / install / log
    ↓
source install/setup.bash
    ↓
让当前 Shell 能发现这个 Workspace 新构建出来的 ROS2 资源
```

### 关键区别

```text
colcon build
→ 把东西构建出来

source install/setup.bash
→ 把构建结果加入当前终端环境
```

它不会下载依赖。

下载 / 安装依赖更接近：

```bash
apt install ...
rosdep install ...
```

---

## 3.2 Workspace

Workspace 不是简单的“当前所在文件夹”。

典型结构：

```text
~/ros2_ws/
├── src/
├── build/
├── install/
└── log/
```

理解：

> Workspace 是用于组织、构建一组 ROS2 Package 的工程目录。

例如：

```text
ros2_ws/
└── src/
    ├── package_A/
    └── package_B/
```

---

## 3.3 Package

Package 是 ROS2 工程中的代码组织单元。

例如：

```text
my_robot_driver
my_robot_description
my_navigation
my_slam
```

一个 Workspace 可以包含多个 Package。

```text
Workspace
   ↓
Package
```

---

## 3.4 Executable

Executable 是操作系统层面的可执行程序。

例如：

```text
publisher.cpp
    ↓
编译 + 链接
    ↓
publisher_node
```

其中：

```text
publisher_node
→ Executable
```

运行：

```bash
ros2 run my_robot publisher_node
```

可拆解为：

```text
ros2 run   my_robot   publisher_node
           ↑          ↑
         Package   Executable
```

---

## 3.5 Node 与 Executable 的区别

最重要的新认识之一：

```text
Executable
→ 操作系统层面的程序

Node
→ ROS2 运行时中的通信 / 功能实体
```

典型链条：

```text
publisher.cpp
    ↓
编译 + 链接
    ↓
publisher_node        ← Executable
    ↓
运行
    ↓
创建 rclcpp::Node
    ↓
/my_publisher         ← Node
```

### 自测结果

```text
ros2_ws          → Workspace
my_robot         → Package
publisher_node   → Executable
/my_publisher    → Node
```

已能正确区分。

---

# 4. Part B：C++ 知识体检

我的 C++ 状态不是零基础，而是：

> 有 C 语言基础，也见过一些 C++ / ROS2 代码，但很多 C++ 概念没有系统建立。

---

## 4.1 class 与 object

示例：

```cpp
class MinimalPublisher
{
};
```

这表示：

> 定义一个类 / 类型。

不是立即创建一个节点实例。

真正创建对象：

```cpp
MinimalPublisher obj;
```

或者：

```cpp
auto node = std::make_shared<MinimalPublisher>();
```

所以：

```text
class
→ 类型 / 蓝图

object
→ 根据 class 创建出来的具体实例
```

---

## 4.2 继承

代码：

```cpp
class MinimalPublisher : public rclcpp::Node
```

表示：

> `MinimalPublisher` 继承自 `rclcpp::Node`。

关系：

```text
rclcpp::Node
     ↑
     │ inheritance
MinimalPublisher
```

因此 `MinimalPublisher` 可以使用 Node 提供的能力：

```cpp
create_publisher(...)
create_subscription(...)
create_wall_timer(...)
get_logger()
```

---

## 4.3 构造函数

```cpp
MinimalPublisher()
{
}
```

这是：

> Constructor（构造函数）

创建对象时自动执行，用于初始化对象。

---

## 4.4 初始化列表

代码：

```cpp
MinimalPublisher()
    : Node("minimal_publisher")
{
}
```

这里：

```cpp
: Node("minimal_publisher")
```

属于：

> Member Initializer List（成员初始化列表）

在这里主要作用是：

```text
创建 MinimalPublisher
        ↓
先构造基类 Node
        ↓
Node("minimal_publisher")
        ↓
设置 ROS2 Node 名称
        ↓
进入 MinimalPublisher 构造函数主体
```

这一部分目前只是初步认识，后续 Day 3 需要系统学习。

---

## 4.5 引用

示例：

```cpp
void changeA(int x)
{
    x = 10;
}

void changeB(int &x)
{
    x = 10;
}
```

调用：

```cpp
int a = 5;
int b = 5;

changeA(a);
changeB(b);
```

结果：

```text
a = 5
b = 10
```

原因：

### 值传递

```cpp
void changeA(int x)
```

会复制：

```text
a
 ↓ copy
x
```

修改 `x` 不影响 `a`。

### 引用传递

```cpp
void changeB(int &x)
```

其中 `x` 是外部变量的引用 / 别名。

```text
b ─────┐
       ├── 同一个对象
x ─────┘
```

修改 `x` 就是在修改 `b`。

---

## 4.6 const reference

```cpp
const int &x
```

可以理解为：

> 只读引用。

即：

```text
&
→ 不复制，直接引用原对象

const
→ 不允许通过这个引用修改原对象
```

例如：

```cpp
int a = 5;
const int &x = a;

a = 10;      // 可以
// x = 10;   // 不可以
```

重要理解：

> `const int &x` 不代表原变量永远不能变化，只代表不能通过 `x` 修改它。

---

## 4.7 namespace 与 ::

代码：

```cpp
std::string
rclcpp::Node
std_msgs::msg::String
```

其中：

```cpp
::
```

叫：

> Scope Resolution Operator（作用域解析运算符）

例如：

```cpp
std::string
```

表示：

> 到 `std` 命名空间中找 `string`。

```cpp
std_msgs::msg::String
```

可以理解为：

```text
std_msgs
└── msg
    └── String
```

---

## 4.8 指针

已有 C 指针基础可直接迁移。

```cpp
int a = 10;
int *p = &a;

*p = 20;
```

理解：

```text
p       → 保存 a 的地址
&a      → 取得 a 的地址
*p      → 解引用，访问 p 指向的对象
```

因此：

```cpp
*p = 20;
```

会修改 `a`。

---

## 4.9 public / private

示例：

```cpp
class Robot
{
public:
    int speed;

private:
    int password;
};
```

外部：

```cpp
Robot robot;

robot.speed = 10;       // 可以
robot.password = 1234;  // 不可以
```

理解：

```text
public
→ 类外可以访问

private
→ 只能在类内部访问
```

---

## 4.10 模板：当前明显缺口

例如：

```cpp
std::vector<int>
```

以及：

```cpp
rclcpp::Publisher<std_msgs::msg::String>
```

尖括号中表示模板参数。

初步理解：

```text
std::vector<int>
→ 一个保存 int 的 vector

Publisher<std_msgs::msg::String>
→ 一个处理 std_msgs::msg::String 消息的 Publisher
```

当前只是建立位置感，模板后续需要系统学习。

---

## 4.11 智能指针：当前明显缺口

常见：

```cpp
rclcpp::Publisher<std_msgs::msg::String>::SharedPtr publisher_;
```

以及：

```cpp
auto node = std::make_shared<MinimalPublisher>();
```

当前只建立基础认识：

> `SharedPtr` / `std::make_shared` 属于智能指针体系，用于管理对象生命周期，减少手动 `new/delete`。

引用计数、所有权、析构时机暂未展开。

---

# 5. C++ 体检结论

## 已经有较好基础

```text
变量 / 函数
C 指针
& 取地址
* 解引用
public / private
namespace / ::
引用的基本效果
const 的基础直觉
```

## 基本理解但需要强化

```text
class / object
constructor
inheritance
reference
const reference
```

## 明显知识缺口

```text
初始化列表
template
STL
shared_ptr / make_shared
auto
this
std::bind
Lambda
CMake
编译 / 链接系统
```

---

# 6. Part C：ROS2 整体框架重构

## 6.1 运行时视角

可以把已有 ROS2 知识重新组织成：

```text
ROS2 System
│
├─ Node
│
├─ Communication
│   ├─ Topic
│   ├─ Service
│   ├─ Action
│   └─ Parameter
│
├─ Callback
│
└─ Executor
```

进一步：

```text
Node
│
├─ Publisher
├─ Subscription
├─ Service
├─ Client
├─ Action
├─ Timer
└─ Parameter
```

事件到来后的执行逻辑：

```text
消息到达
Service Request 到达
Timer 到期
Action 状态变化
        ↓
callback ready
        ↓
Executor 调度
        ↓
执行 callback
```

重要结论：

```text
事件发生
≠
callback 立刻执行
```

---

## 6.2 Node 与 Executor 的职责

### Node

> ROS2 运行时中的基本功能实体，用来承载通信、Timer、Callback 等逻辑。

### Executor

> 负责等待、发现并执行 ready 的工作。

简单理解：

```text
Node
→ 提供“有什么事情可以做”

Executor
→ 决定“什么时候执行 ready 的 callback”
```

更深入的执行顺序、多线程 Executor、Callback Group 后续再学。

---

# 7. ROS2 软件栈：rclcpp → rcl → rmw → DDS

本日只建立位置感，不深入 DDS。

```text
Application
   ↓
rclcpp
   ↓
rcl
   ↓
rmw
   ↓
DDS / Middleware
```

---

## 7.1 rclcpp

ROS2 C++ Client Library。

以后直接接触最多：

```cpp
rclcpp::Node
rclcpp::Publisher
rclcpp::Subscription
rclcpp::spin()
```

因此：

```cpp
this->create_publisher(...)
```

直接使用的是：

```text
rclcpp
```

---

## 7.2 rcl

更底层的 ROS 公共核心接口。

可以先理解：

```text
rclcpp
   ↓
rcl
```

不同语言客户端库可以共享更底层的 ROS 核心能力。

---

## 7.3 rmw

`rmw`：

> ROS Middleware Interface

作用：

> 把上层 ROS2 与具体中间件实现隔离。

结构：

```text
rclcpp
   ↓
rcl
   ↓
rmw
 ↙   ↘
不同 DDS / Middleware 实现
```

因此应用代码不应该直接依赖某个具体 DDS API。

---

## 7.4 DDS

当前只需知道：

> DDS 位于 ROS2 通信栈的较底层，是 ROS2 常用的分布式通信中间件体系。

本日不深入：

```text
Domain
Participant
DataWriter
DataReader
```

---

# 8. Part D：真实开发环境确认

实际环境：

```text
Windows
   ↓
WSL2
   ↓
Ubuntu 22.04
   ↓
ROS2 Humble
```

VS Code 使用 WSL 插件连接到 Ubuntu 环境。

---

## 8.1 实际检查结果

### 当前目录

```bash
pwd
```

结果：

```text
/home/glance
```

说明当前位于 Linux 文件系统。

---

### 当前 Linux 用户

```bash
whoami
```

结果：

```text
glance
```

---

### ROS2 发行版

正确命令：

```bash
echo $ROS_DISTRO
```

结果：

```text
humble
```

曾经错误输入：

```bash
$ROS_DISTRO
```

Shell 会先展开变量：

```text
$ROS_DISTRO
    ↓
humble
```

然后把 `humble` 当成命令执行，因此得到：

```text
Command 'humble' not found
```

这个错误反而说明变量本身已经正确设置为 `humble`。

---

### ros2 命令位置

```bash
which ros2
```

结果：

```text
/opt/ros/humble/bin/ros2
```

说明 ROS2 Humble 安装在：

```text
/opt/ros/humble/
```

---

### g++ 版本

```bash
g++ --version
```

结果：

```text
g++ 11.4.0
```

直接运行：

```bash
g++
```

出现：

```text
fatal error: no input files
```

不是安装失败，而是：

> 编译器已经启动，只是没有给它源文件。

---

## 8.2 Linux 路径认识

推荐 Workspace：

```text
/home/glance/ros2_ws
```

或者：

```text
~/ros2_ws
```

其中：

```text
~
→ /home/glance
```

优先使用 Linux 文件系统，而不是：

```text
/mnt/c/...
```

---

# 9. Part E：第一个普通 C++ 编译实验

## 9.1 创建实验目录

```bash
mkdir -p ~/ros2_cpp_week1/day1
cd ~/ros2_cpp_week1/day1
```

### `mkdir -p`

`-p` 可理解为 parents。

作用：

- 父目录不存在时一并创建；
- 目标目录已经存在时通常不会报错。

例如：

```bash
mkdir -p ~/a/b/c
```

即使 `a`、`b` 不存在，也能一次建立整个目录结构。

---

## 9.2 编写 hello.cpp

```cpp
#include <iostream>

int main()
{
    std::cout << "Hello ROS2 C++!" << std::endl;
    return 0;
}
```

### 基本解释

```cpp
#include <iostream>
```

引入标准输入输出功能。

```cpp
int main()
```

程序入口。

```cpp
std::cout
```

标准输出。

这里再次体现：

```text
std::cout
→ std 命名空间里的 cout
```

```cpp
return 0;
```

程序正常结束。

---

## 9.3 一步完成编译和链接

命令：

```bash
g++ hello.cpp -o hello
```

拆解：

```text
g++
→ C++ 编译工具链

hello.cpp
→ 输入源码

-o hello
→ 指定最终输出文件名为 hello
```

生成：

```text
hello.cpp
hello
```

其中：

```text
hello.cpp → Source Code
hello     → Executable
```

Linux 可执行程序不要求 `.exe` 后缀。

运行：

```bash
./hello
```

结果：

```text
Hello ROS2 C++!
```

---

## 9.4 VS Code 右上角运行按钮

可以直接点击 VS Code 右上角三角形运行 / 调试程序。

但当前阶段暂时不能完全依赖按钮。

原因：

> 目前重点不是“怎么方便运行”，而是理解 VS Code 在背后替我完成了什么。

当前应先熟悉：

```text
写源码
→ 编译
→ 链接
→ 生成 executable
→ 运行
```

以后日常开发完全可以正常使用 VS Code 的运行 / 调试功能。

---

# 10. 把编译和链接拆开

执行：

```bash
g++ -c hello.cpp -o hello.o
```

其中：

```text
-c
→ 只编译，不进行最终链接
```

得到：

```text
hello.o
```

然后：

```bash
g++ hello.o -o hello2
```

完成链接，生成：

```text
hello2
```

运行：

```bash
./hello2
```

仍得到：

```text
Hello ROS2 C++!
```

---

## 10.1 三个文件的关系

```text
hello.cpp
   ↓
编译 Compilation
   ↓
hello.o
   ↓
链接 Linking
   ↓
hello2
```

### hello.cpp

> C++ 源代码。

### hello.o

> Object File（目标文件）。

已经完成编译，但还不是最终可执行程序。

### hello2

> Executable。

已经完成链接，可以运行。

---

# 11. 当前建立的 C++ 编译链

完整第一版模型：

```text
hello.cpp
    ↓
预处理 Preprocessing
    ↓
编译 Compilation
    ↓
hello.o
    ↓
链接 Linking
    ↓
hello / hello2
    ↓
运行
```

Day 1 不深入预处理器、汇编、中间代码、符号解析等内部机制。

核心是先建立：

```text
源代码
→ 编译
→ 目标文件
→ 链接
→ 可执行程序
```

---

# 12. 与后续 ROS2 C++ 工程的连接

普通 C++：

```text
.cpp
 ↓
g++ 编译
 ↓
.o
 ↓
链接
 ↓
Executable
```

以后 ROS2 C++：

```text
.cpp
 ↓
CMakeLists.txt 描述构建规则
 ↓
CMake
 ↓
编译 + 链接
 ↓
Executable
 ↓
colcon 组织整个 Workspace / 多个 Package 的构建
 ↓
install/
 ↓
source install/setup.bash
 ↓
ros2 run package executable
```

因此以后看到：

```cmake
add_executable(my_node src/my_node.cpp)
```

不应该只把它当成“ROS2 模板代码”。

它的本质是：

> 告诉 CMake：把这个 `.cpp` 构建成一个叫 `my_node` 的可执行程序。

---

# 13. 今日知识状态总结

## ROS2 已掌握

```text
Node 基本概念
Publisher / Subscriber
Topic
Service
Client / Server
Action
Goal / Feedback / Result
Timer ready 机制
Callback
Executor 基本职责
Future
Topic / Service / Action 区别
```

---

## ROS2 需要后续强化

```text
Workspace 工程结构
Package
Executable
Node vs Executable
source install/setup.bash
CMakeLists.txt
ament_cmake
colcon
Executor / spin 更深入机制
```

---

## C++ 已有基础

```text
基本变量 / 函数
C 指针
取地址 / 解引用
public / private
namespace / ::
引用基本效果
const 基础
```

---

## C++ 需要强化

```text
class vs object
constructor
inheritance
initializer list
reference
const reference
```

---

## C++ 明显缺口

```text
template
STL
shared_ptr
make_shared
auto
this
std::bind
Lambda
CMake
编译 / 链接工程机制
```

---

# 14. 今日实际完成的实验

环境确认：

```text
当前 Linux 用户：
glance

Linux 用户目录：
/home/glance

ROS2：
Humble

ros2 路径：
/opt/ros/humble/bin/ros2

g++：
11.4.0
```

完成代码：

```text
~/ros2_cpp_week1/day1/hello.cpp
```

完成：

```bash
g++ hello.cpp -o hello
./hello
```

以及：

```bash
g++ -c hello.cpp -o hello.o
g++ hello.o -o hello2
./hello2
```

---

# 15. Day 1 完成标准检查

```text
✅ 能正确说明 Windows / WSL2 / Ubuntu / ROS2 的层级关系

✅ 完成 ROS2 知识体检

✅ 完成 C++ 基础体检

✅ 能把 Node / Topic / Service / Action / Callback / Executor
   放入统一框架

✅ 初步理解：
   rclcpp → rcl → rmw → DDS

✅ 亲手编写普通 C++ 程序

✅ 使用 g++ 编译并运行

✅ 理解：
   源代码 → 编译 → 目标文件 → 链接 → 可执行程序

✅ 已明确 Day 2 的重点
```

结论：

> **Week 1 · Day 1 已完成，达到预定完成标准。**

---

# 16. Day 2 路线调整建议

原计划方向保持不变，但根据 Day 1 体检结果，Day 2 应强化工程结构，而不是重新讲 Node 基础定义。

## Day 2 重点

```text
Workspace
Package
Executable
CMakeLists.txt
add_executable
ament_cmake
colcon build
build / install / log
source install/setup.bash
```

## 配套 C++ / 工程基础

```text
#include
头文件
声明与定义
编译单元
链接
namespace
```

Node 基本概念不需要从零重复。

---

# 17. Day 3 预期接口

Day 3 再正式进入：

```cpp
class MinimalNode : public rclcpp::Node
```

届时同步补：

```text
class
object
inheritance
constructor
initializer list
```

这样更符合当前真实知识水平：

> 不先独立学完 C++，而是在 ROS2 C++ 代码中同步补齐真正需要的语言知识。

---

# 18. 一页快速复习

## ROS2 通信

```text
Topic
→ Publish / Subscribe
→ 数据分发

Service
→ Request / Response
→ 一次请求对应一次响应

Action
→ Goal / Feedback / Result / Cancel
→ 长任务 + 过程反馈 + 中途管理
```

## ROS2 执行

```text
事件发生
   ↓
Callback ready
   ↓
Executor 调度
   ↓
Callback 执行
```

## ROS2 工程对象

```text
Workspace
   ↓
Package
   ↓
源代码
   ↓
Executable
   ↓
运行
   ↓
Node
```

## ROS2 软件栈

```text
Application
   ↓
rclcpp
   ↓
rcl
   ↓
rmw
   ↓
DDS / Middleware
```

## C++ 编译链

```text
.cpp
 ↓
编译
 ↓
.o
 ↓
链接
 ↓
Executable
```

## source 的作用

```text
colcon build
→ 构建结果产生

source install/setup.bash
→ 让当前 Shell 能发现这些构建结果
```

---

# 19. 后续复习时应重点检查的几个问题

1. `create_publisher()` 和 `publish()` 的区别是什么？
2. Timer 到期后，callback 为什么不一定马上执行？
3. Executor 在 ROS2 中承担什么职责？
4. Future 为什么适合 Service 的异步请求？
5. Topic、Service、Action 的通信语义分别是什么？
6. Workspace、Package、Executable、Node 有什么区别？
7. `source install/setup.bash` 到底做了什么？
8. `class` 和 `object` 的区别是什么？
9. `int &x` 与 `const int &x` 有什么区别？
10. `rclcpp::Node` 中的 `::` 是什么意思？
11. `hello.cpp → hello.o → hello` 分别对应什么阶段？
12. 为什么 `hello.o` 还不是最终可执行程序？
13. `rclcpp → rcl → rmw → DDS` 各层大概位于什么位置？
14. 为什么更换底层 DDS 实现时，上层 `rclcpp` 应用代码通常不需要大改？

---

> 本日最重要的成果不是“又学了一遍 ROS2 基础”，而是明确了真实起点：
>
> **ROS2 通信与基本执行模型已经有较好基础；真正需要系统补的是 ROS2 C++ 工程结构、C++ 面向对象机制、模板 / 智能指针，以及 CMake / colcon 构建链。**
