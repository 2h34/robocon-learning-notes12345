# 蓝牙输入与 Operator 上层控制设计（第二版）

> 日期：2026-08-24  
> 文档定位：在既有 `PickBlockTask / PlaceBlockTask / BlockArm / BlockVacuum` 方案之上，设计蓝牙输入、协议解析、业务映射与 Operator 调度层。  
> 版本说明：第二版在第一版基础上补充 `config → Payload` 布局、Parser / Deserializer 概念边界、UART Callback → Protocol 执行边界、RX/TX 对齐规则、Valid Frame → Mapping 更新时间，并将通信 Fail-safe 等内容调整为可选升级项。  
> 本文只描述新增上层方案，不重复既有 Task / Mechanism 内部设计。

---

# 0. 设计范围

本方案新增以下控制链：

```text
手机 / 遥控器
↓ Bluetooth
蓝牙模块
↓ UART
STM32 UART / HAL
↓
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

本方案当前负责：

```text
UART 原始数据接入
Frame 识别与校验
Payload 解包
协议字段到业务字段的映射
一次性命令与持续输入的区分
Operator 对现有 Task / Mechanism 的调度
```

第一版当前不负责：

```text
通信超时与 Fail-safe
复杂输入仲裁
双缓冲 / Queue
通用 Command Scheduler
自动重连状态机
上位机遥测回传
```

---

# 1. 总体分层原则

## 1.1 UART / HAL 层

负责：

```text
接收蓝牙模块通过 UART 发送的原始 byte
```

UART 层不理解：

```text
0xA5 是包头
bool[0] 是 LowPick
某个 Short 是速度
```

---

## 1.2 BluetoothProtocol

负责通信协议本身：

```text
识别完整 Frame
检查 Header
接收 Payload
Checksum 校验
检查 Tail
Payload 解包
生成 BluetoothRxData
```

该层只理解：

```text
bool[0]
u8[0]
u16[0]
short[0]
```

等协议字段，不理解机器人业务。

---

## 1.3 OperatorMapping

负责：

```text
协议字段
↓
业务操作字段
```

例如：

```text
bool[0]
↓
low_pick
```

Mapping 负责“解释”，不负责执行 Task。

---

## 1.4 Operator

负责：

```text
读取 OperatorCommand
判断 Rising Edge / Falling Edge / Level
结合当前 Task / Mechanism 状态
调用现有公开接口
```

最终职责关系：

```text
BluetoothProtocol = 解析协议
OperatorMapping   = 解释业务意义
Operator          = 调度操作意图
Task / Mechanism  = 执行既有流程
```

---

# 2. 推荐模块划分

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

第一版暂不额外拆分：

```text
bluetooth_deserializer.c
```

原因：

```text
Parser
+
Deserializer
```

虽然概念职责不同，但当前协议规模较小，可以先统一封装在：

```text
bluetooth_protocol.c
```

内部，避免为模块化而机械增加文件数量。

---

# 3. bluetooth_config.h —— 协议配置

## 3.1 职责

`bluetooth_config.h` 描述：

```text
通信协议“长什么样”
```

而不是：

```text
机器人“这些字段是什么意思”
```

当前概念配置：

```c
#define BT_FRAME_HEADER      0xA5
#define BT_FRAME_TAIL        0x5A

#define BT_RX_BOOL_NUM       11
#define BT_RX_U8_NUM          0
#define BT_RX_U16_NUM         0
#define BT_RX_SHORT_NUM       0
```

具体宏名最终以老师实际工程为准。

---

## 3.2 config 与 Mapping 的边界

```text
config
↓
决定：
Bool 有几个
U8 有几个
U16 有几个
Short 有几个
Payload 如何排列
Payload 有多长
```

而：

```text
Mapping
↓
决定：
bool[0] 是 LowPick
bool[9] 是 FineUp
```

因此：

```text
config = 协议格式
Mapping = 协议字段的业务意义
```

不能混为一层。

---

# 4. Payload 大小计算

## 4.1 Bool 区

Bool 按 bit 打包：

```text
8 个 Bool
→ 1 byte
```

Bool 所需字节数：

```c
#define BT_RX_BOOL_BYTES \
    ((BT_RX_BOOL_NUM + 7) / 8)
```

当前：

```text
BT_RX_BOOL_NUM = 11
```

因此：

```text
BT_RX_BOOL_BYTES = 2
```

---

## 4.2 其他类型

基本大小：

```text
U8
→ 1 byte

U16
→ 2 byte

Short / int16_t
→ 2 byte
```

若未来配置：

```text
Bool × 11
U8 × 2
U16 × 1
Short × 2
```

则：

```text
Bool  → 2 byte
U8    → 2 byte
U16   → 2 byte
Short → 4 byte

