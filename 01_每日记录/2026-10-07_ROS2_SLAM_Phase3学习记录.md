# 2026-10-07 ROS2 + SLAM Phase 3 学习记录

> 主题：从单帧 `LaserScan` 建立二维几何观测，并形成底盘可用的对正误差  
> 阶段结论：**Phase 3 PASS**  
> 学习方式：真实 LiDAR 数据 + ROS2 节点 + rosbag 回放 + 控制变量实验  
> 核心原则：以真实任务为牵引，以最小闭环验证理论；主线大步推进，调试小步定位。

---

# 1. Phase 3 原定目标与阶段合同

## 1.1 原定主线

Phase 3 的原定目标不是做一个“复杂环境下鲁棒的显示器检测器”，而是完成下面这条最小几何感知闭环：

```text
LaserScan
→ 二维点集
→ 目标候选
→ 目标可见面几何
→ 相对状态
→ 底盘对正误差
```

更具体地说，主线任务是：

```text
/scan
→ sensor_msgs/msg/LaserScan
→ ranges[i]
→ 有效性过滤
→ 极坐标转二维笛卡尔坐标
→ Point2D
→ Cartesian ROI
→ candidate_points
→ PCA / TLS 直线拟合
→ midpoint / distance / yaw / visible width / fit error
→ alignment error
```

最终输出希望形成：

\[
\boxed{
x_{\text{mid}},\quad d,\quad \psi,\quad L,\quad \mathrm{RMSE}
}
\]

以及：

\[
\boxed{
e_x,\quad e_d,\quad e_\psi
}
\]

其中：

- \(x_{\text{mid}}\)：可见线段中点的横向坐标；
- \(d\)：LiDAR 原点到目标面的垂直距离；
- \(\psi\)：目标面的方向角；
- \(L\)：当前可见线段长度；
- RMSE：直线拟合误差；
- \(e_x,e_d,e_\psi\)：底盘对正误差。

## 1.2 Phase 3 的 Must / Optional / Exit Criteria

### Must

1. 能从 `LaserScan` 中读取并解释真实数据；
2. 完成有效点过滤和极坐标转笛卡尔坐标；
3. 用简单 ROI 提取候选点；
4. 用 PCA / TLS 拟合目标可见面；
5. 提取距离、方向、可见长度、中点和拟合误差；
6. 用真实实验验证前后、左右、旋转三类物理变化；
7. 将观测量转化为底盘可用的对正误差。

### Optional

- scan-order segmentation；
- clustering；
- RANSAC；
- 复杂目标识别；
- 动态 ROI；
- 鲁棒 tracking。

### Exit Criteria

当下列链路经过真实数据验证后，Phase 3 即可结束：

\[
\text{LaserScan}
\rightarrow
\text{目标面几何观测}
\rightarrow
\text{控制可用误差}
\]

---

# 2. P3.0：现有代码与数据流审计

## 2.1 LaserScan 的真实数据结构

二维 LiDAR 发布：

```text
sensor_msgs/msg/LaserScan
```

其中最重要的数据是：

```cpp
ranges[i]
```

`ranges[i]` 表示第 \(i\) 束激光测得的距离。

对应角度为：

\[
\boxed{
\theta_i
=
\theta_{\min}
+
i\Delta\theta
}
\]

其中：

- \(\theta_{\min}\)：第一束激光的角度；
- \(\Delta\theta\)：相邻激光束的角度增量；
- \(i\)：扫描索引。

因此 `LaserScan` 并不是“无序点集”，而是一组按扫描角度排列的距离观测。

## 2.2 有效性过滤

对每一个 `range`，只保留：

\[
r_{\min}\le r_i\le r_{\max}
\]

并排除：

- `NaN`；
- `Inf`；
- 其他非有限值。

典型逻辑：

```cpp
if (!std::isfinite(r)) {
    continue;
}

if (r < scan->range_min || r > scan->range_max) {
    continue;
}
```

### 工程意义

这一层不是算法优化，而是数据输入的基本合法性保证。

---

# 3. LaserScan 极坐标到二维笛卡尔坐标

对于有效激光点：

\[
r_i,\quad \theta_i
\]

转换为：

\[
\boxed{
x_i=r_i\cos\theta_i
}
\]

\[
\boxed{
y_i=r_i\sin\theta_i
}
\]

保存为：

```cpp
struct Point2D
{
    double x;
    double y;
};
```

此时数据流变成：

```text
ranges[i]
→ θi
→ (r, θ)
→ (x, y)
→ Point2D
```

这一步是后续几何处理的基础。

---

