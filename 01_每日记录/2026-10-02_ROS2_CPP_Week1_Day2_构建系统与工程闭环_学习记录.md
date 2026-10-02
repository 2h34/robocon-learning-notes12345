# Week 1 · Day 2 学习记录：ROS2 构建系统与工程闭环

> 日期：2026-10-02  
> 环境：Windows + WSL2 + Ubuntu 22.04 + ROS2 Humble  
> 开发方式：VS Code + WSL  
> 工作目录优先：`/home/glance/...`

---

## 1. 今天的学习目标

Day 2 的核心问题是：

> **一个 ROS2 C++ 源文件，如何经过 Workspace、Package、CMake、ament_cmake、colcon，最终变成 `ros2 run` 可以找到并启动的 Executable？**

今天不再从零复习 Node、Topic、Service、Action 等概念，而是在 Day 1 已经掌握的基础上，重点建立 ROS2 工程构建主线。

最终需要形成的完整链条：

```text
源码 .cpp
↓
ROS2 Package
↓
CMakeLists.txt
↓
CMake / ament_cmake
↓
colcon build
↓
build / install / log
↓
source install/setup.bash
↓
ROS2 能发现 Package / Executable
↓
ros2 run package executable
```

---

# 2. Day 1 快速回顾结果

Day 2 开始前先检查了三个基础点。

## 2.1 Workspace / Package / Executable / Node

已经能够正确区分：

```text
ros2_ws          → Workspace
my_robot         → Package
publisher_node   → Executable
/my_publisher    → Node
```

当前理解：

- **Workspace**：组织和构建一组 ROS2 Package 的工程空间。
- **Package**：ROS2 中的代码组织与依赖管理单元。
- **Executable**：操作系统层面的可执行程序。
- **Node**：ROS2 运行时中的功能与通信实体。

注意：

```text
一个 Executable = 一个 Node
```

只是教程和简单工程中的常见情况，并不是绝对关系。

---

## 2.2 普通 C++ 编译链

Day 1 已经掌握：

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

例如：

```bash
g++ -c hello.cpp -o hello.o
g++ hello.o -o hello
./hello
```

对应：

```text
hello.cpp → Source Code
hello.o   → Object File
hello     → Executable
```

Day 2 的重点不是重新学习这条链，而是把它接入 ROS2 工程体系。

---

## 2.3 `source install/setup.bash`

已经纠正之前的错误理解：

错误：

```text
source install/setup.bash
→ 下载依赖
```

正确：

```text
colcon build
→ 构建出结果

source install/setup.bash
→ 修改当前 Shell 环境
→ 让 ROS2 能发现当前 Workspace 新构建出来的资源
```

---

# 3. 为什么不能一直手写 `g++`

简单程序可以直接：

```bash
g++ hello.cpp -o hello
```

但真实机器人项目可能包含：

```text
main.cpp
robot.cpp
sensor.cpp
controller.cpp
多个头文件
多个库
多个 ROS2 依赖
```

如果全部手写 `g++` 命令，会遇到：

- 源文件越来越多；
- include 路径越来越多；
- 库路径和链接参数越来越复杂；
- 修改一个文件后不希望每次全量重编；
- 多个 Package 之间还存在依赖关系。

因此需要 **Build System（构建系统）**。

---

# 4. g++、CMake、ament_cmake、colcon 的层级

这是今天最重要的概念之一。

## 4.1 g++

`g++` 是实际执行 C++ 编译和链接的工具。

例如：

```bash
g++ main.cpp math_utils.cpp -o demo
```

---

## 4.2 CMake

CMake 不是编译器。

它负责：

> **描述和组织“这个 C++ 工程应该怎么构建”。**

例如：

```cmake
add_executable(demo main.cpp math_utils.cpp)
```

表示：

```text
main.cpp
      \
       → demo executable
      /
math_utils.cpp
```

CMake 的职责不是亲自“编译”，而是根据 `CMakeLists.txt` 生成构建规则，再调用底层编译工具完成构建。

可以记成：

```text
g++   = 直接“编”
CMake = 描述“怎么编”
```

---

## 4.3 ament_cmake

普通 CMake 能构建普通 C++ 工程，但 ROS2 还需要：

- ROS2 Package 约定；
- ROS2 依赖处理；
- 安装规则；
- Package 信息；
- ROS2 资源发现。

