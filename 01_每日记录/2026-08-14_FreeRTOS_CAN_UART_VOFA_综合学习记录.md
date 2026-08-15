# 2026-08-14 STM32 FreeRTOS + CAN + UART + VOFA 综合学习记录

## 一、今日学习主题

今天主要围绕 Robocon 结课作业继续完善双 STM32 FreeRTOS 工程，重点学习并实现：

- FreeRTOS Task 与 Queue 的实际使用
- CAN 接收数据后通过 Queue 交给其他 Task
- CAN 中 `float` 数据的打包与恢复
- UART Task 同时处理 VOFA 控制命令与波形数据发送
- VOFA FireWater 数据格式
- 呼吸灯参数 Log 输出
- 主板 / 从板条件编译
- 从板 100Hz `float` 反馈链路设计
- Keil 工程文件 `.uvprojx` XML 损坏排查

---

## 二、当前整体通信结构

```text
                VOFA
                 │
            UART 115200
                 │
                 ▼
              主板 STM32
        ┌────────┴────────┐
        │                 │
   主板呼吸灯          CAN总线
                          │
                  ┌───────┴───────┐
                  │               │
                从板            CANable
                  │
             从板呼吸灯
```

当前团队协议：

```text
VOFA → 主板：
A5 | on_off | period_H | period_L | 5A

主板 → 从板：
CAN 标准帧 0x100
Data[0] = on_off
Data[1] = period_H
Data[2] = period_L

从板 → 主板：
CAN 标准帧 0x200
Data[0..3] = float brightness

CANable → 两块板：
CAN 标准帧 0x123
Data[0] = 蜂鸣次数
```

---

## 三、FreeRTOS Queue 实际使用

### 1. CAN 接收 Queue

CAN Callback 收到 CAN 帧以后，不直接做复杂业务逻辑，而是将数据放入：

```c
can_rx_queue
```

然后 `CAN_Task` 中：

```c
osMessageQueueGet(
    can_rx_queue,
    &Msg,
    0,
    osWaitForever
);
```

数据流：

```text
CAN Callback
↓
osMessageQueuePut()
↓
can_rx_queue
↓
CAN_Task
↓
osMessageQueueGet()
↓
Msg
```

当 Queue 没有消息时，`CAN_Task` 会进入 `Blocked`，不会持续占用 CPU。

### 2. Queue 中传递 float

为主板向 VOFA 输出波形，建立：

```c
vofa_tx_queue
```

每个 Queue 元素保存一个 `float`：

```c
vofa_tx_queue = osMessageQueueNew(
    8,
    sizeof(float),
    NULL
);
```

CAN Task 中：

```c
osMessageQueuePut(
    vofa_tx_queue,
    &brightness,
    0,
    0
);
```

UART Task 中：

```c
osMessageQueueGet(
    vofa_tx_queue,
    &brightness,
    NULL,
    0
);
```

Queue 保存的是数据副本，不是局部变量地址本身。

---

## 四、CAN 中 float 的传输

CAN Data 本质是：

```c
uint8_t data[8];
```

因此 `float` 通过 `memcpy()` 进行原始字节复制。

### 1. 从板发送 float

```c
float brightness;
uint8_t TxData[8];

memcpy(
    TxData,
    &brightness,
    sizeof(float)
);
```

过程：

```text
brightness(float)
↓
复制4Byte
↓
TxData[0..3]
```

### 2. 主板恢复 float

```c
float brightness;

memcpy(
    &brightness,
    Msg.data,
    sizeof(float)
);
```

过程：

```text
Msg.data[0..3]
↓
复制4Byte
↓
brightness
```

今天进一步明确：

> `memcpy()` 不是进行数学意义上的类型转换，而是原样复制内存中的字节。

---

## 五、brightness 与 duty 的关系

呼吸灯模块中：

```c
static uint32_t duty;
```

`duty` 是 PWM 比较值，范围约为：

```text
0 ~ 1999
```

