# 蓝牙输入与 Operator 上层控制设计（第一版）

> 日期：2026-08-24  
> 文档定位：在既有 `PickBlockTask / PlaceBlockTask / BlockArm / BlockVacuum` 方案之上，新增蓝牙输入、协议解析、业务映射与 Operator 调度层。  
> 本文只记录新增内容，不重复已有 Task / Mechanism 设计。

---

# 0. 新增控制链

```text
手机 / 遥控器
↓ Bluetooth
蓝牙模块
↓ UART
BluetoothProtocol
↓
BluetoothRxData
↓
OperatorMapping
↓
OperatorCommand
↓
Operator
↓
现有 Task / Mechanism 对外接口
```

新增上层的目标是：

```text
原始通信数据
→ 合法协议数据
→ 有业务意义的操作意图
→ 正确调用现有控制接口
```

---

# 1. BluetoothProtocol

## 1.1 职责

`BluetoothProtocol` 只负责通信协议本身：

```text
接收 UART 原始字节
识别完整 Frame
检查 Header / Checksum / Tail
提取 Payload
解析 Bool / U8 / U16 / Short 等协议字段
对外提供最新合法接收数据
```

该模块不理解：

```text
LowPick
FineUp
ConfirmGrab
PickBlockTask
BlockArm
```

等机器人业务含义。

---

## 1.2 当前 Frame 模型

课程目前确认的基本结构：

```text
Header
+
Payload
+
Checksum
+
Tail
```

其中：

```text
Header = 0xA5
Tail   = 0x5A
```

Checksum：

```text
相关字节累加
↓
保留低 8 位
```

当前仍需后续依据老师源码确认：

```text
1. Checksum 的实际覆盖范围
2. 是否存在独立 Length 字段
3. U16 / Short 的字节序
4. 实际串口波特率
```

---

## 1.3 Parser 状态机

第一版概念状态：

```c
typedef enum
{
    BT_PARSE_WAIT_HEADER,
    BT_PARSE_RECEIVE_PAYLOAD,
    BT_PARSE_CHECK_CHECKSUM,
    BT_PARSE_CHECK_TAIL

} BluetoothParseState_t;
```

流程：

```text
WAIT_HEADER
↓ 收到 0xA5

RECEIVE_PAYLOAD
↓ Payload 收满

CHECK_CHECKSUM
↓ 校验通过

CHECK_TAIL
↓ 收到 0x5A

FRAME VALID
↓
重置 Parser
↓
WAIT_HEADER
```

关键边界：

```text
收到一个 UART byte
≠
收到一帧合法命令

收到一帧合法数据
≠
已经执行机器人动作
```

---

# 2. BluetoothRxData

`BluetoothRxData` 表示已经完成协议解包的数据，但仍然没有机器人业务意义。

第一版当前主要使用 Bool：

```c
typedef struct
{
    bool bool_data[BT_RX_BOOL_NUM];

} BluetoothRxData_t;
```

如果后续实际协议配置包含：

```text
U8
U16
Short
```

再按真实 `config` 扩展对应字段。

---

# 3. Bool 位打包

当前课程使用：

```text
LSB First
```

即：

```text
bool[0] → 第 1 个 Bool 字节 bit0
...
bool[7] → 第 1 个 Bool 字节 bit7

bool[8] → 第 2 个 Bool 字节 bit0
...
```

当前 11 个 Bool 共占：

```text
2 byte
```

Bool 所需字节数：

```c
bool_bytes = (BOOL_NUM + 7) / 8;
```

单个 Bool 提取：

```c
byte_index = index / 8;
bit_index  = index % 8;

bool_value =
    (buffer[byte_index] >> bit_index) & 0x01;
```

这些位操作全部属于协议解包层，不应出现在 Operator 或 Task 中。

---

# 4. 当前协议字段映射

第一版暂定：

```text
bool[0]  → LowPick
bool[1]  → HighPick

bool[2]  → PlaceBottom
bool[3]  → PlaceLevel1
bool[4]  → PlaceLevel2

bool[5]  → ConfirmGrab
bool[6]  → ConfirmRelease

bool[7]  → FineForward
bool[8]  → FineBackward
bool[9]  → FineUp
bool[10] → FineDown
```

该索引目前属于第一版设计约定。

最终必须满足：

```text
蓝牙上位机 TX config
        =
STM32 Bluetooth RX config
```

否则即使 UART 与 Parser 完全正常，也会发生业务字段错位。

---

# 5. OperatorCommand

`OperatorCommand` 是经过 Mapping 后形成的“操作手输入快照”。

```c
typedef struct
{
    bool low_pick;
    bool high_pick;

    bool place_bottom;
    bool place_level1;
    bool place_level2;

    bool confirm_grab;
    bool confirm_release;

    bool fine_forward;
    bool fine_backward;
    bool fine_up;
    bool fine_down;

} OperatorCommand_t;
```

它表示：

```text
“操作手当前想做什么”
```

不表示：

```text
“机器人当前执行到什么状态”
```

机器人状态仍由已有 Task / Mechanism 自己维护。

---

# 6. OperatorMapping

## 6.1 职责

`OperatorMapping` 负责：

```text
协议字段
↓
业务字段
```

第一版主要是一一映射：

```c
cmd->low_pick        = rx->bool_data[0];
cmd->high_pick       = rx->bool_data[1];

cmd->place_bottom    = rx->bool_data[2];
cmd->place_level1    = rx->bool_data[3];
cmd->place_level2    = rx->bool_data[4];

cmd->confirm_grab    = rx->bool_data[5];
cmd->confirm_release = rx->bool_data[6];

cmd->fine_forward    = rx->bool_data[7];
cmd->fine_backward   = rx->bool_data[8];
cmd->fine_up         = rx->bool_data[9];
cmd->fine_down       = rx->bool_data[10];
```

