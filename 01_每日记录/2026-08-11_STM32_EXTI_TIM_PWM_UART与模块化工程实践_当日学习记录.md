# 2026-08-11｜STM32 中断、定时器、PWM、UART 与模块化工程实践

## 1. 基本信息

- 日期：2026-08-11
- 培训阶段：Robocon 控制组 STM32 综合学习与工程实践
- 学习方向：STM32 / 中断 / Timer / PWM / UART / 嵌入式 C 工程组织
- 主要芯片：
  - STM32F405：EXTI、按键状态机、TIM、PWM 综合实践
  - STM32H723ZETx：USART1 中断接收、字节环形队列、命令处理软件实现
- 内容来源：8.11 课堂学习记录、当日工程实操、本次对话中的 UART 软件实现与代码结构讨论
- 学习时长：未单独统计
- 开发工具：STM32CubeMX、Keil、VS Code、Git
- 当日结果：
  - F405 综合作业完成，编译、下载和实物现象验证通过
  - H723 UART 中断接收软件链完成，Keil `0 Error(s), 0 Warning(s)`，Git 已提交
  - H723 UART 真实串口通信因暂无合适 USB-TTL，暂未完成实物验证

---

## 2. 今日目标

- [x] 复习并重新梳理 STM32 时钟、Timer Clock 与 HAL Timebase
- [x] 完成 STM32F405 新工程基础配置
- [x] 理解 EXTI 外部中断完整触发链
- [x] 理解 NVIC 抢占优先级与子优先级
- [x] 实现按键双边沿检测、消抖、短按/长按识别
- [x] 理解 `__weak`、HAL Callback 与 `##` Token Pasting
- [x] 使用 TIM2 周期 Tick 完成非阻塞任务调度
- [x] 理解并实际使用 PWM 的 Start / Compare / Stop
- [x] 完成按键状态机 + 流水灯 + 呼吸灯 + 蜂鸣器综合作业
- [x] 在 H723 工程中完成 USART1 CubeMX 配置
- [x] 实现 `HAL_UART_Receive_IT()` 单字节中断接收的软件结构
- [x] 新建并实现 `byte_queue.c/.h` 字节环形队列
- [x] 将 UART Callback 与 main 业务处理通过环形队列解耦
- [x] 学习中断与主循环共享数据时的临界区和 PRIMASK
- [x] 实现简单 `command_process()` 命令解析
- [x] 完成 UART 软件链模拟、编译和 Git 提交
- [x] 总结后续 STM32 工程的模块化封装原则
- [ ] 使用 USB-TTL 完成 H723 USART1 真实硬件通信验证
- [ ] 进入 UART DMA 学习

---

## 3. 今日完成内容

### 3.1 STM32F405 工程基础配置与时钟体系

本次重新建立了 F405 工程的基础配置流程：

```text
选择 MCU
↓
SYS / RCC
↓
Clock Configuration
↓
GPIO / TIM / UART 等外设
↓
NVIC / DMA
↓
Generate Code
↓
检查自动生成代码
↓
编写用户逻辑
↓
编译
↓
下载 / 实物验证
```

当前 F405 使用：

```text
HSE = 25 MHz
PLLM = 25
PLLN = 336
PLLP = 2
```

因此：

```text
25 MHz / 25 = 1 MHz
1 MHz × 336 = 336 MHz
336 MHz / 2 = 168 MHz
```

得到：

```text
SYSCLK = 168 MHz
HCLK   = 168 MHz
PCLK1  = 42 MHz
PCLK2  = 84 MHz
```

进一步明确了 F405 的 Timer Clock 规则：

```text
APB Prescaler = 1
→ Timer Clock = PCLK

APB Prescaler > 1
→ Timer Clock = 2 × PCLK
```

因此：

```text
APB1 Timer Clock = 84 MHz
APB2 Timer Clock = 168 MHz
```

#### HAL Timebase

当前工程使用 TIM1 推进 HAL 的 1 ms Tick。

典型链路：

```text
TIM1 每 1 ms 到期
↓
TIM1 IRQ
↓
HAL_TIM_IRQHandler()
↓
HAL_TIM_PeriodElapsedCallback()
↓
HAL_IncTick()
↓
uwTick++
↓
HAL_GetTick()
```

本次进一步明确：

- `HAL_GetTick()` 读取的是 HAL 已维护的 Tick；
- 它不是直接读取 TIM1 的 CNT；
- HAL Timebase 可以来自 SysTick，也可以来自某个 Timer；
- “Timer 函数执行完”与“Timer 硬件任务结束”是两回事。