团队协议要求反馈的 `brightness` 为归一化值：

```text
0.0 ~ 1.0
```

因此：

```c
brightness = (float)duty / 1999.0f;
```

例如：

```text
duty = 0
→ brightness = 0.0

duty ≈ 1000
→ brightness ≈ 0.5

duty = 1999
→ brightness = 1.0
```

为了保持模块封装，不直接暴露 `static duty`，而是在 `Led_Breath` 模块增加接口：

```c
float Led_Breath_GetBrightness(void);
```

实现：

```c
float Led_Breath_GetBrightness(void)
{
    if (!breath_on)
    {
        return 0.0f;
    }

    return (float)duty / 1999.0f;
}
```

这样 CAN 发送部分不需要知道呼吸灯内部如何计算 `duty`。

---

## 六、VOFA FireWater 波形输出

主板从 CAN 收到：

```c
float brightness;
```

之后，将其转换成 FireWater 能识别的文本：

```c
snprintf(
    tx_buffer,
    sizeof(tx_buffer),
    "%.3f\n",
    brightness
);
```

例如：

```text
brightness = 0.53241
```

生成：

```text
0.532\n
```

其中 `%.3f` 表示保留三位小数，`\n` 是 FireWater 一帧数据的结束标志。

UART 发送：

```c
HAL_UART_Transmit(
    &huart1,
    (uint8_t *)tx_buffer,
    strlen(tx_buffer),
    100
);
```

数据流：

```text
brightness
↓
snprintf
↓
"0.532\n"
↓
USART1
↓
VOFA FireWater
↓
波形图
```

---

## 七、UART Task 同时处理收发

UART Task 当前同时承担两类任务：

```text
uart_rx_queue
→ VOFA → STM32 控制指令

vofa_tx_queue
→ STM32 → VOFA 波形数据
```

因此不能只对其中一个 Queue 使用 `osWaitForever`，否则可能一直阻塞，导致另一个 Queue 无法处理。

当前采用非阻塞检查：

```c
rx_status = osMessageQueueGet(
    uart_rx_queue,
    frame,
    NULL,
    0
);
```

以及：

```c
tx_status = osMessageQueueGet(
    vofa_tx_queue,
    &brightness,
    NULL,
    0
);
```

最后：

```c
osDelay(1);
```

防止 Task 高速空转。

结构：

```text
UART_Task
│
├─ 检查 uart_rx_queue
│
├─ 检查 vofa_tx_queue
│
└─ osDelay(1)
```

---

## 八、主板解析 VOFA 控制指令

VOFA 控制协议：

```text
A5 | on_off | period_H | period_L | 5A
```

解析逻辑：

```c
if ((rx_status == osOK) &&
    (frame[0] == 0xA5 && frame[4] == 0x5A))
{
    cmd.on_off = frame[1];

    cmd.period_ms =
        (frame[2] << 8) |
        frame[3];
}
```

例如：

```text
A5 01 07 D0 5A
```

解析：

```text
on_off = 1
period_ms = 0x07D0 = 2000ms
```

主板随后：

```text
① 控制自己的呼吸灯
② CAN发送同样参数给从板
③ 保存当前呼吸灯参数用于 Log
```

---

## 九、current_breath_cmd 与 Log

为 Log 保存当前控制参数：

```c
static BreathCmd_t current_breath_cmd = {
    0,
    2000
};
```

解析成功后：

```c
current_breath_cmd = cmd;
```

注意：

```text
brightness
→ 当前瞬时亮度

current_breath_cmd
→ 当前开关状态 + 呼吸周期
```

两者含义不同，不能通过 `brightness` 反推 `on_off` 和 `period_ms`。

Log 格式化：

```c
snprintf(
    log_buffer,
    sizeof(log_buffer),
    "Breath %s period=%ums\n",
    current_breath_cmd.on_off ? "ON" : "OFF",
    current_breath_cmd.period_ms
);
```

例如：

```text
Breath ON period=2000ms
```

然后通过：