因此有：

```text
CMake
→ 通用 C/C++ 构建系统

ament_cmake
→ ROS2 在 CMake 之上的扩展和约定体系
```

可以暂时记成：

```text
ament_cmake
= 让 CMake 工程按照 ROS2 Package 的方式工作
```

当前不需要深入 ament 的内部实现。

---

## 4.4 colcon

`colcon` 工作在 Workspace 层。

例如：

```text
Workspace
├── Package A
├── Package B
└── Package C
```

若：

```text
C 依赖 B
B 依赖 A
```

那么由 `colcon` 在 Workspace 层组织：

```text
A → B → C
```

所以：

```text
colcon ≠ compiler
colcon ≠ CMake
```

而是：

> **Workspace-level build orchestration tool**
>
> 工作空间级构建组织工具。

---

## 4.5 四者最简记忆

```text
g++            负责“编”
CMake          负责“怎么编”
ament_cmake    负责“按 ROS2 的方式编”
colcon         负责“一堆 Package 怎么一起编”
```

---

# 5. `package.xml` 与 `CMakeLists.txt`

## 5.1 `package.xml`

主要描述：

> **我是谁？我依赖谁？**

包括：

- Package 名称；
- 版本；
- 描述；
- 维护者；
- License；
- Dependencies；
- Build Type 等元信息。

例如：

```xml
<depend>rclcpp</depend>
```

表示当前 Package 依赖 `rclcpp`。

当前理解：

```text
package.xml
→ ROS2 Package / 生态层面的元信息和依赖声明
```

---

## 5.2 `CMakeLists.txt`

主要描述：

> **这个 Package 实际应该如何构建。**

包括：

- 找哪些依赖；
- 哪些源文件生成哪些 executable；
- target 使用哪些依赖；
- 如何安装 executable；
- 如何完成 ament Package 配置。

当前理解：

```text
CMakeLists.txt
→ 实际构建层面的配置
```

---

## 5.3 二者的关系

不要简单理解成“两个文件都写依赖”。

更准确是：

```text
package.xml
→ Package / ROS2 生态层面声明依赖

CMakeLists.txt
→ 构建层面实际找到并使用依赖
```

---

# 6. 创建第一个 ROS2 C++ Package

建立 Workspace：

```bash
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src
```

创建 Package：

```bash
ros2 pkg create hello_ros2 --build-type ament_cmake --dependencies rclcpp
```

生成类似结构：

```text
hello_ros2/
├── CMakeLists.txt
├── package.xml
└── src/
```

其中：

```text
ros2_ws/         → Workspace
hello_ros2/      → Package
```

---

# 7. 第一个 ROS2 C++ Node

最终源码：

```cpp
#include "rclcpp/rclcpp.hpp"

class HelloNode : public rclcpp::Node
{
public:
    HelloNode() : Node("hello_node")
    {
        RCLCPP_INFO(this->get_logger(), "Hello ROS2!");
    }
};

int main(int argc, char *argv[])
{
    rclcpp::init(argc, argv);

    auto node = std::make_shared<HelloNode>();
    rclcpp::spin(node);

    rclcpp::shutdown();
    return 0;
}
```

今天只要求理解最小执行主线：

```text
main()
↓
rclcpp::init(...)
↓
创建 HelloNode
↓
rclcpp::spin(node)
↓
rclcpp::shutdown()
```

以下内容暂时不深入：

- `std::make_shared`
- `shared_ptr`
- `auto`
- 继承内部机制
- `spin()` 与 Executor / Callback 的详细关系

这些放到后续真正用到时再学。

---

# 8. CMakeLists.txt 中真正关键的构建链

今天实际使用的核心配置：

```cmake
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)

add_executable(hello_node src/hello_node.cpp)
ament_target_dependencies(hello_node rclcpp)

install(TARGETS
  hello_node
  DESTINATION lib/${PROJECT_NAME}
)

ament_package()
```

---

## 8.1 `find_package()`

例如：

```cmake
find_package(rclcpp REQUIRED)
```

作用：

> 找到当前构建所需要的依赖。

---

## 8.2 `add_executable()`

```cmake
add_executable(hello_node src/hello_node.cpp)
```

表示：

```text
src/hello_node.cpp
↓
构建
↓
hello_node executable
```

