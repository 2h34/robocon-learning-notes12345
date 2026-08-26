# 双机械臂 BlockArm 改造方案

## 1. 目标与边界

本方案用于两套**结构相同、非镜像安装**的机械臂。两只机械臂都具备归零、移动到安全姿态、取块姿态、放块姿态、位置保持和后续人工微调能力。

本次改造的目标不是增加一套电机驱动，而是让现有 `BlockArm` 同时保存并推进两只机械臂的独立状态。

本方案明确保持以下边界：

- 保留现有 `BlockArm.c` 的状态机和 `switch` 形式的固定姿态设置；
- 不建立姿态表；
- 不复制或重写 `DJmotor.c`、`ZDrive.c`、CAN 中断和定时器控制循环；
- 不把吸盘、电磁阀和双臂防碰撞调度塞入 `BlockArm`；
- 固定支持两只机械臂，使用左臂和右臂编号。

## 2. 当前单机械臂限制

当前 `BlockArm.c` 内部只有一份运行状态：

```c
static BlockArm_t block_arm;
```

这份状态中同时包含：绑定的两台电机指针、归零状态、零点偏移、当前位置、目标位置、到位计数、微调状态和故障状态。

因此，如果直接第二次调用现有 `BlockArm_Init()`：

```text
第一次初始化左臂
→ block_arm 保存左臂的电机和状态

第二次初始化右臂
→ 同一个 block_arm 被右臂覆盖
→ 左臂的电机绑定、零点和状态全部丢失
```

所以不能仅靠“多调用一次现有接口”实现双机械臂。

## 3. 改造后的模块关系

```text
DJmotor[] / Zmotor[]
    ├─ 左臂绑定的一台 DJI 与一台 ZDrive
    └─ 右臂绑定的一台 DJI 与一台 ZDrive

BlockArm 模块
    ├─ block_arm[BLOCK_ARM_LEFT]
    └─ block_arm[BLOCK_ARM_RIGHT]

上层任务
    ├─ 决定左臂执行取块、放块或回 Safe
    ├─ 决定右臂执行取块、放块或回 Safe
    └─ 决定两臂是否允许同时运动
```

电机层仍统一遍历所有 `DJmotor[]` 和 `Zmotor[]`。每只 `BlockArm` 只通过保存的指针操作属于自己的两台电机。

## 4. 机械臂编号

在 `Mechanism/Inc/BlockArm.h` 中增加：

```c
typedef enum
{
    BLOCK_ARM_LEFT = 0,
    BLOCK_ARM_RIGHT,
    BLOCK_ARM_COUNT
} BlockArmId_t;
```

含义如下：

```text
BLOCK_ARM_LEFT  ：左侧机械臂的状态和动作
BLOCK_ARM_RIGHT ：右侧机械臂的状态和动作
BLOCK_ARM_COUNT ：机械臂总数，只用于数组大小和编号检查
```

这里使用固定编号，而不是动态创建对象，原因是当前项目明确只有两只机械臂。这能保持模块内部结构私有，也能避免引入动态内存管理。

## 5. `BlockArm.c` 的核心改动

### 5.1 两份内部状态

保留 `BlockArm_t` 继续定义在 `BlockArm.c` 中，不必暴露结构体字段给其他模块。

将：

```c
static BlockArm_t block_arm;
```

替换为：

```c
static BlockArm_t block_arm[BLOCK_ARM_COUNT];
```

此后：

```text
block_arm[BLOCK_ARM_LEFT]  保存左臂完整状态
block_arm[BLOCK_ARM_RIGHT] 保存右臂完整状态
```

每只臂将独立拥有：

- `dji_motor` 与 `zdrive_motor`；
- `state`；
- `dji_zero_position` 与 `zdrive_zero_position`；
- `dji_current_position` 与 `zdrive_current_position`；
- `dji_target_position` 与 `zdrive_target_position`；
- 归零、到位、微调、停止和复位的其余状态字段。

### 5.2 取得指定机械臂实例

在 `BlockArm.c` 增加一个私有辅助函数：

```c
static BlockArm_t *BlockArm_GetInstance(BlockArmId_t arm_id)
{
    if ((uint32_t)arm_id >= (uint32_t)BLOCK_ARM_COUNT)
    {
        return NULL;
    }

    return &block_arm[arm_id];
}
```

所有公开接口先通过该函数取得目标机械臂。非法编号时直接返回，不操作任何电机。

### 5.3 内部函数传入当前实例

当前内部函数直接读写唯一的 `block_arm`。改造后改为传入当前状态指针：