```c
HAL_UART_Transmit(
    &huart1,
    (uint8_t *)log_buffer,
    strlen(log_buffer),
    100
);
```

发送到 VOFA。

Log 更适合在参数更新时输出一次，而不是 100Hz 重复输出。

---

## 十、主板 / 从板条件编译

当前工程使用：

```c
#if defined(IS_MASTER_BOARD)

#elif defined(IS_SLAVE_BOARD)

#endif
```

生成主板和从板两种不同固件。

今天进一步明确：

```text
Task 是否创建
```

和：

```text
代码是否参与编译
```

是两个不同层级。

例如：

```c
#if defined(IS_MASTER_BOARD)
UARTTaskHandle = osThreadNew(...);
#endif
```

决定主板是否创建 UART Task。

而在 `UART_Task()` 函数内部使用：

```c
#if defined(IS_MASTER_BOARD)
...
#endif
```

则用于防止 Slave 编译时访问主板专用变量。

---

## 十一、发现的重要遗漏：从板 100Hz float 反馈

今天检查工程后发现：

> 从板虽然有呼吸灯功能，但还没有真正发送 `0x200` 的 `float` 反馈。

因此此前链路实际是：

```text
从板
↓
【缺少100Hz float发送】
↓
主板等待 FLOAT_ID
↓
无法收到
```

这意味着 VOFA 波形链还没有真正闭环。

---

## 十二、Float_Task 设计

题目要求：

```text
从板 100Hz 向主板反馈变化的 float
```

100Hz 对应周期：

```text
10ms
```

因此从板增加 `Float_Task`，只在从板创建：

```c
#if defined(IS_SLAVE_BOARD)
```

任务周期：

```c
uint32_t last = osKernelGetTickCount();

for (;;)
{
    last += 10;
    osDelayUntil(last);

    ...
}
```

每 10ms：

```text
读取 brightness
↓
打包 float
↓
CAN 0x200 发送
```

核心发送逻辑：

```c
brightness = Led_Breath_GetBrightness();

memcpy(
    TxData,
    &brightness,
    sizeof(float)
);

HAL_CAN_AddTxMessage(
    &hcan1,
    &TxHeader,
    TxData,
    &TxMailbox
);
```

CAN Header：

```c
TxHeader.StdId = FLOAT_ID;
TxHeader.ExtId = 0;
TxHeader.IDE = CAN_ID_STD;
TxHeader.RTR = CAN_RTR_DATA;
TxHeader.DLC = 4;
TxHeader.TransmitGlobalTime = DISABLE;
```

最终完整链路：

```text
从板 Breath_Task
↓
更新 duty
↓
Led_Breath_GetBrightness()
↓
Float_Task 100Hz
↓
CAN 0x200
↓
主板 CAN Callback
↓
can_rx_queue
↓
CAN_Task
↓
memcpy → brightness
↓
vofa_tx_queue
↓
UART_Task
↓
FireWater
↓
VOFA波形
```

---

## 十三、今天遇到的调试问题

### 1. `protocol.h` 未包含

曾出现：

```text
CanMsg_t undeclared
BreathCmd_t undeclared
FLOAT_ID undeclared
BEEP_ID undeclared
```

原因是 `freertos.c` 没有包含：

```c
#include "protocol.h"
```

补充后大量级联 Error 一起消失。

学习点：

> 编译时优先看第一处真正 Error，不要逐条修后面的级联报错。

### 2. `can.c` 缺少 `protocol.h`

报错：

```text
RX_BUSINESS_ID undeclared
BEEP_ID undeclared
```

同样通过：

```c
#include "protocol.h"
```

解决。

### 3. Queue 被重复 Get

曾错误写成：

```c
tx_status = osMessageQueueGet(...);

if (tx_status == osOK)
{
    osMessageQueueGet(...);
}
```

这样会连续取两个 float。

正确理解：

```text
第一次 Get 成功
↓
brightness 已经拿到了
↓
if 里直接使用
```

