# 2026-08-22 Robocon 控制组学习记录
## 从 Lift 直接依赖 ZDrive 到通用 Motor 抽象、Ops Table 与 Adapter 层

> 日期：2026-08-22  
> 承接记录：`2026-08-21_ZDrive与Lift_Mechanism层学习记录.md`  
> 本阶段主题：**解决 Mechanism 对具体 Driver 的耦合问题，建立通用 Motor 层，并进一步用函数指针 + Ops Table + Adapter 实现运行时分发。**  
> 当前验证状态：**软件结构已完成第一版，Keil 编译 `0 Error(s), 0 Warning(s)`；本阶段未进行新的实物验证。**

---

# 一、承接上一阶段：我们从哪里开始

上一阶段已经完成了一个直接基于 ZDrive 的 `Lift` Mechanism 框架。

当时已经具备：

- `Lift_SetHeight(height_mm)`；
- `Lift_Zero()`；
- `Lift_Process()`；
- `Lift_Update()`；
- `Lift_IsReached()`；
- `Lift_HaveZeroed()`；
- `UNZEROED / HOMING / READY / MOVING / REACHED / FAULT` 状态机；
- 机械零点 `zero_angle_deg`；
- 高度 `mm` 与输出轴角度 `deg` 的换算；
- 软件行程限制；
- 连续多周期到位判定；
- Homing 超时与 Fault；
- 主循环周期调度。

上一阶段最重要的 Mechanism 层认识是：

```text
业务 / 操作手
    ↓
Mechanism
    ↓
底层执行器
```

并且已经明确：

> Mechanism 应该使用“机构语义”，例如 Lift 使用高度 `mm`，而不是直接把电机角度、rpm、CAN 等底层量暴露给业务层。

但是，当时还有一个明显问题：

```text
Lift
 ↓
ZDrive
```

`Lift.c` 里仍然直接调用：

```c
Zdrive_Set_target_mode(...);
Zdrive_Get_Position(...);
Zdrive_GetSpeed(...);
```

也就是说：

> **Mechanism 层虽然建立了，但仍然与 ZDrive 这个具体 Driver 耦合。**

这正是本阶段的起点。

---

# 二、发现新的工程问题：如果 Lift 不再使用 ZDrive 怎么办？

我们首先讨论了一个实际工程问题。

假设当前：

```text
Lift → ZDrive
```

以后因为机械设计、功率、重量、减速比、空间布局等原因，希望改成：

```text
Lift → DJI
```

如果 Lift 内部直接写满：

```c
Zdrive_xxx(...);
```

那么更换执行器意味着：

```text
换 Driver
↓
Lift.c 也必须跟着大改
```

这说明：

> **Lift 知道了太多具体 Driver 的细节。**

进一步考虑 Robocon 实际工程，还可能同时存在：

```text
Lift      → ZDrive
Gripper   → DJI
Wrist     → DJI
Conveyor  → 另一类电机
```

所以我们需要在 Mechanism 和具体 Driver 之间再加一层：

```text
Mechanism
    ↓
Motor
    ↓
Driver
```

---

# 三、Motor 层到底解决什么问题

Motor 层不是一个新的电机驱动器，也不负责 CAN、PID 或具体协议。

它的职责是：

> **定义所有上层 Mechanism 可以共同使用的一套“电机公共语言”。**

例如：

```c
Motor_SetPosition(...);
Motor_SetSpeed(...);
Motor_GetPosition(...);
Motor_GetSpeed(...);
Motor_Disable(...);
```

这样 Lift 不再写：

```c
Zdrive_Set_target_mode(...);
```

而只写：

```c
Motor_SetPosition(lift.motor, target_angle);
```

于是 Lift 的思维变成：

```text
“让我的执行电机到这个输出轴角度”
```

而不是：

```text
“调用 ZDrive 的某个模式接口”
```

这一步真正把机构逻辑和电机品牌 / Driver 细节分开了。

---

# 四、统一 Motor 公共语义

为了让 ZDrive 和 DJI 都能被上层当成“Motor”使用，首先必须统一公共接口的物理语义。

当前第一版规定：

```c
Motor_SetPosition(...); // 输出轴位置，单位 deg
Motor_SetSpeed(...);    // 输出轴速度，单位 rpm

Motor_GetPosition(...); // 输出轴位置，单位 deg
Motor_GetSpeed(...);    // 输出轴速度，单位 rpm
```

也就是说：

```text
Motor 层统一使用“输出轴侧”语义
```

这样 Mechanism 不需要知道电机轴转速、减速比、Driver 内部 raw rpm、编码器原始值。

层级边界：

```text
Driver
负责：
原始反馈 → 输出轴位置/速度
协议
CAN
PID
控制模式

Motor
负责：
统一 Position / Speed / Disable 语义

Mechanism
负责：
机构高度、角度、状态、行程、Homing、到位等
```

---

# 五、第一版 Motor：先用 `type + id + switch`

为了先把问题解决，而不是一开始就追求复杂架构，我们先设计最简单的 Motor 对象：

```c
typedef enum
{
    MOTOR_TYPE_ZDRIVE,
    MOTOR_TYPE_DJI
} MotorType_t;

typedef struct
{
    MotorType_t type;
    uint8_t id;
} Motor_t;
```

这里：

```text
type
→ 这是哪一种 Driver

id
→ 是这种 Driver 里的哪一个具体实例
```

例如：

```c
Motor_t lift_motor;

Motor_Init(
    &lift_motor,
    MOTOR_TYPE_ZDRIVE,
    1
);
```

表示：

```text
lift_motor
→ ZDrive 类型
→ 1 号实例
```

---

# 六、第一版 Motor API 的分发方式

第一版中，Motor 公共 API 内部使用：

```c
switch (motor->type)
```

例如概念上：

```c
bool Motor_SetSpeed(Motor_t *motor, float speed)
{
    switch (motor->type)
    {
        case MOTOR_TYPE_ZDRIVE:
            /* 转成 ZDrive 的实现 */
            break;

        case MOTOR_TYPE_DJI:
            /* 转成 DJI 的实现 */
            break;
    }
}
```

