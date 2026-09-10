# 2026-09-09 ROS2 Humble 环境部署、Linux 基础、ROS2 实操与 MinimalPublisher 学习记录

> 主题：ROS2 阶段 0 收尾 + Linux / Shell 最小前置 + WSL2 / Ubuntu 22.04 / ROS 2 Humble 环境部署 + 官方 Demo 实操 + C → C++ 第一小步 + MinimalPublisher 主干阅读  
> 学习日期：2026-09-09  
> 学习目标：从“知道 ROS2 是什么”，过渡到“真正拥有可用 ROS2 环境，并能运行、观察和初步读懂最小 ROS2 Publisher 程序”。  
> 前置记录：`2026-09-08_ROS2_SLAM第一阶段学习记录.md`  
> 学习原则：真实学习记录优先；一次只推进一个必要小块；新术语首次出现先解释；理论与实际操作同步；充分连接已有 STM32 / FreeRTOS / CAN / Mechanism / Motor / Driver 经验，但明确类比边界。

---

# 一、今日学习总览

今天完整走完了原定路线：

```text
阶段 A：昨日知识快速唤醒
↓
阶段 B：ROS2 阶段 0 收尾
        Topic / Service / Action
↓
阶段 C：Linux / ROS2 环境模型
        Terminal / Shell / Environment Variable / source
↓
阶段 D：部署最小 ROS2 环境
        WSL2 + Ubuntu 22.04 + ROS 2 Humble
↓
阶段 E：最小 Linux 操作 + ROS2 官方 Demo
↓
阶段 F：C → C++ 第一小步
        class / member function / constructor / inheritance
↓
阶段 G：重新阅读 MinimalPublisher
```

今天最终达到的状态：

```text
✓ 能说明 Topic / Service / Action 的用途区别
✓ 理解 Terminal / Shell / Environment Variable / source
✓ ROS 2 Humble 最小环境成功部署
✓ 能使用 pwd / ls / cd / mkdir
✓ 能理解 /、~、.、..
✓ 能区分绝对路径 / 相对路径
✓ 能运行官方 talker / listener Demo
✓ 能实际观察 Node / Topic / Message
✓ 能观察 Publisher / Subscriber 数量变化
✓ 能解释 class / member function / constructor / inheritance 第一版
✓ 能重新看懂 MinimalPublisher 的基本结构和执行主线
```

---

# 二、今日起点：昨日知识状态

昨日已经建立 ROS2 第一版通信模型：

```text
Node
↓
Publisher
↓
Topic + Message
↓
Subscriber
↓
Node
```

并已经理解：

- Node；
- Topic；
- Message；
- Publisher；
- Subscriber；
- Callback；
- Executor 第一版作用；
- Publisher `publish()` 后不会直接跳进另一个 Node 执行；
- Subscriber 收到消息后，由 ROS2 通信 / 调度机制使 Callback ready，再由 Executor 调度；
- Node 与 FreeRTOS Task 只能类比；
- Topic 与 FreeRTOS Queue 只能类比；
- Message 与 C struct 只能类比；
- ROS2 与 SLAM 属于不同层级。

今日没有重新完整教学这些内容，而是在新的 Topic / Service / Action 框架和真实 Demo 中自然复习。

---

# 三、ROS2 阶段 0 收尾：Topic / Service / Action

## 1. 为什么 ROS2 不只有 Topic

机器人系统中的通信需求并不只有一种，因此 ROS2 提供不同语义的通信模型：

```text
Topic
→ 持续的数据流

Service
→ 一次请求 + 一次响应

Action
→ 耗时任务
   Goal
   Feedback
   Result
```

重点不是背 API，而是先判断：

> 当前任务到底是什么通信语义？

---

# 四、Topic

## 1. 第一版定义

Topic = 话题。

可以理解为：

> ROS2 中一条有名字的、用于持续传递某类 Message 的数据流。

典型场景：

```text
IMU Node
↓
Publisher
↓
/imu/data
↓
Subscriber
↓
SLAM Node
```

Topic 特别适合：

```text
IMU
LiDAR
Odometry
Pose
Motor Feedback
```

这类持续产生的数据。

---

## 2. 为什么适合连续数据

例如 IMU 每隔一小段时间就产生新测量：

```text
t0 → 数据
t1 → 数据
t2 → 数据
t3 → 数据
...
```

它不需要等待：

> “SLAM 问一次，IMU 再回一次。”

而是：

> 有新数据就发布。

---

## 3. 与已有 STM32 / CAN 知识的联系

可以把 Topic 的“持续数据流”语义和：

```text
电机周期反馈
传感器周期上报
```

建立类比。

但需要明确：

```text
ROS2 Topic
≠
CAN 周期帧
≠
FreeRTOS Queue
```

只是通信语义相似，底层机制完全不同。

---

## 4. 今日自测

题目：

> LiDAR 持续向 SLAM 发送点云，更适合 Topic 还是 Service？

用户回答：

> B，Topic。

判断正确。

---

# 五、Service

## 1. 第一版定义

Service = 服务。

核心模型：

```text
Client
↓
Request
↓
Server
↓
Response
↓
Client
```

适合：

> 一次请求，一次明确响应。

---

## 2. 为什么不是 Topic

例如：

```text
“机械臂当前是否已经归零？”
```

如果只在需要时偶尔查询，则没有必要持续广播：

```text
homed = true
homed = true
homed = true
...
```

更自然的是：

```text
Request：
“是否已归零？”

↓

Response：
“是”
```

---

## 3. 与 Robocon 经验的联系

可以与之前主从板 / Mechanism 的：

```text
Query
Reset
一次命令
一次返回
```

建立通信语义上的类比。

但：

```text
CAN 请求响应
≠
ROS2 Service
```

---

## 4. Service 的通信实体

用户提出关键问题：

> 通信方式改变以后，应该还是要创建 Publisher 和 Subscriber 之类的吧？

