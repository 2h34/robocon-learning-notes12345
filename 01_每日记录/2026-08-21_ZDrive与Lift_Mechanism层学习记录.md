# 2026-08-21 Robocon 控制组学习记录
## ZDrive 上层接口完善与 Lift Mechanism 层框架学习

> 日期：2026-08-21  
> 学习主题：ZDrive 上层接口整理、Lift Mechanism 层设计、状态机、归零、到位判断、故障与主循环调度  
> 当前目标：**掌握 Mechanism 层的大体书写逻辑，搭出可编译的 Lift 框架，不追求当前阶段的完整实物验证。**

---

# 一、今天的学习目标

今天的主线不是继续深入某一种电机控制算法，而是开始学习：

> **如何把已经能够工作的底层执行器，进一步封装成一个具有“机构语义”的 Mechanism 模块。**

我们选择 `Lift`（升降机构）作为第一个完整例子。

今天主要解决以下问题：

1. ZDrive 怎样向上层提供足够清晰的接口；
2. Mechanism 层为什么不能直接暴露底层电机参数；
3. Lift 应该保存哪些属于“机构本身”的状态和物理量；
4. 如何建立机械零点；
5. 如何设计 Lift 的状态机；
6. `SetHeight()` 为什么只是“发起运动任务”，而不是“任务完成”；
7. 如何根据反馈判断机构真正到位；
8. 如何加入最基本的软件行程、Homing 超时和 Fault；
9. 如何把 Lift 模块接回 `main.c`，保持“初始化 + 调度”的工程结构。

---

# 二、今天开始时已有基础

今天不是从零开始。

前面已经具备的知识包括：

- STM32 基础外设与中断；
- UART / CAN；
- HAL Callback；
- 非阻塞程序和状态机；
- `.c/.h` 模块化；
- PID 与串级 PID；
- DJI 电机基础；
- ZDrive 基本通信、位置模式和速度模式；
- ZDrive 位置/速度实物验证；
- `main.c` 只负责初始化和调度的工程组织原则。

因此今天的重点不是重新学习电机控制，而是：

```text
业务 / 操作手
    ↓
Mechanism
    ↓
Motor / Driver
    ↓
真实执行器
```

开始真正理解中间的 **Mechanism 层**。

---

# 三、第一阶段：整理 ZDrive 给上层使用的接口

## 3.1 `mode` 与 `modeRead`

今天首先重新确认了 ZDrive 中两个容易混淆的量：

```c
mode
modeRead
```

它们不是一回事。

- `mode`：软件希望驱动器进入的**目标模式**；
- `modeRead`：通过 CAN 反馈确认到的驱动器**实际模式**。

因此：

```text
写入 mode
≠
驱动器已经完成模式切换
```

模式切换本身是一个异步过程：

```text
上层请求 Position
↓
mode = Position
↓
ZdriveFunc() 周期运行
↓
不断下发模式命令并查询
↓
驱动器反馈 modeRead = Position
↓
模式切换才真正完成
```

这进一步强化了今天一个反复出现的核心认识：

> **函数调用结束，不等于它启动的物理任务已经完成。**

## 3.2 模式切换期间目标值被覆盖的问题

原来的 ZDrive 在模式切换时会做保护：

- 进入 Speed 模式时，把速度目标先清零；
- 进入 Position 模式时，把目标位置先设置成当前位置，防止突然跳变。

这本身是合理的。

但如果上层在模式还没切换完成时就提前写目标，例如：

```text
请求 Position + 目标角度
↓
ZDrive 正在切模式
↓
SwitchMachine 把位置目标重新对齐当前位置
↓
上层刚写入的目标被覆盖
```

于是我们加入了 pending target 的思想。

核心结构：

```c
typedef struct
{
    float pendingTarget;
    uint8_t isPending;
} ZdriveTarget;
```

上层调用：

```c
Zdrive_Set_target_mode(id, mode, target);
```

时先保存：

```text
目标模式
+
待处理目标值
+
isPending = true
```

