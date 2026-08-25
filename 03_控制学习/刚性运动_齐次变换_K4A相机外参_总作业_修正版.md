# 刚性运动、齐次变换与 K4A 相机外参作业


# 1. 基本题

## 1.1 世界 \(x\) 轴、当前 \(z\) 轴、世界 \(y\) 轴旋转

题目操作顺序：

1. 关于世界坐标系 \(x\) 轴旋转 \(\phi\)；
2. 关于当前坐标系 \(z\) 轴旋转 \(\theta\)；
3. 关于世界坐标系 \(y\) 轴旋转 \(\psi\)。

第一步：

\[
R_1=R_x(\phi)
\]

第二步为当前轴旋转，因此右乘：

\[
R_2=R_x(\phi)R_z(\theta)
\]

第三步为固定轴旋转，因此左乘：

\[
\boxed{
R=R_y(\psi)R_x(\phi)R_z(\theta)
}
\]

---

## 1.2 世界 \(x\)、世界 \(z\)、当前 \(x\)、世界 \(z\) 旋转

题目操作顺序：

1. 世界 \(x\) 轴旋转 \(\phi\)；
2. 世界 \(z\) 轴旋转 \(\theta\)；
3. 当前 \(x\) 轴旋转 \(\psi\)；
4. 世界 \(z\) 轴旋转 \(\alpha\)。

依次有：

\[
R_1=R_x(\phi)
\]

\[
R_2=R_z(\theta)R_x(\phi)
\]

当前 \(x\) 轴旋转，右乘：

\[
R_3=R_z(\theta)R_x(\phi)R_x(\psi)
\]

最后绕固定 \(z\) 轴旋转，左乘：

\[
\boxed{
R=
R_z(\alpha)
R_z(\theta)
R_x(\phi)
R_x(\psi)
}
\]

---

## 1.3 两次旋转的叠加

题目：

1. 首先绕 \(x\) 轴旋转 \(\pi/2\)；
2. 再绕**固定坐标系**的 \(y\) 轴旋转 \(\pi/2\)。

由于第二次为固定轴旋转：

\[
R=
R_y\left(\frac{\pi}{2}\right)
R_x\left(\frac{\pi}{2}\right)
\]

其中：

\[
R_x\left(\frac{\pi}{2}\right)
=
\begin{bmatrix}
1&0&0\\
0&0&-1\\
0&1&0
\end{bmatrix}
\]

\[
R_y\left(\frac{\pi}{2}\right)
=
\begin{bmatrix}
0&0&1\\
0&1&0\\
-1&0&0
\end{bmatrix}
\]

因此：

\[
\boxed{
R=
\begin{bmatrix}
0&1&0\\
0&0&-1\\
-1&0&0
\end{bmatrix}
}
\]

从矩阵三列可以直接读出最终坐标轴：

\[
\boxed{
x_1=-z_0,\qquad
y_1=+x_0,\qquad
z_1=-y_0
}
\]

坐标系关系示意：

```text
初始：
x0, y0, z0 为标准右手系

最终：
x1 指向 -z0
y1 指向 +x0
z1 指向 -y0
```

---

## 1.4 坐标系之间的旋转变换

已知：

\[
{}^1_2R=
\begin{bmatrix}
1&0&0\\
0&\frac12&-\frac{\sqrt3}{2}\\
0&\frac{\sqrt3}{2}&\frac12
\end{bmatrix}
\]

\[
{}^1_3R=
\begin{bmatrix}
0&0&-1\\
0&1&0\\
1&0&0
\end{bmatrix}
\]

要求：

\[
{}^2_3R
\]

变换路径为：

\[
3\rightarrow1\rightarrow2
\]

因此：

\[
{}^2_3R
=
{}^2_1R\,{}^1_3R
\]

而旋转矩阵满足：

\[
{}^2_1R=({}^1_2R)^T
\]

所以：

\[
{}^2_3R
=
({}^1_2R)^T{}^1_3R
\]

最终：

\[
\boxed{
{}^2_3R=
\begin{bmatrix}
0&0&-1\\
\frac{\sqrt3}{2}&\frac12&0\\
\frac12&-\frac{\sqrt3}{2}&0
\end{bmatrix}
}
\]

