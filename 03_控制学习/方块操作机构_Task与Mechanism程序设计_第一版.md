# 方块操作机构 Task 与 Mechanism 程序设计（第一版）

> 日期：2026-08-24  
> 文档定位：大地块拾取、放置机构的第一版编码基线。  
> 适用模块：`PickBlockTask`、`PlaceBlockTask`、`BlockArm`、`BlockVacuum`。

---

## 0. 设计范围

本方案只设计大地块的拾取和放置控制流程：

```text
PickBlockTask       PlaceBlockTask
        ↓                  ↓
        └──── BlockArm + BlockVacuum ────┘
                         ↓
              Motor / Vacuum Driver
```

第一版继续采用：

```text
enum
+ switch-case
+ 非阻塞 Process()
+ 固定内部变量
+ 一个 Task 串行执行
```

当前不采用：

```text
通用 Command Scheduler
资源锁 / Ownership 模块
动态动作表
在线逆运动学
自动视觉对准
自动重试与 Fault Recovery
```

当前机械结构采用侧向吸盘。自动动作只负责把吸盘送到目标附近的粗定位姿态，最终位置由操作手通过四方向末端微调完成。

---

# 1. 统一分层原则

## 1.1 Task

Task 表示完整比赛目标，负责：

```text
决定 Mechanism 的调用顺序
等待 Mechanism 的真实完成状态
等待操作手确认吸附或释放
发现 Mechanism FAULT 后停止流程
```

Task 不负责：

```text
设置两个电机的具体角度或速度
计算连杆运动
控制真空泵、阀、GPIO
判断电机底层通信是否正常
```

## 1.2 Mechanism

`BlockArm` 负责两个耦合电机和连杆末端运动；`BlockVacuum` 负责真空吸附与释放。

Task 调用动作函数，只表示动作已经发起：

```text
函数返回
≠
物理动作已经完成
```

Task 必须通过 `GetState()` 等待 Mechanism 的真实状态。

## 1.3 人工微调

拾取和放置共用同一套四方向微调接口：

```text
FORWARD
BACKWARD
UP
DOWN
```

上层按住方向键时持续微调，松开时停止并保持当前位置。由于连杆耦合，每个方向内部可能需要两个电机共同运动。

---

# 2. PickBlockTask —— 拾取大地块

## 2.1 涉及 Mechanism

```text
BlockArm
BlockVacuum
```

## 2.2 执行流程

```text
StartLowPick() / StartHighPick()
↓
BlockArm 到对应取块预备姿态
↓ 等待 BlockArm == REACHED
进入人工微调
↓
操作手确认吸附
↓
BlockVacuum_Grab()
↓ 等待 BlockVacuum == GRABBED
BlockArm_MoveToSafe()
↓ 等待 BlockArm == REACHED
DONE
```

低位取块适用于单层大地块，或双层上方块取走后的下方块；高位取块适用于双层堆放中的上方块。

## 2.3 Task 状态

```c
typedef enum
{
    PICK_BLOCK_TASK_IDLE,

    PICK_BLOCK_TASK_MOVE_TO_PICK_PREPARE,
    PICK_BLOCK_TASK_MANUAL_ALIGN,
    PICK_BLOCK_TASK_GRAB,
    PICK_BLOCK_TASK_MOVE_TO_SAFE,

    PICK_BLOCK_TASK_DONE,
    PICK_BLOCK_TASK_FAULT

} PickBlockTaskState_t;
```

状态含义：

| 状态 | 含义 |
|---|---|
| `IDLE` | 当前没有执行拾取任务 |
| `MOVE_TO_PICK_PREPARE` | 正在前往高位或低位取块预备姿态 |
| `MANUAL_ALIGN` | 粗定位完成，等待操作手微调并确认吸附 |
| `GRAB` | 已启动真空，等待 `BlockVacuum == GRABBED` |
| `MOVE_TO_SAFE` | 已吸住方块，正在撤离到安全搬运姿态 |
| `DONE` | 拾取任务完成 |
| `FAULT` | 依赖 Mechanism 故障，Task 停止推进 |

## 2.4 对外接口

