# 2026-08-24 蓝牙通信协议、Mapping 与 Operator 上层控制学习记录

> 日期：2026-08-24  
> 学习主题：蓝牙通信链路、自定义数据包协议、STM32 解包状态机、位操作、业务 Mapping 与 Operator 调度  
> 学习背景：在已有 `PickBlockTask / PlaceBlockTask / BlockArm / BlockVacuum` 方块操作机构方案基础上，为后续接入手机/遥控器蓝牙控制补齐上层输入链路。  
> 资料来源：本次培训逐字稿与时间轴、当前《方块操作机构 Task 与 Mechanism 程序设计（第一版）》方案。  
> 说明：目前尚未拿到老师实际蓝牙工程源码，因此涉及真实波特率、Frame Length、Checksum 覆盖范围、U16/Short 字节序等实现细节暂不做硬性结论。

---

# 1. 今日学习目标

今天不是重新学习 UART，而是在已经掌握 UART、中断、DMA、Callback、状态机和模块化的基础上，进一步解决：

```text
UART 收到的一串原始字节
↓
如何识别完整合法的数据帧
↓
如何恢复 Bool / U8 / U16 / Short 等数据
↓
如何把协议字段解释成机器人操作意图
↓
如何由 Operator 正确调度已有 Task / Mechanism
```

最终建立完整链路：

```text
手机 / 遥控器
↓ Bluetooth
蓝牙模块
↓ UART
STM32
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
PickBlockTask / PlaceBlockTask / BlockArm
↓
Mechanism / Driver
```

---

# 2. 与已有知识的衔接

今天直接复用了之前已经掌握的内容：

```text
UART 基本通信
UART 中断接收
DMA
HAL Callback
Buffer
enum + switch-case 状态机
非阻塞 Process()
.c/.h 模块化
Task / Mechanism 分层
函数返回 ≠ 物理动作完成
```

因此今天重点不在 UART 外设本身，而在：

```text
自定义通信协议
+
数据解释
+
业务映射
+
上层输入调度
```

---

# 3. 蓝牙通信整体架构

培训中的硬件链路：

```text
手机 / 遥控器
↓
蓝牙无线通信
↓
蓝牙模块
↓ UART
STM32 主控
```

当前学习模型中：

```text
蓝牙模块
≈ 无线侧与 UART 侧之间的中转
```

STM32 主要面对的是：

```text
UART + 自定义应用层通信协议
```

暂时不深入：

```text
BLE PHY
GATT
ATT
Advertising
Pairing
```

等蓝牙协议栈内部内容。

---

# 4. 软件分层理解

今天进一步明确了四层职责：

## 4.1 UART / Driver 层

负责：

```text
把 byte 收进 STM32
```

UART 不知道：

```text
0xA5 是包头
bool[0] 是 LowPick
某个 Short 是速度
```

---

## 4.2 BluetoothProtocol

负责：

```text
识别 Frame
检查 Header
接收 Payload
Checksum 校验
检查 Tail
Payload 解包
```

它只理解：

```text
bool[0]
u8[0]
short[0]
```

等协议数据，不理解机器人业务。

---

## 4.3 OperatorMapping

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

Mapping 的核心作用是“解释”。

---

## 4.4 Operator

负责：

```text
读取业务操作意图
判断 Rising Edge / Falling Edge / Level
检查当前 Task 状态
调用现有 Task / Mechanism 公开接口
```

最终：

```text
Mapping = 翻译官
Operator = 操作调度者
```

---

# 5. 自定义数据帧 Frame

培训给出的基本协议结构：

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

Checksum 采用：

```text
相关原始字节累加
↓
取低 8 位
```

即基本思想：

```text
checksum = sum mod 256
```

但目前没有老师源码，因此以下内容暂未确认：

```text
Checksum 是否包含 Header
是否存在独立 Length 字段
Length 是帧内字段还是由 config 静态计算
```

---

# 6. 为什么不能只发送裸 Payload

如果只发送：

```text
01 00
```

STM32 无法天然知道：

```text
这一帧从哪里开始
这一帧在哪里结束
是否中途丢字节
收到的数据是否损坏
```

因此需要：

```text
Header
→ 定位一帧开始

Payload
→ 真正的数据

Checksum
→ 检查数据是否可靠

Tail
→ 辅助确认帧结构结束
```

---