Payload Size = 10 byte
```

因此：

```text
config
↓
决定 Payload Size
↓
Parser 知道应接收多少 Payload
↓
Deserializer 知道如何切割 Payload
```

---

# 5. Frame 模型

当前课程确认的基本模型：

```text
Header
+
Payload
+
Checksum
+
Tail
```

目前已知：

```text
Header = 0xA5
Tail   = 0x5A
```

Checksum 基本思想：

```text
相关字节累加
↓
保留低 8 位
```

当前仍需老师真实源码 / PPT 确认：

```text
1. 是否存在独立 Length 字段
2. 如果存在，Length 的位置与含义
3. Checksum 实际覆盖哪些 byte
```

在这些信息确认前，不在本方案中写死。

---

# 6. Parser 与 Deserializer 的概念边界

## 6.1 Parser

Parser 负责：

```text
byte stream
↓
合法 Frame / Payload
```

概念状态：

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

---

## 6.2 Deserializer

Deserializer 负责：

```text
合法 Payload
↓
bool[] / u8[] / u16[] / short[]
```

即：

```text
Parser
回答：
“这是不是一帧合法数据？”

Deserializer
回答：
“这帧 Payload 内每一段是什么数据？”
```

当前两者可以统一实现在：

```text
bluetooth_protocol.c
```

中，但概念职责必须保持分离。

---

# 7. Bool 位打包与解包

当前协议采用：

```text
LSB First
```

因此：

```text
bool[0] → 第 1 个 Bool byte 的 bit0
...
bool[7] → 第 1 个 Bool byte 的 bit7

bool[8] → 第 2 个 Bool byte 的 bit0
...
```

单个 Bool 定位：

```c
byte_index = index / 8;
bit_index  = index % 8;
```

提取：

```c
bool_value =
    (buffer[byte_index] >> bit_index) & 0x01;
```

例如：

```text
bool[10]
```

对应：

```text
buffer[1] 的 bit2
```

这些位操作全部属于协议解包层，不应出现在 Operator / Task / Mechanism 中。

---

# 8. 多字节数据边界说明

当前需要明确：

```text
Bool 的 LSB First
≠
U16 / Short 一定采用 Little Endian
```

LSB First 描述：

```text
Bool bit 在 byte 内如何排列
```

Endianness 描述：

```text
一个多字节整数拆成多个 byte 后如何排序
```

因此：

```text
U16 / Short 的实际字节序
```

必须等待老师源码或协议文档确认。

---

# 9. Buffer 名称与职责

实际工程中建议区分：

```text
UART RX Buffer
→ HAL / DMA 接收原始 UART 数据

Frame Buffer
→ Parser 临时保存当前 Frame

Payload Buffer
→ 去除 Header / Checksum / Tail 后的数据段
```

不要仅因为变量都可能叫 `buffer`，就在概念上把它们视作同一层数据。

---

# 10. BluetoothRxData

`BluetoothRxData` 表示：

```text
已经完成协议解包
但还没有业务语义的数据
```

第一版：

```c
typedef struct
{
    bool bool_data[BT_RX_BOOL_NUM];

} BluetoothRxData_t;
```

若后续实际协议配置包含：

```text
U8
U16
Short
```

再根据真实 config 扩展。

---

# 11. 当前协议字段映射

当前第一版暂定：

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

该索引是当前设计约定。

最终必须与：

```text
蓝牙上位机 TX config
```

完全对齐后才能冻结。

---

# 12. RX / TX 对齐规则

UART 硬件方向：

```text
蓝牙模块 UART_TX
↓
STM32 UART_RX

蓝牙模块 UART_RX
↑
STM32 UART_TX
```

当前遥控命令方向：

```text
上位机 TX config
↓
STM32 RX config
```

双方必须同时对齐：

```text
数据类型
字段数量
字段排列顺序
字段约定
```

数量一致并不代表协议一定正确。

例如：

```text
上位机：
bool[3] = PlaceLevel1