# 4. P3.1：Cartesian ROI 候选点筛选

## 4.1 为什么需要 ROI

整个房间的 LiDAR 点包含：

- 墙；
- 桌子；
- 显示器；
- 家具；
- 其他杂物。

不能直接把全部点送入 PCA。

因此先做一个最小空间筛选：

\[
x_{\min}\le x\le x_{\max}
\]

\[
y_{\min}\le y\le y_{\max}
\]

得到：

```cpp
std::vector<Point2D> candidate_points;
```

## 4.2 典型实现

```cpp
for (std::size_t i = 0; i < points.size(); ++i)
{
    const Point2D point = points[i];

    if (point.x >= x_min && point.x <= x_max &&
        point.y >= y_min && point.y <= y_max)
    {
        candidate_points.push_back(point);
    }
}
```

## 4.3 为什么不用单纯距离阈值

如果只用：

\[
r<r_{\text{threshold}}
\]

得到的是一个圆形区域。

这会把所有“离 LiDAR 较近”的结构都纳入候选。

而 Cartesian ROI 可以独立限制 \(x\) 和 \(y\)，更适合当前受控实验。

## 4.4 必须记住的语义

\[
\boxed{\text{ROI 是空间筛选，不是目标识别}}
\]

ROI 只是利用场景先验减少搜索范围，并不意味着 ROI 中所有点都一定属于目标。

---

# 5. P3.2：PCA / TLS 二维直线拟合

这是 Phase 3 最核心的数学部分。

二维 LiDAR 的扫描平面与一个平面目标相交，在二维扫描平面内表现为一条线。

因此问题变成：

> 给定一组带噪声二维点，寻找最能代表这些点的一条直线。

---

# 6. 质心与中心化

候选点：

\[
P_i=
\begin{bmatrix}
x_i\\
y_i
\end{bmatrix}
\]

首先计算质心：

\[
\boxed{
\bar x=\frac1N\sum_i x_i
}
\]

\[
\boxed{
\bar y=\frac1N\sum_i y_i
}
\]

写成：

\[
\bar P=
\begin{bmatrix}
\bar x\\
\bar y
\end{bmatrix}
\]

对每一个点中心化：

\[
\boxed{
q_i=P_i-\bar P
}
\]

也就是：

\[
\Delta x_i=x_i-\bar x
\]

\[
\Delta y_i=y_i-\bar y
\]

### 物理意义

中心化之后，我们研究的是：

> 点云相对于自身中心往哪些方向展开。

而不再受点云整体位于哪里影响。

---

# 7. Scatter Matrix

构造：

\[
\boxed{
S=
\begin{bmatrix}
S_{xx} & S_{xy}\\
S_{xy} & S_{yy}
\end{bmatrix}
}
\]

其中：

\[
S_{xx}=\sum_i(\Delta x_i)^2
\]

\[
S_{yy}=\sum_i(\Delta y_i)^2
\]

\[
S_{xy}=\sum_i\Delta x_i\Delta y_i
\]

代码：

```cpp
double s_xx = 0.0;
double s_xy = 0.0;
double s_yy = 0.0;

for (std::size_t i = 0; i < candidate_points.size(); ++i)
{
    const double dx = candidate_points[i].x - centroid_x;
    const double dy = candidate_points[i].y - centroid_y;

    s_xx += dx * dx;
    s_xy += dx * dy;
    s_yy += dy * dy;
}
```

这里没有除以 \(N\)，所以严格说是 scatter matrix，而不是标准 covariance matrix。

但：

\[
C=\frac1N S
\]

只是整体乘了一个常数，因此：

\[
\boxed{\text{二者特征向量相同}}
\]

当前任务只需要方向，因此使用 \(S\) 足够。

---

# 8. 特征值 / 特征向量的物理意义

对于单位向量 \(v\)：

\[
\boxed{
v^TSv
=
\sum_i(q_i\cdot v)^2
}
\]

它表示点云沿方向 \(v\) 的总投影平方。

因此：

## 8.1 最大特征值

\[
\lambda_{\max}
\]

对应的特征向量：

\[
\boxed{
t=\text{直线方向}
}
\]

因为该方向上的点云延展最大。

## 8.2 最小特征值

\[
\lambda_{\min}
\]

对应的特征向量：

\[
\boxed{
n=\text{直线法向}
}
\]

因为该方向上的点云离散最小。

二维中：

\[
t\perp n
\]

这就是 PCA 在当前任务中的核心。

---

# 9. PCA 与 TLS 为什么等价

我们真正想最小化的是：

> 所有点到拟合直线的垂直平方距离。

设单位法向：

