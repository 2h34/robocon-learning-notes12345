# 2026-08-20 ZDrive + STM32 CAN 电机驱动学习记录

## 一、今日学习主题

进入 Robocon 控制组下一阶段学习：

**ZDrive + STM32 CAN 电机驱动**

目标：

- 理解 ZDrive 在机器人控制链中的位置；
- 理解 STM32 与 ZDrive 的职责划分；
- 学习 ZDrive CAN 通信模型；
- 阅读真实 ZDrive 驱动代码。

---

## 二、已有知识基础

已掌握：

- STM32 嵌入式 C；
- `.c/.h` 模块化；
- 状态机；
- 非阻塞程序；
- Timer / Interrupt / Callback；
- UART / DMA；
- FreeRTOS Task / Queue；
- CAN 基础；
- PID 与串级控制；
- DJI C610 电机控制链。

本阶段不重新学习这些内容。

---

## 三、ZDrive定位

ZDrive 不是电机，而是驱动器：

```
STM32
↓
ZDrive
↓
无刷电机
```

ZDrive负责：

- 接收控制目标；
- 处理编码器反馈；
- 执行内部控制；
- 驱动电机；
- 返回状态。

---

## 四、STM32与ZDrive职责

控制链：

```
Mechanism
↓
Motor
↓
Driver(ZDrive)
↓
Motor
```

STM32主要负责：

- 上层目标；
- 模式选择；
- 状态管理；
- 通信。

ZDrive主要负责：

- 驱动器内部控制；
- 电机反馈处理；
- 执行控制。

---

## 五、架构边界纠正

需要区分：

```
Driver层
=
软件职责概念

ZDrive.c
=
实际代码文件
```

二者不是一一对应。

不能认为 ZDrive.c 只负责协议转换。

---

## 六、Heartbeat学习

Heartbeat：

不是 STM32 请求。

而是：

```
ZDrive
↓
周期发送状态
↓
STM32接收
```

Ask：

```
STM32请求
↓
ZDrive返回
```

二者区别：

- Ask适合主动查询；
- Heartbeat适合周期状态反馈。

Heartbeat属于ZDrive内部参数配置。

---

## 七、CAN数据与小端

CAN只传输字节：

```
Data[0]
Data[1]
...
```

协议决定数据含义。

例如：

```
78 56 34 12
```

如果双方采用小端：

恢复：

```
0x12345678
```

---

## 八、ZDrive代码分析

重点函数：

### ZdriveInit()

初始化ZDrive对象。

### ZdriveFunc()

周期控制入口。

### ZdriveSet()

发送设置命令。

### ZdriveAsk()

发送查询请求。

### ZdriveReceive()

解析反馈并更新状态。

---

## 九、老师要求对应

### STM32 → ZDrive

包括：

- 模式；
- 目标位置；
- 目标速度；
- 参数配置；
- 查询请求。

### ZDrive → STM32

包括：

- 位置；
- 速度；
- 电流；
- 状态；
- 错误；
- Heartbeat。

---

## 十、今日问题与纠正

1. Driver层与文件结构混淆

纠正：

软件职责和代码文件不是同一概念。

2. Heartbeat与Ask混淆

纠正：

Heartbeat由ZDrive主动发送。

3. 协议学习顺序调整

正确顺序：

```
控制职责
↓
通信模型
↓
协议
↓
代码实现
```

---

## 十一、当前状态

已掌握：

- ZDrive是驱动器，不是电机；
- STM32与ZDrive职责划分；
- Heartbeat与Ask区别；
- CAN数据解释方法。

正在学习：

- ZDrive代码调用链；
- CAN发送链；
- CAN接收链；
- 工程验证。

---

## 十二、下一步

继续：

```
主控 ↔ ZDrive通信内容
↓
ZDrive内部控制职责
↓
ZDrive代码调用链
↓
CAN发送
↓
CAN接收
↓
实机验证
```