---

## 1.5 平移、当前轴旋转、固定轴平移

操作：

1. 沿 \(x\) 轴平移 3；
2. 绕当前 \(z\) 轴旋转 \(\pi/2\)；
3. 沿固定 \(y\) 轴平移 1。

因此：

\[
T
=
T_y(1)
T_x(3)
R_z\left(\frac{\pi}{2}\right)
\]

旋转部分：

\[
R_z\left(\frac{\pi}{2}\right)
=
\begin{bmatrix}
0&-1&0\\
1&0&0\\
0&0&1
\end{bmatrix}
\]

最终齐次变换矩阵为：

\[
\boxed{
T=
\begin{bmatrix}
0&-1&0&3\\
1&0&0&1\\
0&0&1&0\\
0&0&0&1
\end{bmatrix}
}
\]

因此最终原点 \(o_1\) 相对于初始坐标系的位置为：

\[
\boxed{
{}^0p_{o_1}
=
\begin{bmatrix}
3\\
1\\
0
\end{bmatrix}
}
\]

最终坐标轴：

\[
x_1=+y_0,\qquad
y_1=-x_0,\qquad
z_1=+z_0
\]

---

# 2. 应用题

## 2.1 机器人、桌子、立方体与相机

根据题图：

- 基础坐标系为 \(\{0\}\)；
- 桌面距离地面 1 m；
- 机器人距离桌子 1 m；
- 桌面为 \(1\text{ m}\times1\text{ m}\)；
- 立方体边长为 0.2 m，放置于桌面中心；
- 坐标系 \(\{2\}\) 原点位于立方体中心；
- 相机位于立方体中心正上方，并且距离桌面 2 m。

### （1）桌子坐标系 \(\{1\}\) 相对于基础坐标系 \(\{0\}\)

从题图可知 \(\{1\}\) 与 \(\{0\}\) 的轴方向一致：

\[
{}^0_1R=I
\]

桌子原点相对于基础坐标系：

\[
{}^0p_1=
\begin{bmatrix}
0\\
1\\
1
\end{bmatrix}
\]

因此：

\[
\boxed{
{}^0_1T=
\begin{bmatrix}
1&0&0&0\\
0&1&0&1\\
0&0&1&1\\
0&0&0&1
\end{bmatrix}
}
\]

---

### （2）立方体坐标系 \(\{2\}\) 相对于基础坐标系 \(\{0\}\)

立方体中心相对于桌子坐标系：

\[
{}^1p_2=
\begin{bmatrix}
-0.5\\
0.5\\
0.1
\end{bmatrix}
\]

因此：

\[
{}^0p_2
=
{}^0p_1+{}^1p_2
=
\begin{bmatrix}
-0.5\\
1.5\\
1.1
\end{bmatrix}
\]

立方体初始轴方向与基础坐标系一致，因此：

\[
{}^0_2R=I
\]

得到：

\[
\boxed{
{}^0_2T=
\begin{bmatrix}
1&0&0&-0.5\\
0&1&0&1.5\\
0&0&1&1.1\\
0&0&0&1
\end{bmatrix}
}
\]

---

### （3）相机坐标系 \(\{3\}\) 相对于基础坐标系 \(\{0\}\)

根据题图坐标轴方向：

\[
x_3=+y_0
\]

\[
y_3=+x_0
\]

\[
z_3=-z_0
\]

因此：

\[
{}^0_3R=
\begin{bmatrix}
0&1&0\\
1&0&0\\
0&0&-1
\end{bmatrix}
\]

相机位于立方体中心正上方，距离桌面 2 m，因此：

\[
{}^0p_3=
\begin{bmatrix}
-0.5\\
1.5\\
3
\end{bmatrix}
\]

故：

\[
\boxed{
{}^0_3T=
\begin{bmatrix}
0&1&0&-0.5\\
1&0&0&1.5\\
0&0&-1&3\\
0&0&0&1
\end{bmatrix}
}
\]

---

### （4）相机坐标系 \(\{3\}\) 相对于立方体坐标系 \(\{2\}\)