纠正后形成：

```text
Topic
→ Publisher / Subscriber

Service
→ Client / Server

Action
→ Action Client / Action Server
```

即：

> 不同通信模型仍然需要创建对应的通信实体，但不一定还是 Publisher / Subscriber。

---

## 5. Client / Server 判断

场景：

```text
Operator Node：
“机械臂是否已经归零？”

Mechanism Node：
“已经归零。”
```

用户判断：

```text
Operator Node → Client
Mechanism Node → Server
```

判断正确。

---

# 六、Action

## 1. 第一版定义

Action = 动作 / 任务型通信。

适合：

> 需要较长时间完成，而且执行过程中希望持续获得进度反馈的任务。

核心结构：

```text
Goal
↓
Feedback
↓
Result
```

---

## 2. 三个核心概念

### Goal

Goal = 目标。

表示：

> 我要你完成什么。

例如：

```text
机械臂移动到 500 mm
执行一次抓取任务
```

### Feedback

Feedback = 反馈。

表示：

> 任务尚未结束，但当前执行到什么程度。

例如：

```text
当前高度：250 mm
当前状态：GRABBING
```

### Result

Result = 最终结果。

表示：

> 最终执行结果如何。

例如：

```text
success = true
抓取失败
```

---

## 3. 为什么 Service 不够

若机械臂移动需要几秒：

```text
Request
↓
等待……
↓
Response
```

Service 只体现开始和结束，不适合持续表达中间进度。

Action 更适合：

```text
Goal
↓
执行中
↓
Feedback
↓
Feedback
↓
Result
```

---

## 4. 与 Mechanism 状态机的联系

用户之前已有状态机：

```text
未抓取
↓
抓取中
↓
已抓稳
↓
抬升中
↓
抬升完毕
```

可以暂时类比：

```text
Goal
≈ 上层下达任务

Feedback
≈ GRABBING / LIFTING 等过程状态

Result
≈ 成功 / 失败
```

但要明确：

```text
Mechanism 状态机
≠
ROS2 Action
```

状态机解决：

> 机构内部怎么执行。

Action 解决：

> 不同 ROS2 模块之间如何表达和管理耗时任务。

---

## 5. Topic / Service / Action 最终对比

```text
Topic
→ 持续数据流
→ Publisher / Subscriber

Service
→ 一次请求，一次响应
→ Client / Server

Action
→ 耗时任务
→ Goal / Feedback / Result
→ Action Client / Action Server
```

---

# 七、Linux / ROS2 环境模型

今日正式建立：

```text
Windows
↓
WSL2
↓
Ubuntu
↓
Terminal
↓
Shell
↓
Environment Variables
↓
source
↓
ROS2
```

---

# 八、Terminal

## 1. 定义

Terminal = 终端。

第一版理解：

> 提供文字输入与输出的交互窗口。

例如在 Terminal 中输入：

```bash
pwd
ls
cd
ros2 node list
```

---

## 2. 为什么需要

Linux / ROS2 开发中大量操作通过命令完成：

```text
进入目录
运行 ROS2
查看 Node
查看 Topic
编译 Workspace
```

Terminal 是主要交互入口。

---

## 3. Terminal ≠ Shell

用户自测：

> Terminal 负责提供输入/输出界面，Shell 负责理解并执行命令。

用户选择 A，判断正确。

---

# 九、Shell

## 1. 定义

Shell = 命令解释器。

第一版理解：

> 接收用户输入的命令，解析含义并调用系统程序执行。

流程：

```text
用户输入 ls
↓
Terminal
↓
Shell 解析
↓
执行程序
↓
结果显示回 Terminal
```

---

## 2. 与 PowerShell 的联系

PowerShell 本身就是一种 Shell。

Ubuntu 中今日主要使用：

```text
/bin/bash
```

即 bash Shell。

---

## 3. 用户理解

用户判断：

> Shell 是解释命令的环境，ROS2 只是其中可以调用的一套程序和工具。

选择 B，正确。

---

# 十、Environment Variable

## 1. 定义

Environment Variable = 环境变量。

第一版理解：

> 当前 Shell 环境中保存的一些配置参数，供程序查找和读取。

例如：

```text
ROS_DISTRO=humble
PATH=...
```

---

## 2. 为什么需要

当输入：

```bash
ros2
```

Shell 要知道：

> `ros2` 程序在哪里。

如果环境配置没有加载好，可能出现：

```text
command not found
```

这并不一定表示 ROS2 没安装，也可能只是：

> 当前 Shell 不知道去哪里找 ROS2。

---

## 3. 与 C 变量的边界

C：

```c
int speed = 100;
```

属于某个程序内部。

Environment Variable：

```text
属于 Shell 环境
可以传递给 Shell 启动的程序
```

因此：

```text
Environment Variable
≠
C 全局变量
```

---

# 十一、source

## 1. 第一版定义

`source` 是 Shell 命令，用于：

> 让当前 Shell 读取并执行某个脚本中的环境配置。

例如：

```bash
source /opt/ros/humble/setup.bash
```

---

## 2. 关键点：改变“当前 Shell”

假设：

```text
Terminal A
→ Shell A

Terminal B
→ Shell B
```

如果只在 A 中：

```bash
source /opt/ros/humble/setup.bash
```

则：

```text
Shell A
✓ 已加载 ROS2 环境

Shell B
✗ 不一定加载
```

因此：

> 新开终端后可能需要重新 source，除非写入 bash 启动配置。

---

## 3. source 不是“启动 ROS2”

正确理解：

```text
读取 setup.bash
↓
加载环境变量 / 搜索路径
↓
当前 Shell 获得 ROS2 环境
↓
ros2 命令可用
```

错误理解：

```text
source
=
启动 ROS2
```

---

## 4. 与 C `#include` 的类比边界

可以粗略类比：

