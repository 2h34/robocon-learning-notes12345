# 2026-09-12 传感器组考核 Day 3 学习记录
## 主题：Probability / Gaussian / Covariance 与 Kalman Filter

> 学习目标：从“已经能够利用 IMU 做姿态 Prediction”进一步过渡到“能够描述 Prediction 与 Observation 的不确定性，并理解 Kalman Filter 如何依据这些不确定性进行融合”。

---

# 一、Day 3 在整体学习路线中的位置

五天理论路线：

- **Day 1：三维 Rotation 与 Quaternion**
- **Day 2：SO(3) / so(3) / Exp / Log + IMU Quaternion Propagation**
- **Day 3：Probability / Gaussian / Covariance + Kalman Filter**
- **Day 4：EKF + Error-State EKF + 6D Attitude ESKF**
- **Day 5：15D Position Extension + 题目二完整数学模型**

今天的核心不是重新学习姿态表示，也不是提前推 Quaternion Error-State，而是解决：

\[
\boxed{
Prediction + Observation + Uncertainty
\rightarrow Kalman\ Fusion
}
\]

Day 2 已经解决：

\[
\omega_m
\rightarrow
\hat{\omega}
\rightarrow
\Delta\theta
\rightarrow
\Delta q
\rightarrow
q_{k+1}
\]

也就是：

\[
\boxed{
IMU \rightarrow Quaternion\ Prediction
}
\]

Day 3 则进一步解决：

\[
\boxed{
如何描述 Prediction 和 Observation 各自的可信程度，并据此完成融合
}
\]

---

# 二、阶段 0：Day 2 主线快速唤醒

## 2.1 Gyroscope 输出的是什么？

Gyroscope（陀螺仪）直接输出：

**Angular Velocity = 角速度**

记为：

\[
\omega_m=
\begin{bmatrix}
\omega_x\\
\omega_y\\
\omega_z
\end{bmatrix}
\]

单位：

\[
rad/s
\]

它**不是直接输出姿态**。

姿态需要通过：

\[
Angular\ Velocity
\rightarrow
Integration
\rightarrow
Attitude
\]

得到。

---

## 2.2 为什么纯 Gyroscope Integration 会 Drift？

陀螺仪测量模型：

\[
\omega_m
=
\omega_{true}
+
b_g
+
n_g
\]

其中：

- \(b_g\)：Gyroscope Bias，陀螺仪零偏；
- \(n_g\)：Gyroscope Noise，陀螺仪噪声。

进行 Bias Compensation（零偏补偿）后：

\[
\hat{\omega}
=
\omega_m-\hat b_g
\]

但即使已经补偿，也可能仍然存在：

\[
\delta b_g=b_g-\hat b_g
\]

以及测量噪声。

这些 Angular Velocity Error（角速度误差）经过时间积分：

\[
\delta \theta(t)
\approx
\int_0^t \delta \omega(\tau)d\tau
\]

会逐渐累积成 Attitude Error（姿态误差），最终形成 Drift（漂移）。

今天自己的理解：

> \(\omega\) 总会有小偏差，可能来自 noise，也可能来自 bias 的变化。时间长了以后，这个误差会经过积分不断累积，最终产生 drift。

这一理解是正确的。

核心因果链：

\[
\boxed{
Gyro\ Error
\rightarrow
Integration
\rightarrow
Attitude\ Error\ Accumulation
\rightarrow
Drift
}
\]

---

## 2.3 \(\Delta\theta\) 的物理意义

\[
\Delta\theta
=
(\omega_m-\hat b_g)\Delta t
\]

这里的：

\[
\Delta\theta
\]

表示的是：

**Rotation Vector = 旋转向量**

更准确地说，是：

> 在这一小段时间 \(\Delta t\) 内发生的增量旋转对应的旋转向量。

可写成：

\[
\Delta\theta=\theta n
\]

其中：

- \(n\)：旋转轴方向；
- \(\theta\)：这段时间内转过的角度。

然后：

\[
\Delta\theta
\rightarrow
\Delta q
\]

再用于 Quaternion Prediction（四元数姿态预测）。

---

## 2.4 为什么已经有一个具体 Quaternion 仍然不能认为姿态绝对准确？

例如算法得到：

\[
\hat q=
[0.01,0.02,0.10,0.994]
\]

首先需要区分两个问题。

### Quaternion normalization（四元数归一化）

归一化要求：

\[
\|q\|=1
\]

它解决的是：

> 当前 Quaternion 表示是否合法。

但即使一个 Quaternion 完全满足单位约束，也不代表它就是真实姿态。

### State uncertainty（状态不确定性）

姿态估计仍然可能受到：

- Gyroscope noise；
- residual bias；
- 数值积分误差；
- 模型近似；
- 初始姿态误差；

等因素影响。

所以：

\[
\hat q
\]

只是当前的 Estimated State（状态估计值），而不是“绝对真实值”。

---

# 三、为什么状态估计需要 Probability

## 3.1 一个状态估计不能只有一个数值

例如两个算法都输出：

\[
Yaw=10^\circ
\]

但：

算法 A：

\[
10^\circ \pm 0.1^\circ
\]

算法 B：

\[
10^\circ \pm 20^\circ
\]

虽然 Estimated State 一样，但 confidence（可信程度）完全不同。

所以：

\[
\boxed{
State\ Estimation
=
State\ Estimate
+
State\ Uncertainty
}
\]

也就是，状态估计算法必须同时回答：

1. 我认为状态是多少？
2. 我对这个判断有多确定？

---

# 四、Random Variable：随机变量

**Random Variable = 随机变量**

