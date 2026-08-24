# 2027 WHUROBOCON 校内赛上层控制软件第一版总设计

> 日期：2026-08-22  
> 文档定位：**可直接指导后续 `.h/.c` 模块划分、接口声明、状态机实现和主循环调度的第一版软件设计说明**  
> 当前阶段：架构设计完成，真实机械参数、部分传感器与底层执行方式待后续确认  
> 主要依据：`2027 WHUROBOCON 校内赛规则 V6(1).pdf`、当前机械方案讨论、现有 `Motor_t + Adapter + Mechanism` 架构基础

---

## 0. 文档使用说明

本文档不是“知识总结”，而是当前校内赛执行机构控制软件的**第一版设计基线**。后续编码时，应优先按照本文档已经确定的模块边界、状态机、对外 API、内部数据和调用关系实现；真实机械结构尚未确定的部分，不应擅自写死。

本文使用以下标记：

- **✓ 第一版确定**：当前方案已经讨论确认，建议第一版代码直接按此实现。
- **△ 待确认 / 可选**：依赖真实机械、传感器或底层硬件方案，当前保留设计位置但不写死参数。
- **→ 后期优化方向**：不纳入第一版必须结构，待第一版能够正确运行和验证后再考虑。

### 0.1 设计优先级

当前设计目标依次为：

1. **结构正确**：Task、Mechanism、Actuator、Driver 分层清晰；
2. **能够实施**：每个模块有明确 API、状态、内部数据和完成条件；
3. **能够运行与调试**：第一版严格串行，避免多动作并发导致难以定位问题；
4. **保留扩展空间**：机械和传感器变化时尽量只修改对应 Mechanism；
5. **最后再优化效率**：并行动作、多 Task 并发、自动重试等均放在后期。

---

# 1. 规则需求与当前软件范围

## 1.1 校内赛对执行机构的直接要求

根据校内赛规则，当前机器人由一名操作手进行纯手动控制，执行机构研发组负责上半部执行机构；执行机构需要完成方块/灵石采集、堆叠放置以及灵石安置，并具备夹取/吸附方块或球体、不同高度放置的能力。

规则同时要求：

- 方块搬运过程中必须**主动抬起**，不能在地面推、拖、滑；
- 掉落方块允许重新通过机构获取，因此“重新获取掉落方块”可以复用正常取块能力；
- 灵石需要放置到指定基座，且掉落会带来较严重的比赛后果；
- 比赛为纯手动控制，但这不意味着软件内部不能用状态机完成一个由操作手触发的连续动作。

## 1.2 当前软件范围

当前第一版重点设计：

```text
Task
  ↓
Mechanism
  ↓
Actuator / Motor
  ↓
Adapter / Driver
  ↓
HAL / Hardware
```

`Operator Control`（遥控器/上位机输入层）只作为**参考层**记录，不作为当前必须实现内容，因为：

1. 当前尚未系统学习上位机/遥控输入的软件实现；
2. 该部分最终不一定由当前控制模块开发者负责；
3. 只要外部能够调用 Task 的 `Start()` 并读取 Task 状态，就可以与本文设计对接。

---

# 2. 总体分层原则

## 2.1 Task 与 Mechanism 的区别

### Task

Task 表示一个**完整的比赛动作目标**，负责多个 Mechanism 的调用顺序。

例如：

```text
获取方块
= BlockArm 到取块位
→ BlockVacuum 吸取
→ BlockArm 回安全搬运位
```

Task 关心：

- 当前完整任务进行到哪一步；
- 当前应该调用哪个 Mechanism；
- 当前步骤是否完成；
- 某个依赖 Mechanism 是否进入 `FAULT`。

Task **不关心**：

- 电机具体转多少角度；
- 气缸伸还是缩；
- PWM / CAN / GPIO 怎么发；
- Mechanism 内部如何完成 Homing 或到位判断。

### Mechanism

Mechanism 表示一个**具有独立机械职责、状态和执行器资源的真实机械子系统**。

Mechanism 不按“一台电机”划分，也不按“一整个比赛 Task”划分。

例如：

```text
BlockArm
= 两个耦合电机 + 连杆机构
= 一个 Mechanism
```

即使它内部有两个 Motor，只要两个 Motor 共同控制同一个机械子系统，就应该由同一个 Mechanism 负责协调。

### Actuator / Motor

Mechanism 下层使用实际执行器。

当前明确：

```text
DJI / ZDrive 电机
→ 可以进入现有 Motor_t 抽象

普通舵机
气缸
真空吸盘系统
→ 不强行塞入当前 Motor_t
```

判断是否进入 `Motor_t` 的依据不是“内部有没有电机”，而是是否满足当前公共语义：

```c
Motor_SetPosition();
Motor_SetSpeed();
Motor_GetPosition();
Motor_GetSpeed();
Motor_Disable();
```

---

## 2.2 命令调用结束 ≠ 物理任务完成

这是全文最重要的统一原则之一。

例如：

```c
BlockArm_MoveToPick();
```

函数返回只表示：

> 已经向 BlockArm 发起“前往取块位”的任务。

不表示：

> 机械臂已经物理到达取块位。

正确关系：

```text
BlockArm_MoveToPick()
↓
BlockArm state = MOVING
↓
BlockArm_Process() 周期推进
↓
反馈满足到位条件
↓
BlockArm state = REACHED
```

Task 必须查询：

```c
BlockArm_GetState();
```

并等待 `REACHED` 后才允许进入下一步骤。

这一原则同样适用于：

- `BlockVacuum_Grab()`；
- `BallGripper_Open()`；
- `BallGripper_Grab()`；
- `BallRotaryAxis_MoveToXXX()`；
- 所有 Task 的 `Start()`。

---

## 2.3 第一版严格串行

**✓ 第一版确定**

一个 Task 内采取最保守的串行策略：

```text
发起动作 A
↓
等待动作 A 明确完成
↓
发起动作 B
↓
等待动作 B 明确完成
↓
...
```

暂时不做：

- 机械臂还未完全到位就提前吸取；
- 底盘运动和精确抓取同时进行；
- 两个 Mechanism 在 Task 内提前并行；
- 多个 Task 同时运行。

原因：

> 第一版优先保证正确、可实施、容易调试；效率优化放到实物验证之后。

---

# 3. 第一版 Task 划分

当前只保留 4 个核心 Task：

```text
1. PickBlockTask   —— 获取方块
2. PlaceBlockTask  —— 放置方块
3. PickBallTask    —— 获取灵石
4. PlaceBallTask   —— 放置灵石
```

“掉落方块重新获取”不单独建立 Task，第一版直接复用 `PickBlockTask`。

> △ 如果以后发现掉落方块和物资区方块的抓取姿态、路径、机构动作存在本质区别，再考虑增加特殊抓取流程。

---

# 4. Task 1：PickBlockTask —— 获取方块

## 4.1 涉及 Mechanism

```text
BlockArm
BlockVacuum
```

## 4.2 执行流程

```text
BlockArm_MoveToPick()
↓ 等待 BlockArm == REACHED

BlockVacuum_Grab()
↓ 等待 BlockVacuum == GRABBED

BlockArm_MoveToSafe()
↓ 等待 BlockArm == REACHED

Task DONE
```

这里同一个 `BlockArm` 在一个 Task 中被调用两次，但仍然只有一个 `BlockArm Mechanism`。

## 4.3 Task 状态