> 都有“把外部信息引入当前环境”的感觉。

但机制不同：

```text
#include
→ C/C++ 编译期头文件机制

source
→ Shell 运行时环境配置机制
```

---

# 十二、环境部署方案的确定

## 1. 队内要求优先

最初曾考虑：

```text
Ubuntu 24.04 + ROS2 Jazzy
```

但随后用户明确：

> 队里已经要求 Ubuntu 22.04 + ROS2 Humble。

因此最终确定：

```text
Windows 10
↓
WSL2
↓
Ubuntu 22.04
↓
ROS 2 Humble
```

这里体现项目原则：

> 团队环境一致性优先于单纯追求更新版本。

---

# 十三、Codex 环境盘点与安装策略

用户希望让 Codex：

- 检查本地 WSL2；
- 判断已有 Ubuntu；
- 安装缺失组件；
- 避免反复手工截图。

最终采用原则：

```text
先盘点
↓
再安装
↓
缺什么补什么
```

明确禁止：

```text
直接卸载 WSL
删除已有发行版
wsl --unregister
覆盖已有 Ubuntu
删除 workspace
关闭证书校验
```

---

# 十四、实际环境盘点结果

最终环境：

```text
Windows / WSL：
WSL 2 正常

已注册发行版：
docker-desktop（WSL2，保留）
Ubuntu-22.04（WSL2，新装）

Ubuntu：
22.04.5 LTS (Jammy)
x86_64

默认用户：
glance

默认 Shell：
/bin/bash
```

ROS2 最终安装：

```text
ROS 2 Humble
安装路径：
/opt/ros/humble
```

---

# 十五、环境部署中的网络证书问题

这是今天最重要的排障过程之一。

## 1. 问题现象

访问：

```text
packages.ros.org
```

时，Windows 与 WSL 都解析到：

```text
198.18.1.223
```

并收到证书：

```text
*.osuosl.org
```

与：

```text
packages.ros.org
```

不匹配。

因此 ROS2 安装被安全阻止。

---

## 2. 正确处理

没有采用：

```text
--insecure
关闭 apt 证书验证
忽略 TLS 校验
```

而是停止安装并排查网络路径。

这是正确的工程和安全处理。

---

## 3. 可能原因

检测到：

- Nano 虚拟隧道网卡；
- Radmin VPN；
- 代理 / VPN / Fake-IP 可能影响 WSL2 DNS。

当时只能合理推测：

> 某个代理/VPN 的 Fake-IP 或分流配置没有正确覆盖 WSL2。

并未擅自判断具体是哪一个软件。

---

## 4. 网络验证流程

用户手动处理代理/VPN 后：

```powershell
wsl --terminate Ubuntu-22.04
wsl -d Ubuntu-22.04
```

然后在 Ubuntu 中测试：

```bash
getent hosts packages.ros.org
```

结果已经不再是：

```text
198.18.1.223
```

而是正常 IPv6 地址。

随后：

```bash
curl -I http://packages.ros.org/ros2/ubuntu/dists/jammy/InRelease
```

返回：

```text
HTTP/1.1 200 OK
```

说明：

```text
WSL2 DNS 恢复
+
ROS 软件源可访问
```

---

# 十六、Linux 普通用户初始化

第一次进入：

```powershell
wsl -d Ubuntu-22.04
```

创建普通 Linux 用户：

```text
glance
```

强调：

> 日常 ROS2 开发使用普通用户，不长期直接使用 root。

需要系统级操作时再通过：

```text
sudo
```

临时提权。

---

# 十七、ROS2 Humble 最终部署结果

最终由 Codex 完成：

```text
✓ 更新 Jammy 系统包
✓ 配置官方签名 ROS 软件源
✓ 安装 ros-humble-ros-base
✓ 安装 ros-humble-demo-nodes-cpp
✓ 配置用户级 source
✓ talker / listener Demo 验证
```

未安装：

```text
Gazebo
Nav2
MoveIt2
SLAM Toolbox
micro-ROS
Docker ROS 镜像
```

保持最小学习环境。

---

# 十八、`.bashrc` 与自动 source

Codex 在：

```text
~/.bashrc
```

加入：

```bash
source /opt/ros/humble/setup.bash
```

因此新交互式 bash 启动时：

```text
启动 bash
↓
读取 ~/.bashrc
↓
执行 ROS2 source
↓
ROS_DISTRO=humble
↓
ros2 命令可用
```

---

# 十九、今日遇到的第二个环境问题：`ros2: command not found`

## 1. 现象

ROS2 已正确安装，但在一个早已打开的 Shell 中运行：

```bash
ros2 run demo_nodes_cpp talker
```

出现：

```text
ros2: command not found
```

---

## 2. 原因

该 bash 在 `.bashrc` 被修改之前就已经启动。

所以：

```text
旧 bash
↓
没有重新读取最新 ~/.bashrc
↓
ROS2 环境变量还未加载
```

不是 ROS2 安装失败。

---

## 3. 解决

执行：

```bash
source ~/.bashrc
```

随后：

```bash
echo $ROS_DISTRO
```

输出：

```text
humble
```

再执行：

```bash
which ros2
```

输出：

```text
/opt/ros/humble/bin/ros2
```

问题解决。

---

## 4. 关键理解

```bash
source ~/.bashrc
```

不是直接只加载 ROS2。

而是：

> 重新读取整个 bash 用户配置。

因为 `.bashrc` 中已经有：

```bash
source /opt/ros/humble/setup.bash
```

所以会间接加载 ROS2 环境。

---

# 二十、Linux 路径基础

## 1. `pwd`

`pwd` = Print Working Directory。

作用：

> 显示当前工作目录。

第一次运行：

```bash
pwd
```

输出：

```text
/mnt/c/Users/glance
```

说明从 Windows PowerShell 当前路径进入 WSL 时，WSL 保留了对应 Windows 路径映射。

---