---

### 3.2 EXTI、NVIC 与按键事件识别

#### EXTI 基本机制

EXTI = External Interrupt/Event Controller，外部中断/事件控制器。

GPIO 负责表示当前电平：

```text
HIGH / LOW
```

EXTI 负责检测电平变化，例如：

```text
LOW → HIGH：上升沿
HIGH → LOW：下降沿
```

PC11 的完整中断链：

```text
PC11 电平变化
↓
EXTI11 检测有效边沿
↓
EXTI11 Pending = 1
↓
产生 EXTI15_10_IRQn
↓
NVIC 判断是否允许响应
↓
CPU 进入 EXTI15_10_IRQHandler()
↓
HAL_GPIO_EXTI_IRQHandler()
↓
HAL 清除 Pending
↓
HAL_GPIO_EXTI_Callback()
↓
用户按键逻辑
```

关键区分：

```text
EXTI Line
→ 负责监视哪一路边沿

Pending Flag
→ 记录有事件等待处理

IRQ
→ 外设向 NVIC 提出的中断请求

ISR
→ CPU 实际执行的中断服务入口

Callback
→ HAL 提供给用户实现业务逻辑的接口
```

#### EXTI Line 与共享 IRQ

GPIO 外部中断线通常按 Pin 编号映射：

```text
Px11 → EXTI11
```

因此 PA11、PB11、PC11 会竞争同一条 EXTI11。

F405 中：

```text
EXTI0~4   → 独立 IRQ
EXTI5~9   → EXTI9_5_IRQn
EXTI10~15 → EXTI15_10_IRQn
```

所以 PC11 最终进入：

```c
EXTI15_10_IRQHandler()
```

#### NVIC

NVIC = Nested Vectored Interrupt Controller，嵌套向量中断控制器。

主要负责：

- 中断是否使能
- 中断优先级
- 多中断同时到达时的处理顺序
- 是否允许高优先级中断抢占当前 ISR

当前理解：

```text
数值越小
→ 优先级越高
```

其中：

```text
Preempt Priority
→ 决定是否能够抢占

Sub Priority
→ 抢占优先级相同时决定先后顺序
```

---

### 3.3 按键短按 / 长按、消抖与状态机

当前按键电气逻辑：

```text
默认 LOW
按下 HIGH
```

因此：

```text
GPIO_PIN_SET
→ 按下

GPIO_PIN_RESET
→ 松开
```

经典 HAL 的：

```c
HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
```

只告诉用户“哪个 Pin 触发”，不会直接给出“上升沿还是下降沿”。

所以双边沿模式中，需要结合：

```c
HAL_GPIO_ReadPin(...)
```

读取当前电平，推断：

```text
读到 HIGH
→ 按下 / 上升沿

读到 LOW
→ 松开 / 下降沿
```

注意：

> `HAL_GPIO_ReadPin()` 读的是当前电平，不是直接读取“边沿类型”。

#### 短按 / 长按

本次作业采用：

```text
按下
→ 记录 press_time

松开
→ now - press_time
→ 判断短按 / 长按
```

因此当前实现准确地说是：

> 在松开时，根据按压持续时间分类短按或长按。

如果以后要求：

```text
按住满 1000 ms
→ 不等松开立即触发长按
```

则还需要增加一次性长按触发状态，例如：

```text
long_triggered
```

#### 消抖

当前方案：

```c
if (now - last_edge_time < KEY_DEBOUNCE_MS)
{
    return;
}
```

本质属于：

> 边沿时间过滤。

即接受一个边沿后，在一定时间内忽略其它快速抖动边沿。

---

### 3.4 `__weak`、HAL Callback 与 `##`

HAL 常提供：

```c
__weak void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
}
```

用户工程中重新定义同名非 weak 函数：

```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    // 用户实现
}
```

链接阶段使用用户的强定义。

本次形成的精确理解：

```text
__weak
→ 决定最终使用哪个函数实现

HAL 内部调用位置
→ 决定 Callback 什么时候执行
```

另外学习了：

```c
##
```

即 Token Pasting Operator，记号粘贴运算符。

例如：

```c
LED##x##_GPIO_Port
```

在预处理阶段可以拼接成：

```c
LED1_GPIO_Port
```

它只是 C 预处理机制，本身不理解 GPIO 的硬件含义。

---

### 3.5 TIM2 周期 Tick 与非阻塞任务

作业中 TIM2 每 2 ms 产生一个周期 Tick：