它本质上不是 ROS2 魔法，而是普通 CMake 的构建目标描述。

---

## 8.3 `ament_target_dependencies()`

```cmake
ament_target_dependencies(hello_node rclcpp)
```

作用：

> 让 `hello_node` 这个 target 使用 `rclcpp` 依赖。

---

## 8.4 `install(TARGETS ...)`

今天通过真实报错真正理解了这一项。

```cmake
install(TARGETS
  hello_node
  DESTINATION lib/${PROJECT_NAME}
)
```

核心区别：

```text
add_executable()
→ 把程序“做出来”

install()
→ 把程序“放到 ROS2 规定的位置”
```

`ros2 run` 依赖安装后的 ROS2 Package 布局，而不是直接去源码目录寻找 `.cpp` 文件。

---

# 9. `colcon build` 与 Workspace 构建

在 Workspace 根目录：

```bash
cd ~/ros2_ws
colcon build
```

作用：

```text
Workspace
↓
colcon
↓
识别 Package
↓
调用各 Package 的构建系统
↓
生成 build / install / log
```

对于多个 Package：

```text
build/
├── package_A/
├── package_B/
└── package_C/
```

每个 Package 有自己的构建目录，而不是所有中间文件混在一起。

---

# 10. `build / install / log`

这部分不再抠内部文件，只掌握工程职责。

```text
src/
→ 自己写的源码和 Package

build/
→ 构建过程中的中间结果

install/
→ 构建后供运行及其他 Package 使用的结果

log/
→ colcon 构建日志
```

当前阶段不需要研究：

- `CMakeCache.txt`
- `CMakeFiles/`
- install 内部每一个目录
- colcon 内部实现

只需要知道出了构建问题时，这些目录分别承担什么作用。

---

# 11. `source install/setup.bash`

构建成功之后：

```bash
source install/setup.bash
```

它不是：

- 下载依赖；
- 重新构建；
- 安装软件。

而是：

> **在当前 Shell 中执行 `setup.bash`，修改当前终端环境，让 ROS2 能发现当前 Workspace 的安装结果。**

核心关系：

```text
colcon build
= 把东西构建出来

source install/setup.bash
= 告诉当前终端东西在哪里
```

---

## 11.1 为什么只影响当前终端？

执行：

```bash
source install/setup.bash
```

修改的是当前 Shell 环境。

因此：

```text
终端 A：
source 过
→ 能找到 hello_ros2

新开的终端 B：
没有 source
→ 不一定能找到 hello_ros2
```

新的 Shell 不知道另一个已经打开的 Shell 后来修改了什么。

所以新终端通常需要重新：

```bash
source ~/ros2_ws/install/setup.bash
```

---

# 12. 今天完成的普通 C++ 多文件实验

建立：

```text
cpp_multifile/
├── main.cpp
├── math_utils.hpp
└── math_utils.cpp
```

`math_utils.hpp`：

```cpp
#ifndef MATH_UTILS_HPP
#define MATH_UTILS_HPP

int add(int a, int b);

#endif
```

`math_utils.cpp`：

```cpp
#include "math_utils.hpp"

int add(int a, int b)
{
    return a + b;
}
```

`main.cpp`：

```cpp
#include <iostream>
#include "math_utils.hpp"

int main()
{
    std::cout << add(2, 3) << '\n';
    return 0;
}
```

分别编译：

```bash
g++ -c main.cpp -o main.o
g++ -c math_utils.cpp -o math_utils.o
```

链接：

```bash
g++ main.o math_utils.o -o demo
./demo
```

输出：

```text
5
```

---

## 12.1 Translation Unit 初步理解

当前只需要知道：

> 一个 `.cpp` 经过预处理（包括 `#include` 展开）后，作为一个独立单位进行编译。

例如：

```text
main.cpp       → main.o
math_utils.cpp → math_utils.o
```

最后再链接：

```text
main.o + math_utils.o
↓
demo
```

由于已经有较熟练的 C 语言基础，这部分以后不再过度展开。

---

# 13. 使用 CMake 构建同一个普通 C++ 工程

`CMakeLists.txt`：

```cmake
cmake_minimum_required(VERSION 3.10)

project(cpp_multifile)

add_executable(demo main.cpp math_utils.cpp)
```