```c
typedef enum
{
    PICK_BLOCK_TASK_IDLE,

    PICK_BLOCK_TASK_MOVE_TO_PICK,
    PICK_BLOCK_TASK_GRAB,
    PICK_BLOCK_TASK_MOVE_TO_SAFE,

    PICK_BLOCK_TASK_DONE,
    PICK_BLOCK_TASK_FAULT

} PickBlockTaskState_t;
```

状态流：

```text
IDLE
 │ Start()
 ↓
MOVE_TO_PICK
 │ BlockArm == REACHED
 ↓
GRAB
 │ BlockVacuum == GRABBED
 ↓
MOVE_TO_SAFE
 │ BlockArm == REACHED
 ↓
DONE
```

任意执行阶段，如果依赖 Mechanism 进入 `FAULT`：

```text
→ PICK_BLOCK_TASK_FAULT
```

## 4.4 对外 API

```c
PickBlockTask_Init();
PickBlockTask_Start();
PickBlockTask_GetState();
PickBlockTask_Process();
```

### `PickBlockTask_Init()`

**语义：**

初始化 Task 自身的软件状态，第一版至少令：

```text
state = IDLE
```

不负责重新初始化 `BlockArm` 或 `BlockVacuum`。

### `PickBlockTask_Start()`

**语义：**

发起一次完整的获取方块任务。

第一版逻辑：

```text
检查启动前置条件
↓
BlockArm_MoveToPick()
↓
task.state = MOVE_TO_PICK
↓
函数返回
```

注意：

```text
Start() 返回
≠ 已经获取方块
```

### `PickBlockTask_GetState()`

返回 Task 当前阶段，用于外部判断：

- 是否空闲；
- 正在执行哪一步；
- 是否完成；
- 是否故障。

### `PickBlockTask_Process()`

周期推进 Task 状态机。

关键实现原则：

> **切换到下一 Task 状态时只发一次下一动作命令；之后周期只查询 Mechanism 状态，不重复发送同一个动作。**

示意：

```c
case PICK_BLOCK_TASK_MOVE_TO_PICK:

    if (BlockArm_GetState() == BLOCK_ARM_REACHED)
    {
        BlockVacuum_Grab();          // 只调用一次
        task.state = PICK_BLOCK_TASK_GRAB;
    }
    break;
```

## 4.5 启动前置条件

第一版建议：

- `BlockArm` 已经完成 Homing；
- `BlockArm` 当前处于允许接受运动命令的状态；
- `BlockVacuum` 当前没有已经保持一个方块；
- 两个 Mechanism 均不处于 `FAULT`。

△ `Start()` 最终返回 `void` 还是 `bool` 尚未正式定死。考虑到前置条件可能不满足，实际编码时优先考虑 `bool` 返回“是否成功启动”。

## 4.6 Task 内部数据

第一版最小数据：

```text
PickBlockTaskState_t state
```

暂时不需要保存更多任务参数。

---

# 5. Task 2：PlaceBlockTask —— 放置方块

## 5.1 涉及 Mechanism

```text
BlockArm
BlockVacuum
```

## 5.2 三个放置目标

当前暂定三个独立业务位置：

```text
底层放置位
第一层放置位
第二层放置位
```

对应接口：

```c
BlockArm_MoveToPlaceBottom();
BlockArm_MoveToPlaceLevel1();
BlockArm_MoveToPlaceLevel2();
```

> 注：这里的软件命名是当前讨论中的业务占位命名，后续可根据机械组最终层级命名统一调整。

当前**不考虑**其中某个放置位是否与 `MoveToPick()` 机械坐标恰好一致。第一版先保持不同的业务函数，避免把“业务语义相同”和“坐标碰巧相同”混为一谈。

## 5.3 执行流程

根据目标层，从三个入口中选择一个：

```text
BlockArm_MoveToPlaceBottom()
或
BlockArm_MoveToPlaceLevel1()
或
BlockArm_MoveToPlaceLevel2()

↓ 等待 BlockArm == REACHED

BlockVacuum_Release()
↓ 等待 BlockVacuum == RELEASED

BlockArm_MoveToSafe()
↓ 等待 BlockArm == REACHED

Task DONE
```

`MoveToSafe()` 与 PickBlockTask 结束时使用的是同一个 BlockArm 安全搬运位。

## 5.4 Task 状态

```c
typedef enum
{
    PLACE_BLOCK_TASK_IDLE,

    PLACE_BLOCK_TASK_MOVE_TO_PLACE,
    PLACE_BLOCK_TASK_RELEASE,
    PLACE_BLOCK_TASK_MOVE_TO_SAFE,

    PLACE_BLOCK_TASK_DONE,
    PLACE_BLOCK_TASK_FAULT

} PlaceBlockTaskState_t;
```

## 5.5 对外 API

```c
PlaceBlockTask_Init();

PlaceBlockTask_StartBottom();
PlaceBlockTask_StartLevel1();
PlaceBlockTask_StartLevel2();

PlaceBlockTask_GetState();
PlaceBlockTask_Process();
```

三个 `StartXXX()` 在启动时直接向 BlockArm 发出对应的第一条运动命令，因此第一版**不需要长期保存 `target_level`**。

例如：

```text
PlaceBlockTask_StartLevel1()
↓
BlockArm_MoveToPlaceLevel1()
↓
state = MOVE_TO_PLACE
```

之后 Task 只等待 `BlockArm == REACHED`，不再需要知道这一轮业务上是 Level1 还是 Level2。

## 5.6 启动前置条件

建议：

```text
BlockArm 已 Homing 且可接受运动命令
+
BlockVacuum == GRABBED
+
相关 Mechanism 均非 FAULT
```

其中 `BlockVacuum == GRABBED` 表示当前软件认为已有方块处于吸附保持状态。

---

# 6. Task 3：PickBallTask —— 获取灵石

## 6.1 涉及 Mechanism

```text
BallGripper
BallRotaryAxis
```

## 6.2 执行流程

当前明确采用：

```text
BallGripper_Open()
↓ 等待 OPENED

BallRotaryAxis_MoveToPick()
↓ 等待 REACHED

BallGripper_Grab()
↓ 等待 GRABBED

BallRotaryAxis_MoveToSafe()
↓ 等待 REACHED

Task DONE
```

这里特意将 `BallGripper_Open()` 放在旋转轴进入抓球位之前。

理由：

> 爪子需要先形成足够的包络空间，否则旋转到灵石附近后可能无法让球进入爪子内部。

△ 真实机械模型到位后，需要额外检查“张开状态下旋转”是否会与机器人其他结构产生空间干涉。

## 6.3 Task 状态

```c
typedef enum
{
    PICK_BALL_TASK_IDLE,

    PICK_BALL_TASK_OPEN,
    PICK_BALL_TASK_MOVE_TO_PICK,
    PICK_BALL_TASK_GRAB,
    PICK_BALL_TASK_MOVE_TO_SAFE,

    PICK_BALL_TASK_DONE,
    PICK_BALL_TASK_FAULT

} PickBallTaskState_t;
```

## 6.4 对外 API

```c
PickBallTask_Init();
PickBallTask_Start();
PickBallTask_GetState();
PickBallTask_Process();
```

## 6.5 启动前置条件

第一版建议：

- `BallRotaryAxis` 已 Homing，机械角度可信；
- `BallRotaryAxis` 当前允许接受运动命令；
- `BallGripper` 不处于 `FAULT`；
- `BallGripper` 当前没有已经抓着灵石。