只有当：

```c
modeRead == mode
```

以后，`ZdriveFunc()` 才真正把 pending target 写入当前控制目标。

于是流程变成：

```text
请求目标模式 + 目标值
↓
先缓存 target
↓
等待 modeRead == mode
↓
再真正使用 target
```

当前策略采用：

> **last command wins**

即新的目标会覆盖尚未执行的旧 pending target，而不是建立完整命令队列。

对于当前 Lift 框架足够。

## 3.3 补充 ZDrive 上层接口

为了不让 Mechanism 层直接访问：

```c
Zmotor[id].xxx
```

我们补充/确认了几个接口：

```c
void Zdrive_Begin(uint8_t id);
float Zdrive_Get_Position(uint8_t id);
float Zdrive_GetSpeed(uint8_t id);
void Zdrive_Set_target_mode(uint8_t id, ZdriveMode mode, float target);
```

这里特别明确：

### `Zdrive_Begin()`

`Begin` 只是：

> **软件调度参与开关**

它不是硬件意义上的 Enable，也不等价于：

```c
Zdrive_Disable
```

---

# 四、第二阶段：开始设计 Lift Mechanism

今天真正进入 Mechanism 层以后，首先明确了一件事：

> **Lift 层应该使用 Lift 自己的物理语义，而不是直接使用底层执行器的电机语义。**

因此上层希望写的是：

```c
Lift_SetHeight(300.0f);
```

意思是：

> 升降机构运动到 300 mm。

而不是：

```c
Zdrive_Set_target_mode(1, Zdrive_Postion, 5400.0f);
```

后者属于底层执行器语言。

---

# 五、Lift 的基本数据结构

我们逐步建立了 `Lift_t`，其中包含：

```c
float target_height_mm;
float actual_height_mm;
float tolerance_mm;
float zero_angle_deg;
LiftState state;
uint16_t reached_count;
uint32_t homing_count;
```

这些变量分别代表：

| 变量 | 含义 |
|---|---|
| `target_height_mm` | Lift 的目标高度 |
| `actual_height_mm` | Lift 当前实际高度 |
| `tolerance_mm` | 到位允许误差 |
| `zero_angle_deg` | 建立机械零点时对应的 ZDrive 绝对角度 |
| `state` | 当前机构业务状态 |
| `reached_count` | 连续稳定到位计数 |
| `homing_count` | Homing 超时计数 |

这里开始形成一个重要习惯：

> **Mechanism 层保存的是机构自己的状态，而不是简单复制底层驱动器状态。**

---

# 六、机构量与执行器量的转换

暂时假设：

```c
#define LIFT_MM_PER_REV 20.0f
```

即：

> 输出轴每转一圈，Lift 移动 20 mm。

于是我们写了两个内部转换函数：

```c
static float Lift_HeightToAngle(float height_mm);
static float Lift_AngleToHeight(float angle_deg);
```

基本关系：

```text
高度 mm
↓
换算成输出轴相对角度 deg
```

以及：

```text
输出轴相对角度 deg
↓
换算成 Lift 高度 mm
```

注意：

> ZDrive 本身已经处理了 ReductionRatio，所以 Lift 层不再重复乘减速比。

这是一个典型的层级边界问题：

```text
ZDrive
负责电机 / 输出轴角度关系

Lift
负责输出轴角度 / 机构高度关系
```

不能重复换算。

---

# 七、机械零点与坐标系

## 7.1 为什么需要 `zero_angle_deg`

ZDrive 返回的是自己的绝对位置坐标。

但 Lift 真正关心的是：

> 相对于“机构机械零点”的高度。

因此归零后记录：

```c
lift.zero_angle_deg = current_angle;
```

以后当前 Lift 高度使用：

```text
当前 ZDrive 绝对角度
-
zero_angle_deg
=
相对机械零点角度
```

再换算成 mm。

于是反馈关系是：

```text
ZDrive absolute angle
↓
- zero_angle_deg
↓
relative angle
↓
AngleToHeight()
↓
actual_height_mm
```