构建：

```bash
mkdir -p build
cd build
cmake ..
cmake --build .
./demo
```

核心理解：

```text
add_executable(demo main.cpp math_utils.cpp)
```

表示：

> CMake 需要创建 `demo` 这个 executable，它由 `main.cpp` 与 `math_utils.cpp` 构成。

而之前的 `g++` 命令是直接执行编译和链接。

因此：

```text
CMakeLists.txt
→ 描述“我要什么”

CMake
→ 根据描述生成构建规则

g++ 等编译工具
→ 真正执行编译和链接
```

---

# 14. 今天遇到的真实问题与排错

## 14.1 错误一：`undefined reference to main`

构建时出现：

```text
undefined reference to `main'
```

检查真实源码后发现：

```cpp
class HelloNode : public rclcpp::Node
{
    ...
};
```

存在，但没有：

```cpp
int main(...)
```

### 原因

代码已经走到了链接阶段，但链接器找不到 executable 的程序入口。

过程：

```text
hello_node.cpp
↓
编译基本成功
↓
目标文件
↓
链接 executable
↓
找不到 main()
↓
失败
```

### 收获

真正区分：

```text
编译错误
≠
链接错误
```

并建立经验：

```text
undefined reference
→ 优先想到链接阶段
```

---

## 14.2 错误二：`No executable found`

执行：

```bash
ros2 run hello_ros2 hello_node
```

出现：

```text
No executable found
```

### 原因

Package 已经能被 ROS2 找到，但 executable 没有按照 ROS2 约定安装到 install 空间。

缺少：

```cmake
install(TARGETS
  hello_node
  DESTINATION lib/${PROJECT_NAME}
)
```

### 收获

真正理解：

```text
add_executable()
→ 能构建 executable

install()
→ 让 executable 进入 ROS2 可运行的安装布局
```

并建立初步排错规则：

```text
Package 'xxx' not found
→ ROS2 连 Package 都没有发现
→ 优先检查 source / Workspace 环境

No executable found
→ Package 找到了，但运行目标没找到
→ 优先检查 install()、executable 名称等
```

---

## 14.3 错误三：`No such file or directory`

执行：

```bash
g++ -c main.cpp -o main.o
```

出现：

```text
main.cpp: No such file or directory
```

检查终端发现当前位于：

```text
/home/glance
```

而文件在另一个实验目录。

### 收获

Linux 工程中遇到：

```text
No such file or directory
```

优先检查：

```bash
pwd
ls
```

即：

```text
我现在在哪？
我要操作的文件是否真的在这里？
```

先检查路径，再怀疑代码或编译器。

---

# 15. 今天形成的完整 ROS2 构建链

最终已经实际走通：

```text
hello_node.cpp
      ↓
ROS2 Package：hello_ros2
      ↓
CMakeLists.txt
      │
      ├── find_package()
      ├── add_executable()
      ├── ament_target_dependencies()
      └── install(TARGETS ...)
      ↓
ament_cmake
      ↓
colcon build
      ↓
build/   install/   log/
             ↓
source install/setup.bash
             ↓
当前 Shell 能发现 Workspace
             ↓
ros2 run hello_ros2 hello_node
             ↓
hello_node executable 启动
             ↓
创建 ROS2 Node
```

这是 Day 2 最核心的主线。

---

# 16. 当前真实掌握情况

## 已经比较稳固

- Workspace / Package / Executable / Node 区分；
- `.cpp → .o → executable`；
- 编译与链接的区别；
- CMake 为什么存在；
- `g++` 与 CMake 的区别；
- `CMake`、`ament_cmake`、`colcon` 的基本层级；
- `package.xml` 与 `CMakeLists.txt` 的职责区分；
- `add_executable()` 的作用；
- `install(TARGETS ...)` 的作用；
- `build / install / log` 的基本职责；
- `colcon build` 的作用；
- `source install/setup.bash` 的作用；
- source 只修改当前 Shell 环境；
- ROS2 为什么不是直接从 `src/` 读取源码运行；
- 一个 `.cpp` 如何最终成为 `ros2 run` 可以启动的 executable。

---

## 已经接触，但暂时不要求深入

- `std::make_shared`
- `shared_ptr`
- `auto`
- C++ inheritance 细节
- constructor / initializer list 细节
- `rclcpp::spin()` 内部机制
- Translation Unit 细节
- CMake 内部生成 Makefile / Ninja 的机制
- ament 内部实现
- build/install 内部详细目录结构

后续按照实际 ROS2 代码使用场景再补。

---

# 17. 今天对学习路线的重要调整

今天学习过程中发现：

原来的路线对以下内容投入过多：

- 普通 C++ 基础；
- CMake 内部细节；
- build/install 目录细节；
- Translation Unit 等工程基础。

但当前真正的长期目标是：

```text
ROS2
→ 传感器
→ TF / 坐标系
→ 机器人数据流
→ 多传感器系统
→ SLAM
```

因此后续调整为：

> **以 ROS2 真实机器人应用主线为核心，C++ 和构建知识采用 Just-in-time Learning：遇到实际代码时再补足当前需要的部分。**

---

# 18. 调整后的近期学习路线

```text
Day 1  ✅ ROS2 框架 + C++ 编译基本认识