STM32 Mapping：
bool[3] = PlaceLevel2
```

则：

```text
UART
Checksum
Parser
Deserializer
```

都可能完全正常，但最终业务动作错误。

该问题属于：

```text
协议字段 ↔ 业务 Mapping 对应错误
```

---

# 13. OperatorCommand

经过 Mapping 后形成：

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

`OperatorCommand` 表示：

```text
操作手当前想做什么
```

不表示：

```text
机器人当前执行到什么状态
```

Task / Mechanism 状态仍由已有模块维护。

---

# 14. OperatorMapping

## 14.1 职责

```text
BluetoothRxData
↓
OperatorCommand
```

第一版：

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

未来 Mapping 可以扩展：

```text
多对一逻辑
单位换算
模式组合
摇杆缩放
```

但 Mapping 不直接执行 Task / Mechanism。

---

# 15. Valid Frame → Mapping 的更新时间

OperatorCommand 不应随着每个 UART byte 更新。

正确边界：

```text
UART byte
↓
Parser
↓
Frame 尚未完成
→ 不更新业务输入
```

只有：

```text
Header 正确
Payload 完整
Checksum 正确
Tail 正确
↓
FRAME VALID
```

后才：

```text
更新 BluetoothRxData
↓
标记 new_frame / frame_ready
↓
执行 Mapping
↓
生成新的 OperatorCommand
```

概念流程：

```text
Parser 验证合法 Frame
↓
BluetoothRxData 更新
↓
new_frame_ready = true
↓
OperatorMapping
↓
OperatorCommand 更新
```

这样可以保证 Operator 只处理：

```text
完整、通过校验的通信输入
```

---

# 16. Operator 输入语义

当前输入分为：

```text
一次性命令
+
持续命令
```

---

## 16.1 Rising Edge —— 一次性命令

以下命令只在：

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

Operator 内部：

```c
static OperatorCommand_t current_cmd;
static OperatorCommand_t previous_cmd;
```

上升沿：

```c
if (current_cmd.low_pick &&
    !previous_cmd.low_pick)
{
    /* 调用已有 LowPick 启动接口 */
}
```

这样遥控器连续发送：

```text
1 1 1 1 1
```

不会重复启动同一 Task。

---

## 16.2 Level / Falling Edge —— 持续微调

人工微调：

```text
FineForward
FineBackward
FineUp
FineDown
```

采用：

```text
1
→ 当前仍然按住

0
→ 当前已经松开
```

因此：

```text
Level = 1
→ 持续保持 FineAdjust 意图

1 → 0
→ StopFineAdjust()
```

当前方案利用已有 `BlockArm_StartFineAdjust()` / `BlockArm_StopFineAdjust()` 接口，不由 Operator 直接控制双电机。

---

# 17. Operator 调度职责

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
停止人工微调
```

Operator 可以根据当前 Task 状态做上层过滤，但：

```text
Task / Mechanism
仍必须保留自身状态检查
```

职责关系：

```text
Operator
→ 调度层保护

Task / Mechanism
→ 自身状态边界最终负责人
```

---

# 18. UART Callback → Protocol 执行边界

## 18.1 基本接入

若使用逐字节中断：

```text
UART 收到 1 byte
↓
HAL Callback
↓
BluetoothProtocol_InputByte(byte)
↓
Parser
```

Callback 不负责：

```text
OperatorMapping
Operator
Task
Mechanism
```

---

## 18.2 执行上下文说明

如果：

```c
HAL_UART_RxCpltCallback(...)
{
    BluetoothProtocol_InputByte(byte);
}
```

那么：

```text
BluetoothProtocol_InputByte()
```

仍然运行在当前 Callback / 中断处理上下文中。

因此：

```text
函数封装
≠
执行上下文改变
```

Callback 中适合：

```text
短小
确定
可快速结束
```

的通信接收处理。

---

# 19. UART Interrupt 与 DMA 的关系

## 19.1 中断方式

```text
UART
↓
1 byte
↓
Callback
↓
BluetoothProtocol_InputByte()
```

---

## 19.2 DMA 方式

```text
UART
↓
DMA 搬运多个 byte
↓
Callback / Event
↓
逐 byte 喂给 BluetoothProtocol
```

因此：

```text
Interrupt / DMA
```

改变的是：

```text
UART 数据如何到达 BluetoothProtocol
```

而不应改变：

```text
Parser
Deserializer
Mapping
Operator
Task
Mechanism
```

---

# 20. 主循环接入

原有 Task / Mechanism 调度保持不变。

概念上：

```c
while (1)
{
    /* Driver / feedback */

    /* Mechanism Process */

    /* Task Process */

    Operator_Process();
}
```

推荐总体顺序：

```text
UART / Bluetooth 更新最新合法输入
↓
Mechanism 根据最新执行器反馈更新
↓
Task 推进现有流程
↓
Operator 根据最新操作输入发起新的上层命令
```

所有处理保持非阻塞。

---

# 21. 完整 LowPick 数据链