## 7.2 目标高度也必须加零点偏移

同理，上层输入：

```c
Lift_SetHeight(20.0f);
```

得到的是相对于机械零点的目标高度。

所以不能直接把：

```text
20 mm → 360°
```

当成 ZDrive 的绝对目标。

正确关系：

```text
目标高度
↓
HeightToAngle()
↓
相对零点角度
↓
+ zero_angle_deg
↓
ZDrive absolute target
```

例如：

```text
zero_angle_deg = 15320°
目标高度 = 20 mm
LIFT_MM_PER_REV = 20 mm/rev
```

则：

```text
20 mm → 360°
15320° + 360° = 15680°
```

所以 ZDrive 应该收到 `15680°`，而不是 `360°`。

---

# 八、建立 Lift 状态机

最终得到的状态：

```c
typedef enum
{
    LIFT_UNZEROED,
    LIFT_HOMING,
    LIFT_READY,
    LIFT_MOVING,
    LIFT_REACHED,
    LIFT_FAULT
} LiftState;
```

状态含义：

- `LIFT_UNZEROED`：还没有建立机械零点；
- `LIFT_HOMING`：正在寻找机械参考点；
- `LIFT_READY`：机械零点已经建立，可以接受高度运动命令；
- `LIFT_MOVING`：已经接受新的高度目标，机构正在运动；
- `LIFT_REACHED`：机构已经稳定到达目标高度；
- `LIFT_FAULT`：当前机构处于异常状态，不应该继续接受普通运动命令。

这里最关键的是：

```text
LIFT_HOMING
≠
ZDrive Speed Mode
```

前者是**机构业务状态**，后者是**底层执行器控制模式**。

当前只是：

> Homing 这个机构任务，恰好利用 Speed 模式实现。

同理：

```text
LIFT_REACHED
≠
底层控制停止
```

即使 Lift 已经认为这次任务完成，ZDrive 的 Position 闭环仍然可以继续运行并保持位置。

---

# 九、`Lift_SetHeight()` 的设计过程

`Lift_SetHeight()` 最终不是简单地“写一个目标”。

它需要完成完整的命令准入过程：

```text
收到 height_mm
↓
检查当前状态是否允许
↓
检查目标高度是否合法
↓
真正接受任务
↓
清 reached_count
↓
保存 target_height_mm
↓
高度 → 相对角度
↓
+ zero_angle_deg
↓
提交给 ZDrive Position
↓
state = MOVING
```

## 9.1 状态准入

只允许：

```text
READY
REACHED
```

接受新的高度任务。

也就是说：

```text
UNZEROED
HOMING
MOVING
FAULT
```

都不能随意接受新任务。

## 9.2 遇到的逻辑问题：计数器清零位置

一开始写成：

```c
lift.reached_count = 0;

if (状态不允许)
{
    return;
}
```

后来发现这存在隐藏问题。

假设当前已经是：

```text
MOVING
reached_count = 15
```

此时上层误调用一次 `Lift_SetHeight()`：

```text
reached_count 先被清 0
↓
发现当前 MOVING
↓
return
```

于是原任务虽然没有被替换，但到位计数被破坏。

因此改成：

```text
先检查是否合法
↓
真正接受新任务
↓
再清 reached_count
```

对应一个重要工程原则：

> **只有新任务真正被接受以后，才修改当前任务内部状态。**

---

# 十、软件行程限制

加入：

```c
#define LIFT_MIN_HEIGHT_MM ...
#define LIFT_MAX_HEIGHT_MM ...
```

然后在 `Lift_SetHeight()` 中检查：

```text
目标 < 最小高度
或
目标 > 最大高度
↓
直接拒绝
```

我们讨论了两种策略：

1. 自动截断到合法范围；
2. 非法命令直接拒绝。

最终选择：

> **直接拒绝。**

原因是，如果上层错误写成：

```c
Lift_SetHeight(5000.0f);
```