```c
static bool BlockArm_DriversBound(const BlockArm_t *arm);
static void BlockArm_UpdateFeedback(BlockArm_t *arm);
static void BlockArm_ApplyPositionTargets(BlockArm_t *arm);
static void BlockArm_SetAutoTarget(BlockArm_t *arm, BlockArmTarget_t target);
static void BlockArm_StartAutoMove(BlockArm_t *arm, BlockArmTarget_t target);
static bool BlockArm_IsReached(BlockArm_t *arm);
static void BlockArm_ApplyFineAdjust(BlockArm_t *arm);
static void BlockArm_StopAndHold(BlockArm_t *arm);
```

函数体中将：

```c
block_arm.dji_target_position
```

替换为：

```c
arm->dji_target_position
```

这样保持现有逻辑不变，只是逻辑作用于被选中的左臂或右臂。

## 6. 对外接口修改

所有涉及单只机械臂动作的接口增加第一个参数 `BlockArmId_t arm_id`。

```c
void BlockArm_Init(BlockArmId_t arm_id,
                   DJMotor *dji_motor,
                   Zdrive *zdrive_motor);

void BlockArm_Home(BlockArmId_t arm_id);
void BlockArm_Stop(BlockArmId_t arm_id);

void BlockArm_MoveToLowPickReady(BlockArmId_t arm_id);
void BlockArm_MoveToHighPickReady(BlockArmId_t arm_id);
void BlockArm_MoveToPlaceBottomReady(BlockArmId_t arm_id);
void BlockArm_MoveToPlaceLevel1Ready(BlockArmId_t arm_id);
void BlockArm_MoveToPlaceLevel2Ready(BlockArmId_t arm_id);
void BlockArm_MoveToSafe(BlockArmId_t arm_id);

void BlockArm_StartFineAdjust(BlockArmId_t arm_id,
                              BlockArmFineAdjustDirection_t direction);
void BlockArm_StopFineAdjust(BlockArmId_t arm_id);
void BlockArm_Reset(BlockArmId_t arm_id);

BlockArmState_t BlockArm_GetState(BlockArmId_t arm_id);
void BlockArm_Process(BlockArmId_t arm_id);
```

以 `BlockArm_Home()` 为例，其改造逻辑为：

```c
void BlockArm_Home(BlockArmId_t arm_id)
{
    BlockArm_t *arm = BlockArm_GetInstance(arm_id);
    if (arm == NULL)
    {
        return;
    }

    /* 保留原有状态判断和归零启动逻辑，
       但全部访问 arm->xxx。 */
}
```

## 7. 固定姿态保持当前 `switch` 写法

不建立姿态表，继续使用现有的 `BlockArm_SetAutoTarget()` 与 `switch`。

改造后的形式：

```c
static void BlockArm_SetAutoTarget(BlockArm_t *arm,
                                   BlockArmTarget_t target)
{
    switch (target)
    {
        case BLOCK_ARM_TARGET_SAFE:
            arm->fine_adjust_profile = BLOCK_ARM_FINE_PROFILE_NONE;
            arm->dji_target_position = /* Safe 的 DJI 输出轴角度 */;
            arm->zdrive_target_position = /* Safe 的 ZDrive 输出轴角度 */;
            break;

        case BLOCK_ARM_TARGET_LOW_PICK_READY:
            arm->fine_adjust_profile = BLOCK_ARM_FINE_PROFILE_LOW_PICK;
            arm->dji_target_position = /* 低位取块 DJI 输出轴角度 */;
            arm->zdrive_target_position = /* 低位取块 ZDrive 输出轴角度 */;
            break;

        /* 其他固定姿态沿用同样写法。 */
    }
}
```

两只机械臂非镜像且使用相同机构坐标定义时，左右臂共用这一个 `switch` 中的角度值。

这些角度都必须是：

```text
相对于本臂归零点的输出轴角度
```

不得在这里手动加 `dji_zero_position` 或 `zdrive_zero_position`。这两个偏移仍由 `BlockArm_ApplyPositionTargets()` 自动加回。

## 8. 初始化与周期调用

在实际应用初始化位置绑定四台不同的电机。电机数组下标只作示例，必须替换为你的真实 CAN ID / 数组映射。

```c
BlockArm_Init(BLOCK_ARM_LEFT,
              &DJmotor[LEFT_DJI_INDEX],
              &Zmotor[LEFT_ZDRIVE_INDEX]);

BlockArm_Init(BLOCK_ARM_RIGHT,
              &DJmotor[RIGHT_DJI_INDEX],
              &Zmotor[RIGHT_ZDRIVE_INDEX]);
```

要求：