一个重要边界：

机器人某一时刻真实的 Yaw、Position、Velocity 等真实状态，实际上具有一个确定值。

例如：

\[
Yaw_{true}=10.3^\circ
\]

它不是物理上“随机在不同角度之间跳”。

真正随机建模的是：

> 我们对未知真实状态的认识。

因此在状态估计里，用 Random Variable：

\[
X
\]

描述：

\[
\boxed{
真实状态可能位于哪些值，以及这些值分别有多可能
}
\]

需要区分三个量：

### True State = 真实状态

\[
x_{true}
\]

客观存在，但通常未知。

### Estimated State = 状态估计值

\[
\hat x
\]

算法当前给出的具体估计结果。

### Random Variable = 随机变量

\[
X
\]

用于描述我们对未知真实状态的 Probability Distribution（概率分布）。

---

# 五、Mean / Expected Value：均值 / 期望

**Mean / Expected Value = 均值 / 期望**

写作：

\[
\mu=E[X]
\]

例如测量：

\[
9.9,\ 10.1,\ 10.0,\ 10.2,\ 9.8
\]

它们整体围绕：

\[
10
\]

分布，所以 Mean 约为 10。

工程直觉：

\[
\boxed{
Mean
=
Probability\ Distribution\ 的中心位置
}
\]

在 Kalman Filter 中：

\[
\hat x
\]

可以理解为当前状态分布的中心或最佳估计。

注意：

\[
\hat x
\]

是最佳估计，不代表：

\[
\hat x=x_{true}
\]

一定成立。

---

# 六、Variance：方差

**Variance = 方差**

定义：

\[
\boxed{
\sigma^2
=
E[(X-\mu)^2]
}
\]

它描述：

\[
\boxed{
随机变量在 Mean 周围分散得有多厉害
}
\]

例如两个传感器 Mean 都是 10：

Sensor A：

\[
9.99,\ 10.01,\ 10.00,\ldots
\]

Sensor B：

\[
8,\ 12,\ 9,\ 11,\ldots
\]

虽然 Mean 一样，但 B 更不稳定，因此 B 的 Variance 更大。

关系：

\[
\sigma^2\text{ 小}
\Rightarrow
Distribution\ 更集中
\Rightarrow
Uncertainty\ 更小
\]

\[
\sigma^2\text{ 大}
\Rightarrow
Distribution\ 更分散
\Rightarrow
Uncertainty\ 更大
\]

---

## 6.1 Variance 与 Standard Deviation 的区别

**Variance = 方差**

\[
\sigma^2
\]

**Standard Deviation = 标准差**

\[
\sigma
\]

例如：

\[
\sigma=2^\circ
\]

那么：

\[
\sigma^2=4\ deg^2
\]

姿态滤波中一般采用弧度，因此姿态 Variance 常见单位：

\[
rad^2
\]

题目二中：

- `cov_33`
- `cov_44`
- `cov_55`

是姿态误差的 **Variance / Covariance diagonal elements**，不是 Standard Deviation。

---

# 七、Gaussian Distribution：高斯分布 / 正态分布

**Gaussian Distribution = 高斯分布 / 正态分布**

一维形式：

\[
X\sim\mathcal N(\mu,\sigma^2)
\]

其中：

- \(\mu\)：Mean，决定 Distribution 中心；
- \(\sigma^2\)：Variance，决定 Distribution 宽度。

工程上可先记：

\[
\boxed{
Gaussian
=
Mean
+
Variance
}
\]

例如：

\[
Yaw\sim\mathcal N(10^\circ,0.01\ deg^2)
\]

表示当前最可能在 \(10^\circ\) 附近，而且 uncertainty 很小。

而：

\[
Yaw\sim\mathcal N(10^\circ,25\ deg^2)
\]

虽然中心仍然是 \(10^\circ\)，但 uncertainty 大得多。

---

## 7.1 多维 Gaussian

多维状态：

\[
x\sim \mathcal N(\mu,\Sigma)
\]

其中：

- \(\mu\)：Mean Vector，均值向量；
- \(\Sigma\)：Covariance Matrix，协方差矩阵。

一维状态用一个 Variance 就可以描述 uncertainty。

多维状态则需要 Covariance Matrix。

---

# 八、Covariance：协方差

**Covariance = 协方差**

假设：

\[
x=
\begin{bmatrix}
p\\
v
\end{bmatrix}
\]

不仅要知道：

- position 自身多不确定；
- velocity 自身多不确定；

还要知道：

> position error 和 velocity error 是否存在统计关联。

Covariance 定义：

\[
Cov(X,Y)
=
E[(X-\mu_X)(Y-\mu_Y)]
\]

它描述的是：

\[
\boxed{
两个变量相对各自 Mean 的偏差是否具有一起变化的趋势
}
\]

例如：

如果 velocity 估计偏大时，position 通常也偏大，那么：

\[
Cov(p,v)>0
\]

如果一个偏大时另一个往往偏小，则：

\[
Cov(p,v)<0
\]

如果没有明显线性关联：

\[
Cov(p,v)\approx0
\]

---

# 九、Covariance Matrix：协方差矩阵

对于：

\[
x=
\begin{bmatrix}
p\\
v
\end{bmatrix}
\]

可写：

\[
P=
\begin{bmatrix}
\sigma_p^2 & Cov(p,v)\\
Cov(v,p) & \sigma_v^2
\end{bmatrix}
\]

核心结构：

\[
\boxed{
Diagonal\ Terms
=
每个 State 自己的 Variance
}
\]

\[
\boxed{
Off\text{-}Diagonal\ Terms
=
不同 State Error 之间的 Covariance
}
\]