Day 2  ✅ ROS2 工程构建最小闭环

Day 3
ROS2 C++ Node
→ 真正做到会读、会改、会运行

Day 4
Publisher / Subscriber
→ 建立真实 ROS2 数据流

Day 5
sensor_msgs
→ Imu / LaserScan / PointCloud2 / Image
→ timestamp
→ frame_id

Day 6
QoS
→ 从传感器通信角度理解 BEST_EFFORT / RELIABLE
→ 解决“Topic 明明存在但收不到数据”等问题

Day 7
Parameter + Launch
→ 向真实机器人启动和配置方式过渡
```

随后进入：

```text
TF2
↓
URDF / robot_state_publisher
↓
RViz2
↓
rosbag2
↓
真实传感器
↓
时间戳 / frame_id / 频率 / 延迟 / 同步
↓
IMU / LiDAR / Camera 等数据流
↓
多传感器融合
↓
SLAM
```

Service / Action 不删除，但不再为了“基础完整性”专门占用大量时间，后续根据机器人系统需求再补充 C++ 实现。

---

# 19. 后续代码教学规范

今天进一步确定：

> **当前实验闭环所必需的代码、命令和配置必须完整给出；可以延后解释原理，但不能为了分步教学而漏掉必要代码。**

例如：

- `main()` 可以暂时不深入每行机制，但不能遗漏；
- `install(TARGETS ...)` 可以先讲用途，但不能等到报错后才补；
- `make_shared` 可以暂不深入；
- 只要它不是当前实验正确运行的必要理解，就可以后置。

代码展示方面：

- 避免过度换行；
- 一行能清楚表达的代码不拆成多行；
- 遵循工程中常见 C++ / ROS2 代码风格；
- 重点保证代码可读、可理解、可掌控。

---

# 20. Day 2 最终结论

按照最初的完整课程标准，仍有一些 C++ / CMake 细节没有深入。

但按照当前重新明确的目标：

> **能够理解并掌控 ROS2 工程构建主线，并尽快进入机器人数据流与传感器应用**

Day 2 的核心目标已经完成。

今天真正需要长期保留的不是很多命令，而是下面这条关系：

```text
Source Code
↓
Package
↓
CMake
↓
ament_cmake
↓
colcon
↓
install
↓
source
↓
ros2 run
```

以及：

```text
g++            → 真正编译 / 链接
CMake          → 描述怎么构建
ament_cmake    → 让 CMake 按 ROS2 Package 规则工作
colcon         → Workspace 层组织多个 Package 的构建
```

如果能够脱离笔记自行解释这两张图，Day 2 的主干就已经真正掌握。

---

# 21. Day 3 开始前只需快速复习

下一次开始 Day 3 前，不需要重新完整复习 Day 2。

只需确认自己还能回答：

1. Workspace、Package、Executable、Node 有什么区别？
2. `add_executable()` 与 `install()` 有什么区别？
3. `CMake`、`ament_cmake`、`colcon` 分别在哪一层？
4. `source install/setup.bash` 做了什么？
5. 一个 `.cpp` 如何最终被 `ros2 run` 启动？

确认这五点后，直接进入：

> **ROS2 C++ Node：真正会读、会改、会运行。**

后续不再以“补齐所有基础”为主线，而以真实 ROS2 应用能力为主线推进。
