# 2026-08-12｜STM32 UART DMA 与 VOFA+ JustFloat

## 1. 基本信息

- 日期：2026-08-12
- 培训阶段：STM32 串口通信进阶与课堂作业
- 学习方向：UART、DMA、VOFA+ 上位机通信
- 内容来源：课堂逐字稿、老师作业、个人代码实践
- 学习时长：未记录

## 2. 今日目标

- [x] 理解 UART 中断接收与 DMA 接收的基本区别
- [x] 学习 `HAL_UARTEx_ReceiveToIdle_DMA()` 的基本使用
- [x] 理解 UART IDLE 事件、`Size` 与 `RxEventCallback`
- [x] 完成“接收数据后按字符 `1` 数量控制蜂鸣器鸣叫”的作业
- [x] 将蜂鸣器多次鸣叫实现为非阻塞状态机
- [x] 理解 VOFA+ RawData 与 JustFloat 的区别
- [x] 学习 `float` 原始字节与 `memcpy()` 的基本使用
- [x] 完成 JustFloat 数据帧拼接
- [x] 使用 STM32 持续发送 `sin` 数据，在 VOFA+ 中显示正弦波
- [x] 将 UART、VOFA、蜂鸣器功能继续按模块封装
- [x] 完成 Keil 编译、烧录和实际运行验证

## 3. 今日完成内容

### 3.1 UART DMA 与 ReceiveToIdle

今天在原有 UART 中断接收基础上进一步学习了 DMA 接收。

主要使用：

```c
HAL_UARTEx_ReceiveToIdle_DMA(&huart1, rx_buffer, sizeof(rx_buffer));
```

当前理解：

- UART 外设负责接收串口数据；
- DMA 负责将 UART 数据寄存器中的数据搬运到 RAM；
- CPU 不需要逐字节完成数据复制；
- `HAL_UARTEx_ReceiveToIdle_DMA()` 用于启动一次 DMA 接收任务；
- 函数调用返回后，UART 和 DMA 仍会继续完成这一轮接收；
- UART 出现 IDLE 等接收事件后，HAL 可以调用 `HAL_UARTEx_RxEventCallback()`；
- `Size` 在当前 Normal DMA、从缓冲区起始位置接收的场景下，可以理解为当前有效数据长度。

简化流程：

```text
上位机发送数据
→ UART 接收
→ DMA 搬运到 rx_buffer
→ UART 检测到 IDLE
→ IRQ / ISR
→ HAL 处理
→ HAL_UARTEx_RxEventCallback()
→ 主循环处理数据
```

### 3.2 作业一：接收数据并控制蜂鸣器

作业要求：

- 上位机发送一批数据；
- STM32 统计其中字符 `'1'` 的数量；
- 蜂鸣器响对应次数。

例如：

```text
1011  → 响 3 次
10001 → 响 2 次
0000  → 不响
```

程序中判断的是：

```c
if (rx_buffer[i] == '1')
{
    count++;
}
```

这里的 `'1'` 是 ASCII 字符，其值为 `0x31`，并不是数值 `0x01`。

本题将 UART 用户逻辑封装到：

```text
uart_app.c
uart_app.h
```

主循环只负责调用：

```c
uart_app_process(tick_2ms_count);
```

### 3.3 非阻塞蜂鸣器状态机

为了避免使用 `HAL_Delay()` 阻塞主循环，将多次蜂鸣封装成状态机。

状态包括：

```c
BEEP_STATE_IDLE
BEEP_STATE_ON
BEEP_STATE_OFF
```

主要思路：

```text
BEEP_Trigger(count)
→ 开始鸣叫

BEEP_Process()
→ ON 时间结束
→ 关闭蜂鸣器
→ 剩余次数减 1
→ 进入 OFF
→ 间隔结束后继续下一次
→ 全部完成后回到 IDLE
```

这样蜂鸣过程中主循环仍然可以继续处理 UART、VOFA 等任务。

### 3.4 VOFA+ RawData 与 JustFloat

今天区分了 VOFA+ 的两种使用方式。

RawData：

- 更接近普通串口助手；
- 不负责把采样数据解析成浮点通道；
- 适合发送和查看普通字符串、原始数据；
- 作业一可以用 RawData 向 STM32 发送 `1011`。

JustFloat：

- 用于解析二进制 `float` 数据；
- 可以把接收到的浮点数据形成数据通道；
- 适合实时波形显示；
- 作业二使用 JustFloat 显示 STM32 发送的正弦波。

需要注意：

> RawData 和 JustFloat 不是两种 UART，而是 VOFA+ 对串口数据采用的不同解析方式。

### 3.5 float 与 memcpy

JustFloat 需要发送 `float` 的原始二进制数据，而不是字符串。