于是架构变成：

```text
Lift
 ↓
Motor_SetSpeed()
 ↓
switch(type)
 ├─ ZDrive
 └─ DJI
```

虽然内部仍然有 `switch`，但这一版已经解决了最核心的问题：

> **Lift 不再直接依赖 ZDrive。**

---

# 七、依赖注入：谁决定 Lift 使用哪一个 Motor？

这里出现了一个非常重要的设计问题。

如果在 `Lift_Init()` 里写：

```c
void Lift_Init(void)
{
    Motor_Init(&lift.motor, MOTOR_TYPE_ZDRIVE, 1);
}
```

那么 Lift 仍然知道自己使用 ZDrive 1，只是把 ZDrive API 换成了 Motor API，本质耦合没有完全消失。

因此改成：

```c
Motor_t lift_motor;

Motor_Init(
    &lift_motor,
    MOTOR_TYPE_ZDRIVE,
    1
);

Lift_Init(&lift_motor);
```

而 `Lift_t` 中只保存：

```c
Motor_t *motor;
```

所以关系变成：

```text
main / composition root
负责：
“Lift 到底使用哪个 Motor”

Lift
只负责：
“使用别人交给我的 Motor”
```

这就是本阶段第一次真正接触到的：

> **Dependency Injection（依赖注入）**

当前工程中的含义非常具体：

```text
不是 Lift 自己创建 ZDrive
而是外部先准备好 Motor
再把 Motor 交给 Lift
```

---

# 八、Driver Init、Motor Init、Mechanism Init 的生命周期

正确顺序：

```text
HAL / CAN 等底层外设初始化
↓
Driver 全局初始化
↓
Motor 实例绑定
↓
Mechanism 初始化
```

例如：

```c
DJI_motor_init();
ZdriveInit();

Motor_Init(&lift_motor, MOTOR_TYPE_ZDRIVE, 1);

Lift_Init(&lift_motor);
```

## 8.1 `Motor_Init()` 不能调用 `ZdriveInit()` / `DJI_motor_init()`

原因：

```text
Driver Init
通常初始化一整组 Driver 全局实例

Motor_Init
只是绑定一个具体的通用 Motor 实例
```

如果每次 `Motor_Init()` 都重新调用整个 Driver Init，可能把其他已经工作的实例一起重置。

因此：

```text
Driver Init：全局 / backend 级
Motor Init：实例级
```

必须分开。

## 8.2 `Zdrive_Begin(id)` 的特殊性

ZDrive 还需要：

```c
Zdrive_Begin(id);
```

它表示：

> 让这个 ZDrive 实例参与 ZDrive 的软件周期调度。

它不是硬件意义上的 Enable，也不应该被强行抽象成所有 Motor 都必须拥有的 `Enable()`。

因此第一版选择：

```text
ZDrive 特有 readiness
→ 留在 ZDrive 相关绑定逻辑里
```

---

# 九、Motor Disable 的语义边界

我们讨论了：

```c
Motor_Disable(...)
```

它不能被简单理解成“电机立刻物理停止”。

当前通用语义更准确的是：

> **取消当前主动 Position / Speed 控制请求，并请求 backend 停止继续主动驱动。**

具体 backend 再翻译：

```text
ZDrive
→ Zdrive_Disable

DJI
→ DJ_Disable
```

特别注意：

```text
Zdrive_Disable
≠
Zdrive_Brake
```

---

# 十、Lift 完成第一次解耦

完成第一版 Motor 后，Lift 中原来的：

```c
Zdrive_Set_target_mode(...);
Zdrive_Get_Position(...);
```

逐步替换为：

```c
Motor_SetPosition(...);
Motor_SetSpeed(...);
Motor_GetPosition(...);
```

最终 Lift 只需要：

```c
Motor_t *motor;
```

而不需要直接依赖 ZDrive。

于是软件结构第一次真正变成：

```text
Lift
 ↓
Motor
 ↓
ZDrive / DJI
```

到这里已经是一个完整的重要阶段成果。


---

# 十一、继续发现问题：Motor.c 中为什么到处都是 `switch(type)`？

第一版虽然解决了 Lift 对 ZDrive 的耦合，但很快又发现一个重复问题。

每个公共 API 都需要：

```c
switch (motor->type)
```

例如：

```text
Motor_SetPosition → switch
Motor_SetSpeed    → switch
Motor_GetPosition→ switch
Motor_GetSpeed   → switch
Motor_Disable    → switch
```

如果以后再增加 Current、Torque、Reset、Fault 等能力，就会不断复制新的 `switch`。

于是出现新的问题：

> 能不能在初始化的时候只判断一次类型，后面直接记住“这个 Motor 的 SetSpeed 应该调用谁”？

这就进入本阶段第二个核心知识：

> **函数指针。**

---

# 十二、函数指针：把“函数地址”保存起来

普通变量保存数据：

```c
int x;
float speed;
```

函数指针保存的是：

> **某个函数的入口地址。**

例如：

```c
bool (*set_speed)(Motor_t *motor, float speed);
```

不是在定义函数，而是在定义一个变量 `set_speed`，它保存某个函数的地址，并要求这个函数必须满足：

```text
参数 = (Motor_t *, float)
返回值 = bool
```

最开始我们考虑过把每个函数指针都直接放进 `Motor_t`。

但很快发现 DJI 1、DJI 2、DJI 3 实际使用完全相同的一组函数。

如果每个实例都保存一整套相同函数地址，会产生重复，也不利于表达“这些实现属于某种 Motor 类型”。

于是进一步引出了：

> **Ops Table（操作表）**

---

# 十三、`MotorOps_t`：把一整套函数指针打包

定义：