要求：

\[
{}^2_3T
\]

使用：

\[
{}^2_3T
=
({}^0_2T)^{-1}{}^0_3T
\]

得到：

\[
\boxed{
{}^2_3T=
\begin{bmatrix}
0&1&0&0\\
1&0&0&0\\
0&0&-1&1.9\\
0&0&0&1
\end{bmatrix}
}
\]

这里的 1.9 m 来自：

\[
3.0-1.1=1.9\text{ m}
\]

---

## 2.2 相机绕当前 \(z_3\) 轴旋转 \(90^\circ\)

相机绕自身当前 \(z_3\) 轴旋转，因此旋转矩阵右乘：

\[
{}^0_3R'
=
{}^0_3R\,R_z(90^\circ)
\]

即：

\[
{}^0_3R'
=
\begin{bmatrix}
0&1&0\\
1&0&0\\
0&0&-1
\end{bmatrix}
\begin{bmatrix}
0&-1&0\\
1&0&0\\
0&0&1
\end{bmatrix}
\]

得到：

\[
{}^0_3R'
=
\begin{bmatrix}
1&0&0\\
0&-1&0\\
0&0&-1
\end{bmatrix}
\]

相机只发生旋转，原点位置不变：

\[
{}^0p_3=
\begin{bmatrix}
-0.5\\
1.5\\
3
\end{bmatrix}
\]

因此：

\[
\boxed{
{}^0_3T'=
\begin{bmatrix}
1&0&0&-0.5\\
0&-1&0&1.5\\
0&0&-1&3\\
0&0&0&1
\end{bmatrix}
}
\]

桌子和立方体没有发生变化，所以：

\[
{}^0_1T,\quad{}^0_2T
\]

保持不变。

重新计算立方体到相机的变换：

\[
{}^2_3T'
=
({}^0_2T)^{-1}{}^0_3T'
\]

得到：

\[
\boxed{
{}^2_3T'=
\begin{bmatrix}
1&0&0&0\\
0&-1&0&0\\
0&0&-1&1.9\\
0&0&0&1
\end{bmatrix}
}
\]

---

## 2.3 方块绕 \(z_2\) 轴旋转并移动

> 本文按“2.3 是相对于 2.1 初始相机状态的独立变化”进行计算。  


方块绕自身 \(z_2\) 轴旋转 \(90^\circ\)：

\[
{}^1_2R=R_z(90^\circ)
=
\begin{bmatrix}
0&-1&0\\
1&0&0\\
0&0&1
\end{bmatrix}
\]

题目给出方块中心在坐标系 \(\{1\}\) 中：

\[
{}^1p_2=
\begin{bmatrix}
0\\
0.8\\
0.1
\end{bmatrix}
\]

因此：

\[
{}^1_2T=
\begin{bmatrix}
0&-1&0&0\\
1&0&0&0.8\\
0&0&1&0.1\\
0&0&0&1
\end{bmatrix}
\]

方块相对于基础坐标系：

\[
{}^0_2T
=
{}^0_1T\,{}^1_2T
\]

得到：

\[
\boxed{
{}^0_2T=
\begin{bmatrix}
0&-1&0&0\\
1&0&0&1.8\\
0&0&1&1.1\\
0&0&0&1
\end{bmatrix}
}
\]

再求相机相对于方块：

\[
{}^2_3T
=
({}^0_2T)^{-1}{}^0_3T
\]

得到：

\[
\boxed{
{}^2_3T=
\begin{bmatrix}
1&0&0&-0.3\\
0&-1&0&0.5\\
0&0&-1&1.9\\
0&0&0&1
\end{bmatrix}
}
\]

### 若 2.3 承接 2.2 的相机旋转状态

若相机仍保持 2.2 中绕 \(z_3\) 旋转 \(90^\circ\) 后的姿态，则：

\[
{}^2_3T=
\begin{bmatrix}
0&-1&0&-0.3\\
-1&0&0&0.5\\
0&0&-1&1.9\\
0&0&0&1
\end{bmatrix}
\]

---


# 3. K4A RGB 相机坐标系到机器人坐标系的实际变换

## 3.1 任务目标