例如：

\[
P=
\begin{bmatrix}
4&1.5\\
1.5&9
\end{bmatrix}
\]

含义：

\[
4=Var(p)
\]

\[
9=Var(v)
\]

\[
1.5=Cov(p,v)=Cov(v,p)
\]

Covariance Matrix 满足：

\[
P=P^T
\]

即它是对称矩阵。

---

# 十、一个非常重要的边界：Covariance ≠ Error

定义 Estimation Error：

\[
e=x-\hat x
\]

这是：

**Estimation Error = 估计误差**

但因为真实状态 \(x\) 通常未知，所以当前实际：

\[
e
\]

到底是多少，通常也不知道。

而：

\[
\boxed{
P
=
E[ee^T]
}
\]

描述的是：

**State Estimate Error Covariance = 状态估计误差协方差**

也就是：

\[
\boxed{
我们对 Error 的统计不确定性
}
\]

因此：

\[
P_{pp}=100
\]

不能理解为：

> 当前 position 一定错了 100 m。

如果：

\[
P_{pp}=100\ m^2
\]

表示的是 Position Error Variance。

对应 Standard Deviation：

\[
\sigma_p=10m
\]

所以：

\[
\boxed{
P\text{ 大}
\neq
当前实际 Error 一定大
}
\]

更准确的理解：

\[
P\text{ 大}
\Rightarrow
State\ Estimate\ uncertainty\ 大
\]

---

# 十一、P / Q / R：今天最重要的三个量

这是 Day 3 第一个核心验收点，也是最容易混淆的地方。

---

## 11.1 P：State Estimate Error Covariance

**State Estimate Error Covariance = 状态估计误差协方差**

\[
\boxed{
P
=
当前 State Estimate 的 uncertainty
}
\]

它回答：

> 我现在对当前 State Estimate 有多确定？

例如：

\[
P\text{ 小}
\Rightarrow
confidence\ 高
\]

\[
P\text{ 大}
\Rightarrow
uncertainty\ 高
\]

但不能说：

\[
P\text{ 大}
\Rightarrow
实际 State 一定错得多
\]

---

## 11.2 Q：Process Noise Covariance

**Process Noise = 过程噪声**

\[
w_k
\]

状态模型：

\[
x_k
=
Fx_{k-1}+w_k
\]

假设：

\[
w_k\sim \mathcal N(0,Q)
\]

其中：

\[
\boxed{
Q
=
Process\ Noise\ Covariance
}
\]

含义：

\[
\boxed{
Q
=
这一次 Prediction 过程中新增的 uncertainty
}
\]

IMU 题目中，Q 的来源包括：

- Gyroscope Noise；
- Gyroscope Bias Random Walk；
- 模型近似；
- 其他未建模扰动。

如果 IMU 很差：

\[
Q\uparrow
\]

表示每次 Prediction 都应该引入更多 uncertainty。

---

## 11.3 R：Measurement Noise Covariance

**Measurement Noise = 测量噪声**

观测模型：

\[
z_k
=
Hx_k+v_k
\]

假设：

\[
v_k\sim\mathcal N(0,R)
\]

其中：

\[
\boxed{
R
=
Measurement\ Noise\ Covariance
}
\]

含义：

\[
\boxed{
R
=
这一次 Observation 自身的 uncertainty
}
\]

如果 FAST-LIO 某一帧 covariance 很大，则：

\[
R\uparrow
\]

意味着这一帧 Observation 应该更少相信。

---

## 11.4 P / Q / R 的最终区分

今天自己的正确总结：

> P 描述目前 State Estimate 有多不确定；  
> Q 描述往前 Prediction 这一步新增加多少不确定性；  
> R 描述外部传感器的这一次 Observation 有多不确定。

最推荐记忆：

\[
\boxed{
P=\text{我现在有多不确定}
}
\]

\[
\boxed{
Q=\text{我预测这一步又增加多少不确定性}
}
\]

\[
\boxed{
R=\text{外部观测本身有多不确定}
}
\]

---

# 十二、State-Space Model：状态空间模型

**State-Space Model = 状态空间模型**

Linear System（线性系统）中：

状态模型：

\[
\boxed{
x_k
=
F_kx_{k-1}
+
B_ku_k
+
w_k
}
\]

Observation Model：

\[
\boxed{
z_k
=
H_kx_k
+
v_k
}
\]

其中：

- \(x_k\)：State，状态；
- \(u_k\)：Input / Control，输入 / 控制；
- \(F_k\)：State Transition Matrix，状态转移矩阵；
- \(B_k\)：Input Matrix，输入矩阵；
- \(w_k\)：Process Noise，过程噪声；
- \(z_k\)：Measurement / Observation，测量 / 观测；
- \(H_k\)：Observation Matrix，观测矩阵；
- \(v_k\)：Measurement Noise，测量噪声。

---

# 十三、F：State Transition Matrix

**State Transition Matrix = 状态转移矩阵**

\[
F
\]

描述：

\[
\boxed{
上一时刻 State 怎样传播到下一时刻 State
}
\]

例如：

\[
x=
\begin{bmatrix}
p\\
v
\end{bmatrix}
\]

恒速模型：

\[
p_k=p_{k-1}+v_{k-1}\Delta t
\]

\[
v_k=v_{k-1}
\]

对应：

\[
\boxed{
F=
\begin{bmatrix}
1&\Delta t\\
0&1
\end{bmatrix}
}
\]

逐项理解：

\[
F_{11}=1
\]

表示旧 position 保留到新 position。

\[
F_{12}=\Delta t
\]