第一动作本身是 `Open()`，因此不必强制启动前一定处于 `DEFAULT`；具体允许哪些状态调用 `Open()`，在 BallGripper 状态转移实现时细化。

---

# 7. Task 4：PlaceBallTask —— 放置灵石

## 7.1 涉及 Mechanism

```text
BallRotaryAxis
BallGripper
```

## 7.2 执行流程

```text
BallRotaryAxis_MoveToPlace()
↓ 等待 REACHED

BallGripper_Open()
↓ 等待 OPENED

BallRotaryAxis_MoveToSafe()
↓ 等待 REACHED

BallGripper_ReturnToDefault()
↓ 等待 DEFAULT

Task DONE
```

当前采用“先打开爪子释放 → 旋转轴离开 → 爪子恢复默认状态”的顺序。

理由：

> 第一版优先避免爪子在灵石基座附近闭合或恢复时与球体、场地结构发生碰撞。

△ 最终顺序需要结合爪子开合包络、放置区域空间和机械轴旋转路径进行实物确认。

## 7.3 Task 状态

```c
typedef enum
{
    PLACE_BALL_TASK_IDLE,

    PLACE_BALL_TASK_MOVE_TO_PLACE,
    PLACE_BALL_TASK_RELEASE,
    PLACE_BALL_TASK_MOVE_TO_SAFE,
    PLACE_BALL_TASK_RETURN_GRIPPER,

    PLACE_BALL_TASK_DONE,
    PLACE_BALL_TASK_FAULT

} PlaceBallTaskState_t;
```

## 7.4 对外 API

```c
PlaceBallTask_Init();
PlaceBallTask_Start();
PlaceBallTask_GetState();
PlaceBallTask_Process();
```

## 7.5 启动前置条件

建议：

```text
BallRotaryAxis 已 Homing 且可运动
+
BallGripper == GRABBED
+
相关 Mechanism 均非 FAULT
```

## 7.6 关于“稳定放置”的限定

规则要求灵石稳定放置在得分基座。

当前第一版软件完成条件仍暂时定义为：

```text
爪子完成打开
→ 旋转轴回 Safe
→ 爪子回 Default
→ Task DONE
```

这并不能从软件上必然证明“灵石已经稳定在基座上”。

△ 如果未来增加视觉、接触检测或其他可靠反馈，可以进一步增强 `PlaceBallTask` 的完成判据。

---

# 8. Task 层统一设计原则

## 8.1 Task 状态描述“任务进行到哪一步”

例如：

```text
PickBallTask state = GRAB
```

表示：

> 当前 Task 正处于“等待抓球动作完成”的阶段。

而此时 BallGripper 自身可能经历：

```text
GRABBING
→ GRABBED
```

二者属于不同层级，不应混淆。

## 8.2 DONE 与 FAULT

第一版：

```text
DONE
→ 保持 DONE

FAULT
→ 保持 FAULT
```

不在下一个周期自动回到 `IDLE`，以便上层能够可靠读取任务结果。

△ `DONE` 后下一轮是直接允许新的 `Start()`，还是增加专门的 Reset，尚未作为第一版硬性结构确定。

→ `Task_Reset()` 放入后期 Fault Recovery / 生命周期增强方向。

## 8.3 Task FAULT 的第一版边界

第一版只实现到：

```text
Mechanism 检测异常
↓
Mechanism 自己进行必要安全处理
↓
Mechanism → FAULT
↓
Task 检测到依赖 Mechanism == FAULT
↓
Task → FAULT
↓
停止继续发起后续动作
```

Task **不越过 Mechanism 直接操作 Motor / GPIO / Valve 来“救场”**。

具体安全动作属于 Mechanism。

当前不纳入第一版必须结构：

```text
ResetFault()
Task_Reset()
自动重试
自动重新 Homing
自动故障恢复状态机
```

这些统一放在后期优化章节。

## 8.4 Cancel 暂不设计

第一版四个 Task 均不设置：

```c
Task_Cancel();
```

因为取消过程中不同 Mechanism 的安全行为并不统一，例如：

- 机械臂运动可以 Stop 并保持；
- Vacuum 正吸着方块时“取消”不一定应该 Release；
- BallGripper 正抓着球时也不能简单自动 Open。

第一版先完成正常动作链和 FAULT 停止逻辑。

---

# 9. Mechanism 1：BlockArm

## 9.1 机械职责

`BlockArm` 表示：

```text
Motor A
+
Motor B
+
耦合连杆机构
+
方块吸盘末端位置控制
```

两个电机通过连杆耦合，共同决定末端位置，因此不能简单拆成“高度 Mechanism + 前后 Mechanism”。

**✓ 第一版职责：**

> 根据上层业务动作，将吸盘末端移动到取块位、安全搬运位和各放置工作位；内部负责两个 Motor 的目标协调。

上层不关心：

- Motor A / B 分别转多少；
- 连杆运动学细节；
- 电机具体是 DJI 还是 ZDrive；
- 电机底层 CAN / PID。

## 9.2 被哪些 Task 使用

```text
PickBlockTask
PlaceBlockTask
```

同一个 BlockArm 可在同一 Task 中多次调用。

## 9.3 执行器依赖

```c
Motor_t *motor_a;
Motor_t *motor_b;
```

初始化接口：

```c
BlockArm_Init(Motor_t *motor_a, Motor_t *motor_b);
```

采用依赖注入：

```text
main / composition root
↓
创建并初始化两个 Motor_t
↓
BlockArm_Init(&motor_a, &motor_b)
↓
BlockArm 内部保存两个 Motor_t *
```

Task 和上层不直接操作这两个 Motor。

## 9.4 对外 API

```c
BlockArm_Init(Motor_t *motor_a, Motor_t *motor_b);

BlockArm_Home();
BlockArm_Stop();

BlockArm_MoveToPick();
BlockArm_MoveToSafe();

BlockArm_MoveToPlaceBottom();
BlockArm_MoveToPlaceLevel1();
BlockArm_MoveToPlaceLevel2();

BlockArm_GetState();

BlockArm_Process();
```

---

## 9.5 外部函数详细语义

### `BlockArm_Init(Motor_t *motor_a, Motor_t *motor_b)`

**作用：**

- 保存两个 Motor 依赖；
- 初始化 BlockArm 软件内部状态与计数器；
- 第一版初始化后进入 `UNHOMED`。

**不负责：**

- 自动完成机械 Homing；
- 直接认为机械位置已经可信。

---

### `BlockArm_Home()`

**作用：**

启动 BlockArm 的机械归零过程。

调用后：

```text
UNHOMED
↓ Home()
HOMING
```

真正的 Homing 由后续 `BlockArm_Process()` 周期推进。

**调用结束不代表 Homing 完成。**

Homing 成功后：

```text
建立机械参考
↓
停止归零运动
↓
READY
```

△ BlockArm 最终使用何种归零传感器/限位方式尚未确定。

△ `zero_a / zero_b` 的具体定义等待真实机构确定。

---

### `BlockArm_Stop()`

**作用：**

取消当前 BlockArm 运动任务，不再继续前往原目标，并使两个执行器进入可控停止/保持状态。

**明确不是：**

```text
Motor_Disable()
```

因为机械臂可能需要抵抗重力，直接失去控制力可能导致机构下落。

正常运动中：