```c
/** 初始化拾取 Task 自身状态，不初始化两个 Mechanism。 */
void PickBlockTask_Init(void);

/** 启动低位大地块拾取，并发起低位粗定位动作。 */
void PickBlockTask_StartLowPick(void);

/** 启动双层上方大地块拾取，并发起高位粗定位动作。 */
void PickBlockTask_StartHighPick(void);

/**
 * 操作手确认吸盘已经完成侧面对准，允许开始建立真空。
 * 只在 PICK_BLOCK_TASK_MANUAL_ALIGN 状态下生效。
 */
void PickBlockTask_ConfirmGrab(void);

/** 返回当前拾取 Task 状态。 */
PickBlockTaskState_t PickBlockTask_GetState(void);

/** 周期推进拾取 Task 状态机；函数不得阻塞。 */
void PickBlockTask_Process(void);
```

## 2.5 关键接口行为

### `PickBlockTask_StartLowPick()`

```text
检查 Task 与 Mechanism 是否允许启动
↓
BlockArm_MoveToLowPickReady()
↓
state = PICK_BLOCK_TASK_MOVE_TO_PICK_PREPARE
↓
函数立即返回
```

### `PickBlockTask_StartHighPick()`

```text
检查 Task 与 Mechanism 是否允许启动
↓
BlockArm_MoveToHighPickReady()
↓
state = PICK_BLOCK_TASK_MOVE_TO_PICK_PREPARE
↓
函数立即返回
```

### `PickBlockTask_ConfirmGrab()`

```text
确认当前 state == MANUAL_ALIGN
↓
BlockArm_StopFineAdjust()
↓
BlockVacuum_Grab()
↓
state = PICK_BLOCK_TASK_GRAB
```

该接口中的“确认”是操作手确认可以吸附，不代表真空已经建立。Task 后续仍需等待 `BLOCK_VACUUM_GRABBED`。

## 2.6 `Process()` 状态推进

```c
switch (pick_block_task_state)
{
    case PICK_BLOCK_TASK_MOVE_TO_PICK_PREPARE:
        /* BlockArm 到达粗定位姿态后，进入人工微调。 */
        break;

    case PICK_BLOCK_TASK_MANUAL_ALIGN:
        /* 不自动前进，等待操作手调用 ConfirmGrab()。 */
        break;

    case PICK_BLOCK_TASK_GRAB:
        /* 等待真空确认吸住，再发起 MoveToSafe()。 */
        break;

    case PICK_BLOCK_TASK_MOVE_TO_SAFE:
        /* 等待 BlockArm 到达 Safe，随后进入 DONE。 */
        break;

    default:
        break;
}
```

## 2.7 内部数据

```c
static PickBlockTaskState_t pick_block_task_state;
```

第一版不长期保存高低取块参数。高低选择已经在两个启动接口中转换为对应的 `BlockArm` 动作。

---

# 3. PlaceBlockTask —— 放置大地块

## 3.1 涉及 Mechanism

```text
BlockArm
BlockVacuum
```

## 3.2 执行流程

```text
StartBottom() / StartLevel1() / StartLevel2()
↓
BlockArm 到对应层的放块预备姿态
↓ 等待 BlockArm == REACHED
进入人工微调
↓
操作手确认方块已经获得下方支撑，可以释放
↓
BlockVacuum_Release()
↓ 等待 BlockVacuum == RELEASED
BlockArm_MoveToSafe()
↓ 等待 BlockArm == REACHED
DONE
```

三个放块预备姿态分别对应底层、第一层和第二层。它们可能同时具有不同高度、前后距离和吸盘姿态。

## 3.3 Task 状态

```c
typedef enum
{
    PLACE_BLOCK_TASK_IDLE,

    PLACE_BLOCK_TASK_MOVE_TO_PLACE_PREPARE,
    PLACE_BLOCK_TASK_MANUAL_ALIGN,
    PLACE_BLOCK_TASK_RELEASE,
    PLACE_BLOCK_TASK_MOVE_TO_SAFE,

    PLACE_BLOCK_TASK_DONE,
    PLACE_BLOCK_TASK_FAULT

} PlaceBlockTaskState_t;
```

状态含义：