表示 velocity 通过积分影响下一时刻 position。

今天自己的理解：

> 右上角 \(\Delta t\) 存在，是因为表达了 velocity 对下一时刻 position 的积分作用。

这是正确的。

\[
F_{21}=0
\]

表示恒速模型中 position 不影响下一时刻 velocity。

\[
F_{22}=1
\]

表示 velocity 保持不变。

所以：

\[
\boxed{
F
=
State\ propagation\ relationship
}
\]

---

# 十四、H：Observation Matrix

**Observation Matrix = 观测矩阵**

\[
H
\]

核心作用：

\[
\boxed{
把完整 State 映射到 Sensor 能观测的空间
}
\]

例如：

\[
x=
\begin{bmatrix}
position\\
velocity
\end{bmatrix}
\]

如果 Sensor 只测 Position：

\[
H=
\begin{bmatrix}
1&0
\end{bmatrix}
\]

如果 Sensor 只测 Velocity：

\[
H=
\begin{bmatrix}
0&1
\end{bmatrix}
\]

今天这里出现了一个理解问题：

一开始把“只测 velocity”的 H 写成：

\[
\begin{bmatrix}
0&0\\
0&1
\end{bmatrix}
\]

后来明确：

如果：

\[
x\in\mathbb R^n
\]

而：

\[
z\in\mathbb R^m
\]

那么：

\[
\boxed{
H\in\mathbb R^{m\times n}
}
\]

也就是：

\[
\boxed{
H:
State\ Space
\rightarrow
Measurement\ Space
}
\]

如果 State 是 2D，但 Measurement 只有 1D，则：

\[
H\in\mathbb R^{1\times2}
\]

所以正确写法：

\[
H=
\begin{bmatrix}
0&1
\end{bmatrix}
\]

---

# 十五、F 与 H 的区别

这是后面 EKF 必须清楚的边界。

\[
\boxed{
F:
State\ 怎样随时间传播
}
\]

\[
\boxed{
H:
Sensor\ 能从 State 中看到什么
}
\]

即：

\[
x_{k-1}
\xrightarrow{F}
x_k
\]

而：

\[
x_k
\xrightarrow{H}
Predicted\ Measurement
\]

---

# 十六、Prior / Posterior：先验 / 后验

今天第一次系统学习：

**Prior Estimate = 先验估计**

\[
\hat x_k^-
\]

表示：

> 第 \(k\) 帧已经完成 Prediction，但还没有融合当前 Observation。

**Posterior Estimate = 后验估计**

\[
\hat x_k^+
\]

表示：

> 第 \(k\) 帧已经完成 Observation Correction。

注意：

上标：

\[
-
\]

和：

\[
+
\]

不是数学上的“负”和“正”。

而是 Kalman Filter 流程阶段标记。

状态流程：

\[
\boxed{
\hat x_{k-1}^+
\rightarrow
Prediction
\rightarrow
\hat x_k^-
\rightarrow
Correction
\rightarrow
\hat x_k^+
}
\]

Covariance 同理：

\[
P_{k-1}^+
\rightarrow
P_k^-
\rightarrow
P_k^+
\]

今天已经能够正确理解：

\[
P_k^-
\]

表示：

> 第 \(k\) 帧 Observation 融合之前的状态 uncertainty。

---

# 十七、Kalman Filter Prediction —— State

State Prediction：

\[
\boxed{
\hat x_k^-
=
F_k\hat x_{k-1}^+
+
B_ku_k
}
\]

它表示：

> 使用 Motion Model，把上一帧 corrected state 推到当前时刻。

例如：

\[
\hat x_{k-1}^+
=
\begin{bmatrix}
10\\
2
\end{bmatrix}
\]

\[
\Delta t=0.5s
\]

\[
F=
\begin{bmatrix}
1&0.5\\
0&1
\end{bmatrix}
\]

得到：

\[
\hat x_k^-=
\begin{bmatrix}
11\\
2
\end{bmatrix}
\]

也就是：

\[
p_k^-=11m
\]

\[
v_k^-=2m/s
\]

---

# 十八、Kalman Filter Prediction —— Covariance

状态需要 Prediction，uncertainty 也必须 Prediction。

核心公式：

\[
\boxed{
P_k^-
=
F_kP_{k-1}^+F_k^T+Q_k
}
\]

---

## 18.1 为什么是 \(FPF^T\)

先从一维开始。

如果：

\[
y=ax
\]

而：

\[
Var(x)=\sigma_x^2
\]

由于：

\[
\delta y=a\delta x
\]

所以：

\[
Var(y)
=
a^2Var(x)
\]

推广到多维：

\[
y=Fx
\]

误差满足：

\[
\delta y=F\delta x
\]

Covariance：

\[
P_y
=
E[\delta y\delta y^T]
\]

代入：

\[
\delta y=F\delta x
\]

得到：

\[
P_y
=
E[(F\delta x)(F\delta x)^T]
\]

最终：

\[
\boxed{
P_y
=
FP_xF^T
}
\]

所以：

\[
\boxed{
FPF^T
=
旧 uncertainty 根据 State propagation 关系进行传播
}
\]

---

## 18.2 为什么还要 +Q

\[
FPF^T
\]

只考虑：

> 上一帧已经存在的 uncertainty 如何传播。

而本轮 Prediction 本身还会新引入：

- Gyroscope noise；
- bias random walk；
- model uncertainty；

等。

所以：

\[
+Q
\]

表示：

\[
\boxed{
本次 Prediction 新加入的 Process uncertainty
}
\]

最终：