```text
MOVING
↓ Stop()
STOPPED
```

此时：

> 原任务没有成功完成，机械臂保持在中间位置。

Homing 中：

```text
HOMING
↓ Stop()
UNHOMED
```

因为零点尚未建立，不能认为机械坐标有效。

---

### `BlockArm_MoveToPick()`

**业务语义：**

发起“移动到方块取块工作位”的任务。

内部需要：

- 设置当前控制目标；
- 清零本次到位计数；
- 清零本次运动超时计数；
- 向 `motor_a / motor_b` 发出对应 Position 目标；
- `state = MOVING`。

调用返回 ≠ 已到位。

---

### `BlockArm_MoveToSafe()`

**业务语义：**

将 BlockArm 移动到自身定义的 **Safe / Carry Pose（安全搬运位）**。

该位置用于：

- PickBlockTask 获取方块结束后的搬运姿态；
- PlaceBlockTask 放置完成后的回收姿态；
- 作为不同方块 Task 之间的稳定衔接位置。

△ Safe Pose 的真实高度、前后位置和两个电机目标待机械模型确定。

> BlockArm 的 Safe Pose 与 BallRotaryAxis 的 Safe Pose 是两个不同 Mechanism 各自的安全工作位，不是机器人全局唯一的“安全高度”。

---

### `BlockArm_MoveToPlaceBottom()`

发起移动到方块底层放置工作位。

### `BlockArm_MoveToPlaceLevel1()`

发起移动到当前定义的第一层放置工作位。

### `BlockArm_MoveToPlaceLevel2()`

发起移动到当前定义的第二层放置工作位。

三者都只是不同业务位置的固定入口，内部继续使用相同的 Motor 控制机制。

第一版**不设计**通用：

```c
BlockArm_SetPose(...);
```

当前位置数量有限，一个业务位置一个函数更直接。

---

### `BlockArm_GetState()`

返回当前 BlockArm 业务状态，供：

- Task 协调；
- 调试与监控；
- 上层判断动作是否完成。

不应该让 Task 越过该接口直接读取 Motor mode / CAN 数据作为动作完成依据。

---

### `BlockArm_Process()`

BlockArm 周期状态机入口。

主要负责：

- 更新反馈；
- 推进 HOMING；
- MOVING 时进行到位判断；
- 更新计数器；
- 检测超时；
- 状态转换；
- 发生异常时进入 `FAULT` 并执行本机构必要安全处理。

---

## 9.6 状态机

```c
typedef enum
{
    BLOCK_ARM_UNHOMED,
    BLOCK_ARM_HOMING,
    BLOCK_ARM_READY,
    BLOCK_ARM_MOVING,
    BLOCK_ARM_REACHED,
    BLOCK_ARM_STOPPED,
    BLOCK_ARM_FAULT

} BlockArmState_t;
```

### 状态语义

- `UNHOMED`：尚未建立可信机械零点；
- `HOMING`：正在执行机械归零；
- `READY`：已归零，可接受新的运动任务；
- `MOVING`：正在前往某个业务工作位；
- `REACHED`：上一运动任务已稳定到位；
- `STOPPED`：上一运动任务被主动取消，机构保持在某个中间位置；机械零点仍有效；
- `FAULT`：BlockArm 检测到异常，原任务无法正常继续。

### 主状态流

```text
Init
 ↓
UNHOMED
 │ Home()
 ↓
HOMING
 ├─ Homing 成功 → READY
 ├─ Stop()      → UNHOMED
 └─ 超时/异常   → FAULT
```

正常移动：

```text
READY / REACHED / STOPPED
 │ MoveToXXX()
 ↓
MOVING
 ├─ 稳定到位 → REACHED
 ├─ Stop()    → STOPPED
 └─ 超时异常 → FAULT
```

---

## 9.7 BlockArm 内部数据

第一版设计需要保留：

### 执行器依赖

```c
Motor_t *motor_a;
Motor_t *motor_b;
```

### 当前状态

```c
BlockArmState_t state;
```

### 当前运动目标

概念上需要保存两个执行目标，例如：

```text
target_a
target_b
```

△ 最终究竟保存“两个 Motor 输出轴角度”还是更高层的“机构空间目标 + 内部换算”，等待真实连杆模型确定。

当前第一版软件接口只给固定业务位置，因此可以由 `MoveToXXX()` 直接设置相应目标。

### 当前反馈

至少需要：

```text
current_a
current_b
```

来源：

```c
Motor_GetPosition(motor_a);
Motor_GetPosition(motor_b);
```

△ 如果后续建立完整连杆运动学，还可以进一步计算末端高度/前后位置。

### Homing 相关

```text
zero_a
zero_b
homing_count / homing_timer
```

具体零点建立方式待机械确认。

### 到位判断

```text
tolerance
reached_count
```

第一版思想：

```text
两个执行反馈均进入允许误差
+
连续满足一定周期
↓
REACHED
```

这样避免单个采样瞬间进入误差范围就立即判断成功。

△ 最终耦合机构的完成条件可能增加“末端真实位置”判断，待机械模型确定。

### 超时 / 安全

```text
motion_count / motion_timer
```

用于避免：

```text
MOVING 永远无法结束
```

△ 软件角度 / 机构行程限位需要保留设计位置，但真实数值等待机械确认。

---

## 9.8 BlockArm 内部辅助函数

当前只保留真正有独立职责的三个：

```c
static void BlockArm_UpdatePosition(void);
static bool BlockArm_IsReached(void);
static void BlockArm_StopMotion(void);
```

### `BlockArm_UpdatePosition()`

- 获取两个 Motor 的位置反馈；
- 更新 BlockArm 内部当前位置信息。

### `BlockArm_IsReached()`

- 比较当前反馈与当前目标；
- 处理 tolerance；
- 处理连续到位计数；
- 返回是否稳定到位。

### `BlockArm_StopMotion()`

内部统一执行“停止继续向旧目标运动并保持”的底层逻辑。

可被：

- `BlockArm_Stop()`；
- Homing 成功；
- Homing 超时；
- 运动 Fault；

等情况复用。

当前**不增加**：

```c
BlockArm_SetTarget(...)
```

因为各固定 `MoveToXXX()` 直接调用 `Motor_SetPosition()` 已经足够清楚，不为了少几行重复代码机械增加一层包装。

当前也不强制增加：

```c
BlockArm_HomingProcess()
```

Homing 第一版可以直接写在 `BlockArm_Process()` 的 `HOMING` 分支；真实逻辑变复杂后再抽取。

---

# 10. Mechanism 2：BlockVacuum

## 10.1 职责

```text
BlockVacuum
= 方块真空吸附系统
```

负责：

- 吸取方块；
- 保持吸附；
- 释放方块。

不负责：

- 吸盘末端移动到哪里；
- BlockArm 位置控制。

## 10.2 被哪些 Task 使用

```text
PickBlockTask
PlaceBlockTask
```

## 10.3 对外 API

当前第一版按复杂情况设计：

```c
BlockVacuum_Init();

BlockVacuum_Grab();
BlockVacuum_Release();

BlockVacuum_GetState();
BlockVacuum_Process();
```

当前不设计：

```c
BlockVacuum_Home();
BlockVacuum_Stop();
```

### 为什么没有 `Stop()`

Vacuum 的“Stop”语义不明确：

```text
Stop = 关真空？
Stop = 保持吸附？
Stop = 泄压？
```