```c
static volatile uint32_t tick_2ms_count = 0;
```

每次中断：

```c
tick_2ms_count++;
```

#### flag 与 counter 的区别

```text
bool flag
→ 只能表示“至少发生过一次”

counter
→ 可以保留发生次数 / 时间推进量
```

但进一步明确：

```text
counter 没丢
≠
业务任务一定没有漏执行
```

例如电机控制本应每 2 ms 采样一次，如果因为阻塞 20 ms，之后一次性补算 10 次，并不等价于过去真的每 2 ms 执行过一次。

#### 调度时间轴

```c
last_tick = current_tick;
```

表示：

> 从本次实际执行时刻重新计时。

```c
last_tick += interval;
```

表示：

> 尽量保持原来的周期时间轴。

流水灯等视觉任务使用前者即可。

#### `uint32_t` 回绕

2 ms Tick 使用 `uint32_t` 会在长时间运行后回绕。

使用：

```c
current_tick - last_tick >= interval
```

这种无符号时间差判断，可以正确处理正常回绕。

---

### 3.6 PWM 工程应用

PWM 相关操作：

初始化：

```c
MX_TIM3_Init();
```

启动：

```c
HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1);
```

修改 CCR：

```c
__HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, value);
```

停止：

```c
HAL_TIM_PWM_Stop(&htim3, TIM_CHANNEL_1);
```

本次重点理解：

> `HAL_TIM_PWM_Start()` 返回后，TIM3 仍然由硬件继续运行。

即：

```text
函数返回
≠
它启动的硬件任务结束
```

同样：

```c
__HAL_TIM_SET_COMPARE(..., 0);
```

只表示：

```text
占空比 = 0%
```

并不表示 Timer / PWM 已经停止。

#### PWM 频率

当前 TIM3：

```text
Timer Clock = 84 MHz
PSC = 83
ARR = 999
```

所以：

```text
CNT Clock = 1 MHz
PWM 周期 = 1 ms
PWM 频率 = 1 kHz
```

呼吸灯大约每 2 ms 更新一次 CCR，因此：

```text
PWM 载波频率 = 1 kHz
CCR 更新频率 ≈ 500 Hz
```

两者是不同概念。

#### PSC / ARR / CCR

```text
PSC
→ 决定 CNT Clock

ARR
→ 决定一个 PWM 周期需要计多少

CCR
→ 决定一个周期中的比较位置 / 占空比
```

同频率下 ARR 越大，通常可调整的 CCR 档位越多，占空比分辨率越高。

---

### 3.7 F405 综合作业：按键状态机 + 流水灯 + 呼吸灯 + 蜂鸣器

状态：

```c
typedef enum
{
    STATE_OFF = 0,
    STATE_FLOW,
    STATE_BREATH
} led_state_t;
```

事件：

```c
typedef enum
{
    KEY_NONE = 0,
    KEY_SHORT,
    KEY_LONG
} key_event_t;
```

状态切换：

```text
短按
→ BREATH

长按
→ FLOW
```

中断侧：

```text
EXTI Callback
→ 记录按键时间
→ 生成 KEY_SHORT / KEY_LONG
→ 快速退出
```

main / 状态机侧：

```text
消费 key_event
↓
切换状态
↓
调用 LED / BEEP 功能
```

流水灯：

```text
LED1 / LED2
→ 约每 300 ms 交替亮灭
```

呼吸灯：

```c
CCR1 = breath_value;
CCR2 = 999U - breath_value;
```

实现 LED3 / LED4 反向呼吸。

蜂鸣器不再使用长时间 `HAL_Delay()`，而采用：

```text
记录开始时间
↓
主循环中持续检查
↓
到时间关闭
```

完成非阻塞控制。

F405 综合作业最终结果：

```text
✓ GPIO / EXTI 正常
✓ 短按 / 长按正常
✓ 消抖正常
✓ 状态机正常
✓ TIM2 2 ms Tick 正常
✓ 流水灯正常
✓ TIM3 PWM 正常
✓ 反向呼吸正常
✓ 蜂鸣器非阻塞控制正常
✓ 0 Error(s), 0 Warning(s)
✓ 下载成功
✓ 实物现象正确
```

---

### 3.8 H723 USART1 CubeMX 配置

在 `keil_proj_advance-8.9H7` 工程中继续进行 UART 中断接收的软件工程实现。

MCU：

```text
STM32H723ZETx
```

CubeMX 配置：