| 状态 | 含义 |
|---|---|
| `IDLE` | 当前没有执行放置任务 |
| `MOVE_TO_PLACE_PREPARE` | 正在前往所选层的放块预备姿态 |
| `MANUAL_ALIGN` | 粗定位完成，等待操作手微调并确认释放 |
| `RELEASE` | 已发起真空释放，等待 `BlockVacuum == RELEASED` |
| `MOVE_TO_SAFE` | 已确认释放，机械臂正在撤离 |
| `DONE` | 放置任务完成 |
| `FAULT` | 依赖 Mechanism 故障，Task 停止推进 |

## 3.4 对外接口

```c
/** 初始化放置 Task 自身状态，不初始化两个 Mechanism。 */
void PlaceBlockTask_Init(void);

/** 启动底层放置任务，并发起底层粗定位动作。 */
void PlaceBlockTask_StartBottom(void);

/** 启动第一层放置任务，并发起第一层粗定位动作。 */
void PlaceBlockTask_StartLevel1(void);

/** 启动第二层放置任务，并发起第二层粗定位动作。 */
void PlaceBlockTask_StartLevel2(void);

/**
 * 操作手确认方块位置合适且已有下方支撑，允许释放真空。
 * 只在 PLACE_BLOCK_TASK_MANUAL_ALIGN 状态下生效。
 */
void PlaceBlockTask_ConfirmRelease(void);

/** 返回当前放置 Task 状态。 */
PlaceBlockTaskState_t PlaceBlockTask_GetState(void);

/** 周期推进放置 Task 状态机；函数不得阻塞。 */
void PlaceBlockTask_Process(void);
```

## 3.5 关键接口行为

三个启动接口分别调用：

```text
StartBottom() → BlockArm_MoveToPlaceBottomReady()
StartLevel1() → BlockArm_MoveToPlaceLevel1Ready()
StartLevel2() → BlockArm_MoveToPlaceLevel2Ready()
```

随后统一进入：

```c
PLACE_BLOCK_TASK_MOVE_TO_PLACE_PREPARE
```

### `PlaceBlockTask_ConfirmRelease()`

```text
确认当前 state == MANUAL_ALIGN
↓
BlockArm_StopFineAdjust()
↓
BlockVacuum_Release()
↓
state = PLACE_BLOCK_TASK_RELEASE
```

该接口中的“确认”是操作手确认允许放气，不代表真空系统已经实际释放完成。

## 3.6 `Process()` 状态推进

```c
switch (place_block_task_state)
{
    case PLACE_BLOCK_TASK_MOVE_TO_PLACE_PREPARE:
        /* BlockArm 到达所选层粗定位姿态后，进入人工微调。 */
        break;

    case PLACE_BLOCK_TASK_MANUAL_ALIGN:
        /* 不自动前进，等待操作手调用 ConfirmRelease()。 */
        break;

    case PLACE_BLOCK_TASK_RELEASE:
        /* 等待真空确认释放，再发起 MoveToSafe()。 */
        break;

    case PLACE_BLOCK_TASK_MOVE_TO_SAFE:
        /* 等待 BlockArm 到达 Safe，随后进入 DONE。 */
        break;

    default:
        break;
}
```

## 3.7 内部数据

```c
static PlaceBlockTaskState_t place_block_task_state;
```

第一版不长期保存放置层级。层级已经由三个启动接口直接转换为对应的 `BlockArm` 动作。

---

# 4. BlockArm —— 双电机耦合机械臂

## 4.1 机械职责

```text
Motor A
+ Motor B
+ 耦合连杆
+ 侧向吸盘末端位置控制
```

`BlockArm` 负责归零、自动粗定位、人工末端微调和安全撤离。两个电机共同决定末端位置，不能拆成两个互不相关的 Mechanism。

## 4.2 公开状态

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

状态关系：

```text
UNHOMED → Home() → HOMING → READY

READY / REACHED / STOPPED
→ MoveToXXX()
→ MOVING
→ REACHED

按住人工微调键 → MOVING
松开人工微调键 → STOPPED

动作异常或超时 → FAULT
```

第一版不增加公开的 `MANUAL_ADJUSTING` 状态。当前 Task 是否处于 `MANUAL_ALIGN` 用于限制微调接口的使用时机。