如果正在吸着方块，直接“停止”甚至可能让方块掉落。

因此当前只提供明确业务动作：

```text
Grab
Release
```

---

## 10.4 函数语义

### `BlockVacuum_Init()`

初始化 BlockVacuum 软件状态和底层依赖。

△ 底层最终是泵、电磁阀、GPIO 还是其他 Driver，尚未定死。

### `BlockVacuum_Grab()`

发起吸取任务：

```text
开启真空相关执行机构
↓
state = GRABBING
↓
函数返回
```

不代表方块已经确定吸稳。

### `BlockVacuum_Release()`

发起释放任务：

```text
关闭真空 / 泄压等
↓
state = RELEASING
↓
函数返回
```

### `BlockVacuum_GetState()`

供 Task 查询吸取/释放过程当前状态。

### `BlockVacuum_Process()`

**✓ 第一版复杂模型保留。**

负责：

- GRABBING 状态完成判定；
- RELEASING 状态完成判定；
- 计时；
- 可选压力反馈；
- 超时；
- FAULT。

如果未来真实硬件最终只是极简单的 ON/OFF 控制，可以再简化状态机和 Process；第一版设计先按照复杂情况保留。

---

## 10.5 状态机

```c
typedef enum
{
    BLOCK_VACUUM_RELEASED,
    BLOCK_VACUUM_GRABBING,
    BLOCK_VACUUM_GRABBED,
    BLOCK_VACUUM_RELEASING,
    BLOCK_VACUUM_FAULT

} BlockVacuumState_t;
```

状态流：

```text
RELEASED
 │ Grab()
 ↓
GRABBING
 │ GrabComplete
 ↓
GRABBED
 │ Release()
 ↓
RELEASING
 │ ReleaseComplete
 ↓
RELEASED
```

异常：

```text
GRABBING / RELEASING
↓ 超时或反馈异常
FAULT
```

---

## 10.6 `GRABBED` 的语义必须根据传感器区分

### △ 方案 A：存在真空压力反馈

```text
Grab()
↓
开启真空
↓
真空压力达到阈值
↓
GRABBED
```

此时 `GRABBED` 可以比较可靠地表示吸附已经建立。

### △ 方案 B：无压力反馈

```text
Grab()
↓
开启真空
↓
等待固定时间
↓
GRABBED
```

此时：

> `GRABBED` 只能表示“软件认为吸取动作已经执行完成”，不能等价于“传感器确认方块一定吸稳”。

最终设计/代码注释必须保持这个机制边界。

---

## 10.7 BlockVacuum 内部数据

复杂版本至少考虑：

```text
state

grab_count / action_timer
release_count / action_timer

timeout_count

△ vacuum_pressure
△ vacuum_threshold
```

根据最终硬件可以合并计时器，但概念职责必须存在。

不需要：

```text
target_position
current_position
zero_position
position_tolerance
```

因为 Vacuum 不是连续位置控制 Mechanism。

---

## 10.8 内部辅助函数

当前不强制设计额外 `static` 函数。

如果未来压力判断复杂，可以增加：

```c
static bool BlockVacuum_IsGrabComplete(void);
```

否则完成条件直接放在 `BlockVacuum_Process()` 对应状态分支即可。

---

# 11. Mechanism 3：BallGripper

## 11.1 职责

```text
BallGripper
= 气缸 + 爪子机械结构
```

职责：

- 让爪子张开到抓球/释放所需状态；
- 抓取灵石；
- 保持抓取；
- 恢复默认爪子状态。

不负责：

```text
整套抓球机构旋转到哪里
```

旋转位置由 `BallRotaryAxis` 负责。

## 11.2 被哪些 Task 使用

```text
PickBallTask
PlaceBallTask
```

## 11.3 对外 API

```c
BallGripper_Init();

BallGripper_Open();
BallGripper_Grab();
BallGripper_ReturnToDefault();

BallGripper_GetState();
BallGripper_Process();
```

第一版不设计：

```c
BallGripper_Home();
BallGripper_Stop();
```

### 为什么不需要 Home

当前爪子由气缸控制，本质上更接近若干离散机械状态，不要求像连续位置电机一样先建立坐标零点。

△ 如果后续真实机构证明需要特殊初始化，再增加。

### 为什么不需要 Stop

气缸运动中“Stop”到底表示保持、断气、泄压还是回默认位置并不明确，且不一定能稳定停在中间位置。

因此第一版只提供明确动作。

---

## 11.4 函数语义

### `BallGripper_Open()`

发起爪子张开动作。

该接口同时用于：

1. PickBallTask 抓球前打开爪子；
2. PlaceBallTask 放球时打开爪子释放灵石。

调用后：

```text
state = OPENING
```

完成后：

```text
OPENED
```

### `BallGripper_Grab()`

发起抓球动作：

```text
OPENED
↓ Grab()
GRABBING
↓ 完成条件
GRABBED
```

### `BallGripper_ReturnToDefault()`

让爪子从当前张开状态恢复到默认机械状态。

注意：

> 当前不把 Default 写死成 “Closed”，因为默认状态真实机械含义尚未最终确认。

### `BallGripper_GetState()`

供 Task 查询。

### `BallGripper_Process()`

周期推进：

- OPENING；
- GRABBING；
- RETURNING；
- 完成判断；
- 动作计时；
- 可选传感器；
- 超时 Fault。

---

## 11.5 状态机

```c
typedef enum
{
    BALL_GRIPPER_DEFAULT,

    BALL_GRIPPER_OPENING,
    BALL_GRIPPER_OPENED,

    BALL_GRIPPER_GRABBING,
    BALL_GRIPPER_GRABBED,

    BALL_GRIPPER_RETURNING,

    BALL_GRIPPER_FAULT

} BallGripperState_t;
```

抓球：

```text
DEFAULT
 │ Open()
 ↓
OPENING
 │ OpenComplete
 ↓
OPENED
 │ Grab()
 ↓
GRABBING
 │ GrabComplete
 ↓
GRABBED
```

放球：

```text
GRABBED
 │ Open()
 ↓
OPENING
 │ OpenComplete
 ↓
OPENED
```

机械轴回 Safe 后：

```text
OPENED
 │ ReturnToDefault()
 ↓
RETURNING
 │ DefaultComplete
 ↓
DEFAULT
```

---

## 11.6 `GRABBED` 的语义边界

△ 如果存在真正的抓球检测：

```text
传感器确认球已抓稳
→ GRABBED
```

△ 如果只有气缸到位/时间判断：

```text
气缸动作完成
→ 软件暂定 GRABBED
```

后一种情况下，`GRABBED` 不等价于“球一定存在于爪内”。

---

## 11.7 BallGripper 内部数据

复杂版考虑：

```text
state

action_count / action_timer

△ 气缸到位传感器状态
△ 抓球检测状态

timeout_count / timeout_timer
```

不需要连续位置控制数据。

---

## 11.8 BallGripper 待确认项

**△ 必须在总设计中明确保留：**

- `DEFAULT` 实际机械状态；
- 气缸伸/缩分别对应 Open / Grab / Default 的哪一个动作；
- 是否存在磁性开关/限位反馈；
- 是否存在真正抓球检测；
- OPENING / GRABBING / RETURNING 的完成条件；
- 各动作超时时间；
- 释放后保持爪子张开再旋转是否存在机械干涉。

---

# 12. Mechanism 4：BallRotaryAxis