## 2. Windows 路径与 WSL 路径

```text
Windows：
C:\Users\glance

WSL：
/mnt/c/Users/glance
```

但 Linux 用户自己的 Home 是：

```text
/home/glance
```

两者不是同一目录。

---

# 二十一、`~` 与 Home Directory

`~` 表示：

> 当前用户的 Home Directory。

对当前用户：

```text
~
=
/home/glance
```

执行：

```bash
cd ~
pwd
```

实际输出：

```text
/home/glance
```

---

## 工程原则

以后 ROS2 workspace 更适合放在：

```text
/home/glance/ros2_ws
```

而不是长期放在：

```text
/mnt/c/Users/glance/...
```

当前只记工程原则，不展开 WSL 文件系统性能和权限细节。

---

# 二十二、`/`、`~`、`.`、`..`

```text
/
= Linux 根目录

~
= 当前用户 Home
= /home/glance

.
= 当前目录

..
= 当前目录的上一级
```

例如当前：

```text
/home/glance
```

则：

```text
.
→ /home/glance

..
→ /home
```

用户预测：

```bash
cd ..
pwd
```

结果应为：

```text
/home
```

判断正确。

---

# 二十三、绝对路径与相对路径

## 1. 绝对路径

Absolute Path = 绝对路径。

特点：

> 从根目录 `/` 开始写完整路径。

例如：

```text
/home/glance/ros2_ws
/opt/ros/humble
```

---

## 2. 相对路径

Relative Path = 相对路径。

特点：

> 相对于当前工作目录描述位置。

例如当前在：

```text
/home/glance
```

进入：

```text
/home/glance/ros2_ws
```

可以写：

```bash
cd ros2_ws
```

用户选择 B，判断正确。

---

# 二十四、`ls`

`ls`：

> 列出当前目录中的文件和目录。

在 `/home/glance` 执行：

```bash
ls
```

没有输出。

原因：

> 当前没有普通可见文件/目录。

---

# 二十五、隐藏文件与 `ls -a`

执行：

```bash
ls -a
```

实际看到：

```text
.
..
.bash_history
.bash_logout
.bashrc
.cache
.motd_shown
.profile
.ros
```

以 `.` 开头的文件通常被视为隐藏文件。

---

# 二十六、`.bashrc`

`.bashrc` 第一版理解：

> bash 启动新的交互式 Shell 时读取的用户级配置文件。

今日真实关系：

```text
新开 Ubuntu
↓
启动 bash
↓
读取 ~/.bashrc
↓
source /opt/ros/humble/setup.bash
↓
ROS2 环境自动加载
```

用户自测：

> 新终端里能直接使用 `ros2 node list`，是因为 `.bashrc` 自动 source ROS2。

选择 B，正确。

---

# 二十七、`.ros`

在 Home 中看到：

```text
.ros
```

当前只知道：

> 这是 ROS2 用户级运行数据相关目录。

今天不展开具体内容。

---

# 二十八、`mkdir`

`mkdir` = make directory。

作用：

> 创建新目录。

例如当前：

```text
/home/glance
```

执行：

```bash
mkdir demo
```

新目录完整路径：

```text
/home/glance/demo
```

用户回答正确。

---

# 二十九、官方 ROS2 talker Demo

执行：

```bash
ros2 run demo_nodes_cpp talker
```

这里新增一个术语：

## Package

Package = 包。

第一版理解：

> 一组围绕某个功能组织起来的代码、程序和配置，是 ROS2 工程中的基本软件组织单位之一。

当前：

```text
demo_nodes_cpp
```

是 ROS2 官方 C++ Demo Package。

---

## Executable

`talker` 是：

> 该 Package 中可以直接运行的可执行程序。

因此：

```bash
ros2 run demo_nodes_cpp talker
```

可以理解为：

> ROS2，请运行 `demo_nodes_cpp` 包中的 `talker` 程序。

实际成功连续输出：

```text
Publishing: 'Hello World: ...'
```

---

# 三十、第二终端观察 ROS2

保持 talker 运行，打开第二个 Ubuntu 终端。

注意：

在 PowerShell 中：

```powershell
wsl -d Ubuntu-22.04
```

进入 Ubuntu 后提示符变为类似：

```text
glance@LAPTOP-...:~$
```

此时已经位于 Linux bash 内部，不应再次执行：

```bash
wsl -d Ubuntu-22.04
```

---

# 三十一、今日出现的第三个操作问题：在 Ubuntu 中再次执行 `wsl`

## 1. 现象

进入 Ubuntu 后，又输入：

```bash
wsl -d Ubuntu-22.04
```

出现：

```text
Command 'wsl' not found
```

---

## 2. 原因

`wsl` 是：

> Windows 侧进入 WSL 的命令。

进入 Ubuntu 后，当前已经是 Linux Shell。

正确层级：

```text
Windows PowerShell
│
│ wsl -d Ubuntu-22.04
↓
Ubuntu / bash
│
│ ros2 node list
│ ros2 topic list
↓
ROS2
```

所以：

```text
wsl ...
→ Windows 侧命令

ros2 ...
→ Ubuntu / ROS2 环境中的命令
```

---

# 三十二、`ros2 node list`

运行：

```bash
ros2 node list
```

talker 运行时看到：

```text
/talker
```

后来启动 listener 后看到：

```text
/listener
/talker
```

这第一次把抽象的 Node 概念和真实系统实体对应起来。

---

# 三十三、`ros2 topic list`

运行：

```bash
ros2 topic list
```

看到：

```text
/chatter
/parameter_events
/rosout
```

今日重点：

```text
/chatter
= talker 发布 Hello World 的 Topic
```

另外：

```text
/rosout
→ ROS2 日志 Topic

/parameter_events
→ 参数变化相关系统 Topic
```

当前只认识，不深入。

---

# 三十四、`ros2 topic info /chatter`

在只有 talker 时，用户预测：