```text
操作手按 LowPick
↓
上位机：
bool[0] = 1
↓
Bool 打包：
01 00
↓
组成 Frame：
A5 | 01 00 | CK | 5A
↓
蓝牙模块 UART_TX
↓
STM32 UART_RX
↓
Callback
↓
BluetoothProtocol
↓
Parser 验证 Frame
↓
Deserializer
↓
bool_data[0] = true
↓
OperatorMapping
↓
operator_cmd.low_pick = true
↓
Operator 检测 Rising Edge
↓
调用已有 LowPick Task 启动接口
↓
进入既有 Task / Mechanism 流程
```

通信层到这里结束。

后续：

```text
BlockArm 是否 REACHED
BlockVacuum 是否 GRABBED
Task 是否 DONE
```

仍由已有 Task / Mechanism 自己负责。

---

# 22. 编译期裁剪（可选代码优化）

若：

```c
#define BT_RX_U8_NUM 0
```

可以使用：

```c
#if BT_RX_U8_NUM > 0

/* U8 解析代码 */

#endif
```

在编译前裁掉当前完全不需要的协议功能。

该内容属于：

```text
可选代码优化
```

不是当前第一版核心逻辑。

---

# 23. 可选升级项

以下内容本版不实现，仅作为后续升级方向记录：

```text
通信超时 / Communication Watchdog

Stale Command 处理

通信失联时持续人工控制自动停止

重连后第一帧同步，不立即产生 Rising Edge

双缓冲 / Queue

Callback 与主循环之间更严格的数据一致性保护

自动 Task 在通信失联后的继续 / 中止策略

上位机遥测回传

更完整的通信状态机
```

是否加入正式工程，等待：

```text
实际安全需求
机器人运动风险
通信频率
实物测试结果
```

后再决定。

---

# 24. 待冻结：输入冲突策略

当前还未正式规定：

```text
同一帧出现多个互斥输入
```

时如何处理。

例如：

```text
LowPick = 1
HighPick = 1
```

或：

```text
FineUp = 1
FineDown = 1
```

第一版可暂时依赖：

```text
Task / Mechanism 自身状态检查
```

阻止部分非法操作。

但后续建议明确：

```text
互斥输入优先级
或
互斥输入同时有效时全部忽略
```

该策略等待遥控器实际交互方式和机械行为确认后冻结。

---

# 25. 当前待老师源码 / PPT 确认

```text
1. 实际 UART Baud Rate

2. Frame 是否存在独立 Length 字段

3. Length 的实际位置和含义

4. Checksum 的实际覆盖范围

5. U16 / Short 的实际 Endianness

6. 老师真实 BluetoothProtocol API

7. config 文件的真实结构

8. 最终使用 UART IT 还是 DMA

9. 蓝牙上位机真实 TX 字段排列

10. Frame 完成后的实际数据交接方式
```

当前设计不对这些未确认项作硬性假设。

---

# 26. 第二版冻结结论

```text
1. 保留 BluetoothProtocol → BluetoothRxData → OperatorMapping
   → OperatorCommand → Operator → Task / Mechanism 的主链。

2. bluetooth_config.h 只描述协议格式，不描述机器人业务意义。

3. config 决定 Payload 的类型、数量、布局和大小。

4. Parser 与 Deserializer 概念职责分离，但第一版可继续共用
   bluetooth_protocol.c。

5. Parser 只负责形成合法 Frame / Payload。

6. Deserializer 只负责 Payload → bool[] / u8[] / u16[] / short[]。

7. 只有 Valid Frame 才允许更新 BluetoothRxData 和 OperatorCommand。

8. Mapping 只负责协议字段 → 业务字段，不直接执行 Task。

9. Operator 负责 Rising Edge / Falling Edge / Level 和上层调度。

10. 一次性 Task / Confirm 命令采用 Rising Edge。

11. 四方向人工微调采用 Level，松开时通过 Falling Edge 停止。

12. UART Callback 可以把 byte 喂给 BluetoothProtocol，但不进入
    Mapping / Operator / Task / Mechanism。

13. 函数封装不改变执行上下文；Callback 中调用的 Protocol 代码仍在
    当前 Callback / 中断处理上下文执行。

14. UART Interrupt / DMA 只属于底层数据接入方式，不改变上层协议与业务结构。

15. 上位机 TX 与 STM32 RX 必须在类型、数量、顺序和字段约定上严格对齐。

16. Bool 的 LSB First 与 U16 / Short 的 Endianness 是两个不同概念。

17. 通信 Fail-safe、Watchdog、Stale Command、双缓冲等暂列为可选升级，
    不进入当前第一版必做范围。

18. 输入冲突策略暂列待冻结项，等待遥控器交互和实物验证后确定。

19. 未得到老师真实源码 / PPT 前，不写死 Baud Rate、Length、Checksum 范围和字节序。
```