```c
typedef struct
{
    bool  (*set_position)(Motor_t *motor, float position);
    bool  (*set_speed)(Motor_t *motor, float speed);

    float (*get_position)(Motor_t *motor);
    float (*get_speed)(Motor_t *motor);

    void  (*disable)(Motor_t *motor);

} MotorOps_t;
```

可以把它理解成：

> **“一个 Motor backend 必须实现哪些操作”的接口形状。**

每个成员都像一个插槽：

```text
set_position → [某个符合签名的位置函数]
set_speed    → [某个符合签名的速度函数]
get_position→ [某个符合签名的位置读取函数]
get_speed   → [某个符合签名的速度读取函数]
disable     → [某个符合签名的 Disable 函数]
```

---

# 十四、每种 backend 只需要一张 Ops Table

DJI Adapter：

```c
static const MotorOps_t dji_ops =
{
    .set_position = Motor_DJI_SetPosition,
    .set_speed    = Motor_DJI_SetSpeed,
    .get_position = Motor_DJI_GetPosition,
    .get_speed    = Motor_DJI_GetSpeed,
    .disable      = Motor_DJI_Disable,
};
```

ZDrive Adapter：

```c
static const MotorOps_t zdrive_ops =
{
    .set_position = Motor_ZDrive_SetPosition,
    .set_speed    = Motor_ZDrive_SetSpeed,
    .get_position = Motor_ZDrive_GetPosition,
    .get_speed    = Motor_ZDrive_GetSpeed,
    .disable      = Motor_ZDrive_Disable,
};
```

于是多个 Motor 可以共享同一张表：

```text
DJI Motor 1 ─┐
DJI Motor 2 ─┼──→ dji_ops
DJI Motor 3 ─┘
```

形成重要认识：

```text
Ops Table
= 类型级别的信息
= “DJI 这种 Motor 怎么工作”

Motor_t
= 实例级别的信息
= “这是具体哪一个 Motor”
```

---

# 十五、为什么 `Motor_t` 里只需要一个 `ops` 指针

新的 Motor 结构变成：

```c
struct Motor
{
    MotorType_t type;
    uint8_t id;

    const MotorOps_t *ops;
};
```

例如：

```text
lift_motor
├─ type = DJI
├─ id   = 1
└─ ops ─────→ dji_ops
```

另一个：

```text
wrist_motor
├─ type = DJI
├─ id   = 2
└─ ops ─────→ dji_ops
```

两者共享相同实现方法，但因为 `id` 不同，最终控制不同的 Driver 实例。

---

# 十六、为什么要出现 `Bind()`？

这里出现了一个讨论中的关键疑问：

> 如果 `dji_ops` 是 `static`，只存在于 `motor_dji.c`，那么 `motor.c` 怎么把 `motor->ops` 指向它？

一种办法是直接把 `dji_ops` 暴露成 `extern`，但这样会让 `motor.c` 直接知道 Adapter 的内部数据结构。

最终采用更清晰的方式：

```c
bool Motor_DJI_Bind(Motor_t *motor, uint8_t id);
bool Motor_ZDrive_Bind(Motor_t *motor, uint8_t id);
```

由 Adapter 自己完成：

```c
motor->ops = &dji_ops;
```

因此：

```text
Motor_Init
只知道：
“请 DJI Adapter 帮我绑定”

Adapter
自己知道：
“我的 ops 表在哪里”
```

---

# 十七、Bind 的准确含义

`Bind()` 不是绑定 CAN、绑定硬件引脚或初始化 PID。

它更准确的含义是：

> **把当前这个通用 `Motor_t` 对象，与某一种 backend 的实现方式建立关联。**

DJI：

```c
bool Motor_DJI_Bind(Motor_t *motor, uint8_t id)
{
    if (DJI_motor_GetById(id) == NULL)
    {
        return false;
    }

    motor->ops = &dji_ops;

    return true;
}
```

ZDrive 还额外需要 backend-specific readiness：

```text
检查 id
↓
Zdrive_Begin(id)
↓
motor->ops = &zdrive_ops
```

于是：

```text
Motor_Init()
↓
switch(type)        ← 只判断一次
↓
Adapter_Bind()
↓
motor->ops = 对应 ops
```

---

# 十八、`switch` 并没有消失，而是被集中到了初始化阶段

第一版：

```text
每次 Motor_SetSpeed
→ switch(type)

每次 Motor_GetSpeed
→ switch(type)

每次 Motor_SetPosition
→ switch(type)
```

第二版：

```text
Motor_Init
→ switch(type) 一次
→ Bind 对应 ops
```

之后：

```text
Motor_SetSpeed
→ motor->ops->set_speed(...)

Motor_SetPosition
→ motor->ops->set_position(...)
```

所以不是“消灭了判断”，而是：

> **把类型选择从每一次操作，提前到实例初始化阶段完成一次。**

---

# 十九、最关键的一句：`motor->ops->set_speed(motor, speed)`

这一句最开始容易机械照抄，因此后来专门拆开理解。

## 第一步

```c
motor->ops
```

从当前 `Motor_t` 中取出它绑定的 Ops Table 地址。

如果是 DJI：

```text
motor->ops → dji_ops
```

## 第二步

```c
motor->ops->set_speed
```

从 `dji_ops` 中取得 `set_speed` 这个函数指针。

因为：

```c
dji_ops.set_speed = Motor_DJI_SetSpeed;
```

所以它实际得到：

```text
Motor_DJI_SetSpeed
```

## 第三步

```c
motor->ops->set_speed(motor, speed);
```

真正调用：

```c
Motor_DJI_SetSpeed(motor, speed);
```

完整链路：

```text
Motor_SetSpeed(...)
↓
motor->ops
↓
dji_ops / zdrive_ops
↓
对应 set_speed 函数指针
↓
Adapter 的具体 SetSpeed()
↓
Driver
```

---

# 二十、为什么已经找到 DJI 实现，还要再传 `motor`

这是本阶段另一个很关键的疑问。

`ops` 只能告诉我们：

```text
“应该使用 DJI 的实现”
```