## 4.3 微调方向

```c
typedef enum
{
    BLOCK_ARM_FINE_FORWARD,
    BLOCK_ARM_FINE_BACKWARD,
    BLOCK_ARM_FINE_UP,
    BLOCK_ARM_FINE_DOWN

} BlockArmFineAdjustDirection_t;
```

方向以机器人自身坐标为基准：

```text
FORWARD  → 末端朝目标区域移动
BACKWARD → 末端朝车体方向移动
UP       → 末端上移
DOWN     → 末端下移
```

## 4.4 对外接口

```c
/**
 * 保存两个电机依赖并初始化 BlockArm 软件状态。
 * 初始化完成后进入 UNHOMED，不自动执行归零。
 */
void BlockArm_Init(Motor_t *motor_a, Motor_t *motor_b);

/** 启动双电机耦合机构的机械归零过程。 */
void BlockArm_Home(void);

/** 中止当前 BlockArm 动作，并让两个电机受控停止、保持。 */
void BlockArm_Stop(void);

/** 两个电机共同移动到低位取块粗定位姿态。 */
void BlockArm_MoveToLowPickReady(void);

/** 两个电机共同移动到双层上方块的取块粗定位姿态。 */
void BlockArm_MoveToHighPickReady(void);

/** 两个电机共同移动到底层放块粗定位姿态。 */
void BlockArm_MoveToPlaceBottomReady(void);

/** 两个电机共同移动到第一层放块粗定位姿态。 */
void BlockArm_MoveToPlaceLevel1Ready(void);

/** 两个电机共同移动到第二层放块粗定位姿态。 */
void BlockArm_MoveToPlaceLevel2Ready(void);

/**
 * 发起到安全搬运姿态的自动动作。
 * 拾取后应先让方块脱离周围物块再收回；放置后应先撤开吸盘再收回。
 * 具体多阶段路径等待机械验证后在 BlockArm 内部实现。
 */
void BlockArm_MoveToSafe(void);

/**
 * 启动一个方向的人工末端微调。
 * 上层按住方向键时可以周期调用；重复调用不得反复重置动作。
 */
void BlockArm_StartFineAdjust(
    BlockArmFineAdjustDirection_t direction
);

/**
 * 停止人工微调，把两个电机当前反馈位置设置为保持目标。
 * 不等同于 Motor_Disable()，应继续提供机构保持力。
 */
void BlockArm_StopFineAdjust(void);

/** 返回当前 BlockArm 运行状态。 */
BlockArmState_t BlockArm_GetState(void);

/** 周期更新反馈、自动动作、人工微调、到位判断和 Fault。 */
void BlockArm_Process(void);
```

## 4.5 内部微调区域

不同粗定位姿态附近，同一个末端方向所需的双电机组合可能不同。因此 `BlockArm.c` 内部需要保存当前微调区域：

```c
typedef enum
{
    BLOCK_ARM_FINE_PROFILE_NONE,

    BLOCK_ARM_FINE_PROFILE_LOW_PICK,
    BLOCK_ARM_FINE_PROFILE_HIGH_PICK,

    BLOCK_ARM_FINE_PROFILE_PLACE_BOTTOM,
    BLOCK_ARM_FINE_PROFILE_PLACE_LEVEL1,
    BLOCK_ARM_FINE_PROFILE_PLACE_LEVEL2

} BlockArmFineAdjustProfile_t;
```

该类型只在 `block_arm.c` 内部使用，不对 Task 公开。

## 4.6 内部数据

```text
Motor_t *motor_a
Motor_t *motor_b

BlockArmState_t state

motor_a_current_position
motor_b_current_position

motor_a_target_position
motor_b_target_position

zero_a
zero_b

tolerance
reached_count

homing_timer
motion_timer

两个电机的软件行程限制

低位取块预备姿态的两电机目标
高位取块预备姿态的两电机目标

底层放块预备姿态的两电机目标
第一层放块预备姿态的两电机目标
第二层放块预备姿态的两电机目标

Safe 姿态的两电机目标

BlockArmFineAdjustProfile_t fine_adjust_profile
BlockArmFineAdjustDirection_t fine_adjust_direction
```