```text
Publisher count = 1
Subscription count = 0
```

实际验证：

```text
Type: std_msgs/msg/String
Publisher count: 1
Subscription count: 0
```

预测完全正确。

---

# 三十五、Message Type：`std_msgs/msg/String`

真实观察到：

```text
Type: std_msgs/msg/String
```

含义：

```text
std_msgs
→ ROS2 标准消息接口包

msg
→ Message 类型

String
→ 具体字符串 Message
```

代码中常写为：

```cpp
std_msgs::msg::String
```

它和 CLI 中的：

```text
std_msgs/msg/String
```

表达同一个 Message 类型。

---

# 三十六、listener Demo

第三个终端执行：

```bash
ros2 run demo_nodes_cpp listener
```

系统形成：

```text
/talker
↓
Publisher
↓
/chatter
↓
Subscriber
↓
/listener
```

listener 连续收到：

```text
I heard: [Hello World: ...]
```

---

# 三十七、Publisher / Subscriber 数量变化

启动 listener 后，用户预测：

```text
Publisher count = 1
Subscription count = 1
```

实际一致。

因此：

```text
/talker
→ 1 个 Publisher

/listener
→ 1 个 Subscriber
```

---

# 三十八、`ros2 topic echo /chatter`

执行：

```bash
ros2 topic echo /chatter
```

作用：

> 临时订阅某个 Topic，并把收到的 Message 内容打印到终端。

实际可看到类似：

```text
data: 'Hello World: 42'
---
data: 'Hello World: 43'
---
```

其中：

```text
data
```

是 `std_msgs/msg/String` Message 的数据字段。

---

# 三十九、Topic 与 Message 的真实对应

今日通过真实 CLI 建立：

```text
Topic：
/chatter

Message Type：
std_msgs/msg/String

Message 字段：
data

运行时具体值：
"Hello World: ..."
```

因此：

```text
Topic
= 数据流叫什么

Message Type
= 数据结构长什么样

具体 Message
= 某次实际传输的数据
```

---

# 四十、`ros2 topic echo` 也是 Subscriber

在 listener 已运行时，再执行：

```bash
ros2 topic echo /chatter
```

用户预测：

```text
Publisher count = 1
Subscription count = 2
```

实际结果一致。

原因：

```text
/listener
= 一个 Subscriber

ros2 topic echo
= 又创建一个临时 Subscriber
```

因此一个 Topic 可以同时被多个 Subscriber 订阅。

---

# 四十一、C → C++ 第一小步

今天没有完整学习 C++，而是只补 ROS2 代码阅读所需的最小知识。

---

# 四十二、class

## 1. 第一版理解

过去 C 中常见：

```c
typedef struct
{
    float target_speed;
    float actual_speed;
} Motor;

void Motor_SetSpeed(Motor *motor, float speed);
void Motor_Update(Motor *motor);
```

工程上其实已经是：

```text
Motor 数据
+
操作 Motor 的函数
```

C++ 可以通过 class 更直接地组织：

```cpp
class Motor
{
public:
    float target_speed;
    float actual_speed;

    void setSpeed(float speed);
    void update();
};
```

第一版理解：

> class 把一类对象的数据和相关行为组织进同一个类型。

---

## 2. 类比边界

当前可以建立：

```text
C：
struct + 相关函数

≈

C++：
class
```

但严格来说：

```text
class
≠
struct + 函数
```

class 还有访问控制、继承、多态等能力。

---

## 3. 用户理解

用户判断：

> Motor class 可以把 position / speed 以及 setPosition() / update() 作为同一个 Motor 类型中的成员组织起来。

选择 B，正确。

---

# 四十三、Member Function

Member Function = 成员函数。

第一版理解：

> 属于某个 class 的函数。

例如：

```cpp
motor1.update();
```

可以理解为：

> `motor1` 对象调用属于 `Motor` class 的 `update()` 成员函数。

---

## 与 C 调用方式对比

C：

```c
Motor_SetSpeed(&motor1, 100.0f);
```

C++：

```cpp
motor1.setSpeed(100.0f);
```

工程语义上：

> C++ 更直接表达“这个操作属于哪个对象”。

---

## 用户理解

用户判断：

```cpp
motor1.update();
```

表示：

> motor1 这个对象调用 Motor 类的 update() 成员函数。

选择 B，正确。

---

# 四十四、Constructor

Constructor = 构造函数。

## 1. 第一版定义

> 当一个 class 的对象被创建时，用来完成该对象初始设置的特殊成员函数。

---

## 2. 与 C Init 的联系

C：

```c
Motor motor1;
Motor_Init(&motor1);
```

C++：

```cpp
Motor motor1;
```

创建时自动执行：

```cpp
Motor()
```

因此：

```text
C Init
→ 通常手动调用

C++ constructor
→ 对象创建时自动调用
```

---

## 3. Constructor 名称

类：

```cpp
Motor
```

构造函数：

```cpp
Motor()
```

即：

> constructor 名称与 class 名称相同。

---

## 4. 带参数初始化

例如：

```cpp
Motor(int id)
{
    motor_id = id;
}
```

创建：

```cpp
Motor motor1(1);
```

流程：

```text
创建 motor1
↓
执行 Motor(1)
↓
初始化 motor_id
```

---

## 5. 与 ROS2 的联系

官方：

```cpp
MinimalPublisher()
```

就是：

> `MinimalPublisher` class 的 constructor。

创建 Node 对象时自动执行初始化。

---

# 四十五、Inheritance

Inheritance = 继承。

## 1. 第一版定义

> 一个新的 class 可以在已有 class 的基础能力上继续扩展。

例如：

```cpp
class Motor
{
public:
    void enable();
    void disable();
};

class ZDriveMotor : public Motor
{
public:
    void setPosition(float pos);
};
```

第一版理解：

```text
ZDriveMotor
=
Motor 的已有能力
+
自己的新能力
```