但不能告诉我们：

```text
“具体是 DJI 1、DJI 2 还是 DJI 3”
```

因为多个 DJI Motor 共享同一张 `dji_ops`。

因此 Adapter 还必须得到当前实例：

```c
motor
```

再通过：

```c
motor->id
```

找到真实 Driver：

```c
DJI_motor_t *dji = DJI_motor_GetById(motor->id);
```

所以可以记成：

```text
ops
→ 解决“用哪一种实现”

motor->id
→ 解决“这一种实现里的哪个实例”
```

---

# 二十一、Adapter 到底在翻译什么？

Motor 公共层说：

```text
“把输出轴速度设置为 100 rpm”
```

DJI Adapter 翻译成：

```text
找到 DJI_motor_t
↓
设置 DJ_RPM 模式
↓
调用 DJI_motor_Set_Speed()
```

ZDrive Adapter 翻译成：

```text
Zdrive_Set_target_mode(
    id,
    Zdrive_Speed,
    speed
)
```

所以：

```text
Motor
定义共同语言

Adapter
翻译共同语言

Driver
真正执行具体实现
```

Adapter 不负责 CAN、PID、编码器处理、反馈解析；也不负责高度、Homing、限位、REACHED、业务状态机。

---

# 二十二、为什么这已经属于 C 语言中的“多态”

Lift 永远只调用：

```c
Motor_SetSpeed(lift.motor, speed);
```

如果：

```text
lift.motor->ops = &dji_ops
```

则实际运行：

```text
Motor_DJI_SetSpeed()
```

如果：

```text
lift.motor->ops = &zdrive_ops
```

则实际运行：

```text
Motor_ZDrive_SetSpeed()
```

也就是说：

```text
同一个 Motor_SetSpeed 接口
+
不同 Motor 实例
↓
执行不同实际代码
```

这就是当前工程中非常具体的运行时多态。

C 语言本身没有 C++ 的 `class / virtual / inheritance`，这里是使用：

```text
struct
+
函数指针
+
Ops Table
```

手动实现动态分发。

---

# 二十三、结构体前向声明：为什么突然多了一个 `Motor`

写 `MotorOps_t` 时遇到了一个新的 C 语法问题。

`MotorOps_t` 里函数参数需要 `Motor_t *`，但 `Motor_t` 自己又需要 `MotorOps_t *`，形成：

```text
MotorOps_t
需要 Motor_t *

Motor_t
又需要 MotorOps_t *
```

因此使用：

```c
typedef struct Motor Motor_t;
```

然后定义 `MotorOps_t`，最后再完整定义：

```c
struct Motor
{
    MotorType_t type;
    uint8_t id;
    const MotorOps_t *ops;
};
```

这里：

```text
struct Motor
```

中的 `Motor` 是 struct tag（结构体标签名），而：

```text
Motor_t
```

是 typedef 出来的类型别名。

这一次之所以必须给 struct 自己一个名字，是因为：

> **在结构体完整定义之前，我们就需要先引用它的指针类型。**

---

# 二十四、第一次大规模编译报错：72 个 Error 实际只有几个根因

写完第一版 Ops + Adapter 后第一次编译，出现大量：

```text
unknown type name 'Motor_t'
typedef redefinition
incomplete definition of type 'struct Motor'
```

但没有逐条处理，而是先寻找第一批真正的根错误。

## 24.1 根因一：头文件循环依赖

当时大致出现：

```text
motor_zdrive.h
→ motor.h
→ motor_dji.h
→ ...
```

后来明确依赖方向：

```text
motor.h
= 公共基础定义

motor_dji.h
→ include motor.h

motor_zdrive.h
→ include motor.h
```

真正需要 Adapter 的地方是：

```c
motor.c
```

所以 Adapter 头文件应由 `motor.c` include，而不是 `motor.h` 反过来 include 具体 Adapter。

这里再次验证：

> **几十个编译错误很可能是一个类型/头文件根错误引起的连锁报错。**

---

# 二十五、第二个根错误：前向声明和真正定义不是同一个 struct

前向声明：

```c
typedef struct Motor Motor_t;
```

表示：

```text
Motor_t
就是 struct Motor
```

但后面一度写成了另一个名字的结构体。

于是编译器认为：

```text
struct Motor
只声明过
从未完整定义
```

访问：

```c
motor->id
motor->ops
```

时全部出现：

```text
incomplete definition of type 'struct Motor'
```

正确关系必须始终是：

```c
typedef struct Motor Motor_t;

struct Motor
{
    ...
};
```

---

# 二十六、封装 Adapter 内部函数

最开始 Adapter `.h` 中公开了很多内部函数。

后来发现外部真正需要的只有：

```c
Motor_DJI_Bind(...);
Motor_ZDrive_Bind(...);
```

因此改成：

```text
motor_dji.h
→ 只公开 Motor_DJI_Bind()

motor_dji.c
→ static Motor_DJI_SetSpeed()
→ static Motor_DJI_GetSpeed()
→ ...
```

ZDrive 同理。

这让 Adapter 的对外接口很小：

```text
Motor 层只需要知道：
“请你把这个 Motor 绑定成 DJI / ZDrive”
```

---

# 二十七、`static` 与 `const` 的一次实际错误

在封装内部函数时，曾误写：

```c
static const bool Motor_DJI_SetSpeed(...);
static const float Motor_DJI_GetSpeed(...);
```

导致：

```text
incompatible function pointer types
conflicting types
```

后来明确：

函数应该写：

```c
static bool Motor_DJI_SetSpeed(...);
static float Motor_DJI_GetSpeed(...);
static void Motor_DJI_Disable(...);
```

Ops Table 写：

```c
static const MotorOps_t dji_ops;
```

这里：

```text
函数的 static
→ 只在当前 .c 内部可见

ops 表的 static
→ 只在当前 .c 内部可见

ops 表的 const
→ 初始化后不允许被修改
```

所以：

```text
函数前的 const
≠
Ops Table 对象的 const
```