```text
USART1
Mode        = Asynchronous
Baud Rate   = 115200
Word Length = 8 Bits
Parity      = None
Stop Bits   = 1
Direction   = Receive and Transmit
```

即：

```text
115200 8N1
```

引脚：

```text
PB14 → USART1_TX
PB15 → USART1_RX
```

并开启：

```text
USART1 global interrupt
```

生成后检查：

```c
UART_HandleTypeDef huart1;
```

以及：

```c
void USART1_IRQHandler(void)
{
    HAL_UART_IRQHandler(&huart1);
}
```

因此 UART 中断的软件链明确为：

```text
USART1 硬件事件
↓
IRQ
↓
NVIC
↓
USART1_IRQHandler()
↓
HAL_UART_IRQHandler()
↓
HAL 根据任务状态处理
↓
HAL_UART_RxCpltCallback()
```

进一步强化：

> Callback 不是中断本身，但它把“中断发生后的用户业务逻辑”和底层中断处理解耦了。

---

### 3.9 UART 接收任务与 `rx_data`

本轮使用：

```c
HAL_UART_Receive_IT(&huart1, &rx_data, 1);
```

表示：

> 启动一次“通过中断方式接收 1 字节”的异步任务。

第一次接收任务放在：

```text
USART1 初始化完成
↓
队列初始化完成
↓
while(1) 之前
```

不能简单放进 `while(1)` 中不断调用，因为前一个异步任务未完成时再次调用，可能得到 `HAL_BUSY`。

#### `rx_data` 为什么使用长期变量

当前定义：

```c
uint8_t rx_data;
```

使用全局 / 长生命周期存储。

原因不是“HAL 强制必须全局”，而是：

- `Receive_IT()` 返回后异步任务仍在继续；
- 接收缓冲地址必须在任务完成前持续有效；
- Callback 后续需要访问收到的字节。

真正要求是：

```text
生命周期足够长
+
地址持续有效
+
需要处理的代码能够访问
```

---

### 3.10 新建 `byte_queue.c/.h` 字节环形队列

当前 8.9 H7 工程没有原来的队列，因此重新新建：

```text
Core/Inc/byte_queue.h
Core/Src/byte_queue.c
```

并把 `.c` 文件加入 Keil 工程。

本轮队列存储：

```c
uint8_t
```

而不是 `command_packet`。

原因：

```text
HAL_UART_Receive_IT(..., 1)
→ 每次直接得到一个原始字节
→ 先缓存原始字节
→ 后续更高层再进行协议解析 / 封包拆包
```

#### 队列结构

```c
#define BYTE_QUEUE_SIZE 16

typedef struct
{
    uint8_t buf[BYTE_QUEUE_SIZE];
    uint8_t head;
    uint8_t tail;
    uint8_t count;
} byte_queue_t;
```

含义：

```text
buf
→ 存数据

head
→ 下一次 pop 位置

tail
→ 下一次 push 位置

count
→ 当前有效元素数
```

初始化：

```text
head = 0
tail = 0
count = 0
```

#### 空 / 满

```text
count == 0
→ 空

count == 16
→ 满
```

学习过程中曾误认为：

```text
count == 15
→ 满
```

后来纠正：

```text
数组下标 0~15
≠
容量只有 15

实际元素容量 = 16
```

#### 环形回绕

push 后：

```c
q->tail = (q->tail + 1) % BYTE_QUEUE_SIZE;
```

pop 后：

```c
q->head = (q->head + 1) % BYTE_QUEUE_SIZE;
```

因此：

```text
0 → 1 → ... → 15 → 0
```

形成真正的环形队列。

#### FIFO

FIFO = First In, First Out，先进先出。

例如：

```text
push 10
push 20
push 30
```

则：

```text
pop → 10
pop → 20
pop → 30
```

UART 字节流必须保持 FIFO 顺序，否则协议数据会乱序。

---

### 3.11 队列满策略

最开始曾考虑：

```text
队列满
→ 丢最旧数据
→ 保留最新数据
```

理由是误以为：

> 最旧数据可能已经被 CPU 处理。

后来纠正：

> 如果最旧数据已经被处理，它应该已经被 `pop()`，不会还占据队列。

所以当前策略：

```text
队列满
→ push 返回 false
→ 保留已有旧数据
→ 丢弃新数据
```

同时明确：

> 无论丢新还是丢旧，只要队列满，就说明软件处理速度已经跟不上输入速度，数据完整性已经受到影响。

后续更完善的工程应加入：

```text
overflow flag
```