\[
n
\]

中心化点：

\[
q_i
\]

那么点到直线的有符号垂直距离为：

\[
n^Tq_i
\]

代价函数：

\[
\boxed{
J
=
\sum_i(n^Tq_i)^2
}
\]

这正好等于：

\[
n^TSn
\]

因此最小化 \(J\) 就是在寻找 scatter matrix 的最小特征值方向。

所以：

\[
\boxed{\text{PCA 在这里等价于二维正交最小二乘 TLS}}
\]

---

# 10. 使用 Eigen 求特征值

实现使用：

```cpp
Eigen::SelfAdjointEigenSolver<Eigen::Matrix2d> solver(scatter);
```

之所以使用 `SelfAdjointEigenSolver`，是因为：

\[
S=S^T
\]

即 scatter matrix 是实对称矩阵。

Eigen 默认将特征值按升序排列，因此：

```cpp
const double lambda_min = solver.eigenvalues()(0);
const double lambda_max = solver.eigenvalues()(1);

const Eigen::Vector2d normal =
    solver.eigenvectors().col(0);

const Eigen::Vector2d direction =
    solver.eigenvectors().col(1);
```

即：

\[
\lambda_{\min}
\leftrightarrow
n
\]

\[
\lambda_{\max}
\leftrightarrow
t
\]

---

# 11. 从法向得到完整直线

一般直线：

\[
ax+by+c=0
\]

单位法向：

\[
n=
\begin{bmatrix}
a\\
b
\end{bmatrix}
\]

由于最佳正交最小二乘直线通过质心：

\[
a\bar x+b\bar y+c=0
\]

所以：

\[
\boxed{
c=-(a\bar x+b\bar y)
}
\]

最终得到：

\[
\boxed{
ax+by+c=0
}
\]

---

# 12. centroid residual 的作用

代码中验证：

\[
a\bar x+b\bar y+c
\]

理论应等于 0。

因此：

```text
centroid_residual = 0
```

说明：

> 直线构造与质心关系一致。

但必须注意：

\[
\boxed{\text{centroid residual = 0 不等于拟合质量好}}
\]

因为这是由直线定义本身保证的。

它只是内部一致性检查。

---

# 13. P3.3：从直线提取几何观测

得到拟合线以后，开始把数学结果转化为机器人真正有物理意义的量。

---

# 14. 目标面距离

因为法向已经归一化：

\[
a^2+b^2=1
\]

原点到直线：

\[
ax+by+c=0
\]

的距离为：

\[
\boxed{
d=|c|
}
\]

### 物理意义

\[
\boxed{\text{LiDAR 原点到目标拟合面的垂直距离}}
\]

它不是某一束 LiDAR 的 `range`。

---

# 15. 目标面方向 yaw

直线方向向量：

\[
t=
\begin{bmatrix}
t_x\\
t_y
\end{bmatrix}
\]

方向角：

\[
\boxed{
\psi=
\operatorname{atan2}(t_y,t_x)
}
\]

`atan2()` 能保留象限信息，比单纯：

\[
\arctan(y/x)
\]

更可靠。

---

# 16. PCA 特征向量的符号二义性

如果 \(t\) 是特征向量，那么：

\[
-t
\]

同样是特征向量。

所以同一条物理直线可能输出：

\[
3^\circ
\]

也可能输出：

\[
-177^\circ
\]

二者代表同一条直线。

因此将直线方向统一到：

\[
\boxed{
[-90^\circ,90^\circ)
}
\]

即：

\[
\boxed{
\left[-\frac{\pi}{2},\frac{\pi}{2}\right)
}
\]

### 关键区别

当前拟合的是：

\[
\boxed{\text{直线轴方向}}
\]

它具有：

\[
\pi
\]

周期。

以后机器人真实姿态 heading 常常具有：

\[
2\pi
\]

周期。

二者不能混淆。

---

# 17. 沿直线方向的投影

对于每个 candidate point：

\[
q_i=P_i-\bar P
\]

沿直线方向 \(t\) 投影：

\[
\boxed{
s_i=q_i\cdot t
}
\]

其中：

\[
s_i
\]

表示：

> 第 \(i\) 个点沿拟合直线方向、相对质心的一维坐标。

找到：

\[
s_{\min}
\]

和：

\[
s_{\max}
\]

即可描述当前观测线段。

---

# 18. Visible Length

定义：

\[
\boxed{
L=s_{\max}-s_{\min}
}
\]

为什么不用：

\[
x_{\max}-x_{\min}
\]

因为目标一旦旋转，\(x\) 方向不再等于目标自身方向。