每次启动粗定位动作时，同时更新内部微调区域：

```text
MoveToLowPickReady()       → LOW_PICK
MoveToHighPickReady()      → HIGH_PICK
MoveToPlaceBottomReady()   → PLACE_BOTTOM
MoveToPlaceLevel1Ready()   → PLACE_LEVEL1
MoveToPlaceLevel2Ready()   → PLACE_LEVEL2
```

## 4.7 内部辅助函数

```c
/** 更新两个电机的当前位置反馈。 */
static void BlockArm_UpdateFeedback(void);

/**
 * 设置两个电机的自动位置目标并进入 MOVING。
 * profile 用于记录动作完成后允许使用哪一套微调组合。
 */
static void BlockArm_StartAutoMove(
    float motor_a_target,
    float motor_b_target,
    BlockArmFineAdjustProfile_t profile
);

/** 判断两个电机是否同时稳定进入目标误差范围。 */
static bool BlockArm_IsReached(void);

/**
 * 根据当前微调区域和微调方向，选择双电机低速组合并输出。
 * 第一版可使用 switch-case 和实测参数，不做在线复杂轨迹规划。
 */
static void BlockArm_ApplyFineAdjust(void);

/**
 * 读取两个电机当前反馈位置，并将其设为新的位置保持目标。
 * 用于松开微调按键、确认吸附或确认释放时稳定末端姿态。
 */
static void BlockArm_StopAndHold(void);
```

---

# 5. BlockVacuum —— 真空吸附机构

## 5.1 机械职责

```text
Grab
→ 建立真空
→ 确认吸住
→ 保持吸附

Release
→ 释放真空
→ 确认方块脱离
```

`BlockVacuum` 不关心机械臂的高度、微调方向和当前 Task 步骤。

## 5.2 状态

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
RELEASED → Grab() → GRABBING → GRABBED

GRABBED → Release() → RELEASING → RELEASED

动作异常或超时 → FAULT
```

## 5.3 对外接口

```c
/** 初始化真空机构的软件状态和底层控制依赖。 */
void BlockVacuum_Init(void);

/** 启动建立真空；函数返回不表示已经吸住方块。 */
void BlockVacuum_Grab(void);

/** 启动释放真空；函数返回不表示方块已经实际脱离。 */
void BlockVacuum_Release(void);

/** 返回真空机构当前状态。 */
BlockVacuumState_t BlockVacuum_GetState(void);

/** 周期推进吸附、释放、完成判断、超时和 Fault。 */
void BlockVacuum_Process(void);
```

## 5.4 内部数据

```text
BlockVacuumState_t state

真空泵 / 电磁阀控制状态

grab_timer
release_timer

grab_timeout
release_timeout

压力传感器反馈值
或
吸附 / 释放确认信号
```

完成判据：

```text
有压力反馈：
达到吸附阈值 → GRABBED
压力恢复或释放信号有效 → RELEASED

无压力反馈：
达到预设动作时间 → GRABBED / RELEASED
```

无压力反馈的计时方案只能作为第一版临时实现，不能真正证明方块已经吸住或完全脱离。

---

# 6. 上层输入调用约定

上层只产生操作意图：

```text
启动低位 / 高位取块
启动底层 / 第一层 / 第二层放块
前后上下人工微调
确认吸附
确认释放
```

对应调用：

```text
低位取块 → PickBlockTask_StartLowPick()
高位取块 → PickBlockTask_StartHighPick()

底层放块   → PlaceBlockTask_StartBottom()
第一层放块 → PlaceBlockTask_StartLevel1()
第二层放块 → PlaceBlockTask_StartLevel2()

按住微调方向 → BlockArm_StartFineAdjust(direction)
松开微调方向 → BlockArm_StopFineAdjust()