记录发生过队列溢出。

---

### 3.12 UART Callback

最终 UART Callback 逻辑：

```c
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        byte_queue_push(&rx_queue, rx_data);
        HAL_UART_Receive_IT(huart, &rx_data, 1);
    }
}
```

Callback 只做：

```text
确认 USART1
↓
保存当前字节到队列
↓
重新启动下一次 1 字节 Receive_IT
↓
快速退出
```

没有在 Callback 中做：

- 命令解析
- LED 控制
- 长循环
- `HAL_Delay()`
- 复杂业务

因此当前连续接收模型：

```text
main 中启动第一次 Receive_IT
↓
收到 1 字节
↓
任务完成
↓
Callback
↓
重新挂下一次 Receive_IT
↓
等待下一字节
```

---

### 3.13 main 与中断共享队列：临界区和 PRIMASK

接入队列后发现新的并发问题：

```text
UART Callback
→ push
→ 修改 tail / count

main
→ pop
→ 修改 head / count
```

其中 `count` 会被中断和主循环共同读改写。

因此学习 Critical Section（临界区）。

当前 main 中：

```c
primask = __get_PRIMASK();

__disable_irq();

result = byte_queue_pop(&rx_queue, &data);

__set_PRIMASK(primask);
```

含义：

```text
保存进入前的中断状态
↓
暂时屏蔽普通可屏蔽中断
↓
完成一次很短的 pop
↓
恢复原来的 PRIMASK
```

#### 为什么不是直接 `__enable_irq()`

```text
__enable_irq()
→ 无条件打开

__set_PRIMASK(old_state)
→ 恢复原来的状态
```

如果进入临界区前中断本来就已经关闭，无条件 `__enable_irq()` 会改变原状态，因此恢复旧 PRIMASK 更严谨。

#### 临界区范围

只保护：

```c
byte_queue_pop(...)
```

不把：

```text
command_process()
LED 控制
HAL_Delay()
复杂业务
```

放在关中断区域内。

本次进一步明确：

> `__disable_irq()` 不只影响 USART1，也会短暂影响 TIM6 等普通可屏蔽中断，所以临界区必须尽量短。

---

### 3.14 命令处理与软件模拟

为了把业务处理从 UART Callback 中解耦，建立：

```c
void command_process(uint8_t data);
```

最小命令集：

```text
'0' → signal = 0
'1' → signal = 1
'2' → signal = 2
'3' → signal = 3
其他 → 忽略
```

这里重新确认：

```c
'1'
```

和：

```c
1
```

不是同一个值。

键盘发送字符 `1`，UART 得到的是 ASCII 字符 `'1'`。

main 数据链：

```text
临界区 pop
↓
恢复中断
↓
result == true
↓
command_process(data)
```

#### 无硬件条件下的软件模拟

因为没有合适 USB-TTL，本轮没有冒充真实 UART 验证，而是临时向队列人工 push：

```c
byte_queue_push(&rx_queue, '1');
```

以及连续：

```c
byte_queue_push(&rx_queue, '1');
byte_queue_push(&rx_queue, '2');
byte_queue_push(&rx_queue, '3');
byte_queue_push(&rx_queue, '0');
```

用于逻辑推演：

```text
字节进入队列
↓
main FIFO pop
↓
command_process()
↓
signal 依次变化
```

这只能验证：

```text
byte_queue
→ main
→ command_process
```

的软件结构和顺序。

不能据此声称：

```text
USART1 硬件接收
IRQ
ISR
HAL_UART_RxCpltCallback
```

已经完成真实运行验证。

最终删除模拟代码，恢复正式 UART 版本。

结果：

```text
✓ Keil 0 Error(s)
✓ 0 Warning(s)
✓ Git 已提交
```

---

### 3.15 本轮实际遇到的编译错误：`unknown type name 'byte_queue_t'`

第一次编译 `byte_queue` 时出现：

```text
unknown type name 'byte_queue_t'
```

原因：

> 在 `byte_queue.h` 中，函数声明写在 `typedef` 之前。

错误结构类似：

```c
void byte_queue_init(byte_queue_t *q);

typedef struct
{
    ...
} byte_queue_t;
```

C 编译器从上到下读取，在看到函数声明时还不认识 `byte_queue_t`。

修改为：

```text
先 typedef
↓
再函数声明
```

之后多个相关 error 一次性消失。

最终重新 Build：

```text
0 Error(s)
0 Warning(s)
```

---

### 3.16 当日形成的模块化代码组织原则