沿 \(t\) 投影以后，长度定义与目标旋转无关。

### 语义边界

\[
\boxed{L=\text{当前可见线段长度}}
\]

而不是保证：

\[
L=\text{物体真实宽度}
\]

遮挡、ROI 截断、激光缺失都会使 \(L\) 变化。

---

# 19. Visible Midpoint

先求：

\[
\boxed{
s_{\text{mid}}
=
\frac{s_{\min}+s_{\max}}2
}
\]

再恢复到二维空间：

\[
\boxed{
P_{\text{mid}}
=
\bar P+s_{\text{mid}}t
}
\]

得到：

\[
P_{\text{mid}}
=
\begin{bmatrix}
x_{\text{mid}}\\
y_{\text{mid}}
\end{bmatrix}
\]

它表示：

\[
\boxed{\text{当前可见线段的几何中点}}
\]

---

# 20. Visible Endpoints

两个拟合端点：

\[
\boxed{
P_{\min}
=
\bar P+s_{\min}t
}
\]

\[
\boxed{
P_{\max}
=
\bar P+s_{\max}t
}
\]

因为：

\[
\|t\|=1
\]

所以：

\[
\boxed{
\|P_{\max}-P_{\min}\|
=
L
}
\]

这也是我们用来检查代码正确性的几何关系。

---

# 21. 为什么 midpoint 不等于真实物体中心

如果整个物体边缘完整可见：

```text
──────────────
```

可见中点可能接近真实中心。

但如果只看到一部分：

```text
──────
```

则：

\[
P_{\text{mid}}
\]

只是可见部分的中点。

因此：

\[
\boxed{\text{visible midpoint ≠ guaranteed object center}}
\]

---

# 22. 可观测性 Observability

无限直线能够决定：

- 方向；
- 法向；
- 到原点的垂直距离。

但沿直线方向平移：

```text
────────────
    →
```

并不会改变无限直线本身。

因此单独一条无限直线不能确定完整：

\[
(x,y,\theta)
\]

目标 pose。

这就是一个简单的可观测性问题。

加入：

\[
s_{\min},s_{\max}
\]

以后，可以得到有限可见段和可见中点。

但仍不能自动声称得到了真实物体几何中心。

---

# 23. Fit RMSE：正式的拟合质量指标

最小特征值：

\[
\lambda_{\min}
\]

满足：

\[
\boxed{
\lambda_{\min}
=
\sum_i d_i^2
}
\]

其中：

\[
d_i
\]

是每个点到拟合直线的垂直距离。

因此定义：

\[
\boxed{
\mathrm{RMSE}
=
\sqrt{
\frac{\lambda_{\min}}N
}
}
\]

单位是米。

### 实验现象

干净目标：

\[
\mathrm{RMSE}
\approx1\sim2\,\mathrm{mm}
\]

复杂 ROI 污染或移动过渡阶段：

\[
\mathrm{RMSE}
\approx5\sim14\,\mathrm{mm}
\]

### 当前结论

RMSE 已经可以作为：

\[
\boxed{\text{直线拟合质量指标}}
\]

但暂时没有设置固定阈值。

原因：

\[
\boxed{\text{目前样本不足以支撑可靠 threshold}}
\]

---

# 24. 最终 Observation

Phase 3 最终几何观测可以写成：

\[
\boxed{
z=
\begin{bmatrix}
x_{\text{mid}}\\
d\\
\psi
\end{bmatrix}
}
\]

并同时保留：

\[
L,\quad \mathrm{RMSE}
\]

程序输出：

```text
observation:
mid=(...)
distance=...
yaw=...
width=...
fit_rmse=...
```

它已经比早期的：

```text
scatter
eigen
centroid
...
```

更接近后级机器人模块真正需要的接口。

---

# 25. P3.4：控制变量实验验证

我们不是只看“数字像不像”，而是主动改变物理世界，再观察输出。

---

## 25.1 前后移动实验

实验状态：

```text
near
mid
far
```

最终有效结果大致为：

\[
d_{\text{near}}
\approx0.274m
\]

\[
d_{\text{mid}}
\approx0.321m
\]

\[
d_{\text{far}}
\approx0.388m
\]

满足：

\[
\boxed{
d_{\text{near}}
<
d_{\text{mid}}
<
d_{\text{far}}
}
\]

因此：

\[
\boxed{\text{distance 对真实前后变化响应正确}}
\]

由于没有尺子，这里验证的是：

- 单调性；
- 物理对应关系；

而不是绝对精度标定。

---

## 25.2 横向移动实验

稳定基准状态：