## 12.1 职责

```text
BallRotaryAxis
= 一个 Motor
+ 旋转机械轴
+ 整套 BallGripper 的角度位置控制
```

职责：

> 将抓球机构旋转到 Pick / Place / Safe 三个固定业务位置。

不负责爪子开合。

## 12.2 被哪些 Task 使用

```text
PickBallTask
PlaceBallTask
```

## 12.3 执行器依赖

```c
Motor_t *motor;
```

通过依赖注入：

```c
BallRotaryAxis_Init(Motor_t *motor);
```

内部只依赖公共 Motor API，不关心实际 Motor 是 DJI 还是 ZDrive。

---

## 12.4 对外 API

```c
BallRotaryAxis_Init(Motor_t *motor);

BallRotaryAxis_Home();
BallRotaryAxis_Stop();

BallRotaryAxis_MoveToPick();
BallRotaryAxis_MoveToPlace();
BallRotaryAxis_MoveToSafe();

BallRotaryAxis_GetState();
BallRotaryAxis_Process();
```

---

## 12.5 函数详细语义

### `BallRotaryAxis_Init(Motor_t *motor)`

绑定 Motor 并初始化软件状态。

初始化后：

```text
UNHOMED
```

### `BallRotaryAxis_Home()`

**✓ 明确保留，不是可选。**

当前已经确定：

> BallRotaryAxis 后续会设置机械零位限位，通过该限位建立机械零点。

逻辑：

```text
UNHOMED
↓ Home()
HOMING
↓
低速向 Homing 方向运动
↓
检测到机械零位限位
↓
停止
↓
建立 zero_angle
↓
READY
```

如果 Homing 超时：

```text
StopMotion()
↓
FAULT
```

如果 Homing 中主动 `Stop()`：

```text
HOMING
↓
UNHOMED
```

### `BallRotaryAxis_Stop()`

与 `BlockArm_Stop()` 采用相同业务语义：

> 取消当前旋转目标，不继续向旧目标运动，并让旋转轴进入可控保持状态。

不是：

```text
Motor_Disable()
```

正常运动中：

```text
MOVING
↓ Stop()
STOPPED
```

### `MoveToPick / MoveToPlace / MoveToSafe`

三个固定业务位置接口。

第一版不向 Task 暴露通用角度接口：

```c
BallRotaryAxis_SetAngle(...);   // 当前不作为 Task API
```

真实角度全部留在 Mechanism 内部。

---

## 12.6 状态机

```c
typedef enum
{
    BALL_AXIS_UNHOMED,
    BALL_AXIS_HOMING,
    BALL_AXIS_READY,
    BALL_AXIS_MOVING,
    BALL_AXIS_REACHED,
    BALL_AXIS_STOPPED,
    BALL_AXIS_FAULT

} BallRotaryAxisState_t;
```

状态流：

```text
UNHOMED
 │ Home()
 ↓
HOMING
 ├─ 限位触发成功 → READY
 ├─ Stop()        → UNHOMED
 └─ 超时/异常     → FAULT
```

正常运动：

```text
READY / REACHED / STOPPED
 │ MoveToXXX()
 ↓
MOVING
 ├─ 稳定到位 → REACHED
 ├─ Stop()    → STOPPED
 └─ 超时异常 → FAULT
```

---

## 12.7 BallRotaryAxis 内部数据

```text
Motor_t *motor

BallRotaryAxisState_t state

target_angle
current_angle

zero_angle
homing_count / homing_timer

tolerance
reached_count

motion_count / motion_timer

软件角度范围
```

当前不保存：

```text
target_type = PICK / PLACE / SAFE
```

原因：

> `MoveToXXX()` 已经表达业务语义，进入运动后 Process 只需要知道真实目标角度。

---

## 12.8 内部辅助函数

```c
static void BallRotaryAxis_UpdatePosition(void);
static bool BallRotaryAxis_IsReached(void);
static void BallRotaryAxis_StopMotion(void);
```

职责与 BlockArm 对应，但这里是单 Motor 版本。

当前不增加额外 `SetTarget()` 包装。

---

## 12.9 BallRotaryAxis 待确认项

**✓ 已确定：**

- 必须 Home；
- 使用机械零位限位建立零点；
- Stop 保留；
- Pick / Place / Safe 三个业务位置。

**△ 待确认：**

- 限位传感器具体形式；
- Homing 方向；
- Homing 速度；
- Homing timeout；
- `zero_angle` 定义；
- Pick / Place / Safe 的具体角度；
- 软件角度限位；
- 到位 tolerance；
- 连续到位计数；
- 运动 timeout。

---

# 13. Safe / Carry Pose 设计

当前存在两个不同的安全工作位：

```text
BlockArm Safe / Carry Pose
BallRotaryAxis Safe Pose
```

二者属于各自 Mechanism，不应混成一个全局坐标。

## 13.1 BlockArm Safe Pose

用于：

- 获取方块结束后的安全搬运；
- 放置方块结束后的机构回收；
- 方块 Task 之间的统一衔接状态。

△ 具体是否同时包含“抬高 + 缩回”、具体高度和伸出量，等待机械模型确定。

## 13.2 BallRotaryAxis Safe Pose

用于：

- 抓到灵石后把抓球机构转回安全搬运位置；
- 灵石释放后离开基座附近。

其角度与 BlockArm Safe Pose 完全独立。

---

# 14. 完成判据设计总则

## 14.1 连续位置型 Mechanism

适用：

```text
BlockArm
BallRotaryAxis
```

第一版采用：

```text
当前反馈进入 tolerance
+
连续满足 N 个 Process 周期
↓
REACHED
```

原因：

> 避免振动或单次采样进入误差范围就错误认为动作稳定完成。

计数器的物理时间必须结合 `Process()` 周期解释：

```text
N 次 × Process 周期 = 实际稳定时间
```

调度周期变化时，阈值不能机械沿用。

## 14.2 Vacuum

完成判据存在两条路径：

```text
压力反馈
或
动作延时
```

第一版保留两种可能。

## 14.3 Pneumatic Gripper

完成判据可能来自：

- 气缸磁性开关；
- 限位开关；
- 抓球传感器；
- 固定动作时间。

具体待机械和电气方案确定。

---

# 15. FAULT 与安全边界

## 15.1 Mechanism 负责自己的硬件安全

例如：

```text
BlockArm Fault
→ BlockArm 自己停止/保持两个 Motor

BallRotaryAxis Fault
→ 自己停止/保持旋转轴

Vacuum Fault
→ 根据真实气路方案决定安全状态

BallGripper Fault
→ 根据真实气路与机械状态决定安全处理
```

Task 不应该直接写：

```c
Motor_Disable(...);
HAL_GPIO_WritePin(...);
```

来处理底层安全。

## 15.2 Task 负责流程停止

Task 看到任意依赖 Mechanism `FAULT`：

```text
Task → FAULT
```

此后不再发起后续动作。

## 15.3 第一版暂不实现 Fault Recovery

**→ 后期优化方向**

未来可以考虑：

```text
实际排障
↓
Mechanism_ResetFault()
↓
位置机构必要时重新 Home
↓
Mechanism 可工作
↓
Task_Reset()
↓
重新 Start
```

但：

```c
ResetFault();
Task_Reset();
```

当前不纳入必须结构。

---

# 16. Ownership / 控制权原则

**✓ 第一版架构原则**