\[
\boxed{
P_k^-
=
旧 uncertainty 的传播
+
新的 Process uncertainty
}
\]

今天正确回答：

即使：

\[
Q=0
\]

也不等于：

\[
P_k^-=0
\]

因为旧 uncertainty 仍会通过：

\[
FPF^T
\]

继续传播。

---

# 十九、Observation 到来后：Residual / Innovation

Prediction 完成后：

\[
\hat x_k^-
\]

Sensor 提供：

\[
z_k
\]

Prediction 认为 Sensor 理论上应该看到：

\[
H\hat x_k^-
\]

所以：

\[
\boxed{
r_k
=
z_k-H\hat x_k^-
}
\]

其中：

**Residual = 残差**

**Innovation = 新息 / 创新量**

例如：

\[
z=5.0
\]

\[
H\hat x^-=4.8
\]

则：

\[
r=0.2
\]

---

## 19.1 Residual ≠ True Error

这是一个重要边界。

\[
r=0.2
\]

只能说明：

> Observation 和 Prediction 相差 0.2。

不能说明：

> Prediction 相对真实状态一定错了 0.2。

因为：

\[
z
\]

自己也存在 Measurement Noise。

今天自己的正确理解：

> 不能直接把 residual 当真实误差，因为观测值本身也可能不准确。

---

# 二十、Innovation Covariance S：新息协方差

**Innovation Covariance = 新息协方差**

\[
\boxed{
S
=
HP^-H^T+R
}
\]

它描述：

\[
\boxed{
Residual / Innovation 自身的不确定性
}
\]

其中：

\[
HP^-H^T
\]

表示：

> Prediction uncertainty 映射到 Measurement Space 后的 uncertainty。

而：

\[
R
\]

表示：

> Measurement 自己的 uncertainty。

所以：

\[
\boxed{
S
=
Prediction\ uncertainty\ in\ measurement\ space
+
Measurement\ uncertainty
}
\]

S 不是没有物理意义的“中间矩阵”。

---

# 二十一、Kalman Gain：卡尔曼增益

**Kalman Gain = 卡尔曼增益**

\[
\boxed{
K
=
P^-H^TS^{-1}
}
\]

即：

\[
K
=
P^-H^T
(HP^-H^T+R)^{-1}
\]

一维并且：

\[
H=1
\]

时：

\[
\boxed{
K=
\frac{P^-}{P^-+R}
}
\]

---

## 21.1 K 的核心直觉

如果：

\[
P^-\gg R
\]

说明：

- Prediction 很不确定；
- Measurement 比较可靠。

则：

\[
K\rightarrow1
\]

所以：

\[
\boxed{
更相信 Measurement
}
\]

如果：

\[
P^-\ll R
\]

说明：

- Prediction 很可靠；
- Measurement 很不可靠。

则：

\[
K\rightarrow0
\]

所以：

\[
\boxed{
更相信 Prediction
}
\]

因此：

\[
\boxed{
Kalman\ Gain
=
根据 Prediction uncertainty 和 Measurement uncertainty 自动计算的动态融合权重
}
\]

它不是人为固定设置的常数。

---

# 二十二、State Update：状态更新

\[
\boxed{
\hat x^+
=
\hat x^-+Kr
}
\]

也就是：

\[
\hat x^+
=
\hat x^-
+
K(z-H\hat x^-)
\]

直觉：

\[
\boxed{
从 Prediction 出发，沿着 Observation 指出的方向修正一部分
}
\]

修正多少由：

\[
K
\]

决定。

若：

\[
K\approx1
\]

则结果更靠近 Measurement。

若：

\[
K\approx0
\]

则结果更靠近 Prediction。

---

# 二十三、Covariance Update：协方差更新

融合 Observation 后，不仅 State 要更新，uncertainty 也要更新。

基础形式：

\[
\boxed{
P^+
=
(I-KH)P^-
}
\]

一维、\(H=1\)：

\[
P^+=(1-K)P^-
\]

如果 Measurement 很可靠：

\[
K\text{ 大}
\]

通常：

\[
P^+\ll P^-
\]

说明融合新信息后，我们对 State 更确定。

如果 Measurement 很差：

\[
R\text{ 大}
\Rightarrow
K\approx0
\]

则：

\[
P^+\approx P^-
\]

也就是说：

> 一个很不可靠的 Observation 不会让滤波器凭空变得非常有信心。

---

## 23.1 Joseph Form：Joseph 形式

扩展认识：

\[
\boxed{
P^+
=
(I-KH)P^-(I-KH)^T
+
KRK^T
}
\]

称为：

**Joseph Form = Joseph 形式**

目前只需要知道：

- 和基础 covariance update 目标相同；
- 工程实现中数值稳定性通常更好；
- 后续实际实现 EKF 时可再深入。

---

# 二十四、完整 Kalman Filter 循环

上一帧 Correction 后：

\[
\hat x_{k-1}^+,\quad P_{k-1}^+
\]

---

## Step 1：State Prediction

\[
\boxed{
\hat x_k^-
=
F\hat x_{k-1}^+
+
Bu
}
\]

---

## Step 2：Covariance Prediction

\[
\boxed{
P_k^-
=
FP_{k-1}^+F^T+Q
}
\]

---

## Step 3：Measurement

\[
z_k
\]

---

## Step 4：Residual / Innovation

\[
\boxed{
r_k
=
z_k-H\hat x_k^-
}
\]

---

## Step 5：Innovation Covariance

\[
\boxed{
S_k
=
HP_k^-H^T+R
}
\]

---

## Step 6：Kalman Gain

\[
\boxed{
K_k
=
P_k^-H^TS_k^{-1}
}
\]