\[
x_{\text{mid}}
\approx-0.007m
\]

横向移动以后：

\[
x_{\text{mid}}
\approx-0.04m
\]

与此同时：

\[
d\approx0.291\sim0.292m
\]

\[
\psi\approx0^\circ
\]

基本保持稳定。

因此验证：

\[
\boxed{\text{横向移动主要反映到 }x_{\text{mid}}}
\]

---

## 25.3 yaw 旋转实验

基准状态：

\[
\psi\approx0.5^\circ
\]

旋转以后：

\[
\psi\approx-10^\circ\sim-12^\circ
\]

同时距离仍大约保持在：

\[
0.29m
\]

附近。

因此：

\[
\boxed{\text{yaw 对真实目标旋转响应正确}}
\]

---

# 26. 控制变量实验的重要认识

真实机器人环境中，各变量并不会完全解耦。

例如：

- 旋转目标时，中点位置也可能变化；
- 横向移动时，距离也可能轻微变化；
- 手动移动无法做到精密纯平移。

因此实验目标不是：

> 每次只允许一个数字变化。

而是验证：

\[
\boxed{\text{主要观测量对主要物理变化具有正确响应}}
\]

---

# 27. 从 Observation 到 Alignment Error

当前观测：

\[
z=
\begin{bmatrix}
x_{\text{mid}}\\
d\\
\psi
\end{bmatrix}
\]

期望状态：

\[
z^*=
\begin{bmatrix}
x^*\\
d^*\\
\psi^*
\end{bmatrix}
\]

定义误差：

\[
\boxed{
e=z-z^*
}
\]

展开：

\[
\boxed{
e_x=x_{\text{mid}}-x^*
}
\]

\[
\boxed{
e_d=d-d^*
}
\]

\[
\boxed{
e_\psi=\psi-\psi^*
}
\]

其中角度误差还需要按直线的 \(\pi\) 周期进行 wrap。

---

# 28. 三个误差的物理意义

## 28.1 横向误差

\[
e_x
\]

表示：

> 当前可见面中点相对期望横向位置偏了多少。

如果：

\[
x^*=0
\]

那么：

\[
e_x=x_{\text{mid}}
\]

---

## 28.2 距离误差

\[
e_d=d-d^*
\]

如果：

\[
e_d>0
\]

说明：

\[
d>d^*
\]

即当前距离比期望更远。

如果：

\[
e_d<0
\]

则说明当前比期望更近。

---

## 28.3 角度误差

\[
e_\psi=\psi-\psi^*
\]

经过 \(\pi\) 周期 wrap 后，表示：

> 当前目标面相对期望方向的最小角度偏差。

---

# 29. 实际误差输出验证

测试时暂时采用：

\[
x^*=0
\]

\[
d^*=0.300m
\]

\[
\psi^*=0^\circ
\]

注意：

\[
\boxed{d^*=0.30m\text{ 只是 TEST ONLY 测试值}}
\]

真实机器人中应该来自：

- 机构尺寸；
- 对接要求；
- 实际任务设计。

在旧 rosbag `p2_box_static_v2` 回放中，典型观测：

\[
x_{\text{mid}}\approx0.019m
\]

\[
d\approx0.294m
\]

\[
\psi\approx3.39^\circ
\]

得到：

\[
\boxed{
e_x\approx0.019m
}
\]

\[
\boxed{
e_d\approx-0.006m
}
\]

\[
\boxed{
e_\psi\approx3.39^\circ
}
\]

程序输出与理论完全一致。

---

# 30. 观测量 ≠ 误差量 ≠ 控制命令

这是 Phase 3 最后必须建立的概念边界。

## 观测量

回答：

> 当前状态是什么？

例如：

\[
d=0.294m
\]

## 误差量

回答：

> 当前状态离目标状态差多少？

例如：

\[
e_d=-0.006m
\]

## 控制命令

回答：

> 底盘现在应该如何运动？

例如未来可能出现：

\[
v_x,\quad v_y,\quad \omega
\]

但：

\[
\boxed{\text{Phase 3 不负责从误差直接生成控制命令}}
\]

因为控制还需要结合：

- `base_link` 坐标系；
- LiDAR 安装方向；
- 底盘运动学；
- 控制律；
- 速度 / 加速度限制。

---

# 31. 闭环控制思想的初步引入

真实机器人不会依靠“一次测量 + 一次运动”完成对正。

正确结构更像：

```text
LaserScan
↓
估计目标几何
↓
计算误差 e
↓
机器人移动一点
↓
新的 LaserScan
↓
重新估计
↓
重新计算误差
↓
继续修正
```