同一个 Mechanism 在同一时刻不应该同时接受多个上层控制者发来的相互冲突命令。

正常比赛控制推荐：

```text
Operator
↓
Task
↓
Mechanism
```

当一个 Task 正在执行时，不应该同时从 Operator 或其他 Task 越过当前流程直接修改其正在使用的 Mechanism 目标。

例如错误情况：

```text
PickBlockTask 正等待 BlockArm 到 Pick
↓
外部直接 BlockArm_MoveToSafe()
```

此时 Task 仍然等待 Pick 完成，但 BlockArm 目标已经被覆盖，流程会失效。

---

# 17. Operator Control —— 仅作为参考设计

**△ 当前不作为必须实现模块。**

原因：

- 当前对上位机/遥控输入的软件实现学习较少；
- 最终可能由其他成员实现；
- 只需要对方遵守 Task API 即可与本文方案对接。

## 17.1 推荐职责

Operator 只负责：

```text
读取操作手意图
↓
选择一个 Task
↓
调用对应 Start()
```

示例：

```text
获取方块
→ PickBlockTask_Start()

放底层方块
→ PlaceBlockTask_StartBottom()

放第一层方块
→ PlaceBlockTask_StartLevel1()

放第二层方块
→ PlaceBlockTask_StartLevel2()

获取灵石
→ PickBallTask_Start()

放置灵石
→ PlaceBallTask_Start()
```

实际遥控器按键映射待后续确定。

## 17.2 第一版推荐一次只运行一个 Task

为了避免资源冲突和调试复杂度，Operator 参考设计可以维护：

```text
active_task
```

第一版：

```text
active_task == NONE
→ 才允许启动新 Task
```

即使方块和灵石 Mechanism 从物理资源上可能互不冲突，也暂时不做多 Task 并发。

## 17.3 输入边沿

如果未来实现按键控制，不能在“按键持续按下”期间每个周期重复调用 `Task_Start()`。

应该采用：

```text
未按
→ 按下
```

这一瞬间产生一次启动事件。

但这一部分只作为 Operator 实现参考。

---

# 18. 主循环与模块调度建议

第一版总体调用关系：

```text
Driver 周期逻辑
↓
Mechanism_Process()
↓
Task_Process()
↓
Operator_Process()   // 如果实现
```

推荐思想：

1. Driver / 反馈先更新；
2. Mechanism 根据最新反馈更新状态；
3. Task 读取最新 Mechanism 状态并决定是否进入下一步；
4. Operator 再根据操作输入启动新的 Task。

示意：

```c
while (1)
{
    /* 各 Driver 自己所需的周期任务 */

    BlockArm_Process();
    BlockVacuum_Process();
    BallGripper_Process();
    BallRotaryAxis_Process();

    PickBlockTask_Process();
    PlaceBlockTask_Process();
    PickBallTask_Process();
    PlaceBallTask_Process();

    /* Operator_Process();  可选 */
}
```

### 重要：不要强行统一 Driver Process

当前 DJI / ZDrive 等 Driver 的周期要求可能不同。

不应该为了表面统一强行创建一个没有真实共同语义的：

```c
Motor_Process();
```

Driver 按自身协议和控制要求调度；Mechanism 只依赖对外执行器 API。

---

# 19. 系统初始化建议

第一版可以按以下层级：

```text
HAL / 外设初始化
↓
Driver Init
↓
Motor_t Init / Bind
↓
Mechanism Init
↓
Task Init
↓
需要 Homing 的 Mechanism 执行 Home
↓
确认位置型 Mechanism Ready
↓
进入正常比赛操作
```

例如概念上：

```c
DJI_motor_init();
ZdriveInit();

/* Motor */
Motor_Init(&block_motor_a, ...);
Motor_Init(&block_motor_b, ...);
Motor_Init(&ball_axis_motor, ...);

/* Mechanism */
BlockArm_Init(&block_motor_a, &block_motor_b);
BlockVacuum_Init();
BallGripper_Init();
BallRotaryAxis_Init(&ball_axis_motor);

/* Task */
PickBlockTask_Init();
PlaceBlockTask_Init();
PickBallTask_Init();
PlaceBallTask_Init();

/* Homing */
BlockArm_Home();
BallRotaryAxis_Home();
```

注意：

> `Mechanism_Init()` ≠ `Mechanism_Home()`。

Init 建立软件对象和依赖；Home 是真实机械运动任务。

---

# 20. 推荐代码目录

建议继续采用“按功能模块封装”，让 `main.c` 保持为初始化 + 调度入口。

```text
Mechanism/
├─ Inc/
│  ├─ block_arm.h
│  ├─ block_vacuum.h
│  ├─ ball_gripper.h
│  └─ ball_rotary_axis.h
│
└─ Src/
   ├─ block_arm.c
   ├─ block_vacuum.c
   ├─ ball_gripper.c
   └─ ball_rotary_axis.c


Task/
├─ Inc/
│  ├─ pick_block_task.h
│  ├─ place_block_task.h
│  ├─ pick_ball_task.h
│  └─ place_ball_task.h
│
└─ Src/
   ├─ pick_block_task.c
   ├─ place_block_task.c
   ├─ pick_ball_task.c
   └─ place_ball_task.c


Motor/
├─ Inc/
└─ Src/


Driver/
├─ ...
```

原则：

```text
.h
→ 对外公开 enum / 类型 / API

.c
→ 状态机
→ 内部数据
→ static 辅助函数
→ 具体实现
```

仅供模块内部使用的函数和变量尽量 `static`。

---

# 21. 第一版当前明确不做的内容

以下内容不属于当前第一版必须完成：

```text
Operator / 上位机完整实现
遥控器具体按键映射
多 Task 并行
Task 内多动作并行
自动重试
Task_Cancel
完整 Fault Recovery
Task_Reset
Mechanism_ResetFault
复杂资源锁 / Resource Manager
自动比赛策略
视觉自动对准
底盘与机构联合闭环
```

---

# 22. 待机械 / 电气确认清单

后续拿到真实机械模型后，需要按此清单逐项回填。

## 22.1 BlockArm

- 两个 Motor 的真实安装方向；
- 两 Motor 与连杆末端位置之间的关系；
- Pick Pose；
- Safe / Carry Pose；
- 三个 Place Pose；
- 是否只需要高度控制，还是需要固定前后位置一起确定；
- Homing 传感器与 Homing 方向；
- zero 定义；
- 软件行程限位；
- 到位 tolerance；
- reached_count；
- motion timeout；
- Homing timeout。

## 22.2 BlockVacuum

- 真空泵 / 电磁阀方案；
- 底层 Driver；
- 是否有压力传感器；
- Grab 完成条件；
- Release 完成条件；
- Grab / Release 所需时间；
- 超时条件；
- Fault 时继续保持真空还是切换其他安全状态。

## 22.3 BallGripper

- 气缸伸/缩与各业务动作的真实对应关系；
- DEFAULT 机械状态；
- 是否有磁性开关；
- 是否有抓球检测；
- Open / Grab / Return 完成条件；
- 各动作 timeout；
- 张开状态旋转时的空间包络。

## 22.4 BallRotaryAxis

- 机械零位限位开关类型；
- Homing 方向和速度；
- zero_angle；
- Pick angle；
- Place angle；
- Safe angle；
- 软件角度范围；
- tolerance；
- reached_count；
- motion timeout；
- Homing timeout。

---