完成当前作业后，对工程结构进行了进一步复盘。

本次作业不再重构，但后续 STM32 / Robocon 工程默认采用：

> 按功能模块封装，`main.c` 只负责初始化和调度。

理想结构：

```text
main
↓
应用模块
uart_app / timer_app / command / state_machine / led_flow
↓
通用工具
byte_queue
↓
硬件抽象
led / buzzer / motor
↓
CubeMX / HAL
usart / tim / gpio / stm32xxxx_it
```

例如：

```text
uart_app.c / uart_app.h
→ UART 接收任务、Callback、UART 数据处理

timer_app.c / timer_app.h
→ Timer 用户层功能和周期事件分发

byte_queue.c / byte_queue.h
→ 通用环形队列

command.c / command.h
→ 命令解析

state_machine.c / state_machine.h
→ 状态机

led_flow.c / led_flow.h
→ LED 流水灯 / 呼吸灯业务

main.c
→ 系统初始化 + 模块初始化 + while(1) 调度
```

#### 不能机械移动的底层代码

CubeMX 生成并与中断向量绑定的：

```c
USART1_IRQHandler()
TIM6_DAC_IRQHandler()
EXTI15_10_IRQHandler()
```

仍保留在：

```text
stm32xxxx_it.c
```

而：

```c
HAL_UART_RxCpltCallback()
HAL_TIM_PeriodElapsedCallback()
```

等 HAL 用户 Callback，可以根据功能职责放在对应用户模块的 `.c` 中。

#### 模块内部隐藏

只供模块内部使用的：

```text
rx_data
rx_queue
内部状态变量
辅助函数
```

以后优先使用：

```c
static
```

限制作用域。

最终目标：

> `main.c` 尽可能像“系统目录 + 调度入口”，而不是成为所有功能实现的集中位置。

---

## 4. 核心概念

### HAL Timebase

- 是什么：HAL 内部统一使用的时间基准。
- 作用：推进 `uwTick`，供 `HAL_GetTick()`、`HAL_Delay()` 等使用。
- 关键边界：`HAL_GetTick()` 不等于直接读取 Timer CNT。

### EXTI

- 全称：External Interrupt/Event Controller。
- 作用：检测 GPIO 等外部信号的有效边沿并产生中断/事件。
- 关键链路：GPIO 边沿 → EXTI Pending → IRQ → NVIC → ISR。

### NVIC

- 全称：Nested Vectored Interrupt Controller。
- 作用：管理中断使能、优先级、抢占和响应顺序。
- 规则：通常数值越小，优先级越高。

### ISR

- 全称：Interrupt Service Routine。
- 作用：CPU 响应 IRQ 后进入的真正中断服务入口。
- 示例：`USART1_IRQHandler()`、`EXTI15_10_IRQHandler()`。

### HAL Callback

- 是什么：HAL 提供给用户实现高层逻辑的回调接口。
- 关键理解：Callback 不是硬件中断机制的必需组成，也不等于 ISR。
- 工程作用：把底层中断处理与用户业务逻辑解耦。

### PWM

- 全称：Pulse Width Modulation，脉宽调制。
- 作用：通过固定周期内高低电平时间比例控制平均输出效果。
- 关键参数：PSC、ARR、CCR。
- 关键边界：PWM 硬件启动后会持续运行，函数返回不等于 PWM 停止。

### FIFO

- 全称：First In, First Out。
- 中文：先进先出。
- 当前用途：保证 UART 原始字节按照接收顺序被 main 处理。

### 环形队列

- 作用：在固定数组空间中通过 head / tail 回绕重复利用存储空间。
- 当前结构：`buf + head + tail + count`。
- 空：`count == 0`
- 满：`count == BYTE_QUEUE_SIZE`

### Critical Section

- 中文：临界区。
- 作用：保护中断和主循环共同访问的共享状态，避免数据竞争。
- 当前方法：保存 PRIMASK → 暂时关中断 → pop → 恢复 PRIMASK。

### 异步任务

- 当前典型接口：`HAL_UART_Receive_IT()`。
- 关键理解：函数调用返回只代表“任务已启动/配置”，不代表接收任务已经结束。

### 模块化封装

- 核心原则：按功能职责拆分 `.c/.h`。
- `.h`：公开接口、必要数据类型和宏。
- `.c`：具体实现、内部变量、辅助函数。
- `static`：隐藏仅模块内部使用的实现。
- `main.c`：初始化 + 调度。

---

## 5. 今天遇到的问题