---

## Step 7：State Correction

\[
\boxed{
\hat x_k^+
=
\hat x_k^-+K_kr_k
}
\]

---

## Step 8：Covariance Correction

\[
\boxed{
P_k^+
=
(I-K_kH)P_k^-
}
\]

然后：

\[
\hat x_k^+,P_k^+
\]

进入下一帧。

---

# 二十五、一维 Kalman Filter 手算例子 1

给定：

\[
x^-=4.8
\]

\[
P^-=0.4
\]

\[
z=5.0
\]

\[
R=0.1
\]

\[
H=1
\]

---

## 25.1 Kalman Gain

\[
K
=
\frac{0.4}{0.4+0.1}
=
0.8
\]

---

## 25.2 Residual

\[
r
=
5.0-4.8
=
0.2
\]

---

## 25.3 State Update

\[
x^+
=
4.8+0.8\times0.2
\]

\[
\boxed{
x^+=4.96
}
\]

结果明显更靠近 Measurement：

\[
5.0
\]

原因：

\[
P^->R
\]

所以 Measurement 相对更可靠。

---

## 25.4 Covariance Update

\[
P^+
=
(1-0.8)\times0.4
\]

\[
\boxed{
P^+=0.08
}
\]

说明 uncertainty 从：

\[
0.4
\]

降低到：

\[
0.08
\]

融合有效 Observation 后，滤波器对 State 更有信心。

---

## 25.5 今天这里出现的一个小错误

最初把：

\[
0.2\times0.4
\]

算成了：

\[
0.04
\]

后修正为：

\[
0.08
\]

这是单纯算术错误，不影响 Kalman Filter 概念理解。

---

# 二十六、一维 Kalman Filter 手算例子 2

第二组：

\[
x^-=4.8
\]

\[
z=5.0
\]

\[
P^-=0.1
\]

\[
R=10
\]

---

## 26.1 Kalman Gain

\[
K
=
\frac{0.1}{0.1+10}
\approx0.0099
\]

非常接近：

\[
0
\]

所以更相信 Prediction。

---

## 26.2 Residual

\[
r=0.2
\]

Residual 和第一组相同。

这说明：

\[
\boxed{
Residual 只描述 Prediction 与 Measurement 差多少
}
\]

而：

\[
\boxed{
这个差应该用于修正多少，由 K 决定
}
\]

---

## 26.3 State Update

\[
x^+
=
4.8+0.0099\times0.2
\]

\[
\boxed{
x^+\approx4.802
}
\]

几乎仍在 Prediction：

\[
4.8
\]

附近。

---

## 26.4 Covariance Update

\[
P^+
=
(1-0.0099)\times0.1
\]

\[
\boxed{
P^+\approx0.099
}
\]

几乎没有下降。

原因：

\[
R\gg P^-
\]

这一帧 Measurement 太不可靠，没有提供多少有效信息。

---

# 二十七、一维 KF 与矩阵 KF 的关系

一维里：

\[
P,Q,R,K
\]

是数字。

多维中：

\[
P,Q,R,K,F,H
\]

变成 Matrix（矩阵）。

但思想没有改变。

多维只是：

- 同时估计多个 State；
- State 之间可能存在 covariance；
- Sensor 不一定能观测全部 State；
- 需要 H 完成 State Space → Measurement Space 映射；
- uncertainty 需要 Covariance Matrix 表达。

所以：

\[
\boxed{
Matrix\ KF
\neq
另一套算法
}
\]

而是：

\[
\boxed{
一维 Kalman Filter 在多维相互关联 State 上的推广
}
\]

---

# 二十八、与题目二：IMU 姿态 EKF 的连接

## 28.1 Day 2 的 Gyroscope Integration 属于 Prediction

Day 2：

\[
\omega_m
\rightarrow
\hat{\omega}
\rightarrow
\Delta\theta
\rightarrow
\Delta q
\rightarrow
q_{k+1}
\]

在 EKF 框架中属于：

\[
\boxed{
Prediction
}
\]

---

## 28.2 FAST-LIO Pose 属于 Observation

题目中的：

`pose_cov.csv`

提供 FAST-LIO 位姿结果。

它在 EKF 中属于：

\[
\boxed{
Observation
}
\]

因此：

\[
\boxed{
IMU
\rightarrow
Prediction
}
\]

\[
\boxed{
FAST\text{-}LIO
\rightarrow
Observation
}
\]

---

## 28.3 pose_cov.csv 中 covariance 属于 R

题目规定：

position covariance：

- `cov_00`
- `cov_11`
- `cov_22`

attitude covariance：

- `cov_33`
- `cov_44`
- `cov_55`

所以姿态 Observation Noise Covariance 可概念上写成：

\[
\boxed{
R_\theta
=
diag(
cov_{33},
cov_{44},
cov_{55}
)
}
\]

位置选做：

\[
\boxed{
R_p
=
diag(
cov_{00},
cov_{11},
cov_{22}
)
}
\]

这些 covariance 属于：

\[
\boxed{
R
}
\]

因为它们描述的是 FAST-LIO Observation 本身的 uncertainty。

---

# 二十九、Day 3 的最终核心心智模型

今天所有公式背后的真正核心是：

\[
\boxed{
Kalman\ Filter
不是只维护 State Estimate，
还同时维护 State Estimate 的 Uncertainty
}
\]

每一帧有两条并行主线。

---

## State Estimate 主线

\[
\hat x_{k-1}^+
\rightarrow
\hat x_k^-
\rightarrow
\hat x_k^+
\]

---

## State Uncertainty 主线