---

# 二十八、`NULL` 未定义的问题

加入：

```c
if (motor == NULL)
```

以及：

```c
if (motor->ops == NULL)
```

时出现：

```text
use of undeclared identifier 'NULL'
```

处理：

```c
#include <stddef.h>
```

也讨论过：

```c
if (!motor)
```

同样可以判断空指针。

当前为了语义直观，仍使用 `== NULL`。

---

# 二十九、`void` 函数不能 `return false`

在：

```c
void Motor_Disable(Motor_t *motor)
```

中一度机械地写成：

```c
return false;
```

编译器报：

```text
void function should not return a value
```

正确：

```c
return;
```

这属于机械错误，但也提醒：

> **增加统一错误检查时，不能机械复制，要看函数真实返回类型。**

---

# 三十、为什么还要检查 `motor->ops == NULL`

最初 Motor 公共 API 只检查：

```c
if (motor == NULL)
```

后来发现：

```text
Motor 对象存在
≠
这个 Motor 已经成功绑定 backend
```

因此增加：

```c
if (motor->ops == NULL)
```

保护。

现在可以区分：

```text
motor != NULL
→ Motor 对象存在

motor->ops != NULL
→ Motor 已经绑定了一套具体实现
```

Getter 失败应返回 `0.0f`，`void` 函数直接 `return;`。

---

# 三十一、一个小问题：Getter 里 `return false`

最新检查中发现：

```c
float Motor_GetPosition(...)
```

某些失败分支写：

```c
return false;
```

C 中 `false` 会转换成 `0.0f`，所以不会导致编译错误。

但接口语义上更合适：

```c
return 0.0f;
```

这是小问题：需要知道，但不影响当前核心结构。

---

# 三十二、Lift 对 Motor 命令失败的处理讨论

Lift 已经开始检查：

```c
if (Motor_SetPosition(...) == false)
{
    return;
}
```

但还讨论到一些未完全完善的边界。

例如：

```text
先更新 target_height
↓
Motor_SetPosition 失败
↓
状态没有进入 MOVING
```

可能造成部分内部数据已变化。

`Lift_Zero()` 中如果：

```text
先把 state 改成 HOMING
↓
Motor_SetSpeed 失败
```

也可能产生：

```text
软件认为 HOMING
但电机任务没有真正启动
```

目前这不是主阻塞，但后续如果做完整状态/错误系统，需要继续考虑：

> **Mechanism 状态变化最好与 Motor 命令是否真正接受保持一致。**

---

# 三十三、为什么没有把 `Process()` 周期强行抽象成 Motor API

当前两个 Driver 的周期任务并不等价。

ZDrive 有自己的 `ZdriveFunc()`；DJI 有自己的周期控制，而且当前 DJI PID 内部使用固定的 `Ts = 0.002 s`。

因此不能简单认为：

```text
所有 Motor backend
都可以统一每 100 ms 调一次 Motor_Process()
```

当前阶段没有强行做：

```c
Motor_Process();
```

原因是：

> **公共抽象必须来自真正共有的语义，而不是为了接口整齐硬把不同机制塞到一起。**

当前 Motor 第一版只统一：

```text
Position
Speed
Feedback
Disable
```

暂不统一：

```text
Driver 周期调度
Current / Torque
Homing
Fault
Enable
```

---

# 三十四、为什么暂时不加入 Current / Torque / Zero

## Current / Torque

DJI 当前 `current_cmd` 仍带有较强协议/raw 控制量语义，并没有完全统一成物理电流或物理转矩。

因此现在强行定义：

```c
Motor_SetCurrent(...)
```

容易产生假的统一。

## Zero / Homing

Homing 属于：

```text
Mechanism 建立机械坐标系
```

不应该简单变成所有 Motor 的公共能力。

例如 Lift Homing 涉及：

```text
向下移动
检测机械参考
记录 zero_angle
建立机构坐标
```

这是机构逻辑，不是单纯 Motor Driver 功能。

因此当前保持：

```text
Homing
→ Mechanism

Motor
→ 执行 Position / Speed
```


---

# 三十五、进一步讨论：一个 Motor 是不是“绑死”了？

理解 Bind 后产生了一个新的疑问：

> 一旦对一个 Motor 执行 Bind，它是不是就永远只对应一个电机？那多个 Mechanism 怎么办？

答案分两层。

## 35.1 一个具体 `Motor_t` 实例通常固定绑定一个具体执行器

例如：

```c
Motor_t lift_motor;

Motor_Init(
    &lift_motor,
    MOTOR_TYPE_DJI,
    1
);
```

之后通常把 `lift_motor` 稳定理解成 DJI 1。

运行时不建议频繁重新 Bind，因为上层已经把它当作一个稳定的执行器依赖。

## 35.2 但整个 Motor 层并没有只能存在一个 Motor

系统完全可以：

```c
Motor_t lift_motor;
Motor_t gripper_motor;
Motor_t wrist_motor;
```

分别：

```c
Motor_Init(&lift_motor, MOTOR_TYPE_ZDRIVE, 1);
Motor_Init(&gripper_motor, MOTOR_TYPE_DJI, 1);
Motor_Init(&wrist_motor, MOTOR_TYPE_DJI, 2);
```

于是：

```text
Lift
→ lift_motor
→ ZDrive 1

Gripper
→ gripper_motor
→ DJI 1

Wrist
→ wrist_motor
→ DJI 2
```

这里再次明确：

```text
Motor 层
= 公共机制

Motor_t
= 一个具体实例
```

---

# 三十六、一个 Mechanism 也可以拥有多个 Motor

例如双电机升降：

```c
typedef struct
{
    Motor_t *left_motor;
    Motor_t *right_motor;

    ...
} Lift_t;
```

初始化：

```c
Motor_t lift_left;
Motor_t lift_right;

Motor_Init(&lift_left, MOTOR_TYPE_DJI, 1);
Motor_Init(&lift_right, MOTOR_TYPE_DJI, 2);

Lift_Init(&lift_left, &lift_right);
```