求 K4A RGB camera frame \(\{C\}\) 相对于机器人坐标系 \(\{R\}\) 的齐次变换：

\[
\boxed{{}^R_CT}
\]

其形式为：

\[
{}^R_CT=
\begin{bmatrix}
{}^R_CR&{}^Rp_C\\
0&1
\end{bmatrix}
\]

并满足：

\[
{}^Rp={}^R_CR\,{}^Cp+{}^Rp_C
\]

---

## 3.2 坐标系定义

### 机器人坐标系 \(\{R\}\)

- 原点 \(O_R\)：四个轮中心连线的交点；
- \(+X_R\)：机器人开口方向；
- \(+Y_R\)：指向红色相机方向；
- \(+Z_R\)：竖直向上；
- 满足右手系。

### K4A RGB 相机坐标系 \(\{C\}\)

由于深度图最终会对齐到 RGB camera frame，本题采用 RGB 相机坐标系。

本次 CAD 测量中，相机坐标系原点取：

\[
\boxed{O_C=\text{RGB 大镜头圆心}}
\]

该取点作为本次几何外参作业中的相机原点。

---

## 3.3 最终采用方向向量法

最初通过 SolidWorks 测量相机轴与机器人参考轴之间的夹角，可以近似推得旋转关系；但由于夹角测量会涉及锐角、钝角、补角以及选取对象误差，最终改用更直接的方法：

> 直接读取 K4A 坐标轴在机器人坐标系中的三维方向向量，然后归一化构造方向余弦矩阵。

旋转矩阵为：

\[
\boxed{
{}^R_CR=
\begin{bmatrix}
{}^R\hat x_C&
{}^R\hat y_C&
{}^R\hat z_C
\end{bmatrix}
}
\]

即三列分别表示相机 \(x_C,y_C,z_C\) 三根单位轴在机器人坐标系中的表达。

---

## 3.4 SolidWorks 实测 \(x_C\) 方向

重新测得 K4A \(+x_C\) 方向向量：

\[
v_x=
\begin{bmatrix}
-17.57\\
-0.33\\
-66.73
\end{bmatrix}
\]

其长度为：

\[
\|v_x\|\approx 69.00512
\]

归一化：

\[
{}^R\hat x_C
=
\frac{v_x}{\|v_x\|}
\approx
\boxed{
\begin{bmatrix}
-0.254619\\
-0.004782\\
-0.967030
\end{bmatrix}
}
\]

---

## 3.5 SolidWorks 实测 \(y_C\) 方向

重新测得 K4A \(+y_C\) 方向向量：

\[
v_y=
\begin{bmatrix}
-11.85\\
37.04\\
2.94
\end{bmatrix}
\]

其长度为：

\[
\|v_y\|\approx 39.00036
\]

直接归一化得到：

\[
{}^R\hat y_{C,\mathrm{meas}}
\approx
\begin{bmatrix}
-0.303843\\
0.949735\\
0.075384
\end{bmatrix}
\]

两根实测单位向量点积：

\[
({}^R\hat x_C)^T({}^R\hat y_{C,\mathrm{meas}})
\approx -0.00007614
\approx0
\]

说明两次测量已经非常接近严格正交。

---

## 3.6 利用右手系构造 \(z_C\)，并消除微小测量误差

相机坐标系满足右手系：

\[
x_C\times y_C=z_C
\]

因此利用两根实测方向构造：

\[
{}^R\hat z_C
=
\frac{{}^R\hat x_C\times{}^R\hat y_{C,\mathrm{meas}}}
{\left\|{}^R\hat x_C\times{}^R\hat y_{C,\mathrm{meas}}\right\|}
\]

得到：

\[
\boxed{
{}^R\hat z_C
\approx
\begin{bmatrix}
0.918061\\
0.313020\\
-0.243273
\end{bmatrix}
}
\]

由于 SolidWorks 测量存在极小误差，为使最终旋转矩阵严格满足正交条件，再利用：

\[
{}^R\hat y_C={}^R\hat z_C\times{}^R\hat x_C
\]

得到修正后的：