偷偷截断成最大高度，机器人仍然会运动，只是运动到错误位置。

直接拒绝更容易暴露逻辑错误。

---

# 十一、`Lift_IsReached()`：从“瞬时到位”到“稳定到位”

## 11.1 最初版本

最开始只是：

```c
fabsf(actual_height_mm - target_height_mm) <= tolerance_mm
```

这只能说明：

> 当前这一瞬间进入了误差范围。

但不一定代表已经稳定。

## 11.2 连续到位判定

今天提出了一个重要想法：

> 如果连续多次进入误差范围，就认为真正到位。

于是加入：

```c
reached_count
```

逻辑：

```text
当前误差在 tolerance 内
↓
reached_count++

当前误差离开 tolerance
↓
reached_count = 0

reached_count 达到阈值
↓
Lift_IsReached() = true
```

关键点：

> 必须是**连续计数**，而不是累计计数。

否则振荡过程中多次进入误差范围，也可能最终累计到阈值形成误判。

## 11.3 饱和计数

为了避免计数器一直增长：

```c
if (lift.reached_count < LIFT_REACHED_COUNT_THRESHOLD)
{
    lift.reached_count++;
}
```

达到阈值以后就不再继续增加。

## 11.4 `Lift_IsReached()` 的副作用

因为函数内部会：

```c
reached_count++;
```

所以它不仅是在“查询”，还会修改状态。

这叫：

> **Side Effect（副作用）**

因此同一个周期调用两次 `Lift_IsReached()`，可能会计数两次。

当前框架接受这种写法，但约定：

> `Lift_IsReached()` 只在 `LIFT_MOVING` 中由 `Lift_Process()` 每周期调用一次。

## 11.5 要不要加入速度条件

我们讨论过是否改成：

```text
位置进入容差
+
速度足够小
+
连续 N 个周期
```

最后决定：

> **当前不加速度条件。**

原因：

1. ZDrive 已经处于 Position 模式；
2. 底层位置 PID 本身会负责收敛和保持；
3. 目标附近一定程度的速度反馈波动可能是正常的；
4. 连续多个周期保持在位置容差内，本身已经能过滤快速经过目标的问题；
5. 当前只是 Mechanism 框架学习，不需要为了“更严谨”继续复杂化。

因此当前定义：

> **实际高度连续多个控制周期保持在目标高度允许误差内，即认为 Lift 已稳定到位。**

---

# 十二、Homing 的基本逻辑

## 12.1 `Lift_Zero()` 不是归零完成

这一点今天反复强调：

```c
Lift_Zero();
```

只是：

> **启动 Homing 任务。**

不是：

> 机构已经归零。

流程：

```text
UNZEROED
↓ Lift_Zero()
HOMING
↓
寻找参考位置
↓
确认参考点
↓
记录 zero_angle_deg
↓
READY
```

## 12.2 Homing 找到零点后的处理

当前框架中：

```text
检测到零点
↓
读取 current_angle
↓
zero_angle_deg = current_angle
↓
Position 模式目标 = current_angle
↓
保持当前位置
↓
target_height = 0
actual_height = 0
↓
READY
```

这里的重点：

> Homing 成功以后，建立的是 Lift 的机械坐标系。

## 12.3 为什么不先发 `Speed = 0` 再发 Position

因为 ZDrive 当前采用 pending target，并且策略是：

> last command wins

如果同一周期：

```text
Speed = 0
↓
Position = current_angle
```

第二条命令可能覆盖第一条 pending target。

因此当前简单方案直接切换到：

```text
Position(current_angle)
```

来保持当前位置。

---

# 十三、Homing 超时与 Fault

Homing 不能无限运行。

否则：

```text
零点检测永远失败
↓
Speed 模式一直运行
↓
机构可能持续向一个方向运动
```

因此加入：

```c
homing_count
```

逻辑：

```text
HOMING
↓
还没找到零点
↓
homing_count++

达到阈值
↓
FAULT
```

## 13.1 一个重要问题：进入 FAULT 不等于电机自动停止