# 7. Bool 位打包

培训协议中 Bool 采用按位压缩：

```text
8 个 Bool
→ 1 byte
```

当前方块机构第一版暂定 11 个输入：

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

因此：

```text
11 Bool
→ 2 byte
```

Bool 所需字节数：

```c
(BOOL_NUM + 7) / 8
```

这是整数运算实现的向上取整。

---

# 8. LSB First 与 Bool 排列

课程采用：

```text
LSB First
```

因此：

```text
bool[0] → 第一个 Bool byte 的 bit0
bool[1] → bit1
...
bool[7] → bit7

bool[8] → 第二个 Bool byte 的 bit0
bool[9] → bit1
bool[10] → bit2
```

例如只按：

```text
LowPick
```

则：

```text
Byte0 = 0000 0001 = 0x01
Byte1 = 0000 0000 = 0x00
```

所以 Payload 写成：

```text
01 00
```

这里表示：

```text
buf[0] = 0x01
buf[1] = 0x00
```

不是把 `01 00` 当成一个 `0x0100` 整数。

---

# 9. 今日重点位操作

定位某个 Bool：

```c
byte_index = index / 8;
bit_index  = index % 8;
```

解释：

```text
index / 8
→ 找第几个 byte

index % 8
→ 找这个 byte 内的第几个 bit
```

提取：

```c
bool_value =
    (buffer[index / 8] >> (index % 8)) & 0x01;
```

例如：

```text
bool[10]
```

有：

```text
10 / 8 = 1
10 % 8 = 2
```

因此：

```text
bool[10]
→ buffer[1] 的 bit2
```

本次学习中已能正确独立判断。

---

# 10. 一个容易混淆的问题：数组顺序与 bit 顺序

今天专门澄清了：

```text
buf[0]、buf[1]、buf[2]
```

没有天然的“左”和“右”。

我们通常把它写成：

```text
buf[0] → buf[1] → buf[2]
左                    右
```

只是为了表示：

```text
第一个
→ 第二个
→ 第三个
```

或者 UART 的先后到达顺序。

而单个 byte 一般写成：

```text
bit7 ... bit1 bit0
```

即：

```text
高位在左
低位在右
```

因此：

```text
buf[0] 通常画在最左边
```

与：

```text
bool[0] 位于 byte 的最右边 bit0
```

并不矛盾。

---

# 11. Parser 帧解析状态机

UART 实际收到的是连续 byte：

```text
A5
↓
01
↓
00
↓
Checksum
↓
5A
```

而不是天然得到一个完整 Frame。

因此使用状态机：

```c
typedef enum
{
    BT_WAIT_HEADER,
    BT_RECEIVE_PAYLOAD,
    BT_CHECK_CHECKSUM,
    BT_CHECK_TAIL

} BluetoothParseState_t;
```

基本流程：

```text
WAIT_HEADER
↓ 找到 0xA5

RECEIVE_PAYLOAD
↓ Payload 收满

CHECK_CHECKSUM
↓ 校验通过

CHECK_TAIL
↓ 收到 0x5A

FRAME VALID
↓
重置
↓
WAIT_HEADER
```

---

# 12. Parser 状态决定 byte 的角色

同一个：

```text
0xA5
```

在不同状态下意义不同。

如果：

```text
state = WAIT_HEADER
```

则：

```text
0xA5
→ Header
```

如果：

```text
state = RECEIVE_PAYLOAD
```

则：

```text
0xA5
→ 普通 Payload byte
```

因此：

```text
byte 本身
+
Parser 当前状态
```

共同决定当前字节的意义。

这与之前学习的 Task 状态机思想一致，但职责不同：

```text
Bluetooth Parser 状态
→ 当前 Frame 收到哪一步

PickBlockTask 状态
→ 当前拾取任务执行到哪一步
```

---

# 13. Frame、Payload 与业务命令的任务边界

今天进一步强化了以下关系：

```text
收到一个 UART byte
≠
收到完整 Frame

收到完整 Frame
≠
执行了机器人命令

启动 PickBlockTask
≠
拾取任务已经完成

调用 BlockArm_MoveToXXX()
≠
机械动作已经完成
```

这是今天通信层与原有 Task / Mechanism 层连接时的重要边界。

---

# 14. Payload 反序列化

Parser 验证完整 Frame 后，得到的是原始 Payload。

接下来需要：