---

## 2. Base Class / Derived Class

```text
Base Class
= 基类 / 父类

Derived Class
= 派生类 / 子类
```

---

## 3. ROS2 中的核心例子

```cpp
class MinimalPublisher : public rclcpp::Node
```

可以读成：

> `MinimalPublisher` 是从 `rclcpp::Node` 派生出来的具体 ROS2 Node class。

其中：

```text
rclcpp::Node
= Base Class

MinimalPublisher
= Derived Class
```

---

# 四十六、MinimalPublisher 第一行

```cpp
class MinimalPublisher : public rclcpp::Node
```

今天形成的正确理解：

```text
rclcpp::Node
↓
提供 ROS2 Node 基础能力
↓
MinimalPublisher
↓
在此基础上加入 Publisher / Timer / Callback 等功能
```

不是：

> 两个互不相关的名字。

---

# 四十七、`MinimalPublisher()` 与 `Node("minimal_publisher")`

官方结构类似：

```cpp
MinimalPublisher()
: Node("minimal_publisher")
{
    ...
}
```

---

## 1. `MinimalPublisher()`

是 constructor。

执行时机：

> 创建 `MinimalPublisher` 对象时自动执行。

主要负责：

```text
初始化 Node
创建 Publisher
创建 Timer
初始化成员
```

---

## 2. `Node("minimal_publisher")`

今日一个关键纠错：

用户第一次表述：

> `Node("minimal_publisher")` 创建了一个名叫 `"minimal_publisher"` 的 Node。

更准确地说：

> 它是在初始化 `MinimalPublisher` 继承来的 `rclcpp::Node` 基类部分，并把这个 Node 的名字设为 `minimal_publisher`。

因此：

```text
MinimalPublisher 本身就是这个 Node
```

而不是：

```text
MinimalPublisher 内部又额外创建一个独立 Node
```

---

# 四十八、`create_publisher()`

官方类似：

```cpp
publisher_ =
    this->create_publisher<std_msgs::msg::String>("topic", 10);
```

当前不深入：

```text
this
模板语法
QoS 的 10
智能指针
```

只抓主干。

第一版理解：

> 创建一个向 `"topic"` 发布 `std_msgs::msg::String` 类型 Message 的 Publisher。

---

## 关键边界

```text
create_publisher()
=
创建 Publisher 通信实体

≠
真正发送 Message
```

---

# 四十九、`publish()`

官方：

```cpp
publisher_->publish(message);
```

第一版理解：

> 使用之前创建并保存的 Publisher，把 `message` 真正发布到对应 Topic。

通信链：

```text
Publisher
↓
Topic
↓
ROS2 通信系统
↓
Subscriber
```

不是：

```text
publish()
↓
直接进入另一个 Node 的 Callback
```

---

# 五十、`publisher_`

## 1. 第一版作用

`publisher_` 是：

> `MinimalPublisher` class 内保存已创建 Publisher 的成员变量。

为什么要保存：

```text
constructor
↓
create_publisher()
↓
得到 Publisher
↓
保存到 publisher_

……

未来 Timer Callback
↓
仍然需要使用同一个 Publisher
↓
publisher_->publish(message)
```

---

## 2. 末尾 `_`

```text
publisher_
timer_
count_
```

末尾 `_`：

> 只是常见成员变量命名习惯。

不是 C++ 特殊语法，也不是 ROS2 操作符。

---

# 五十一、Timer Callback

官方结构可以简化为：

```cpp
void timer_callback()
{
    ...
    publisher_->publish(message);
}
```

今天重点理解：

```text
初始化阶段：
constructor
↓
create_publisher()
↓
create_timer()

运行阶段：
Timer 到期
↓
Callback ready
↓
Executor 调度
↓
timer_callback()
↓
publish()
```

---

# 五十二、Timer 到期与 Callback 执行的关键纠错

用户表述：

> timer_callback() 是只在到达时间时进入 ready，并由 executor 调度后进行触发。

方向正确，但更严谨应为：

```text
Timer 到期
↓
产生待处理的 Timer Callback 工作
↓
Callback 处于 ready / 可执行状态
↓
Executor 调度
↓
timer_callback() 真正执行
```

因此：

```text
Timer 到期
≠
Callback 已经执行
```

---

# 五十三、MinimalPublisher 最终执行主线

今天已经能够完整解释：

```text
定义 MinimalPublisher class
↓
它继承 rclcpp::Node
↓
创建 MinimalPublisher 对象
↓
constructor 自动执行
↓
初始化继承来的 Node 基础部分
↓
Node 名称设为 minimal_publisher
↓
create_publisher()
↓
Publisher 保存到 publisher_
↓
create_timer()
↓
Node 进入运行
↓
Timer 到期
↓
Callback ready
↓
Executor 调度
↓
timer_callback()
↓
准备 Message
↓
publisher_->publish(message)
↓
Message 发布到 Topic
↓
Subscriber 接收
```

---

# 五十四、用户最终整体验收回答

用户能够独立解释：

1. `class MinimalPublisher : public rclcpp::Node`
   - 理解为基于 Node 定义新的 MinimalPublisher class。
2. `MinimalPublisher()`
   - 理解为 constructor，在创建对象时自动运行并初始化。
3. `Node("minimal_publisher")`
   - 最初表述为“创建一个名叫 minimal_publisher 的 Node”，后纠正为“初始化继承来的 Node 基类部分并设置 Node 名”。
4. `create_publisher(...)`
   - 理解为创建 Publisher，但不真正发送 Message。
5. `timer_callback()`
   - 理解为 Timer 到期后 Callback ready，再由 Executor 调度执行。
6. `publisher_->publish(message)`
   - 理解为真正执行消息发布。

整体主线理解已经建立。

---

# 五十五、今日主要问题与纠错过程

## 问题 1：队内版本要求

最初曾考虑：