如果 Homing 时最后一个 ZDrive 命令是：

```text
Speed = -50 rpm
```

后来软件只是：

```c
lift.state = LIFT_FAULT;
```

那么 Lift 状态虽然已经是 FAULT，但 ZDrive 仍可能继续执行之前的速度命令。

因此再次得到今天非常重要的结论：

> **修改软件状态，不会自动取消底层正在执行的物理控制任务。**

对于当前“归零超时”这种明确 Fault，可以采用：

```text
读取当前位置
↓
切 Position
↓
目标 = 当前角度
↓
保持当前位置
↓
FAULT
```

真实机构以后还需要根据制动、重力、传感器状态设计更完整的安全策略，本阶段不展开。

---

# 十四、关于 `Lift_Update()` 与未归零高度

今天还讨论了：

> 在还没建立机械零点时，`actual_height_mm` 是否应该更新？

如果 `UNZEROED / HOMING` 时直接用：

```text
current_angle - zero_angle_deg
```

计算高度，那么这个“高度”实际上没有可靠机械意义。

因此更合理的是让 `Lift_Update()` 自己保护：

```text
UNZEROED / HOMING
↓
不更新机构高度
```

这里体现了一个封装思想：

> **模块应该自己守住自己的前置条件，而不是要求所有调用者永远记得正确调用顺序。**

同时 `lift` 本身使用：

```c
static Lift_t lift;
```

避免模块外部直接访问内部状态。

---

# 十五、主循环调度

最后把 Lift 接入 `main.c`。

当前 TIM2：

```text
2 ms 产生一次 tick
```

原来的控制调度：

```c
if (tick_2ms_count - last_control_tick >= 50U)
```

即：

```text
50 × 2 ms = 100 ms
```

因此当前可以：

```text
每 100 ms：
ZdriveFunc();
Lift_Process();
```

初始化顺序：

```text
ZdriveInit()
↓
Lift_Init()
```

因为：

```text
先初始化底层执行器
↓
再初始化依赖它的 Mechanism
```

## 15.1 计数阈值必须结合调用周期理解

例如：

```c
LIFT_REACHED_COUNT_THRESHOLD = 20
```

如果 `Lift_Process()` 每 100 ms 一次：

```text
20 × 100 ms = 2 s
```

意思就是：

> 连续约 2 秒处于容差范围内才认为稳定到位。

同样：

```c
LIFT_HOMING_COUNT_THRESHOLD = 500
```

则代表：

```text
500 × 100 ms = 50 s
```

所以：

> **计数值本身没有时间意义，必须结合任务周期理解。**

---

# 十六、编译与当前验证结果

当前阶段已经完成：

```text
Lift.c / Lift.h
↓
接入 Keil 工程
↓
main.c 初始化与周期调度
↓
编译通过
```

当前学习目标只是：

> **搭一个 Mechanism 框架并掌握其书写逻辑。**

因此本阶段不要求继续做：

- 真实限位开关；
- 真实 Homing；
- 真实机械参数；
- 实物 Lift 运动；
- 完整安全系统；
- 完整执行器抽象层；
- 复杂 Fault 恢复。

---

# 十七、今天遇到的主要问题与修正过程

## 问题 1：模式切换后目标值被覆盖

### 原因

ZDrive 切模式时为了安全会重置目标。

### 修正

增加 pending target：

```text
先保存目标
↓
确认 modeRead == mode
↓
再真正应用目标
```

## 问题 2：`LiftState` 类型定义顺序

曾出现：

```text
Lift_t 中先使用 LiftState
↓
后面才 typedef enum LiftState
```

### 修正

先定义 `enum`，再定义 `struct`。

## 问题 3：`reached_count` 清零太早

### 原逻辑

```text
函数一进来先 count = 0
↓
再判断命令是否合法
```

### 问题

非法/不允许的命令也会破坏正在执行任务的计数状态。

### 修正

```text
先完成所有合法性检查
↓
真正接受新任务
↓
再清 count
```