此时：

```text
          Lift
         /    \
        ↓      ↓
   Motor 1   Motor 2
      ↓         ↓
    DJI 1     DJI 2
```

Mechanism 负责同步、目标、状态、完成条件、故障；Motor 只负责单个执行器的统一操作。

---

# 三十七、不同 Mechanism 不应无协调地抢同一个物理 Motor

如果：

```text
Lift ─────┐
          ├→ 同一个 Motor
Gripper ──┘
```

两个 Mechanism 都直接拥有控制权，就可能出现：

```text
Lift：
SetSpeed(+100)

Gripper：
SetSpeed(-50)
```

因此形成工程原则：

> **一个物理执行器通常应该有一个明确的上层所有者。**

如果多个业务确实需要共享一个执行器，应该在更高层加入协调逻辑，而不是让多个 Mechanism 无约束争抢同一个 Motor。

---

# 三十八、结合校内赛的实际软件例子

为了理解多 Mechanism + 多 Motor，我们用校内赛执行机构做了一个工程示例。

假设机器人有：

```text
Lift
→ 负责不同高度放置 / 堆叠

Gripper
→ 负责抓取物块

Wrist
→ 负责调整姿态
```

假设执行器选择：

```text
Lift
→ ZDrive 1

Gripper
→ DJI 1

Wrist
→ DJI 2
```

则：

```c
Motor_t lift_motor;
Motor_t gripper_motor;
Motor_t wrist_motor;

Motor_Init(&lift_motor, MOTOR_TYPE_ZDRIVE, 1);
Motor_Init(&gripper_motor, MOTOR_TYPE_DJI, 1);
Motor_Init(&wrist_motor, MOTOR_TYPE_DJI, 2);

Lift_Init(&lift_motor);
Gripper_Init(&gripper_motor);
Wrist_Init(&wrist_motor);
```

软件关系：

```text
               操作手 / 上层
                    │
        ┌───────────┼───────────┐
        ↓           ↓           ↓
      Lift       Gripper      Wrist
        │           │           │
        ↓           ↓           ↓
 lift_motor   gripper_motor  wrist_motor
        │           │           │
        ↓           └────┬──────┘
 zdrive_ops            dji_ops
        │                 │
        ↓                 ↓
 ZDrive Adapter       DJI Adapter
        │                 │
        ↓             ┌───┴───┐
   ZDrive 1         DJI 1    DJI 2
```

例如：

```c
Lift_SetHeight(650.0f);
```

Lift 内部只调用：

```c
Motor_SetPosition(lift.motor, target_angle);
```

最终：

```text
lift.motor
→ zdrive_ops
→ Motor_ZDrive_SetPosition()
→ ZDrive 1
```

而 `Gripper_Grab()` 内部也可以调用同一个公共：

```c
Motor_SetPosition(gripper.motor, target);
```

但最终：

```text
gripper.motor
→ dji_ops
→ Motor_DJI_SetPosition()
→ DJI 1
```

同一个 Motor API，根据不同实例自动走不同 backend。

这个例子第一次把依赖注入、Motor 实例、Ops Table、Adapter、多态和实际机器人执行机构完整连接起来。

---

# 三十九、当前文件职责

## `Lift.c / Lift.h`

负责：

```text
机构高度
状态机
Homing
机械零点
到位判断
机构行程
Fault
```

只认识：

```c
Motor_t *
```

## `motor.h`

负责：

```text
Motor 公共类型
MotorOps_t
Motor 公共 API
公共语义
```

## `motor.c`

负责：

```text
Motor_Init
初始化时选择 Adapter
公共 API
通过 ops 进行分发
公共参数有效性保护
```

## `motor_dji.c`

负责：

```text
DJI Motor Adapter
dji_ops
Motor 语义 → DJI Driver 语义
```

## `motor_zdrive.c`

负责：

```text
ZDrive Motor Adapter
zdrive_ops
Motor 语义 → ZDrive Driver 语义
Zdrive_Begin 等 backend-specific readiness
```

## `dji_motor.c / ZDrive.c`

负责真正的 Driver 逻辑：

```text
CAN
反馈
协议
PID
模式
编码器
Driver 内部状态
```

完整链：

```text
业务 / 操作手
      ↓
Mechanism
      ↓
Motor 公共 API
      ↓
Ops Table
      ↓
Adapter
      ↓
Driver
      ↓
真实执行器
```

---

# 四十、当前最终软件调用链

以 Lift + DJI 为例：

```text
main
↓
DJI_motor_init()
↓
Motor_Init(&lift_motor, MOTOR_TYPE_DJI, 1)
↓
Motor_DJI_Bind()
↓
lift_motor.ops = &dji_ops
↓
Lift_Init(&lift_motor)
```

以后：

```text
Lift_SetHeight()
↓
高度 mm → 输出轴 deg
↓
Motor_SetPosition(lift.motor, angle)
↓
检查 motor
↓
检查 motor->ops
↓
motor->ops->set_position(...)
↓
Motor_DJI_SetPosition()
↓
motor->id
↓
DJI_motor_GetById()
↓
DJI_motor_SetMode(DJ_Position)
↓
DJI_motor_Set_Position()
↓
DJI Driver 周期控制
↓
PID / current_cmd / CAN
```

如果初始化改成：

```c
Motor_Init(
    &lift_motor,
    MOTOR_TYPE_ZDRIVE,
    1
);
```

那么 Lift 主体代码不需要改，运行链自动变成：

```text
Lift
↓
Motor_SetPosition
↓
zdrive_ops
↓
Motor_ZDrive_SetPosition
↓
ZDrive Driver
```

这就是当前架构最核心的收益。

---

# 四十一、本阶段最新编译与代码状态

当前工作分支：

```text
Lift-Motor-Adapter
```

本阶段最新检查的提交：

```text
8.21 Lift-motor-Adapter 纠错
```