这就是：

\[
\boxed{\text{closed-loop feedback，闭环反馈}}
\]

核心思想：

\[
\boxed{\text{不断重新测量，并让误差逐渐趋近于 0}}
\]

Phase 3 为后续闭环控制提供了观测和误差基础。

---

# 32. 扩展知识 1：复杂环境下固定 ROI 的失败

这一部分不是 Phase 3 原定 Must，但真实实验中发现了重要失败模式。

将 ROI 放宽以后，候选点中出现了多个不同物理结构。

表现为：

\[
y\text{ 范围明显变厚}
\]

以及：

\[
\lambda_{\min}
\]

从：

\[
10^{-4}
\]

量级上升到：

\[
10^{-2}
\]

量级。

PCA 仍然输出一个数学上完全合法的方向，例如约：

\[
10^\circ
\]

但这个方向已经不再代表真正目标面。

### 关键认识

\[
\boxed{\text{PCA 一定会给主方向，但不保证输入点属于正确目标}}
\]

因此：

\[
\boxed{\text{算法数学正确} \neq \text{输入语义正确}}
\]

---

# 33. 扩展知识 2：为什么重新加入 scan_index

最初：

```cpp
struct Point2D
{
    double x;
    double y;
};
```

已经足够完成 ROI 和 PCA。

后来为了分析：

> ROI 中是否存在多个空间结构？

原始扫描顺序第一次真正有用。

因此扩展：

```cpp
struct Point2D
{
    double x;
    double y;
    std::size_t scan_index;
};
```

这里体现了一个很好的工程原则：

\[
\boxed{\text{只有当信息真正解决当前问题时，才引入它}}
\]

而不是提前保存所有可能有用的信息。

---

# 34. 扩展知识 3：扫描索引连续不等于表面连续

我们比较相邻 candidate point：

\[
\Delta i
=
i_k-i_{k-1}
\]

以及空间距离：

\[
\boxed{
g_k=
\sqrt{
(x_k-x_{k-1})^2+
(y_k-y_{k-1})^2
}
}
\]

实测发现：

\[
\Delta i\approx1\sim2
\]

但：

\[
g_{\max}
\approx0.06m
\]

而在 \(r\approx0.39m\) 时，正常相邻激光束的横向采样尺度大约：

\[
r\Delta\theta
\approx0.39\times0.00582
\approx0.0023m
\]

即：

\[
2.3mm
\]

因此：

\[
\boxed{60mm \gg 2.3mm}
\]

说明相邻扫描方向虽然连续，但激光已经从一个物理表面突然跳到了另一个表面。

因此：

\[
\boxed{\text{扫描角度连续} \neq \text{空间表面连续}}
\]

---

# 35. 扩展知识 4：Euclidean Gap Segmentation

由上面的失败模式，自然引出了：

**Euclidean gap segmentation（欧氏距离间隙分割）**

基本思想：

```text
● ● ● ● ●  |  ● ● ● ● ●
            ↑
         大空间 gap
```

如果：

\[
g_k>g_{\text{threshold}}
\]

则认为进入新的 segment。

实验阶段使用：

\[
g_{\text{threshold}}=0.02m
\]

其依据是：

\[
2.3mm
\ll
20mm
\ll
60mm
\]

实际一度稳定分成：

```text
segment[0] ≈ 20 points
segment[1] ≈ 100 points
```

说明 ROI 中确实存在至少两个不同空间结构。

---

# 36. 为什么没有继续做 segmentation / RANSAC

虽然已经证明固定 ROI 在复杂环境下有局限，但这并不阻塞 Phase 3 原定任务。

因此该问题被分类为：

\[
\boxed{\text{Limitation}}
\]

而不是：

\[
\boxed{\text{Blocker}}
\]

继续深入可能自然走向：

```text
segment selection
→ clustering
→ RANSAC
→ robust target recognition
```

但这会把当前主线从：

> 学习 LaserScan 二维几何观测

悄悄变成：

> 做一个复杂环境中的鲁棒目标检测器。

因此最终停止扩展。

### 正确结论

已经知道：

- 固定 ROI 的失败边界；
- 为什么 PCA 会被污染；
- scan-order segmentation 可以作为未来扩展；
- clustering / RANSAC 可能进一步提高鲁棒性。

但当前不实现。

---

# 37. 今天暴露出的教学节奏问题

一度出现了这种循环：

```text
加几行代码
→ 编译
→ 回放
→ 发几帧日志
→ 再加几行代码
```

这种方式适合：

\[
\boxed{\text{故障定位}}
\]

但不适合作为主线长期学习方式。