\[
\boxed{
{}^R\hat y_C
\approx
\begin{bmatrix}
-0.303863\\
0.949735\\
0.075310
\end{bmatrix}
}
\]

修正量极小，说明原始实测数据本身已经具有较好一致性。

---

## 3.7 最终旋转矩阵

将三根单位轴作为三列：

\[
{}^R_CR=
\begin{bmatrix}
{}^R\hat x_C&
{}^R\hat y_C&
{}^R\hat z_C
\end{bmatrix}
\]

得到：

\[
\boxed{
{}^R_CR
\approx
\begin{bmatrix}
-0.254619&-0.303863&0.918061\\
-0.004782&0.949735&0.313020\\
-0.967030&0.075310&-0.243273
\end{bmatrix}
}
\]

由构造方法可知：

\[
R^TR=I
\]

且：

\[
\det R=1
\]

因此这是一个合法的三维旋转矩阵。

---

## 3.8 旋转过程分解

为了按照“先绕哪个轴，再绕哪个轴”的形式描述最终姿态，选择：

\[
\boxed{
{}^R_CR=
R_y(\alpha)R_x(\beta)R_z(\gamma)
}
\]

其物理含义为：

1. 先绕**机器人固定 \(Y_R\) 轴**旋转 \(\alpha\)；
2. 再绕**当前 \(X\) 轴**旋转 \(\beta\)；
3. 最后绕**当前 \(Z\) 轴**旋转 \(\gamma\)。

对于：

\[
R_y(\alpha)R_x(\beta)R_z(\gamma)
\]

有：

\[
R_{23}=-\sin\beta
\]

\[
R_{21}=\sin\gamma\cos\beta,\qquad
R_{22}=\cos\beta\cos\gamma
\]

\[
R_{13}=\sin\alpha\cos\beta,\qquad
R_{33}=\cos\alpha\cos\beta
\]

所以：

\[
\beta=-\arcsin(R_{23})
\approx
\boxed{-18.241^\circ}
\]

\[
\gamma=\operatorname{atan2}(R_{21},R_{22})
\approx
\boxed{-0.289^\circ}
\]

\[
\alpha=\operatorname{atan2}(R_{13},R_{33})
\approx
\boxed{104.841^\circ}
\]

因此可以采用的一组旋转过程为：

\[
\boxed{
\text{固定 }Y_R:+104.841^\circ
\rightarrow
\text{当前 }X:-18.241^\circ
\rightarrow
\text{当前 }Z:-0.289^\circ
}
\]

即：

\[
\boxed{
{}^R_CR=
R_y(104.841^\circ)
R_x(-18.241^\circ)
R_z(-0.289^\circ)
}
\]

> 旋转分解并不唯一。  
> 上述三步是对最终相对姿态的一种数学分解，并不表示安装相机时必须实际依次执行这三个机械旋转动作。

---

## 3.9 平移向量

![alt text](86bf587953c002ffdae48717ed405279.png)

机器人坐标系原点取四轮中心连线交点。

K4A RGB 相机坐标系原点取 RGB 大镜头圆心。

SolidWorks 实测：

\[
x=322.17\text{ mm}
\]

\[
y=-334.86\text{ mm}
\]

\[
z=612.94\text{ mm}
\]

因此：

\[
\boxed{
{}^Rp_C=
\begin{bmatrix}
322.17\\
-334.86\\
612.94
\end{bmatrix}
\text{ mm}
}
\]

换成 SI 单位：

\[
\boxed{
{}^Rp_C=
\begin{bmatrix}
0.32217\\
-0.33486\\
0.61294
\end{bmatrix}
\text{ m}
}
\]

---

## 3.10 最终齐次变换矩阵

以 mm 为平移单位：

\[
\boxed{
{}^R_CT
\approx
\begin{bmatrix}
-0.254619&-0.303863&0.918061&322.17\\
-0.004782&0.949735&0.313020&-334.86\\
-0.967030&0.075310&-0.243273&612.94\\
0&0&0&1
\end{bmatrix}
}
\]

采用 SI 单位 m：