```text
Payload byte stream
↓
Bool / U8 / U16 / Short
```

这个过程称为：

```text
Deserialization
反序列化
```

可以理解为：

```text
发送端：
变量 → byte 序列

接收端：
byte 序列 → 变量
```

---

# 15. Bool / U8 / U16 / Short

基本尺寸：

```text
Bool
→ 按 bit 打包

U8
→ 1 byte

U16
→ 2 byte

Short / int16_t
→ 2 byte
```

当前方块机构第一版主要使用离散操作意图，因此：

```text
Bool
```

已经足够满足当前输入需求。

不因为协议支持 U8 / U16 / Short 就强行使用。

---

# 16. 多字节数据与大小端

以：

```text
0x1234
```

为例。

大端：

```text
12 34
```

小端：

```text
34 12
```

重点结论：

```text
buf[0]
```

只表示：

```text
第一个 byte
```

它并不天然等于：

```text
高字节
```

或：

```text
低字节
```

具体由协议字节序决定。

同时明确：

```text
Bool 的 LSB First
≠
U16 / Short 一定 Little Endian
```

两者不是同一个概念。

目前课程逐字稿没有确认 U16 / Short 实际字节序，等待源码或 PPT 核对。

---

# 17. config 的作用

config 决定：

```text
Bool 有几个
U8 有几个
U16 有几个
Short 有几个
```

从而决定：

```text
Payload Size
```

以及：

```text
Deserializer 应该怎样切 Payload
```

例如：

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

Payload 总计 = 10 byte
```

所以：

```text
config
↓
决定 Payload 布局
↓
Parser 知道应接收多少数据
↓
Deserializer 知道每一段如何解释
```

---

# 18. RX / TX 对齐

TX / RX 必须明确设备视角。

UART 接线：

```text
蓝牙模块 TX
↓
STM32 RX

蓝牙模块 RX
↑
STM32 TX
```

当前遥控控制方向：

```text
上位机 TX config
↓
STM32 RX config
```

必须严格一致。

一致不仅包括：

```text
字段数量
```

还包括：

```text
数据类型
字段顺序
字段业务语义
```

如果 UART、Checksum、Parser 都正常，但：

```text
上位机 bool[3] = PlaceLevel1
```

STM32 Mapping 却解释：

```text
bool[3] = PlaceLevel2
```

那么错误发生在：

```text
协议字段 ↔ 业务 Mapping 约定
```

而不是 UART 或 Parser。

---

# 19. Mapping 层

今天对 Mapping 的最终理解：

```text
BluetoothRxData
↓
OperatorMapping
↓
OperatorCommand
```

例如：

```c
cmd->low_pick =
    rx->bool_data[0];

cmd->fine_up =
    rx->bool_data[9];
```

Mapping 负责：

```text
“这些协议字段代表什么？”
```

但不负责：

```text
“现在应该调用哪个 Task？”
```

---

# 20. Operator 层

Operator 负责：

```text
操作输入语义
+
当前机器人流程
↓
调度已有公开接口
```

今天区分了两种输入。

## 20.1 一次性命令：Rising Edge

例如：

```text
LowPick
HighPick
PlaceBottom
PlaceLevel1
PlaceLevel2
ConfirmGrab
ConfirmRelease
```

采用：

```text
0 → 1
```

时触发一次。

概念代码：

```c
if (current.low_pick &&
    !previous.low_pick)
{
    PickBlockTask_StartLowPick();
}
```

这样遥控器连续多帧保持：

```text
1 1 1 1
```

不会重复启动 Task。

---

## 20.2 持续命令：Level

人工微调：

```text
FineForward
FineBackward
FineUp
FineDown
```

需要表达：

```text
按住
→ 持续运动

松开
→ 停止并保持
```

所以主要看当前：

```text
Level
```

松开时再处理：

```text
1 → 0
```

即 Falling Edge。

这与当前 `BlockArm_StartFineAdjust()` / `BlockArm_StopFineAdjust()` 设计相匹配。

---

# 21. Mapping 与 Operator 的最终区分

今天曾提出理解：

> Mapping 是用来调度 `bool[]` 的不同含义，改变状态，Operator 检测状态变化后执行 Task。

最终精确化为：

```text
Protocol
→ 第几个 bool 是多少

Mapping
→ 这个 bool 在机器人业务中是什么意思