因为：

\[
\boxed{\text{最小可验证闭环} \neq \text{最小代码增量}}
\]

正确的主线节奏应该是：

\[
\boxed{
\text{一个完整能力块}
\rightarrow
\text{一次实现}
\rightarrow
\text{一次集中验证}
}
\]

只有真正出现 Blocker 时，才切换到：

\[
\text{提出假设}
\rightarrow
\text{只改一个变量}
\rightarrow
\text{小步验证}
\]

最终形成新的固定原则：

\[
\boxed{\text{主线大步推进，调试小步推进}}
\]

---

# 38. 防止学习主线再次漂移：阶段合同机制

这次 Phase 3 中后期一度因为 ROI 污染问题偏离主线。

原因不是局部判断错误，而是：

> 一串局部合理的下一步，不一定组成全局合理的学习路线。

因此以后每个 Phase 固定维护：

## Goal

当前阶段最终要形成什么能力。

## Must

不完成就不能退出当前 Phase 的内容。

## Optional

可以扩展，但当前没有必要深入。

## Exit Criteria

满足什么条件就应该立即结束当前 Phase。

---

# 39. 新问题的三级分类

遇到新问题时，不再自动深挖，而是分类：

## Blocker

不解决就无法完成当前 Phase 的 Must。

→ 必须处理。

## Limitation

当前方案存在的真实局限，但不影响完成阶段目标。

→ 记录并返回主线。

## Extension

以后任务需要时可能值得增加的高级能力。

→ 暂不实现。

只有：

\[
\boxed{\text{Blocker}}
\]

允许插队主线。

---

# 40. Phase 3 完整数据流

今天最终形成的完整链路：

```text
物理世界中的目标平面
        ↓
2D LiDAR
        ↓
sensor_msgs/msg/LaserScan
        ↓
ranges[i]
        ↓
有效性过滤
        ↓
θi = angle_min + i·angle_increment
        ↓
极坐标 → 笛卡尔坐标
        ↓
Point2D
        ↓
Cartesian ROI
        ↓
candidate_points
        ↓
质心
        ↓
scatter matrix
        ↓
Eigen 特征分解
        ↓
PCA / TLS
        ↓
拟合直线 ax + by + c = 0
        ↓
────────────────────────────
distance = |c|
yaw = atan2(ty, tx)
si = (Pi - P̄) · t
visible length
visible midpoint
visible endpoints
fit RMSE
────────────────────────────
        ↓
observation
(xmid, d, ψ, L, RMSE)
        ↓
与期望状态比较
        ↓
alignment error
(ex, ed, eψ)
        ↓
后续控制阶段
```

---

# 41. Phase 3 目前真正掌握到什么程度

## 已达到“能理解 + 能读懂 + 能修改 + 能验证”的内容

- `LaserScan` 数据结构；
- 有效距离过滤；
- 极坐标到笛卡尔坐标；
- Cartesian ROI；
- `std::vector` 在当前数据流中的使用；
- 质心；
- scatter matrix；
- PCA / TLS；
- Eigen 二维特征分解；
- 直线一般式；
- 法向 / 方向向量；
- 点到直线距离；
- 投影；
- visible length；
- visible midpoint；
- endpoint；
- fit RMSE；
- observation；
- alignment error。

## 已理解概念，但当前不继续实现

- scan-order segmentation；
- Euclidean gap segmentation；
- clustering；
- RANSAC；
- 鲁棒目标识别；
- 动态 ROI；
- tracking。

---

# 42. Phase 3 的已知局限

当前系统仍然存在：

1. 固定 ROI 依赖场景先验；
2. 复杂环境可能混入其他结构；
3. visible midpoint 不保证是真实物体中心；
4. visible width 不保证是真实物理宽度；
5. RMSE 尚未通过大规模数据建立可靠 threshold；
6. 当前没有完整目标类别识别；
7. 还没有接入 TF2；
8. 还没有转换到 `base_link`；
9. 还没有真正输出底盘控制速度；
10. 当前 `target_distance = 0.30 m` 只是测试参数，不是实际机构参数。

这些不是 Phase 3 的遗漏，而是后续阶段或真实任务需要时再解决的问题。

---

# 43. Phase 3 最终验收结论

## P3.0 数据与代码审计

\[
\boxed{\text{PASS}}
\]

## P3.1 Cartesian ROI

\[
\boxed{\text{PASS}}
\]

## P3.2 PCA / TLS 直线拟合

\[
\boxed{\text{PASS}}
\]

## P3.3 几何观测量

已完成：