### 4. `continue` 导致 TX 分支被跳过

曾有：

```c
if (frame非法)
{
    continue;
}
```

由于 UART Task 后面还要处理 `vofa_tx_queue`，`continue` 会跳过本轮 TX。

因此改成：

```c
if ((rx_status == osOK) &&
    (frame[0] == 0xA5 && frame[4] == 0x5A))
{
    处理RX
}
```

然后继续执行后面的 TX 检查。

---

## 十四、Keil 工程文件问题

今天还遇到 `.uvprojx` 无法打开：

```text
Fatal Message: element name expected
```

以及：

```text
processing instruction cannot start with 'xml'
```

说明 `F4_FreeRTOS.uvprojx` 本身 XML 结构可能被破坏。

可能原因包括：

```text
重复 <?xml ... ?>
Git冲突标记
XML标签缺失
文件内容重复拼接
```

这属于工程配置文件错误，而不是 C 源码编译错误。

---

## 十五、今天最重要的知识关系

```text
FreeRTOS
负责：
什么时候运行哪个Task

Queue
负责：
Task之间安全传递数据

CAN
负责：
主板 / 从板之间传输字节

memcpy
负责：
float ↔ 原始4Byte

Led_Breath
负责：
生成实际亮度 duty

Float_Task
负责：
100Hz读取并发送brightness

UART_Task
负责：
VOFA控制指令 + 波形数据输出

FireWater
负责：
把UART文本数据解析成波形
```

---

## 十六、当前掌握状态

### 已基本掌握

- FreeRTOS Task 基本结构
- Ready / Running / Blocked
- `osDelay()` 与 `osDelayUntil()`
- Queue Put / Get
- `osWaitForever`
- Queue 在 Task 间传数据
- CAN 标准帧基本发送与接收
- `memcpy()` 传输 float
- 条件编译 `#if defined`
- FireWater 基本格式
- `snprintf()`
- `HAL_UART_Transmit()`
- 呼吸灯 duty 与 brightness 的关系

### 正在掌握

- 多个 Queue 由一个 Task 同时处理
- 主从板 FreeRTOS 软件架构
- UART + CAN + FreeRTOS 多模块协同
- Task 职责划分
- 完整通信链调试

---

## 十七、当前任务进度

```text
✓ FreeRTOS基础结构
✓ 流水灯Task
✓ 呼吸灯Task
✓ 蜂鸣器Task
✓ CAN接收Queue
✓ CAN噪声Task 500Hz
✓ VOFA控制协议结构
✓ 主板解析呼吸灯参数
✓ 主板控制本机呼吸灯
✓ 主板CAN控制从板
✓ 主板接收0x200的float逻辑
✓ float → VOFA FireWater
✓ Log设计
✓ brightness getter设计
→ 从板 Float_Task 100Hz 实现与检查

□ UART实际接收链完整检查
□ 主板/从板全部 Rebuild 0E0W
□ CAN实物联调
□ VOFA实际波形
□ Log实际显示
□ 100Hz反馈验证
□ 500Hz噪声过滤验证
□ 最终双板联调
□ Git整理与提交
```

---

## 十八、今日最值得记住的一点

今天最大的收获不是某一个 HAL 函数，而是开始真正理解：

> 一个完整功能不是“写一个函数”，而是一条完整的数据链。

例如今天的 VOFA 波形需要：

```text
duty
→ brightness
→ Float_Task
→ CAN
→ CAN Callback
→ Queue
→ CAN_Task
→ Queue
→ UART_Task
→ FireWater
→ VOFA
```

任何一环缺失，程序即使能够编译，也不代表功能真正完成。

---

## 十九、今日简短复盘

今天已经把“主板接收从板 float → VOFA 打波 + Log”的整体软件链基本理清，同时发现此前遗漏的“从板 100Hz float 发送链”。

今天最关键的进步，是开始从“单个函数怎么写”转向理解 Task、Queue、CAN、UART 和业务模块之间的完整数据流。