当前 Keil Build：

```text
0 Error(s), 0 Warning(s)
```

说明当前 Motor、Adapter、Lift、头文件依赖、函数指针类型、链接关系至少已经能够正确编译链接。

本阶段没有重新进行实际电机运动验证，因此：

```text
编译通过
≠
整个机器人执行机构已经完成实物闭环验证
```

当前结论仅是：

> **第一版 Motor + Adapter 软件架构已经建立并编译通过。**

---

# 四十二、本阶段主要问题与“发现 → 分析 → 修正”汇总

## 问题 1：Lift 直接依赖 ZDrive

### 发现
以后如果 Lift 改用 DJI，会发现必须修改大量 Lift 代码。

### 分析
Mechanism 只应该知道机构语义，不应该知道具体 Driver。

### 修正

```text
Lift → Motor → Driver
```

---

## 问题 2：Motor 内每个 API 都重复 `switch(type)`

### 发现
增加一个 Motor 功能，就需要在各个 backend 分支中继续复制判断。

### 分析
类型选择其实只需要在初始化时确定一次。

### 修正

```text
函数指针
+
MotorOps_t
+
Ops Table
```

---

## 问题 3：每个 Motor_t 都保存一套函数指针太重复

### 分析
函数实现属于“类型”，不是“实例”。

### 修正
每种 backend 一张共享：

```text
dji_ops
zdrive_ops
```

每个 Motor 只保存：

```c
const MotorOps_t *ops;
```

---

## 问题 4：`dji_ops` 是 static，Motor.c 怎么访问？

### 修正
建立：

```c
Motor_DJI_Bind();
Motor_ZDrive_Bind();
```

让 Adapter 自己完成：

```c
motor->ops = &xxx_ops;
```

---

## 问题 5：`MotorOps_t` 和 `Motor_t` 相互需要

### 修正
学习并使用：

```c
typedef struct Motor Motor_t;
```

前向声明。

---

## 问题 6：几十个 `unknown type / incomplete type`

### 分析
不是几十个独立问题，而是：

```text
头文件依赖
+
struct 名称不一致
```

造成的连锁报错。

### 修正
重新理清 include 方向，并保证：

```c
typedef struct Motor Motor_t;

struct Motor
{
    ...
};
```

使用同一个 struct tag。

---

## 问题 7：Adapter 内部函数暴露过多

### 修正
只公开：

```c
Bind()
```

其余放 `.c`：

```c
static ...
```

---

## 问题 8：把函数写成 `static const bool`

### 原因
误把 `static const MotorOps_t dji_ops` 的 `const` 机械套到了函数返回值。

### 修正

函数：

```c
static bool ...
static float ...
static void ...
```

Ops 表：

```c
static const MotorOps_t ...
```

---

## 问题 9：`NULL` 未定义

### 修正

```c
#include <stddef.h>
```

---

## 问题 10：`void` 函数机械 `return false`

### 修正

```text
bool  → false
float → 0.0f
void  → return
```

---

## 问题 11：只检查 `motor != NULL` 仍然不够

### 修正
增加：

```c
if (motor->ops == NULL)
```

---

## 问题 12：Bind 后是不是整个 Motor 层只能对应一个电机？

### 原因
把“Motor 层”误理解成“一个 Motor 对象”。

### 修正

```text
Motor 层
= 一套公共机制

Motor_t
= 一个具体实例
```

---

# 四十三、本阶段最重要的思维变化

## 43.1 不要先问“调用哪个 Driver API”

以前面对 Lift 容易直接想：

```text
ZDrive 应该切哪个模式？
```

现在应该先想：

```text
Lift 需要什么 Motor 能力？
```

然后由 Adapter 解决不同 Driver 的实现差异。

## 43.2 抽象不是“把代码移到另一个文件”

曾经讨论过：

> 如果只是把原来 `switch` 里的 DJI case 搬到 `motor_dji.c`，但 Motor 每次还是 switch，那是不是只是为了封装而封装？

结论是：

> **机械移动文件并没有真正改变分发机制。**

因此不是为了“文件漂亮”强行拆层，而是在理解函数指针、Ops Table、Bind 后再建立 Adapter。

## 43.3 抽象不能制造“假的共同点”

Driver 周期、Current、Homing 并不天然统一。

因此当前没有为了 API 看起来完整而强行增加：

```text
Motor_Process
Motor_Zero
Motor_Enable
Motor_SetCurrent
```

只抽象真实共同语义。

## 43.4 编译错误优先找根因

这次几十个错误再次证明：

```text
第一处真正错误
```

比：

```text
最后几十条连锁报错
```

更重要。

---

# 四十四、当前知识掌握状态

| 知识 / 能力 | 当前状态 |
|---|---|
| Mechanism 与 Driver 解耦的必要性 | 已掌握 |
| Motor 层职责 | 基本掌握 |
| Motor 公共 Position / Speed 语义 | 已掌握 |
| `Motor_t` 的 type / id 含义 | 已掌握 |
| 依赖注入 `Lift_Init(Motor_t *)` | 基本掌握 |
| Driver Init / Motor Init / Mechanism Init 层级 | 基本掌握 |
| 第一版 `switch(type)` 分发 | 已掌握 |
| 函数指针基本意义 | 基本掌握 |
| `MotorOps_t` Ops Table | 基本掌握 |
| 每 backend 一张共享 ops 表 | 基本掌握 |
| Bind 的作用 | 基本掌握 |
| `motor->ops->set_speed(...)` 调用链 | 基本掌握，仍值得复习 |
| `ops` 与 `id` 的职责区别 | 已掌握 |
| C 结构体前向声明 | 基本掌握 |
| struct tag 与 typedef alias | 基本掌握 |
| `static` 用于模块内部隐藏 | 基本掌握 |
| `const` Ops Table | 基本掌握 |
| Adapter 层职责 | 基本掌握 |
| C 语言函数指针实现运行时多态 | 基本掌握 |
| 多个 Motor_t / 多个 Mechanism 的关系 | 已掌握 |
| 一个 Mechanism 多 Motor 的概念 | 已理解 |
| backend/context 指针 | 尚未正式学习 |
| capability / status / fault 通用抽象 | 暂未学习 |
| Motor 层 Driver 周期统一 | 当前明确不强行统一 |
| 本阶段实物综合验证 | 尚未进行 |