\[
x_{\text{mid}},\quad
d,\quad
\psi,\quad
L,\quad
\mathrm{RMSE}
\]

\[
\boxed{\text{PASS}}
\]

## P3.4 控制变量实验

已验证：

- 前后距离；
- 横向位置；
- yaw。

\[
\boxed{\text{PASS}}
\]

## 对正误差

已完成：

\[
e_x,\quad e_d,\quad e_\psi
\]

并使用旧 rosbag 完成回归验证。

\[
\boxed{\text{PASS}}
\]

---

# 44. Phase 3 最终结论

Phase 3 真正完成的不是：

> “一个显示器检测程序”。

而是：

\[
\boxed{
\text{从真实 LaserScan 中建立二维目标面几何观测的能力}
}
\]

并进一步得到：

\[
\boxed{
\text{机器人后级控制能够消费的相对对正误差}
}
\]

同时通过真实失败案例补充了两条非常重要的工程认识：

\[
\boxed{
\text{输入选择正确，比盲目相信算法输出更重要}
}
\]

以及：

\[
\boxed{
\text{系统复杂度应由真实 Blocker 驱动，而不是由技术兴趣驱动}
}
\]

因此：

\[
\boxed{\text{Phase 3：PASS}}
\]

---

# 45. 后续复习时最应该记住的 10 个问题

1. `LaserScan` 的 `ranges[i]` 为什么不是无序点？
2. 如何由 \(i\) 得到对应角度 \(\theta_i\)？
3. 为什么需要先做有效距离过滤？
4. ROI 为什么只是候选筛选，而不是目标识别？
5. 为什么 PCA 最大特征值方向对应直线方向？
6. 为什么最小特征值和垂直拟合误差有关？
7. 为什么 \(d=|c|\) 的前提是单位法向？
8. 为什么 visible length 要沿 \(t\) 投影，而不能直接用 \(x_{\max}-x_{\min}\)？
9. 为什么 visible midpoint 不等于真实物体中心？
10. 为什么 observation、error、control command 必须严格区分？

---

# 46. 本阶段关键公式速查

## LaserScan 角度

\[
\boxed{
\theta_i=\theta_{\min}+i\Delta\theta
}
\]

## 极坐标转笛卡尔坐标

\[
\boxed{
x_i=r_i\cos\theta_i
}
\]

\[
\boxed{
y_i=r_i\sin\theta_i
}
\]

## 质心

\[
\boxed{
\bar x=\frac1N\sum_i x_i,\qquad
\bar y=\frac1N\sum_i y_i
}
\]

## Scatter Matrix

\[
\boxed{
S=
\begin{bmatrix}
\sum\Delta x_i^2 & \sum\Delta x_i\Delta y_i\\
\sum\Delta x_i\Delta y_i & \sum\Delta y_i^2
\end{bmatrix}
}
\]

## 直线

\[
\boxed{
ax+by+c=0
}
\]

\[
\boxed{
c=-(a\bar x+b\bar y)
}
\]

## 距离

\[
\boxed{
d=|c|
}
\]

## yaw

\[
\boxed{
\psi=\operatorname{atan2}(t_y,t_x)
}
\]

## 投影

\[
\boxed{
s_i=(P_i-\bar P)\cdot t
}
\]

## 可见长度

\[
\boxed{
L=s_{\max}-s_{\min}
}
\]

## 可见中点

\[
\boxed{
s_{\text{mid}}
=
\frac{s_{\min}+s_{\max}}2
}
\]

\[
\boxed{
P_{\text{mid}}
=
\bar P+s_{\text{mid}}t
}
\]

## 拟合误差

\[
\boxed{
\mathrm{RMSE}
=
\sqrt{\frac{\lambda_{\min}}N}
}
\]

## 对正误差

\[
\boxed{
e_x=x_{\text{mid}}-x^*
}
\]

\[
\boxed{
e_d=d-d^*
}
\]

\[
\boxed{
e_\psi=
\operatorname{wrap}_{\pi}
(\psi-\psi^*)
}
\]

---

# 47. 今日阶段状态

```text
Phase 3：二维 LiDAR 几何观测与对正误差

P3.0 代码 / bag / 数据流审计        PASS
P3.1 Cartesian ROI                  PASS
P3.2 PCA / TLS 直线拟合             PASS
P3.3 几何观测量提取                 PASS
P3.4 控制变量实验                   PASS
Alignment Error                     PASS

扩展：
ROI contamination                   已分析
scan_index continuity               已分析
Euclidean gap segmentation          已理解并验证基本现象
clustering / RANSAC                 暂不进入

最终状态：
Phase 3 PASS
```