\[
\boxed{
{}^R_CT
\approx
\begin{bmatrix}
-0.254619&-0.303863&0.918061&0.32217\\
-0.004782&0.949735&0.313020&-0.33486\\
-0.967030&0.075310&-0.243273&0.61294\\
0&0&0&1
\end{bmatrix}
}
\]

于是 K4A 中任意一点：

\[
{}^Cp=
\begin{bmatrix}
x_C\\
y_C\\
z_C
\end{bmatrix}
\]

变换到机器人坐标系：

\[
\boxed{
{}^Rp
=
{}^R_CR\,{}^Cp
+
{}^Rp_C
}
\]

---

# 4. 最终结果汇总

## 4.1 《刚性运动和齐次变换》基本题

### 1.1

\[
\boxed{R=R_y(\psi)R_x(\phi)R_z(\theta)}
\]

### 1.2

\[
\boxed{
R=
R_z(\alpha)R_z(\theta)R_x(\phi)R_x(\psi)
}
\]

### 1.3

\[
\boxed{
R=
\begin{bmatrix}
0&1&0\\
0&0&-1\\
-1&0&0
\end{bmatrix}
}
\]

### 1.4

\[
\boxed{
{}^2_3R=
\begin{bmatrix}
0&0&-1\\
\frac{\sqrt3}{2}&\frac12&0\\
\frac12&-\frac{\sqrt3}{2}&0
\end{bmatrix}
}
\]

### 1.5

\[
\boxed{
T=
\begin{bmatrix}
0&-1&0&3\\
1&0&0&1\\
0&0&1&0\\
0&0&0&1
\end{bmatrix}
}
\]

\[
\boxed{o_1=(3,1,0)^T}
\]

应用题 2.1～2.3 的完整计算见正文。

---

## 4.2 K4A 最终结果

### 旋转矩阵

\[
\boxed{
{}^R_CR
\approx
\begin{bmatrix}
-0.254619&-0.303863&0.918061\\
-0.004782&0.949735&0.313020\\
-0.967030&0.075310&-0.243273
\end{bmatrix}
}
\]

### 一组旋转分解

\[
\boxed{
{}^R_CR=
R_y(104.841^\circ)
R_x(-18.241^\circ)
R_z(-0.289^\circ)
}
\]

即：

\[
\boxed{
\text{固定 }Y_R:+104.841^\circ
\rightarrow
\text{当前 }X:-18.241^\circ
\rightarrow
\text{当前 }Z:-0.289^\circ
}
\]

### 平移向量

\[
\boxed{
{}^Rp_C=
\begin{bmatrix}
0.32217\\
-0.33486\\
0.61294
\end{bmatrix}
\text{ m}
}
\]

### 最终齐次变换

\[
\boxed{
{}^R_CT
\approx
\begin{bmatrix}
-0.254619&-0.303863&0.918061&0.32217\\
-0.004782&0.949735&0.313020&-0.33486\\
-0.967030&0.075310&-0.243273&0.61294\\
0&0&0&1
\end{bmatrix}
}
\]

---

# 5. 本次作业中的关键结论与说明

1. 旋转矩阵的三列分别表示目标坐标系三根单位轴在参考坐标系中的表达：

\[
R=
\begin{bmatrix}
\hat x&\hat y&\hat z
\end{bmatrix}
\]

2. 固定轴旋转左乘，当前轴旋转右乘。

3. 相对位姿应沿正确的坐标变换链计算，例如：

\[
{}^2_3T=({}^0_2T)^{-1}{}^0_3T
\]

4. 平移向量需要明确“从哪个原点指向哪个原点”以及“用哪个坐标系表达”。

5. 实际 K4A 安装存在三维倾斜，不能假设相机坐标轴与机器人坐标轴简单平行或反平行。

6. 本次最终使用 SolidWorks 直接测得的轴方向向量构造旋转矩阵，比通过多个夹角间接推算更可靠。

7. 由于 CAD 测量存在微小误差，最终利用右手系和正交条件对坐标轴进行了极小的正交化修正。

8. 外参本质上是一条固定刚体变换：

\[
T_{\text{robot}\leftarrow\text{camera}}
\]

其中旋转 \(R\) 描述两套坐标轴方向的差异，平移 \(p\) 描述两套坐标系原点的差异。