例如：

```c
float value = 0.5f;
```

它和：

```c
char str[] = "0.5";
```

完全不同。

在 STM32F405 中，`float` 通常占 4 Byte。

使用：

```c
memcpy(tx_buffer, &value, sizeof(float));
```

可以把 `value` 在 RAM 中的 4 个原始字节复制到发送数组。

当前理解：

```text
memcpy(目标地址, 源地址, 复制字节数)
```

`memcpy()` 只是复制原始内存字节，并不会把 `float` 转换成字符串。

### 3.6 JustFloat 数据拼接

本次只发送一个 `float`，因此数据结构为：

```text
4 Byte float 数据
+
4 Byte JustFloat 包尾
=
8 Byte
```

发送数组：

```c
uint8_t tx_buffer[8];
```

JustFloat 包尾：

```c
uint8_t tail[4] = {0x00, 0x00, 0x80, 0x7F};
```

拼接方式：

```c
memcpy(&tx_buffer[0], &value, sizeof(float));
memcpy(&tx_buffer[4], tail, 4);
```

数组结构：

```text
[0] [1] [2] [3] | [4] [5] [6] [7]
 └ float 4 Byte ┘   └ JustFloat 尾 ┘
```

### 3.7 作业二：VOFA+ 正弦波

新建：

```text
vofa_app.c
vofa_app.h
```

核心逻辑：

```c
static float x = 0.0f;

float value = sinf(x);

x += 0.1f;

if (x >= 6.2831853f)
{
    x -= 6.2831853f;
}

memcpy(&tx_buffer[0], &value, sizeof(float));
memcpy(&tx_buffer[4], tail, 4);

HAL_UART_Transmit(&huart1, tx_buffer, sizeof(tx_buffer), 10);
```

运行过程：

```text
STM32 计算 sinf(x)
→ 得到一个 float
→ 拼接 JustFloat 数据帧
→ USART1 发送
→ VOFA+ JustFloat 解析
→ 出现 i0 数据通道
→ 波形图 Y 轴绑定 i0
→ 显示正弦波
```

VOFA+ 中：

- Y 轴选择 `i0`；
- X 轴保持默认，可以理解为采样点到达的先后顺序；
- STM32 内部变量 `x` 并没有发送给 VOFA，因此它不是 VOFA 图中的 X 轴。

### 3.8 当前程序模块划分

今天继续按照功能模块封装：

```text
main.c
→ 初始化 + 主循环调度

uart_app.c / uart_app.h
→ UART DMA 接收与数据处理

vofa_app.c / vofa_app.h
→ JustFloat 数据生成、拼接与发送

beep.c / beep.h
→ 蜂鸣器非阻塞控制

usart.c / dma.c / stm32f4xx_it.c
→ CubeMX / HAL 底层代码
```

`main.c` 中主要调用：

```c
while (1)
{
    vofa_app_process();
    uart_app_process(tick_2ms_count);
    BEEP_Process(tick_2ms_count);
}
```

## 4. 核心概念

### DMA

- 全称：Direct Memory Access，直接存储器访问。
- 是什么：一种让外设和内存之间可以由 DMA 控制器完成数据搬运的机制。
- 主要作用：减少 CPU 逐字节搬运数据的负担。
- 当前用途：将 USART1 接收到的数据自动搬运到 `rx_buffer`。

### UART IDLE

- 是什么：UART 硬件检测到接收线路处于空闲状态的事件。
- 当前作用：帮助判断当前这一批不定长数据暂时接收完成。
- 注意事项：IDLE 不是业务协议中的包尾，UART 并不知道一个协议包是否真正完整。

### `HAL_UARTEx_RxEventCallback()`

- 是什么：HAL 提供给用户处理 UART ReceiveToIdle 接收事件的回调接口。
- 当前用途：记录接收数据长度 `Size`、设置处理标志并准备下一轮接收。
- 注意事项：Callback 属于 HAL 封装层，不是硬件中断机制本身必需的组成。

### `Size`

- 是什么：HAL 传给 `RxEventCallback()` 的接收位置/有效数据范围信息。
- 当前场景：Normal DMA 且从 buffer 开头接收时，可近似理解为当前有效数据长度。
- 注意事项：后续 Circular DMA 中不能简单把每次 `Size` 都理解成“本次新收到的字节数”。

### `memcpy()`

- 是什么：C 标准库中的内存复制函数。
- 主要作用：按字节原样复制一段内存。
- 当前用途：把 `float` 的 4 Byte 原始数据复制到 UART 发送数组中。
- 注意事项：它不进行类型转换，也不会自动变成字符串。

### JustFloat