## 问题 4：瞬时进入误差范围就算到位

### 修正

加入：

```text
连续 N 个周期处于 tolerance
```

才进入 `REACHED`。

## 问题 5：累计计数和连续计数的区别

错误：

```text
进入一次 +1
离开不清零
```

正确：

```text
进入范围 → ++
离开范围 → 0
```

## 问题 6：Fault 只是改状态，但底层命令还在运行

### 原因

软件状态和底层控制任务属于不同层级。

```text
Lift state = FAULT
```

不会自动取消：

```text
ZDrive Speed = -50 rpm
```

### 修正思路

对于当前 Homing timeout：

```text
先发安全保持命令
↓
再进入 FAULT
```

## 问题 7：未归零时的高度是否有效

初始化：

```c
actual_height_mm = 0;
```

并不能说明 Lift 真实高度就是 0 mm。

只有：

```text
Homing
↓
建立 zero_angle_deg
↓
机构坐标系有效
```

以后，高度才真正具有机械意义。

---

# 十八、今天形成的 Mechanism 层核心认识

## 18.1 Mechanism 层的职责

可以概括为：

> **把“上层想让机构做什么”，转换成“底层执行器应该怎么动”，同时维护机构自己的坐标、状态、约束和动作完成条件。**

例如：

```text
上层：
升到 300 mm

↓ Lift

Mechanism：
检查状态
检查行程
300 mm → 电机目标角度
维护 MOVING
读取反馈
判断稳定到位

↓ ZDrive

执行位置闭环
```

## 18.2 机构状态和电机模式不是一个层级

例如：

```text
LIFT_HOMING
```

是 Mechanism 状态。

```text
Zdrive_Speed
```

是底层控制模式。

两者可能存在当前实现中的对应关系，但不能混为一谈。

## 18.3 命令下发和任务完成必须分开

```text
Lift_SetHeight()
```

只代表：

> 新运动任务已经被接受并提交。

真正完成需要：

```text
周期读取反馈
↓
判断状态
↓
满足完成条件
↓
REACHED
```

## 18.4 Mechanism 层应该使用机构物理量

Lift 使用 `mm`，而不是把电机角度直接暴露给业务层。

因此：

```text
业务层
认识 height

Lift
认识 height + actuator mapping

ZDrive
认识 angle / speed / mode
```

## 18.5 `main.c` 只做初始化和调度

最终希望：

```text
main.c
├─ 外设初始化
├─ ZdriveInit()
├─ Lift_Init()
└─ while(1)
    ├─ ZdriveFunc()
    └─ Lift_Process()
```

而具体 Lift 功能全部留在：

```text
Lift.c / Lift.h
```

---

# 十九、以后设计一个 Mechanism 前，先回答这五个问题

这是今天最后形成的一套非常重要的自检模板。

## 1. 机构输入是什么？

对于 Lift：

```c
Lift_Zero();
Lift_SetHeight(height_mm);
```

也就是说上层告诉机构：

```text
建立零点
运动到某个高度
```

而不是告诉底层电机：

```text
转多少度
跑多少 rpm
```

### 核心问题

> 上层真正希望这个机构“做什么”？

## 2. 机构状态有哪些？

对于 Lift：

```text
UNZEROED
HOMING
READY
MOVING
REACHED
FAULT
```

这些状态描述的是：

> **机构当前处于哪个业务阶段。**

不是简单复制电机模式。

### 核心问题

> 为了完整描述机构动作过程，需要哪些业务状态？

## 3. 机构自己的物理量是什么？

对于 Lift：

```text
目标高度 mm
实际高度 mm
到位误差 mm
机械零点
```

底层角度只是实现机构运动需要的执行器量。

### 核心问题

> 对这个机构来说，真正有物理意义的量是什么？

## 4. 底层执行器接口是什么？

当前 Lift 使用：

```c
Zdrive_Begin();
Zdrive_Set_target_mode();
Zdrive_Get_Position();
```

它们负责：