```text
Ubuntu 24.04 + Jazzy
```

用户补充：

> 队内要求 Ubuntu 22.04 + ROS2 Humble。

调整为：

```text
Ubuntu 22.04 + ROS2 Humble
```

说明：

> 团队工程环境一致性优先。

---

## 问题 2：ROS2 软件源证书异常

表现：

```text
packages.ros.org → 198.18.1.223
收到 *.osuosl.org 证书
```

处理：

```text
停止安装
↓
不关闭 TLS 校验
↓
检查 Windows 代理/VPN
↓
重启 Ubuntu WSL 实例
↓
重新验证 DNS / HTTP
↓
恢复正常
```

---

## 问题 3：ROS2 安装成功但 `ros2` 命令不可用

表现：

```text
ros2: command not found
```

原因：

> 当前 bash 没有重新加载最新 `.bashrc`。

解决：

```bash
source ~/.bashrc
```

验证：

```bash
echo $ROS_DISTRO
# humble

which ros2
# /opt/ros/humble/bin/ros2
```

---

## 问题 4：进入 Ubuntu 后重复执行 `wsl`

表现：

```text
Command 'wsl' not found
```

纠正层级：

```text
PowerShell
↓
wsl -d Ubuntu-22.04
↓
Ubuntu bash
↓
ros2 ...
```

---

## 问题 5：class 定义与对象创建混淆

用户最初表述：

> 首先创建一个基于 Node 的 class 的新 class。

更准确：

```text
class MinimalPublisher ...
→ 定义 class

后续创建 MinimalPublisher 对象
→ 才创建 object
```

因此：

```text
class 定义
≠
object 创建
```

---

## 问题 6：`Node("minimal_publisher")` 是否创建第二个 Node

初始理解略不严谨。

纠正：

```text
Node("minimal_publisher")
→ 初始化继承来的 Node 基类部分
→ 设置 Node 名称
```

不是：

```text
MinimalPublisher 内部再创建一个独立 Node
```

---

## 问题 7：Timer 到期与 Callback 执行

纠正为：

```text
Timer 到期
↓
Callback ready
↓
Executor 调度
↓
Callback 执行
```

---

# 五十六、今日最重要的类比与边界

## Topic 与 CAN 周期数据

```text
Topic
≈ 持续数据流语义
```

但：

```text
Topic
≠ CAN 帧
```

---

## Service 与查询 / Reset

```text
Service
≈ 一次查询 / 一次请求响应
```

但：

```text
Service
≠ 你之前的 CAN 请求响应协议
```

---

## Action 与 Mechanism 状态机

```text
Action
≈ Goal / Feedback / Result 的任务语义
```

但：

```text
Action
≠ Mechanism 状态机
```

---

## class 与 C 模块化

```text
class
≈ struct + 相关函数的语言级组织
```

但：

```text
class
≠ 简单 struct + 函数
```

---

## constructor 与 Init

```text
constructor
≈ 初始化目的
```

但：

```text
C Init
→ 手动调用

C++ constructor
→ 对象创建时自动调用
```

---

## ROS2 Callback 与 MCU Callback

共同点：

> 事件发生后执行提前注册的用户逻辑。

但：

```text
ROS2 Callback
≠ MCU 硬件中断 Callback
```

---

# 五十七、今日用户已经展示出的理解

用户已经能够正确判断和表达：

1. LiDAR 连续点云适合 Topic。
2. 查询归零状态适合 Service。
3. 耗时机械臂任务适合 Action。
4. Operator Node 更像 Service Client。
5. Mechanism Node 更像 Service Server。
6. Terminal 提供界面，Shell 解释命令。
7. Shell 不是 ROS2 专属工具。
8. ROS2 已安装但 Shell 未加载环境时，仍可能 `command not found`。
9. `source` 改变的是当前 Shell 环境。
10. `~` 对应 `/home/glance`。
11. `..` 表示上一级目录。
12. `cd ros2_ws` 属于相对路径写法。
13. `mkdir demo` 在 `/home/glance` 下创建 `/home/glance/demo`。
14. talker 对 `/chatter` 是一个 Publisher。
15. 未启动 listener 时 `/chatter` 的 Subscription count 为 0。
16. 启动 listener 后 Subscription count 为 1。
17. `ros2 topic echo` 再增加一个 Subscriber，因此 Subscription count 为 2。
18. `std_msgs/msg/String` 是 `/chatter` 的 Message Type。
19. class 可以组织数据成员与成员函数。
20. `motor1.update()` 表示对象调用成员函数。
21. constructor 在对象创建时自动执行。
22. `MinimalPublisher` 继承 `rclcpp::Node`。
23. `create_publisher()` 只创建 Publisher，不真正发消息。
24. `publisher_` 用于保存 Publisher，供后续 Callback 使用。
25. `publish()` 才真正发布 Message。
26. constructor 主要负责初始化，周期发送主要由 Timer Callback 完成。
27. MinimalPublisher 主干已经可以按执行顺序解释。

---

# 五十八、当前知识状态

## 已掌握 / 基本掌握

```text
✓ Topic / Service / Action 用途区别
✓ Topic → Publisher / Subscriber
✓ Service → Client / Server
✓ Action → Goal / Feedback / Result
✓ Terminal
✓ Shell
✓ Environment Variable
✓ source
✓ ~/.bashrc
✓ WSL2 / Ubuntu 22.04 / ROS2 Humble 环境模型
✓ /、~、.、..
✓ 绝对路径 / 相对路径
✓ pwd / ls / ls -a / cd / mkdir
✓ ROS2 官方 talker / listener
✓ ros2 node list
✓ ros2 topic list
✓ ros2 topic info
✓ ros2 topic echo
✓ Message Type 与实际 Message 的关系
✓ 一个 Topic 可被多个 Subscriber 订阅
✓ class 第一版
✓ member function 第一版
✓ constructor 第一版
✓ inheritance 第一版
✓ Base Class / Derived Class
✓ MinimalPublisher 主执行链
✓ create_publisher() 与 publish() 区别
✓ publisher_ 的保存作用
✓ Timer → Callback ready → Executor → Callback
```