确认吸附 → PickBlockTask_ConfirmGrab()
确认释放 → PlaceBlockTask_ConfirmRelease()
```

人工微调只允许在以下情况使用：

```text
PickBlockTask == PICK_BLOCK_TASK_MANUAL_ALIGN
或
PlaceBlockTask == PLACE_BLOCK_TASK_MANUAL_ALIGN
```

第一版一次只运行一个方块 Task，不实现复杂资源管理模块。

---

# 7. 主循环调度

```c
while (1)
{
    /* Driver / 电机反馈更新 */

    BlockArm_Process();
    BlockVacuum_Process();

    PickBlockTask_Process();
    PlaceBlockTask_Process();

    /* Operator_Process(); 后续接入 */
}
```

推荐顺序：

```text
先更新执行器反馈
↓
Mechanism 根据最新反馈更新状态
↓
Task 读取 Mechanism 状态并推进流程
↓
Operator 根据最新输入发起新命令
```

所有 `Process()` 必须非阻塞，不使用长延时等待物理动作完成。

---

# 8. 启动条件与 Fault 传播

## 8.1 PickBlockTask 启动条件

```text
BlockArm 已完成 Homing
BlockArm 能接受新的自动运动命令
BlockVacuum == RELEASED
两个 Mechanism 均非 FAULT
当前没有另一个方块 Task 正在运行
```

## 8.2 PlaceBlockTask 启动条件

```text
BlockArm 已完成 Homing
BlockArm 能接受新的自动运动命令
BlockVacuum == GRABBED
两个 Mechanism 均非 FAULT
当前没有另一个方块 Task 正在运行
```

## 8.3 Fault 传播

```text
BlockArm 或 BlockVacuum 检测异常
↓
对应 Mechanism 执行自身安全处理并进入 FAULT
↓
当前 Task 检测到 Mechanism == FAULT
↓
Task → FAULT
↓
Task 不再发起后续动作
```

Task 不越过 Mechanism 直接操作 Motor、GPIO 或真空阀进行故障恢复。

---

# 9. 编码前仍需机械验证的参数

接口和状态机可以先按本方案编写，但以下内容暂时不能写死：

```text
两个电机的真实零位与 Home 顺序

LowPick / HighPick 的双电机目标

PlaceBottom / PlaceLevel1 / PlaceLevel2 的双电机目标

Safe 的双电机目标和完整撤离路径

每个粗定位区域中：
FORWARD / BACKWARD / UP / DOWN
对应的双电机速度、方向和允许范围

两个电机的位置容差和连续到位次数

自动动作与 Homing 的超时

真空吸附与释放的完成判据
```

必须特别验证：

```text
微调方向在局部范围内是否稳定
是否会接近死点、过中心或奇异位置
取块撤离是否会拖动相邻方块
放块撤离是否会碰撞或带偏已放方块
```

---

# 10. 推荐代码目录

```text
Mechanism/
├─ Inc/
│  ├─ block_arm.h
│  └─ block_vacuum.h
└─ Src/
   ├─ block_arm.c
   └─ block_vacuum.c

Task/
├─ Inc/
│  ├─ pick_block_task.h
│  └─ place_block_task.h
└─ Src/
   ├─ pick_block_task.c
   └─ place_block_task.c
```

头文件负责：

```text
公开 enum
公开类型
公开函数声明
必要的函数注释
```

源文件负责：

```text
内部 static 数据
状态机
内部辅助函数
具体电机和真空控制
```

---

# 11. 当前冻结结论

```text
1. 保留 PickBlockTask 和 PlaceBlockTask。

2. 两个 Task 共用 BlockArm 和 BlockVacuum。

3. Pick 分低位和高位两个自动粗定位姿态。

4. Place 分底层、第一层、第二层三个自动粗定位姿态。

5. 两个 Task 均在粗定位后进入 MANUAL_ALIGN。

6. 人工微调统一为末端前、后、上、下四方向。

7. 四方向微调由 BlockArm 内部协调两个电机完成。

8. Pick 由 ConfirmGrab() 触发吸附。

9. Place 由 ConfirmRelease() 触发释放。

10. Task 必须等待 BlockVacuum 的真实 GRABBED / RELEASED 状态。

11. 自动粗定位、人工微调和 Safe 撤离均由 BlockArm 负责。

12. 第一版采用 enum + switch-case + 非阻塞 Process() 实现。
```

本文可作为 `block_arm.c`、`block_vacuum.c`、`pick_block_task.c` 和 `place_block_task.c` 的第一版编码依据。