```text
让执行器参与调度
提交目标
读取反馈
```

### 核心问题

> Mechanism 通过什么最小接口控制并读取底层执行器？

## 5. 如何判断动作真正完成？

对于 Lift：

```text
目标已经下发
≠
动作完成
```

而是：

```text
读取实际高度
↓
计算高度误差
↓
连续多个周期处于 tolerance
↓
REACHED
```

### 核心问题

> 哪些反馈条件真正说明“机构任务已经完成”？

---

# 二十、Lift 的完整思维链

把今天全部内容压缩成一条链：

```text
上层：
Lift_SetHeight(300 mm)

↓ 状态准入
READY / REACHED 才允许

↓ 行程检查
目标必须合法

↓ Mechanism 量转换
300 mm
→ 相对角度
→ + zero_angle_deg
→ ZDrive absolute position

↓ 下发执行器
ZDrive Position

↓ 状态切换
MOVING

↓ 周期反馈
ZDrive position
→ 减 zero_angle
→ angle to mm
→ actual_height_mm

↓ 完成判断
连续多个周期：
|actual - target| <= tolerance

↓ 完成
REACHED
```

这就是今天学习的 **Mechanism 基本控制框架**。

---

# 二十一、当前掌握状态

| 内容 | 当前状态 |
|---|---|
| ZDrive mode / modeRead | 已掌握 |
| 异步模式切换 | 已掌握 |
| pending target 思路 | 基本掌握并已应用 |
| Mechanism 层作用 | 基本掌握 |
| 机构语义与执行器语义分层 | 基本掌握 |
| Height ↔ Angle 映射 | 已掌握 |
| mechanical zero / zero offset | 基本掌握 |
| Mechanism 状态机 | 基本掌握 |
| SetHeight 状态准入 | 已掌握 |
| 命令提交 ≠ 任务完成 | 已掌握 |
| 连续到位判定 | 已掌握 |
| 软件行程 | 基本掌握 |
| Homing timeout / Fault | 基本掌握 |
| 主循环周期调度 | 已掌握 |
| 真实 Lift 安全系统 | 暂未学习/本阶段不要求 |
| 执行器抽象层 | 后续再考虑 |

---

# 二十二、最终总结

今天真正完成的不是一个“能直接比赛使用的 Lift”，而是第一次较完整地走通了：

```text
底层执行器
↓
Mechanism
↓
上层业务
```

中间这一层应该怎么写。

最值得记住的不是某一行代码，而是以下几句话：

1. **Mechanism 层使用机构自己的语义和物理量。**
2. **机构状态和底层电机模式不是一个概念。**
3. **发出命令不等于物理动作完成。**
4. **Mechanism 要负责机构坐标、状态、约束和完成条件。**
5. **底层执行器只通过明确接口被 Mechanism 使用，不向上层泄漏内部实现。**
6. **周期状态机负责把一次“命令”变成一个真正可跟踪、可完成、可失败的机构任务。**

以后面对新的机构，不应该第一反应就去想：

```text
我要调用哪个电机 API？
```

而应该先回答：

```text
① 机构输入是什么？
② 机构状态有哪些？
③ 机构自己的物理量是什么？
④ 底层执行器接口是什么？
⑤ 如何判断动作真正完成？
```

只要这五个问题能够回答清楚，一个新的 Mechanism 模块的骨架通常就已经基本确定了。

---

# 二十三、下一阶段

当前 Lift 作为 **Mechanism 学习框架** 可以暂时收尾。

后续真正需要继续时，可以选择：

```text
方向 A：
再设计一个新的 Mechanism，
由自己先回答五个问题并搭框架，
验证是否真正掌握。

方向 B：
在真实机构需求出现以后，
补充真实 Homing / 限位 / Fault / 安全策略。

方向 C：
当 Lift 与更多机构都稳定以后，
再考虑 Motor / Actuator 抽象，
降低 Mechanism 对 ZDrive / DJI 等具体驱动的依赖。
```

当前不急于进入扩展。