# 23. 关于底盘对准与机构运动的当前策略

当前初步机械思路是：

- BlockArm 主要移动到若干固定业务工作位；
- 前后方向的最终对准较多依赖底盘；
- 两个 BlockArm 电机的耦合运动在 Mechanism 内部完成。

第一版不做底盘与机械臂联合控制。

建议操作策略：

```text
粗定位 / 预动作阶段
→ 后续可以允许底盘和机构同时运动

最终抓取 / 最终放置阶段
→ 第一版优先底盘停止或低速微调
→ Mechanism 完成精确动作
```

这不是第一版 Task 的自动互锁逻辑，只是操作与后期优化参考。

→ 实物验证后，可以研究底盘移动 + Mechanism 预定位并行，提高比赛效率。

---

# 24. 后期优化方向

以下内容全部明确为**后期方向**，不影响当前第一版落代码。

## 24.1 Fault Recovery

可能增加：

```c
Mechanism_ResetFault();
Task_Reset();
```

恢复思路：

```text
实际排障
↓
恢复 Mechanism
↓
位置型机构必要时重新 Home
↓
Mechanism Ready
↓
清除 Task Fault
↓
重新决定是否 Start
```

不自动假设 Fault 可以直接回 READY。

## 24.2 Task Cancel

以后如果需要操作手中途取消整个 Task，需要为不同阶段明确安全策略。

不能简单统一：

```text
Cancel → 所有执行器 Disable
```

必须根据当前抓取/承载状态设计。

## 24.3 自动重试

例如 Vacuum 未吸住后自动重新尝试。

需要进一步设计：

- 最大重试次数；
- 重试前机械臂是否重新定位；
- 物体可能被移动后的处理；
- 重试失败后的 Fault。

## 24.4 Task 内并行动作

实物稳定后可研究：

```text
BlockArm 接近 Safe
+
底盘提前移动
```

或机构动作提前量。

第一版不做。

## 24.5 多 Task 并发

未来可以根据 Mechanism 资源集合判断：

```text
方块 Task
→ BlockArm + BlockVacuum

灵石 Task
→ BallRotaryAxis + BallGripper
```

理论上资源可能不冲突。

但并发会引入：

- Ownership；
- Resource Lock；
- Task 优先级；
- 取消和 Fault 联动；

因此第一版全局一次只运行一个 Task 更稳妥。

## 24.6 Operator / 上位机完善

以后可进一步设计：

- 遥控器按键映射；
- 上位机协议；
- Task 状态回传；
- 错误提示；
- 手动/自动模式；
- Debug 模式下直接控制 Mechanism。

正常比赛模式仍建议：

```text
Operator → Task → Mechanism
```

Debug 模式才允许：

```text
Operator → Mechanism
```

两种控制权不能同时抢同一个 Mechanism。

---

# 25. 推荐编码实施顺序

第一版真正开始写代码时，建议按依赖关系推进，而不是四个 Task 同时开工。

```text
阶段 1：建立四个 Mechanism 头文件骨架
↓
状态 enum
外部 API
结构体/内部数据规划

阶段 2：实现 BlockArm
↓
双 Motor 依赖
Home
固定业务位置
反馈更新
到位判断
Stop
Process

阶段 3：实现 BallRotaryAxis
↓
单 Motor
机械限位 Homing
Pick / Place / Safe
到位 / Stop / Process

阶段 4：实现 BlockVacuum
↓
复杂版状态机
Grab / Release
Process
实际反馈方案后补

阶段 5：实现 BallGripper
↓
Open / Grab / Return
状态机
Process
实际气缸反馈后补

阶段 6：分别进行 Mechanism 单独测试
↓
先确认一个 Mechanism 自己能正确运行
不要立刻上 Task

阶段 7：实现四个 Task
↓
先 PickBlockTask 建模板
再 PlaceBlock / PickBall / PlaceBall

阶段 8：Task + Mechanism 联合调试
↓
严格串行
确认状态转移

阶段 9：如有需要再对接 Operator
```

---

# 26. 编码验证原则

每个 Mechanism / Task 不以“代码写完”作为完成标准。

至少检查：

```text
1. 接口声明和职责一致
2. 编译 0 Error
3. Warning 有明确处理
4. 状态转移符合设计
5. 动作接口只发起一次任务
6. Process 能正确推进
7. GetState 能被上层可靠读取
8. 真实执行器动作符合预期
9. 到位 / 完成判据经过实物验证
10. Fault 不会继续执行后续危险动作
```

对于真实硬件：

> 下载前先明确“正常情况下应该看到什么”，再根据实物反馈判断是否通过。

---

# 27. 当前第一版设计最终总览

```text
                        [ Operator / Remote ]
                         △ 参考，不是当前必做
                                 │
                                 ↓
 ┌────────────────────────────────────────────────────┐
 │                      Task 层                        │
 │                                                    │
 │ PickBlockTask       PlaceBlockTask                 │
 │ PickBallTask        PlaceBallTask                  │
 │                                                    │
 │ 职责：完整任务顺序、等待 Mechanism 完成、Fault传播 │
 └────────────────────────────────────────────────────┘
                                 │
                          command / state
                                 ↓
 ┌────────────────────────────────────────────────────┐
 │                   Mechanism 层                     │
 │                                                    │
 │ BlockArm          BlockVacuum                      │
 │ BallGripper       BallRotaryAxis                   │
 │                                                    │
 │ 职责：独立机械子系统的动作、状态和完成判断          │
 └────────────────────────────────────────────────────┘
                                 │
                                 ↓
 ┌────────────────────────────────────────────────────┐
 │                  Actuator 层                       │
 │                                                    │
 │ Motor_t       Pneumatic       Vacuum ...           │
 └────────────────────────────────────────────────────┘
                                 │
                                 ↓
                        Adapter / Driver
                                 │
                                 ↓
                            HAL / Hardware
```

---

# 28. 当前结论

当前第一版已经完成以下设计闭环：

- ✓ 从校内赛需求拆分 4 个核心 Task；
- ✓ 从 Task 拆分 4 个 Mechanism；
- ✓ 区分 Task 与 Mechanism 的职责；
- ✓ 明确一个 Mechanism 可以被多个 Task 复用；
- ✓ 明确一个 Task 可以多次调用同一个 Mechanism；
- ✓ 完成每个 Task 的第一版状态机；
- ✓ 完成每个 Task 的对外 API；
- ✓ 完成 4 个 Mechanism 的职责、依赖、状态机和 API；
- ✓ 明确连续位置机构的 Home / Stop / Reached 语义；
- ✓ 明确 Vacuum / Pneumatic 的完成判据存在硬件相关分支；
- ✓ 明确每个 Mechanism 需要保存的主要内部数据；
- ✓ 明确内部 `static` 辅助函数范围；
- ✓ 确定第一版严格串行；
- ✓ 确定命令调用结束不等于动作完成；
- ✓ 确定 Mechanism Fault → Task Fault 的第一版传播边界；
- ✓ 确定 Ownership 原则；
- ✓ 给出 Operator 层参考方案但不列为当前必做；
- ✓ 区分第一版确定内容、待确认内容和后期优化内容。

因此：

> **本文可以作为当前校内赛上层控制软件第一版编码基线。**

后续不需要重新从“整体软件怎么分”开始思考，而应按本文顺序逐个实现、编译、测试、实物验证，再根据真实机械模型补齐所有 `△` 项。