| 问题现象 | 原因 | 解决过程 | 当前状态 |
|---|---|---|---|
| F405 Timer Clock ×2 规则不熟 | 对 APB 分频和 Timer Clock 关系记忆不牢 | 重新按 `PCLK + APB Prescaler` 梳理，明确分频为 1 和大于 1 的两种规则 | 基本掌握 |
| Timer 周期单位一度计算错误 | 1 MHz CNT 下 tick 与 μs / ms 的换算不熟 | 明确 `1 tick = 1 μs`，`1000 tick = 1 ms` | 已纠正 |
| 按键按下 / 松开电平判断写反 | 没先确认硬件默认电平 | 重新确认“默认 LOW、按下 HIGH”，再解释双边沿 Callback | 已解决 |
| 呼吸灯 CH2 与 CH1 同步变化 | 两路 CCR 都使用了相同 `breath_value` | CH2 改为 `999U - breath_value` | 已解决 |
| 误认为停止调用处理函数就等于 PWM 停止 | 混淆了 CPU 函数边界和 Timer 硬件任务边界 | 区分 `SetCompare(0)`、不再更新 CCR、`HAL_TIM_PWM_Stop()` | 已掌握 |
| `tick_flag` 与计数器的能力混淆 | 没区分“发生过”与“发生次数” | 对比 flag 和 counter，并讨论实时任务补算问题 | 已纠正 |
| UART 配置好后对“第三层”理解不够准确 | 把接收任务和 Callback 混成一层 | 重新区分“外设准备 → NVIC → Receive_IT 任务 → Callback” | 已纠正 |
| 不清楚 `rx_data` 应放哪里 | 没把异步任务生命周期和变量地址有效性联系起来 | 从生命周期和 Callback 访问两个角度分析，使用长期有效变量 | 已掌握 |
| 队列容量 16 时误认为 `count=15` 已满 | 混淆数组最大下标和元素数量 | 明确下标 `0~15`，容量仍是 16 | 已纠正 |
| `byte_queue_t` 编译时报 unknown type name | 头文件中函数声明早于 `typedef` | 调整为先定义类型，再声明接口 | 已解决，0 Error / 0 Warning |
| 队列满时最初想覆盖最旧数据 | 误认为最旧数据可能已经被 CPU 处理 | 认识到“还在队列里就代表尚未 pop”，当前改为拒绝新数据 | 已纠正 |
| 不清楚 main 和 Callback 同时操作队列是否安全 | `count` 等状态由中断和主循环共同修改 | 学习 Critical Section、PRIMASK，并只保护 `byte_queue_pop()` | 初步掌握 |
| 一开始想用 `__enable_irq()` 恢复中断 | 会无条件改变原中断状态 | 改为保存并恢复 PRIMASK | 已理解 |
| 没有 USB-TTL，无法完成真实 UART 验证 | 当前手头硬件条件不足，现有板卡也不能仅凭外观确认可作为 USB-UART | 软件链继续完成，真实硬件链明确列为待验证，不冒险接线 | 待硬件 |
| 复盘后发现 `main.c` 功能过多 | 当前作业以学习链路为主，UART、Callback、command 等仍集中在 main | 制定以后按 `uart_app / timer_app / queue / command / state_machine` 等功能模块封装的原则 | 已形成新工程规范 |

---

## 6. 当前掌握情况

### 已经可以独立完成

- 使用 CubeMX 配置 STM32 常见基础外设并 Generate Code；
- 根据 HSE / PLL / AHB / APB 分析 SYSCLK、HCLK、PCLK；
- 根据 APB 分频判断常见 Timer Clock；
- 区分 EXTI Line、Pending、IRQ、ISR、HAL Handler 和 Callback；
- 配置 EXTI + NVIC，并理解共享 IRQ；
- 根据双边沿 GPIO 当前电平判断按下 / 松开；
- 实现简单按键消抖、短按 / 长按识别；
- 使用 Timer Tick 完成简单非阻塞周期调度；
- 启动、修改和停止 PWM；
- 计算 PSC / ARR / CCR 与 PWM 基本关系；
- 使用状态机组织 LED、按键和蜂鸣器业务；
- 使用 `HAL_UART_Receive_IT()` 启动单字节异步接收任务；
- 编写简单 `HAL_UART_RxCpltCallback()`，完成 push + 重新 Receive_IT；
- 从零实现基本 `uint8_t` 环形队列；
- 理解并实现 FIFO、空 / 满、head / tail 回绕；
- 用 PRIMASK 建立很短的临界区保护共享队列操作；
- 将 UART 接收与 `command_process()` 业务处理解耦；
- 在无串口硬件条件下进行软件链逻辑模拟；
- 通过 Keil 编译结果定位并解决声明顺序等 C 语言错误；
- 完成 Git 提交；
- 判断 `main.c` 是否承担了过多具体实现，并规划模块化结构。