Operator
→ 这个业务输入应该按边沿还是持续状态处理，以及当前应该调用谁

Task / Mechanism
→ 真正管理完整动作流程
```

简化：

```text
Mapping = 解释
Operator = 调度
```

---

# 22. 与当前方块机构方案的接入

完整 LowPick 链路：

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
蓝牙模块 UART TX
↓
STM32 UART RX
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
Operator 检测 0 → 1
↓
PickBlockTask_StartLowPick()
↓
已有 Task / Mechanism 流程继续
```

蓝牙层不等待：

```text
BlockArm REACHED
BlockVacuum GRABBED
Task DONE
```

这些仍由已有 Task / Mechanism 自己处理。

---

# 23. 新增上层模块方案

今天确定的新增模块：

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
→ 协议配置

bluetooth_protocol.c/.h
→ Frame Parser + Payload 解包

operator_mapping.c/.h
→ BluetoothRxData → OperatorCommand

operator.c/.h
→ Edge / Level 判断 + 上层调度
```

第一版暂不单独增加：

```text
bluetooth_deserializer.c
```

避免为了模块化机械增加文件数量。

---

# 24. UART 中断 / DMA 与协议层关系

蓝牙模块对 STM32 来说就是普通 UART 对端。

CubeMX 主要负责：

```text
UART
TX / RX
Baud Rate
NVIC
DMA（可选）
```

两种接收方式：

## 中断

```text
UART
↓
收到 1 byte
↓
Callback
↓
BluetoothProtocol_InputByte()
```

## DMA

```text
UART
↓
DMA 搬运一批 byte
↓
Callback / Event
↓
逐个 byte 喂给 BluetoothProtocol
```

重要结论：

```text
中断 / DMA
```

只改变：

```text
UART 数据如何送到 Protocol
```

不应该改变：

```text
Parser
Mapping
Operator
Task
Mechanism
```

---

# 25. Callback 的机制边界

如果：

```c
HAL_UART_RxCpltCallback(...)
{
    BluetoothProtocol_InputByte(byte);
}
```

那么 `BluetoothProtocol_InputByte()` 仍然运行在当前 Callback / 中断处理上下文中。

函数封装：

```text
改变代码职责
```

但不会：

```text
自动改变执行上下文
```

因此 Callback 中适合：

```text
短小、确定的通信接收处理
```

不应直接塞入：

```text
Mapping
Operator
Task
Mechanism
```

等业务流程。

---

# 26. 编译期裁剪

老师最后提到：

> 某类字段数量为 0 时，对应代码可以不编译。

今天只学习到概念层级。

例如：

```c
#define BT_RX_U8_NUM 0

#if BT_RX_U8_NUM > 0

/* U8 解析代码 */

#endif
```

这里：

```text
#if
```

属于预处理阶段。

与普通：

```c
if (...)
```

运行时判断不是同一个机制。

当前只需知道：

```text
config 为固定编译期参数
→ 可以用于裁剪当前根本不需要的协议功能
```

无需深入编译器优化。

---

# 27. 学习过程中出现的主要疑问与纠正

## 27.1 `01 00` 为什么这样排列

最初疑问：

> 为什么 LowPick 对应 Payload `01 00`？

澄清：

```text
LowPick = bool[0]
↓
第一个 Bool byte 的 bit0 = 1
↓
Byte0 = 0x01

后面的 bool 全为 0
↓
Byte1 = 0x00
```

因此：

```text
01 00
```

---

## 27.2 `bool[0]` 是不是写在最左边

最初混淆了：

```text
字节顺序
```

与：

```text
单个 byte 内的 bit 书写顺序
```

最终明确：

```text
buf[0]
通常作为第一个 byte 写在左侧

bool[0]
由于采用 LSB First
位于这个 byte 的 bit0
而 bit0 通常写在二进制最右侧
```

---

## 27.3 为什么 `buf[0]` 习惯写在左边

最初感觉反直觉。

最终理解：

```text
buf[0]
```

并没有天然“左边”的含义。

只是通常把：

```text
第一个 → 第二个 → 第三个
```

按阅读习惯画成：

```text
左 → 右
```

真正重要的是：

```text
数组下标顺序 / 数据到达顺序
```

而不是图上的空间方向。

---

## 27.4 LSB First 是否等于 Little Endian

明确纠正：

```text
不是。
```

LSB First 在当前课程中主要描述：

```text
Bool 位在 byte 中的编号与打包顺序
```

Little Endian 描述：

```text
一个多字节整数拆成多个 byte 后的排列顺序
```

属于不同层级。

---

## 27.5 Mapping 是否直接执行 Task

最初理解：

> Mapping 解释 bool 后改变状态，Operator 看到变化执行 Task。

最终精确为：

```text
Mapping
→ 更新有业务意义的 OperatorCommand 输入快照