未来 Mapping 可以承担：

```text
多对一逻辑
单位换算
摇杆缩放
模式组合
```

但不直接执行 Task / Mechanism。

---

# 7. Operator 输入语义

当前输入分为两类。

## 7.1 一次性命令

以下操作只在：

```text
0 → 1
```

时触发一次：

```text
LowPick
HighPick

PlaceBottom
PlaceLevel1
PlaceLevel2

ConfirmGrab
ConfirmRelease
```

Operator 内部需要保存：

```c
static OperatorCommand_t current_cmd;
static OperatorCommand_t previous_cmd;
```

上升沿判断：

```c
if (current_cmd.low_pick &&
    !previous_cmd.low_pick)
{
    /* 调用已有 LowPick 启动接口 */
}
```

这样即使遥控器连续多帧保持：

```text
1 1 1 1 1
```

也只在第一次：

```text
0 → 1
```

时启动一次。

---

## 7.2 持续命令

人工微调：

```text
FineForward
FineBackward
FineUp
FineDown
```

采用当前电平语义：

```text
1 → 操作手仍在按住
0 → 操作手已经松开
```

因此：

```text
按住
→ 持续调用 / 保持对应 FineAdjust 意图

松开
→ 检测 1 → 0
→ 调用 StopFineAdjust()
```

Operator 在这里需要同时处理：

```text
Level
+
Falling Edge
```

而不是像一次性命令那样只检测 Rising Edge。

---

# 8. Operator 调度职责

Operator 只调用现有公开接口。

概念逻辑：

```text
LowPick Rising Edge
↓
启动低位取块 Task

HighPick Rising Edge
↓
启动高位取块 Task

PlaceXXX Rising Edge
↓
启动对应放置 Task

ConfirmGrab Rising Edge
↓
提交吸附确认

ConfirmRelease Rising Edge
↓
提交释放确认

FineXXX == 1
↓
请求对应方向人工微调

FineXXX 1 → 0
↓
停止微调并保持
```

Operator 可以根据当前 Task 状态进行上层过滤，但：

```text
Task / Mechanism 自己仍必须保留内部状态检查
```

即：

```text
Operator
→ 调度层保护

Task / Mechanism
→ 自身状态边界最终负责人
```

---

# 9. 推荐模块划分

```text
Communication/
├─ Inc/
│  ├─ bluetooth_protocol.h
│  └─ bluetooth_config.h
└─ Src/
   └─ bluetooth_protocol.c

Operator/
├─ Inc/
│  ├─ operator.h
│  └─ operator_mapping.h
└─ Src/
   ├─ operator.c
   └─ operator_mapping.c
```

职责：

```text
bluetooth_config.h
→ 协议字段数量、Header、Tail 等配置

bluetooth_protocol.c/.h
→ Parser + Payload 解包

operator_mapping.c/.h
→ BluetoothRxData → OperatorCommand

operator.c/.h
→ Edge / Level 判断 + 上层调度
```

第一版暂不单独创建：

```text
bluetooth_deserializer.c
```

因为当前解包规模较小，可先留在 `bluetooth_protocol.c` 中。

---

# 10. 对主循环的新增接入

原有 Mechanism / Task 调度顺序保持不变，只新增 Operator：

```c
while (1)
{
    /* Driver / feedback */

    /* 现有 Mechanism Process */

    /* 现有 Task Process */

    Operator_Process();
}
```

UART / DMA / Callback 负责把接收到的原始数据送入 BluetoothProtocol。

推荐总体时序：

```text
UART 数据更新
↓
BluetoothProtocol 得到最新合法输入
↓
Mapping 更新 OperatorCommand
↓
Mechanism 根据最新执行器反馈更新
↓
Task 推进现有流程
↓
Operator 根据最新操作输入发起下一条上层命令
```

所有处理保持非阻塞。

---

# 11. 当前冻结结论

```text
1. 新增 BluetoothProtocol、OperatorMapping、Operator 三层。

2. BluetoothProtocol 只处理通信协议，不理解机器人业务。

3. OperatorMapping 只负责协议字段到业务操作意图的解释。

4. Operator 负责边沿、持续状态判断以及现有接口调度。

5. 第一版输入暂定 11 个 Bool，占 2 byte。

6. 一次性 Task / Confirm 命令采用 Rising Edge。

7. 四方向人工微调采用 Level 控制，松开时检测 Falling Edge。

8. BluetoothProtocol 不直接依赖 Task / Mechanism。

9. Operator 不越过现有 Task / Mechanism 直接操作 Motor 或 Vacuum Driver。

10. 最终 bool 索引、帧长度、Checksum 覆盖范围、字节序和波特率必须依据老师实际工程重新核对后冻结。
```

---

# 12. 后续待确认项

```text
老师实际蓝牙工程源码

蓝牙上位机 TX 字段顺序

STM32 RX config

真实 Frame Length 规则

Checksum 计算范围

U16 / Short 大小端

UART 实际波特率

UART 使用逐字节中断还是 DMA 的最终接入方式

有效 Frame 到 OperatorCommand 的更新时机

通信超时后 Operator 输入是否主动清零
```

其中最后一项尤其需要后续补充：

```text
如果遥控器突然断开，
最后一帧 FineUp = 1 不能无限保持。
```

因此正式工程还需要增加：

```text
通信新鲜度 / 超时保护
```

但暂不在当前第一版中展开实现。