\[
P_{k-1}^+
\rightarrow
P_k^-
\rightarrow
P_k^+
\]

Prediction 做两件事：

\[
\boxed{
传播 State
+
传播 / 增加 Uncertainty
}
\]

Observation 也携带两类信息：

\[
\boxed{
Measurement
+
Measurement\ Uncertainty
}
\]

Kalman Filter 再根据：

\[
P^-
\]

和：

\[
R
\]

判断：

> Prediction 与 Measurement 各应该相信多少。

---

# 三十、今天最重要的概念边界

以下边界必须保持清楚。

---

## 30.1 True State ≠ Estimated State

\[
x_{true}
\neq
\hat x
\]

一般情况下不能认为两者严格相同。

---

## 30.2 Error ≠ Covariance

\[
e=x-\hat x
\]

是实际 Estimation Error。

\[
P=E[ee^T]
\]

描述的是 Error 的统计 uncertainty。

---

## 30.3 Variance ≠ Standard Deviation

\[
Variance=\sigma^2
\]

\[
Standard\ Deviation=\sigma
\]

---

## 30.4 Covariance ≠ Correlation Coefficient

Covariance 能反映两个变量误差之间的联合变化关系，但不能直接和标准化后的 Correlation Coefficient（相关系数）等同。

---

## 30.5 P ≠ Q ≠ R

\[
P:
当前 State Estimate uncertainty
\]

\[
Q:
Prediction 新增 process uncertainty
\]

\[
R:
Observation uncertainty
\]

---

## 30.6 F ≠ H

\[
F:
State\ propagation
\]

\[
H:
State\rightarrow Measurement
\]

---

## 30.7 Residual ≠ True Error

\[
r=z-H\hat x^-
\]

只是 Observation 与 Prediction 的差。

因为 Observation 也有 Noise，所以它不是实际 State Error。

---

## 30.8 Kalman Gain 不是人为固定权重

\[
K
\]

由：

\[
P^-,H,R
\]

动态计算。

---

# 三十一、今天学习过程中出现的问题与修正

## 问题 1：Quaternion 归一化和 State Accuracy 混淆

开始时想到：

> 已经算出 Quaternion 后还需要归一化，所以不能认为绝对准确。

这个说法里包含两个不同层级。

### 归一化解决：

\[
\|q\|=1
\]

即 Quaternion 表示是否合法。

### Probability / Covariance 解决：

\[
\text{这个合法 Quaternion 到底有多可信}
\]

最终明确：

\[
\boxed{
Normalization
\neq
Uncertainty\ Estimation
}
\]

---

## 问题 2：Observation Matrix H 的维度

最初在“State 为 [position, velocity]，Sensor 只测 velocity”时写成：

\[
2\times2
\]

矩阵。

后来明确：

\[
H\in\mathbb R^{m\times n}
\]

其中：

- \(n\)：State Dimension；
- \(m\)：Measurement Dimension。

这里只有一个 measurement，所以：

\[
H=
\begin{bmatrix}
0&1
\end{bmatrix}
\]

---

## 问题 3：P 的物理含义容易和 Error 混淆

最终明确：

\[
P
\]

不是当前实际 Error。

它描述：

\[
\boxed{
Error 的统计 uncertainty
}
\]

所以不能说：

> P 大就说明当前一定错得很多。

---

## 问题 4：Covariance Update 中一次算术错误

\[
(1-0.8)\times0.4
\]

正确：

\[
0.08
\]

概念判断没有问题。

---

# 三十二、今天的新英文概念词汇表

| English | 中文 | 当前阶段含义 |
|---|---|---|
| Probability | 概率 | 描述未知状态的不确定认识 |
| Random Variable | 随机变量 | 用概率方式描述未知状态 |
| Probability Distribution | 概率分布 | 描述不同状态值可能性的整体分布 |
| Mean | 均值 | 分布中心 |
| Expected Value | 期望 | 概率意义上的平均值 |
| Variance | 方差 | 一维变量的不确定程度 |
| Standard Deviation | 标准差 | 方差的平方根 |
| Gaussian Distribution | 高斯分布 / 正态分布 | 用 Mean 与 Variance/Covariance 描述的常见分布 |
| Covariance | 协方差 | 两个变量误差联合变化的统计关系 |
| Covariance Matrix | 协方差矩阵 | 多维 State uncertainty 的矩阵表达 |
| State Estimate Error Covariance | 状态估计误差协方差 | KF 中的 P |
| Process Noise | 过程噪声 | Prediction model 中未准确建模的随机扰动 |
| Process Noise Covariance | 过程噪声协方差 | KF 中的 Q |
| Measurement Noise | 测量噪声 | Observation 自身的随机误差 |
| Measurement Noise Covariance | 测量噪声协方差 | KF 中的 R |
| State-Space Model | 状态空间模型 | 由状态模型和观测模型组成 |
| State Transition Matrix | 状态转移矩阵 | F，描述 State 如何传播 |
| Observation Matrix | 观测矩阵 | H，把 State 映射到 Measurement Space |
| Prediction | 预测 | 用 Motion Model 推进 State |
| Prior Estimate | 先验估计 | 当前帧观测融合前的估计 |
| Posterior Estimate | 后验估计 | 当前帧观测融合后的估计 |
| Residual | 残差 | Observation 与 Predicted Measurement 的差 |
| Innovation | 新息 / 创新量 | KF 中通常与 Residual 指同一量 |
| Innovation Covariance | 新息协方差 | S，描述 Residual uncertainty |
| Kalman Gain | 卡尔曼增益 | K，动态决定 Prediction 与 Measurement 融合权重 |
| State Update | 状态更新 | 用 Observation 修正 State |
| Covariance Update | 协方差更新 | Observation 融合后更新 uncertainty |
| Joseph Form | Joseph 形式 | 更数值稳定的 Covariance Update 表达 |
| Linear Kalman Filter | 线性卡尔曼滤波 | Day 3 学习的标准 KF |
| Measurement Space | 观测空间 | Sensor 实际输出所在空间 |
| State Space | 状态空间 | 完整 State 所在空间 |
| Confidence | 置信程度 / 可信程度 | 对 Estimate 的信任程度 |
| Uncertainty | 不确定性 | 对未知 State/Error 的统计不确定程度 |