---

# 五十九、今天明确没有深入的内容

以下内容今天仍然只预览或明确暂不学习：

```text
std::bind
this 的完整机制
Lambda
SharedPtr
智能指针
C++ 模板
复杂 STL
QoS 深入
DDS
Callback Group
Executor 线程模型
Workspace / Package 正式创建
colcon 构建流程
自己编写完整 Publisher / Subscriber
TF2
URDF
RViz 深入
rosbag2 深入
Nav2
MoveIt2
ros2_control
micro-ROS
SLAM 数学
KF / EKF
ICP
Graph Optimization
LIO
FAST-LIO2
```

这些不是遗漏，而是按当前阶段刻意延后。

---

# 六十、今日最值得记住的核心句

## Topic

> 持续产生的数据，优先想到 Topic。

## Service

> 一次请求、一次响应，优先想到 Service。

## Action

> 一个需要时间完成，并希望中途获取进度的任务，优先想到 Action。

## Terminal / Shell

> Terminal 提供交互窗口；Shell 负责解释命令。

## Environment Variable

> 当前 Shell 环境中的配置参数，帮助程序找到对应工具、库和资源。

## source

> `source` 不是启动 ROS2，而是把 ROS2 环境配置加载进当前 Shell。

## `.bashrc`

> 新交互式 bash 启动时会读取 `.bashrc`，因此可以通过它自动加载 ROS2 环境。

## Linux 路径

```text
/
= 根目录

~
= Home

.
= 当前目录

..
= 上一级目录
```

## ROS2 Demo

```text
/talker
↓
Publisher
↓
/chatter
↓
std_msgs/msg/String
↓
Subscriber
↓
/listener
```

## class

> class 把一类对象的数据和相关行为组织进同一个类型。

## constructor

> 对象创建时自动执行，用于初始化。

## inheritance

> 派生类在基类已有能力上继续扩展。

## MinimalPublisher

> 一个继承 `rclcpp::Node` 的具体 Node class；构造时创建 Publisher / Timer，运行阶段由 Timer 触发 Callback，Callback 使用 Publisher 真正发布 Message。

---

# 六十一、MinimalPublisher 最终复习图

```text
class MinimalPublisher : public rclcpp::Node
│
├─ 继承 rclcpp::Node
│
├─ constructor：MinimalPublisher()
│      ↓
│   初始化 Node 基类部分
│      ↓
│   Node 名：minimal_publisher
│      ↓
│   create_publisher()
│      ↓
│   publisher_ 保存 Publisher
│      ↓
│   create_timer()
│
└─ 运行阶段
       ↓
    Timer 到期
       ↓
    Callback ready
       ↓
    Executor 调度
       ↓
    timer_callback()
       ↓
    准备 Message
       ↓
    publisher_->publish(message)
       ↓
    Topic
       ↓
    Subscriber
```

---

# 六十二、今日阶段完成情况

```text
✓ 阶段 A：昨日知识快速唤醒
✓ 阶段 B：Topic / Service / Action
✓ 阶段 C：Linux / ROS2 环境模型
✓ 阶段 D：WSL2 + Ubuntu 22.04 + ROS2 Humble
✓ 阶段 E：最小 Linux 操作 + ROS2 官方 Demo
✓ 阶段 F：C → C++ 第一小步
✓ 阶段 G：重新阅读 MinimalPublisher
```

今日原定学习路线已完成。

---

# 六十三、下一阶段自然衔接

下一阶段不应立即深入：

```text
std::bind
SharedPtr
模板
复杂 C++
```

更合理的下一步是：

```text
建立自己的 ROS2 Workspace
↓
理解 Workspace / Package
↓
创建第一个自己的 Package
↓
自己编写最小 Publisher
↓
运行并验证
↓
自己编写最小 Subscriber
↓
再次观察 Node / Topic / Message
```

即：

> 从“能够运行和读懂官方 ROS2 程序”，过渡到“能够建立自己的 ROS2 工程并编写最小通信程序”。

---

# 六十四、建议复习时优先检查的 10 个问题

1. Topic / Service / Action 分别适合什么通信需求？
2. Service 中 Client / Server 的职责是什么？
3. Action 中 Goal / Feedback / Result 分别是什么？
4. Terminal 与 Shell 有什么区别？
5. `source /opt/ros/humble/setup.bash` 真正在做什么？
6. `/`、`~`、`.`、`..` 分别代表什么？
7. `/chatter` 与 `std_msgs/msg/String` 分别代表 Topic 还是 Message Type？
8. `create_publisher()` 与 `publish()` 有什么区别？
9. `MinimalPublisher : public rclcpp::Node` 表达了什么继承关系？
10. Timer 到期、Callback ready、Executor 调度、Callback 执行的顺序是什么？

---

# 六十五、今日总结

今天完成了从“ROS2 系统认知”到“ROS2 真实环境与最小工程理解”的关键跨越。

昨日主要停留在：

```text
知道 ROS2 是什么
知道 Node / Topic / Message / Publisher / Subscriber 是什么
知道 ROS2 如何与 SLAM 分层
```

今日进一步完成：

```text
真正部署 ROS2 Humble
↓
真正进入 Ubuntu / bash
↓
真正理解 source 与环境变量
↓
真正运行 talker / listener
↓
真正观察 Node / Topic / Message
↓
真正看到 Publisher / Subscriber 数量变化
↓
补齐最小 C++ 前置
↓
重新读懂 MinimalPublisher 主干
```

当前已经具备进入下一阶段：

> **ROS2 Workspace / Package / 自己编写 Publisher 与 Subscriber**

的前置条件。