### 基本掌握，但仍需后续实践强化

- HAL Timebase 的具体底层实现；
- NVIC 抢占和多中断同时发生时的复杂场景；
- 中断与主循环并发访问数据的完整同步方法；
- PRIMASK 的更底层机制；
- 高速 UART 输入情况下环形队列容量设计；
- 队列 overflow 检测与异常处理；
- 模块间接口设计、`static` 隐藏和依赖关系控制；
- 将当前“功能正确”的代码重构为更规范的多模块工程。

### 尚未完成 / 尚未正式学习

- H723 USART1 + USB-TTL 真实硬件收发；
- UART IRQ / Callback 的 H723 实物连续触发验证；
- TIM6 + UART 在 H723 上的真实并发验证；
- UART 完整协议的封包 / 拆包；
- 帧头、长度、校验、丢包与错误恢复；
- UART DMA；
- CPU / UART / DMA / Memory 的职责边界；
- DMA Buffer 与后续协议解析结构。

---

## 7. 今日总结

今天的学习可以分成两条主线。

第一条是 STM32F405 的综合学习与实物作业。完成了从时钟体系、HAL Timebase、EXTI、NVIC、按键双边沿检测、短按/长按和消抖，到 TIM2 周期 Tick、PWM、状态机、流水灯、呼吸灯和蜂鸣器的完整实践。过程中进一步明确了一个重要原则：不能把“常见代码执行流程”直接当成“硬件机制上的必然关系”，需要区分硬件事件、IRQ/ISR、HAL Handler、Callback 和用户业务逻辑。F405 综合作业最终达到 `0 Error(s), 0 Warning(s)`，下载成功且实物现象正确。

第二条是在 STM32H723 的 8.9 工程上完成 UART 中断接收的软件工程实现。先通过 CubeMX 配置 USART1 115200 8N1、PB14/PB15 和 NVIC，再实现第一次 `HAL_UART_Receive_IT()`、UART Rx Callback、字节环形队列、main 中临界区 pop 和 `command_process()`。在这个过程中实际遇到了 `byte_queue_t` 声明顺序错误、队列容量判断、队列满策略、异步缓冲区生命周期以及中断与主循环共享数据等问题，并逐一修正。由于当前没有合适 USB-TTL，本次只完成软件链、逻辑推演、编译和 Git 闭环，没有把模拟输入误写成真实 UART 硬件验证。

今天还形成了后续工程的重要代码组织原则：以后状态机、UART、Timer、Queue、Command、电机等功能尽量分别封装到对应 `.c/.h` 中，模块内部实现尽量通过 `static` 隐藏，`main.c` 只保留系统初始化、模块初始化和主循环调度。CubeMX 自动生成的 ISR 入口仍保留在 `stm32xxxx_it.c` 等底层文件中，HAL Callback 和用户业务逻辑则根据功能职责进入相应模块。

今天最值得记住的工程思想是：

```text
硬件机制
≠ HAL封装
≠ Callback
≠ 用户业务逻辑
```

以及：

```text
中断负责快速响应
队列负责缓存和解耦
main负责业务处理
模块负责隐藏具体实现
```

---

## 8. 下一步任务

- [ ] 有合适 USB-TTL 后完成 H723 USART1 硬件验证
- [ ] 确认 PB14 / PB15 的实际板级引出和电气条件
- [ ] 实测 PC 115200 8N1 → STM32 USART1
- [ ] 验证 `USART1_IRQHandler()` 和 `HAL_UART_RxCpltCallback()` 连续触发
- [ ] 实测 `'0'/'1'/'2'/'3'` 命令是否能正确进入队列并改变业务状态
- [ ] 验证 TIM6 与 USART1 同时运行时的实际表现
- [ ] 后续工程默认按功能模块建立 `.c/.h`
- [ ] 优先采用 `uart_app / timer_app / byte_queue / command / state_machine` 等职责划分
- [ ] 继续学习 UART DMA
- [ ] 重点理解 CPU / UART / DMA / Memory 的职责边界
- [ ] 后续再进入 Buffer、协议解析、封包 / 拆包等更高层通信内容