---

# 四十五、复习时最值得重新回答的问题

以后复习这部分，不需要重新背所有代码。优先回答：

1. 为什么上一版 `Lift → ZDrive` 会产生耦合问题？
2. Motor 层和 Driver 层分别负责什么？
3. 为什么 `Lift_Init()` 要接收 `Motor_t *`，而不是 Lift 内部自己选择 ZDrive？
4. `Motor_t` 的 `type`、`id`、`ops` 分别表达什么？
5. 为什么第一版先用 `switch(type)`，后来又改成 Ops Table？
6. `MotorOps_t` 是“电机对象”还是“操作方法集合”？
7. 为什么多个 DJI Motor 可以共享一张 `dji_ops`？
8. `motor->ops->set_speed(motor, speed)` 从头到尾发生了什么？
9. 为什么已经找到 DJI 的 `set_speed` 后，还要再传 `motor`？
10. Bind 到底绑定的是什么？
11. 为什么 `dji_ops` 适合 `static const`，而 Adapter 函数适合 `static`？
12. 为什么需要 `typedef struct Motor Motor_t;` 前向声明？
13. 为什么几十个编译 Error 最后可能只有两个根因？
14. 一个 `Motor_t` 绑定一个电机，为什么仍然可以支持很多 Mechanism？
15. 哪些功能当前故意没有放进 Motor 抽象？为什么？

---

# 四十六、当前整体架构

上一阶段：

```text
操作手 / 业务
     ↓
    Lift
     ↓
  ZDrive
     ↓
真实执行器
```

当前：

```text
操作手 / 业务
      ↓
  Mechanism
      ↓
   Motor API
      ↓
 MotorOps_t
      ↓
   Adapter
      ↓
    Driver
      ↓
 真实执行器
```

以当前 Lift 为例：

```text
Lift
↓
Motor_SetPosition()
↓
lift_motor.ops
↓
dji_ops / zdrive_ops
↓
Motor_DJI_SetPosition()
或
Motor_ZDrive_SetPosition()
↓
具体 Driver
```

---

# 四十七、本阶段最值得记住的结论

1. **Mechanism 不应该绑死具体 Driver。**
2. **Motor 层定义共同的执行器语言，Adapter 负责翻译，Driver 负责真正的硬件控制。**
3. **一个 `Motor_t` 是一个具体实例，不是整个 Motor 层。**
4. **系统可以拥有很多 `Motor_t`，分别注入不同 Mechanism。**
5. **`ops` 决定使用哪一种实现，`id` 决定这一种实现里的哪个实例。**
6. **Ops Table 是类型级共享数据，Motor_t 是实例级数据。**
7. **Bind 是在初始化阶段建立 `Motor_t → backend implementation` 的关联。**
8. **运行时不必反复 switch，可以通过函数指针直接动态分发。**
9. **C 语言可以通过 `struct + function pointer + ops table` 实现类似多态的效果。**
10. **抽象只应该统一真实共有的语义，不能为了接口整齐强行统一不同 Driver 的机制。**
11. **模块化不是机械拆文件，真正重要的是依赖方向和职责边界。**
12. **大量编译错误优先寻找最前面的类型、声明和 include 根因。**
13. **软件命令成功提交仍然不等于真实机构动作已经完成。**
14. **编译 `0 Error / 0 Warning` 只证明软件构建通过，不代表实物闭环已经验证。**

---

# 四十八、下一阶段建议

当前 `Motor + Adapter` 第一版已经可以暂时收尾。

下一阶段不建议立刻继续堆更多抽象，而应该先确保自己能够不看代码讲清：

```text
初始化
↓
Motor_Init
↓
Bind
↓
ops
↓
Motor API
↓
Adapter
↓
Driver
```

之后再考虑更进一步的：

```text
void *backend / context
```

它要解决的问题是：

> 现在 DJI Adapter 每次都要通过 `motor->id` 再执行 `DJI_motor_GetById()`，能不能在 Bind 时直接保存真正 backend 实例的地址？

但这属于下一阶段。

当前先保留：

```text
type + id + ops
```

因为它结构清楚，已经足以支持：

```text
多个 Motor
多个 Mechanism
不同 Driver 混用
```

---

# 四十九、阶段结论

本阶段真正完成的是一次完整的软件架构演进：

```text
Lift 直接调用 ZDrive
↓
发现 Mechanism 与 Driver 耦合
↓
建立 Motor 公共层
↓
统一 Position / Speed 语义
↓
使用 type + id 完成第一版 switch 分发
↓
通过依赖注入让 Lift 只持有 Motor_t *
↓
Lift 与 ZDrive 解耦
↓
发现 Motor.c 重复 switch
↓
学习函数指针
↓
建立 MotorOps_t
↓
每种 backend 建立共享 ops table
↓
使用 Adapter + Bind 隐藏具体实现
↓
学习结构体前向声明
↓
处理头文件循环依赖
↓
处理 static / const / NULL / 返回类型等 C 语言问题
↓
增加 ops 有效性保护
↓
理解多 Motor、多 Mechanism 的实际机器人应用
↓
Keil 编译 0 Error / 0 Warning
```

这次最重要的收获不是某一段固定代码，而是逐渐开始从：

```text
“这个函数该怎么写？”
```

转向：

```text
“这个职责应该属于哪一层？”
“如果以后换 Driver，哪一层应该变化？”
“这个依赖应该由谁决定？”
“哪些东西是真正共同的，哪些只是当前实现碰巧相似？”
```

这就是从“能写功能代码”向“能组织机器人工程软件结构”迈出的一步。