- 是什么：VOFA+ 用于二进制浮点数据采样和波形显示的数据引擎。
- 当前用途：解析 STM32 发来的 `float` 数据并生成波形通道。
- 本次结构：`float 数据 + 4 Byte 包尾`。

## 5. 今天遇到的问题

| 问题现象 | 原因 | 解决过程 | 当前状态 |
|---|---|---|---|
| 一开始担心 DMA 重新接收会覆盖 `rx_buffer`，影响主循环处理 | Callback 重新启动 DMA 后，DMA 与 CPU 理论上可能同时访问同一 buffer | 分析当前 115200 baud、人工低频发送和主循环处理速度后，确认本次作业实际风险很低；后续高速通信再学习双缓冲/环形缓冲 | 当前作业无需修改 |
| Half Transfer 知识点在代码检查时出现得过早 | 前面只简单提到 HT/TC，没有系统教学，却一度被当成当前代码问题 | 重新划分为“知道存在、当前不要求掌握”的 DMA 进阶内容 | 后续专题学习 |
| 一开始误以为 VOFA 勾选变量后会自动产生正弦波 | 没有区分“STM32 生成数据”和“VOFA 显示数据” | 对照老师逐字稿后明确：STM32 已经持续发送 `sin` 值，VOFA 勾选 `i0` 只是把数据通道绑定到波形图 | 已理解 |
| 一开始把 VOFA 作业设计得过于复杂 | 提前考虑了固定采样周期、精确频率、非阻塞发送等工程化问题 | 回到老师的最小作业要求，只实现 `sinf + memcpy + JustFloat + UART发送` | 作业完成 |
| 蜂鸣器重复调用 `BEEP_Trigger()` | 把“响 count 次”错误理解为外部循环调用 count 次 | 改为只调用一次 `BEEP_Trigger(current_tick, count)`，内部状态机负责完成多次鸣叫 | 已解决 |

## 6. 当前掌握情况

### 已经可以独立完成

- 配置并使用 UART 基本收发；
- 理解 DMA 在 UART 接收中的基本作用；
- 使用 `HAL_UARTEx_ReceiveToIdle_DMA()` 启动不定长 DMA 接收；
- 在 `HAL_UARTEx_RxEventCallback()` 中获取当前有效数据；
- 在主循环中处理 UART 接收结果；
- 统计 ASCII 字符 `'1'` 并触发对应次数蜂鸣；
- 使用非阻塞状态机实现蜂鸣器多次鸣叫；
- 使用 `memcpy()` 复制 `float` 原始字节；
- 拼接单通道 JustFloat 数据帧；
- 使用 `sinf()` 生成正弦采样点；
- 通过 UART 向 VOFA+ 持续发送数据；
- 在 VOFA+ 中使用 JustFloat 和 `i0` 显示正弦波；
- 将 UART、VOFA、蜂鸣器功能拆分到独立 `.c/.h` 模块中。

### 尚未系统理解

- DMA Half Transfer（HT）与 Transfer Complete（TC）的完整机制；
- `HAL_UARTEx_GetRxEventType()` 的使用；
- Normal DMA 与 Circular DMA 的完整区别；
- Circular DMA 中 `Size` 的精确含义；
- 高速连续接收时的 buffer 竞争；
- 双缓冲、环形缓冲和消息队列；
- UART DMA 发送；
- DMA TX buffer 生命周期；
- 精确采样周期与波形频率控制。

## 7. 今日总结

今天主要完成了 UART DMA 不定长接收和 VOFA+ JustFloat 波形显示的学习与实践。通过 `ReceiveToIdle_DMA` 理解了 UART、DMA、IDLE、Callback 和主循环之间的数据处理关系，并完成了串口数据统计 `'1'` 后控制蜂鸣器鸣叫的作业。随后学习了 `float` 原始字节、`memcpy()` 和 JustFloat 数据拼接，最终让 STM32 持续发送 `sin` 数据，并在 VOFA+ 中成功显示正弦波。

相比只记 HAL 函数，今天更重要的是开始从“数据从哪里产生、谁负责搬运、什么时候处理、如何封装、如何发送和显示”的完整数据链理解 UART 通信。

## 8. 下一步任务

- [x] 完成 UART DMA 接收基础
- [x] 完成 ReceiveToIdle DMA 作业
- [x] 完成非阻塞蜂鸣器多次鸣叫
- [x] 完成 VOFA+ JustFloat 正弦波作业
- [x] 完成本次编译、烧录和实际验证
- [ ] 后续系统学习 DMA HT / TC / IDLE 事件区别
- [ ] 学习 Normal DMA 与 Circular DMA
- [ ] 学习双缓冲、环形缓冲等连续数据接收方案
- [ ] 学习 UART DMA 发送和发送缓冲区生命周期