Operator
→ 根据 Edge / Level 和当前流程决定调用哪个接口
```

Mapping 本身不执行 Task。

---

# 28. 暂不深入的扩展内容

学习过程中讨论到通信失联可能留下：

```text
FineUp = 1
```

等陈旧命令。

可以进一步设计：

```text
通信超时
Communication Watchdog
Stale Command 处理
失联后 FineAdjust 自动停止
重连首帧同步
双缓冲 / Queue
自动 Task 失联策略
```

但这些不属于今天课程主线。

当前决定：

```text
统一作为后续方案“可选升级项”
```

暂不深入实现。

---

# 29. 当前仍未确认的内容

由于没有老师实际源码 / PPT，以下内容不能仅凭逐字稿确定：

```text
1. 实际 UART Baud Rate
   逐字稿识别为 15200，但暂不视为已核实值。

2. Frame 是否存在独立 Length 字段。

3. 如果有 Length，具体位置与含义。

4. Checksum 具体覆盖哪些 byte。

5. U16 / Short 实际采用 Big Endian 还是 Little Endian。

6. 老师实际 BluetoothProtocol 函数名。

7. 老师实际 config 文件结构和宏定义。

8. 实际使用逐字节 UART IT 还是 DMA。

9. 上位机真实 TX 字段排列。
```

后续拿到：

```text
PPT
源码
.ioc
config
```

后，应以真实工程为最高优先级重新核对。

---

# 30. 今日知识掌握状态

## 已掌握 / 基本掌握

```text
蓝牙模块与 STM32 UART 的职责关系

Header / Payload / Checksum / Tail 基本意义

Bool 位压缩

LSB First

index / 8
index % 8

右移 + & 0x01 的 Bool 提取

Parser 状态机的作用

“状态决定当前 byte 的角色”

Frame 与 Payload 的区别

反序列化基本概念

Bool / U8 / U16 / Short 的数据宽度

大小端基本概念

config 与 Payload 布局的关系

STM32 RX ↔ 上位机 TX 的对应关系

Mapping 与 Operator 的职责差异

Rising Edge / Level / Falling Edge 在当前输入中的作用

UART Interrupt / DMA 与 Protocol 的分层关系
```

## 已理解但需真实工程验证

```text
BluetoothProtocol 的实际 API 设计

config 的具体形式

实际 Frame 布局

实际 Checksum 算法范围

实际 UART 参数

多字节类型真实字节序
```

---

# 31. 今日形成的核心心智模型

今天最值得保留的一条链：

```text
UART
只负责“把 byte 收进来”

↓

Parser
负责“这些 byte 能不能组成一帧合法消息”

↓

Deserializer
负责“Payload 里面分别是什么数据”

↓

Mapping
负责“这些协议数据在机器人中是什么意思”

↓

Operator
负责“这些操作意图现在应该如何调度”

↓

Task
负责“完整任务怎样一步一步执行”

↓

Mechanism
负责“机构动作怎样真正完成”
```

每一层都有明确职责，不能因为调用链连续，就把它们理解成同一个机制。

---

# 32. 今日阶段结论

今天完成了从：

```text
UART 原始字节
```

一直到：

```text
PickBlockTask / PlaceBlockTask / BlockArm
```

之间缺失的上层通信与输入控制链设计。

当前新的整体架构已经能够解释：

```text
操作手按一个按钮
↓
为什么最终能够正确触发一个 Task

操作手按住一个微调方向
↓
为什么能够持续控制 BlockArm

协议中的 bool[]
↓
为什么不能直接等同于机器人业务状态
```

下一阶段最有价值的工作不是继续扩展理论，而是在老师发布 PPT / 源码后进行：

```text
真实工程对照
↓
确认 config
↓
确认 Frame
↓
确认 Checksum
↓
确认 UART 接入
↓
把今天建立的模型逐项对应到真实代码
```