---

# 三十三、当前掌握状态

## 已掌握

- Random Variable 的状态估计含义；
- Mean / Expected Value；
- Variance；
- Standard Deviation；
- Gaussian Distribution 基本含义；
- Covariance；
- Covariance Matrix 对角线 / 非对角线；
- Covariance ≠ Error；
- P / Q / R 的区别；
- State-Space Model；
- F 的作用；
- H 的作用；
- Prior / Posterior；
- State Prediction；
- Covariance Prediction；
- \(FPF^T\) 的基本来源；
- Residual / Innovation；
- Residual ≠ True Error；
- Innovation Covariance S；
- Kalman Gain K；
- State Update；
- Covariance Update；
- 一维 KF 手算；
- IMU = Prediction；
- FAST-LIO pose = Observation；
- pose covariance = R。

---

## 基本理解，但 Day 4 还需要进一步强化

- 多维 KF 中所有矩阵的具体维度关系；
- \(FPF^T\) 在真实 Error-State System 中的具体形式；
- H 在姿态 Error-State Observation 中如何构造；
- S / K 在多维情况下的具体矩阵运算；
- Quaternion Observation 如何转成 3D Attitude Residual。

---

## 今天尚未学习，留给 Day 4

- Extended Kalman Filter（EKF，扩展卡尔曼滤波）正式推导；
- Error-State EKF（误差状态扩展卡尔曼滤波）；
- Quaternion Error-State；
- \(\delta\theta\)；
- \(\delta b_g\)；
- 6D Error State；
- 6×6 \(F\)；
- Jacobian（雅可比矩阵）；
- Error Injection（误差注入）；
- Quaternion Residual；
- FAST-LIO Attitude Observation 与 ESKF 的具体结合。

---

# 三十四、Day 1 → Day 2 → Day 3 主线总结

## Day 1

解决：

\[
\boxed{
三维姿态怎样表示
}
\]

学习：

- Rotation Matrix；
- Rotation Vector；
- Euler Angle；
- Quaternion。

---

## Day 2

解决：

\[
\boxed{
IMU 怎样预测姿态
}
\]

学习：

- SO(3)；
- so(3)；
- Exp / Log；
- Gyroscope Measurement Model；
- Bias Compensation；
- \(\Delta\theta\)；
- \(\Delta q\)；
- Quaternion Propagation；
- Drift。

---

## Day 3

解决：

\[
\boxed{
Prediction 和 Observation 怎样根据 uncertainty 进行融合
}
\]

学习：

- Probability；
- Gaussian；
- Mean；
- Variance；
- Covariance；
- P / Q / R；
- F / H；
- Prediction；
- Residual；
- S；
- Kalman Gain；
- State Update；
- Covariance Update；
- 完整 Linear Kalman Filter。

---

# 三十五、进入 Day 4 前的最终理解

现在已经能够形成这一整条链：

\[
\boxed{
IMU
\rightarrow
Prediction
}
\]

Prediction 不只有：

\[
\hat x^-
\]

还同时有：

\[
P^-
\]

外部 FAST-LIO 提供：

\[
Observation
\]

以及：

\[
R
\]

Prediction 和 Observation 的差形成：

\[
Residual
\]

再通过：

\[
P^-、R、H
\]

得到：

\[
Kalman\ Gain
\]

最终完成：

\[
State\ Correction
\]

和：

\[
Covariance\ Correction
\]

所以今天真正理解的核心是：

\[
\boxed{
Kalman\ Filter
=
同时维护 State Estimate 与 State Uncertainty，
并依据双方的不确定性动态融合 Prediction 与 Observation
}
\]

---

# 三十六、Day 4 的衔接问题

Day 3 学习的是：

\[
\boxed{
Linear\ Kalman\ Filter
}
\]

但题目二真正的姿态状态包含：

\[
Quaternion
\]

Quaternion Rotation 是 Nonlinear（非线性）的，而且：

\[
Quaternion:
4\ parameters
\]

但 Rotation 实际只有：

\[
3\ DOF
\]

因此 Day 4 将回答：

> 普通 KF 的线性 State 模型无法直接完整处理 Quaternion Rotation，那么应该如何设计姿态 EKF？

最终目标：

\[
\boxed{
Quaternion\ Prediction
+
Error\ State
+
EKF
+
FAST\text{-}LIO\ Attitude\ Observation
}
\]

得到：

\[
\boxed{
6D\ Attitude\ Error\text{-}State\ EKF
}
\]

---

# 最终一句话总结

\[
\boxed{
Day\ 1：姿态怎么表示
\rightarrow
Day\ 2：IMU 怎么预测姿态
\rightarrow
Day\ 3：Prediction 和 Observation 怎么依据 Uncertainty 融合
}
\]

Day 4 才会进一步解决：

\[
\boxed{
如何把 Kalman Filter 的思想真正落到非线性的 Quaternion 姿态估计上
}
\]