- 左右臂不能绑定同一台 `DJmotor[]`；
- 左右臂不能绑定同一台 `Zmotor[]`；
- 四台物理电机的 CAN ID 与数组下标必须逐一确认。

在周期任务中分别推进两只机械臂：

```c
BlockArm_Process(BLOCK_ARM_LEFT);
BlockArm_Process(BLOCK_ARM_RIGHT);
```

当前工作区中未找到 `BlockArm_Init()` 与 `BlockArm_Process()` 的应用调用点。因此在实际改造时，需要把上述初始化和周期调用接入你实际使用的任务或循环；本方案不假定其具体文件位置。

## 9. 归零与参考系在双臂下的行为

每只机械臂独立归零：

```text
左臂 Home
→ 左臂自己的 DJI / ZDrive 归零
→ 左臂保存自己的两个零点偏移

右臂 Home
→ 右臂自己的 DJI / ZDrive 归零
→ 右臂保存自己的两个零点偏移
```

即使两臂同时归零，也不会交换零点，因为两份 `BlockArm_t` 保存的是不同电机指针和不同偏移字段。

每只臂仍遵守现有参考系公式：

```text
机构当前位置 = 驱动原始反馈 − 本臂零点偏移
驱动原始目标 = 机构目标 + 本臂零点偏移
```

DJI 驱动会在归零完成时清自身位置，因此其偏移通常为 `0°`；ZDrive 保留原始位置，由各自的 `zdrive_zero_position` 完成软件偏移。

## 10. 不需要修改的模块

### 10.1 电机驱动层

`Motor/Src/DJmotor.c` 和 `Motor/Src/ZDrive.c` 已经以数组方式管理多台电机。双臂改造不需要复制任何电机控制、堵转判定或位置控制代码。

只有在实际电机数量超过当前配置数量，或某台电机的归零方向不同，才需要额外检查电机配置。

### 10.2 中断与 CAN

定时器中断已经统一运行 DJI 与 ZDrive 电机控制。CAN 接收层也按电机 ID 将反馈写回相应数组元素。只要电机 ID、总线和数组绑定正确，双臂不需要增加第二套中断。

### 10.3 吸盘与双臂调度

`BlockArm` 保持“关节归零、姿态移动、位置保持”的职责。吸盘开关、取放步骤、两臂互锁和防碰撞应留在上层任务逻辑。

例如上层可以做：

```text
左臂去低位取块
右臂去二层放块
等待右臂离开共享空间
左臂再进入放块姿态
```

但单个 `BlockArm` 不需要知道另一只机械臂要做什么。

## 11. 实施顺序

1. 在 `BlockArm.h` 增加 `BlockArmId_t`，并修改公开接口声明。
2. 在 `BlockArm.c` 将单个 `block_arm` 改成两个元素的数组。
3. 增加私有的实例取得函数，并为所有内部函数增加 `BlockArm_t *arm` 参数。
4. 将所有 `block_arm.xxx` 改为 `arm->xxx`，保持状态机判断顺序不变。
5. 修改应用调用处：分别初始化左臂、右臂，并周期调用两个 `BlockArm_Process()`。
6. 先只测试左臂归零，再只测试右臂归零。
7. 测试两臂同时归零，确认完成信号、零点偏移和位置保持互不影响。
8. 再填 Safe 与各取放姿态，最后实现任务层的双臂协作。

## 12. 验证清单

### 软件检查

- 编译时，所有旧的无参数 `BlockArm_*` 调用都已改为传入 `BlockArmId_t`；
- 两个 `BlockArm_Init()` 分别绑定不同电机；
- 两个 `BlockArm_Process()` 都被周期调用；
- 任一无效 `arm_id` 不会访问数组越界；
- 左臂调用位置接口时，不会改写右臂的目标和状态。

### 实机验证

1. 左臂单独归零：确认只左臂运动、完成后左臂机构坐标为零。
2. 右臂单独归零：确认只右臂运动、完成后右臂机构坐标为零。
3. 同时归零：确认两臂均按正确方向运动并分别完成。
4. 左臂保持 Safe、右臂移动：确认左臂目标没有被右臂动作覆盖。
5. 左右臂分别执行相同固定姿态：确认两臂机构运动方向和姿态一致。

## 13. 当前不纳入本次改造的待办

以下工作仍可按现有计划后续完成，不应混入本次“单臂改双臂”改造：

- Safe 姿态的实际角度标定；
- `BlockArm_IsReached()` 的到位判断；
- 归零中止和超时的安全输出；
- 微调的其余方向与双电机组合；
- 吸盘控制与取放任务流程；
- 两臂共享空间时的防碰撞策略。
