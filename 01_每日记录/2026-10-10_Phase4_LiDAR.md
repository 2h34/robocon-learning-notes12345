# Phase 4｜ROS2 LiDAR 正方形箱体位姿估计：阶段复盘学习记录

> **记录日期**：2026-10-10  
> **项目**：`rcccc-ROS2-slam` / `2h34/ROS2`  
> **本次任务性质**：回顾已完成的 Phase 4 实现，依据真实代码理解数据流、核心几何算法、验证结果和局限；**不新增功能、不改代码**。  
> **阶段结论**：现阶段最初设定的学习要求已经达成，Phase 4 暂时结束。后续仅在出现真实机器人或比赛任务需求时，再围绕实际问题继续优化。

## 0. 本文的使用方式与代码依据

本文是**今天代码复盘与自测的学习笔记**，同时整合此前 Phase 4 已有的实验结论，方便以后恢复项目上下文；不是一份声称今天重新完成全部实验的测试报告。

- 仓库：[2h34/ROS2](https://github.com/2h34/ROS2)
- 核心文件：[`src/lidar_geometry/src/scan_geometry_node.cpp`](https://github.com/2h34/ROS2/blob/ac23ba1437b530d087aa06cc7ca2cd13fbbccdbf/src/lidar_geometry/src/scan_geometry_node.cpp)
- 复盘参考修订：`ac23ba1437b530d087aa06cc7ca2cd13fbbccdbf`（复盘时定位的源码修订）
- 今日再次读取 `main` 中该文件，源码共 **1051 行**，Git blob SHA 为 `3e7365849bbc82c8613436ea27c14bb04558e5d0`，与本次复盘使用的代码结构一致。
- 重点复习函数：`scan_callback()`、`fit_line_tls()`、`canonicalize_square_yaw()`、`line_direction_dot()`、`intersect_lines()`、`distance_to_nearest_endpoint()`、`edge_direction_from_corner()`。

**阅读原则**：按真实执行顺序理解「数据从哪里来 → 经过哪些处理 → 怎样计算 → 如何决定有效性 → 输出到哪里」，不凭文件名和注释猜测行为。文中“源码事实”“已有实验记录”“本次分析”应区分看待。

---

## 1. 最初要解决的实际任务

目标：利用二维 LiDAR 扫描，估计已知边长 **350 mm（0.350 m）**、无特殊正面标记的正方形箱体，在**激光雷达扫描坐标系**中的二维位置和几何朝向，并通过 ROS2 话题输出。

根据可见面数量，支持两种观测模型：

1. **`SINGLE_FACE`**：看见近似完整的一条箱边；利用可见边的中点及向内法向恢复箱体中心。
2. **`CORNER`**：看见相邻两条箱边；拟合两条直线，以其交点为角点，沿两条边各走半个边长恢复中心。
3. **`INVALID`**：未通过当前几何有效性判断；该帧不输出新的箱体位姿。

本阶段的目标不是实现通用场景理解、复杂多目标识别或 SLAM 全流程，而是形成一个可以运行、理解、调试与验证的 **LiDAR → 几何处理 → 箱体位姿 → ROS2 发布** 最小闭环。

### 1.1 Phase 4 分阶段完成的工作

| 阶段 | 已完成内容 | 作用 |
|---|---|---|
| P4.1 | 单面 TLS 拟合与中心/朝向恢复 | 在单面观测下估计箱体位姿 |
| P4.2 | 有序点集两线分割、角点拟合与中心恢复 | 支持 Corner 观测 |
| P4.3 | Single / Corner 的几何有效性检查与模式判断 | 拒绝明显不符合箱体模型的观测 |
| P4.4 | `/box_mode`、`/box_pose` 发布，复用输入数据的时间戳和坐标系 | 形成 ROS2 输入输出闭环 |
| 本日复盘 | 按执行路径逐段核对源码、推导公式和完成自测 | 确保能解释核心代码而不只是运行它 |

---

## 2. 实际程序入口与完整数据流

### 2.1 ROS2 节点入口

源码末尾 `main()` 执行的逻辑：

```cpp
rclcpp::init(argc, argv);
auto node = std::make_shared<ScanGeometryNode>();
rclcpp::spin(node);
rclcpp::shutdown();
```

`ScanGeometryNode` 继承 `rclcpp::Node`，节点名为 `scan_geometry_node`。构造函数创建：

```cpp
// 输入：LiDAR 扫描
subscription_ = this->create_subscription<sensor_msgs::msg::LaserScan>(
    "/scan", qos,
    std::bind(&ScanGeometryNode::scan_callback, this, std::placeholders::_1));

// 输出：箱体位姿与观测模式
pose_publisher_ = this->create_publisher<geometry_msgs::msg::PoseStamped>(
    "/box_pose", 10);
mode_publisher_ = this->create_publisher<std_msgs::msg::String>(
    "/box_mode", 10);
```

输入订阅配置为 `KeepLast(10)`、`best_effort` QoS；两项输出使用 `10` 深度的默认发布配置。

**本次理解**：`create_subscription()` 注册订阅及回调，并不主动获取一次扫描或直接执行算法。节点被 `rclcpp::spin(node)` 驱动后，执行器才会在消息到达且回调可执行时调度 `scan_callback()`。

### 2.2 从输入到输出的主流程

```text
物理箱体、环境
    ↓
2D LiDAR → /scan : LaserScan
    ↓
scan_callback(msg)
    ├─ 距离合法性检查
    ├─ 极坐标 → 雷达平面二维坐标 Point2D
    ├─ ROI 过滤 → candidate_points
    ├─ 点间距分段 → segments
    ├─ 选点数最多的段 → target_points
    │
    ├─ 单条 TLS → Single Face 候选位姿 + 有效性
    │
    ├─ 遍历分割位置 → 两条 TLS → Corner 候选位姿 + 有效性
    │
    └─ 模式优先级：SINGLE_FACE / CORNER / INVALID
                ├─ 每帧 /box_mode
                └─ 仅有效时 /box_pose : PoseStamped
```

**自测回顾**：`points[100].scan_index` 未必等于 `100`。因为 `points` 是经过无效测距过滤之后的容器，`scan_index` 保存的是它在**原始** `LaserScan.ranges` 中的下标，不能把当前 vector 下标与原始扫描序号混为一谈。

---

## 3. 点云预处理与连续段提取

### 3.1 原始扫描点转换

对 `ranges[i]`：

1. 检查 `std::isfinite(r)` 等数据合法性及传感器距离范围；非法测量跳过。
2. 根据 `angle_min + i * angle_increment` 得到该束激光的扫描角度 `theta`。
3. 极坐标转换：

   $$x_i=r_i\cos\theta_i,\qquad y_i=r_i\sin\theta_i$$

4. 将 `(x, y, scan_index=i)` 保存为 `Point2D`。

`Point2D` 结构体只有 `x`、`y`、`scan_index` 三个成员；它保留扫描顺序，不携带目标身份信息。

### 3.2 ROI（Region of Interest，感兴趣区域）

本版代码筛选范围：

```cpp
const double x_min = -0.25;
const double x_max =  0.29;
const double y_min = -1.10;
const double y_max = -0.24;
```

在这个矩形范围内的点进入 `candidate_points`。若少于 3 个点，本帧直接发布 `INVALID` 并返回。

这些范围是**当前传感器和实验布置相关的运行参数**，不是适用于一切机器人任务的物理常量。

### 3.3 用空间点距分段，而非扫描下标差分段

当前实现：

```cpp
const double gap_threshold = 0.02; // m
const double point_gap = std::hypot(dx, dy);

if (point_gap > gap_threshold) {
    // 开启新 segment
}
```

注意：代码同时计算 `index_gap` 并统计日志，但**实际分段条件只有 `point_gap > 0.02 m`**，不是 `index_gap` 超阈值，也不是两者同时满足。

本日自测：即使原始扫描序号相差 15，只要相邻候选点在空间上相距 `0.008 m`，就**不会**因此分成两段。

接下来：

```cpp
// 选择点数最多的 segment
if (segments[i].size() > segments[target_segment_index].size()) {
    target_segment_index = i;
}
const std::vector<Point2D> &target_points = segments[target_segment_index];
```

此处没有逐个评估所有候选段的箱体几何模型，而是直接**选取单个最大点数段**作为后续 TLS 的输入。这是本次复盘识别出的重要边界，详见第 11 节。

---

## 4. TLS（Total Least Squares，总最小二乘）直线拟合

实际核心函数：`LineFit fit_line_tls(const std::vector<Point2D>& points)`（源码约第 50～189 行）。

### 4.1 输入与输出

**输入**：同一个候选连续段的一组二维点。  
**输出**：一个 `LineFit`，核心字段包含：

| 成员 | 作用 |
|---|---|
| `centroid` | 点云质心，TLS 拟合直线经过它 |
| `normal`、`direction` | 拟合直线的单位法向、切向 |
| `lambda_min`、`lambda_max` | 散布矩阵的两个特征值 |
| `c` | 一般式直线 `ax + by + c = 0` 中的常数项 |
| `distance` | **雷达原点到拟合无限直线**的垂直距离 |
| `projection_min/max` | 点云沿直线方向的最小/最大投影 |
| `endpoint_min/max` | 该点云在拟合直线上的投影端点 |
| `visible_mid`、`visible_length` | 可见段几何中点及长度 |
| `rmse` | 点到拟合直线的垂直距离均方根 |
| `yaw_rad` | 无向直线方向角，归一化到 `[-π/2, π/2)` |

### 4.2 质心与散布矩阵

$$\bar{\mathbf p}=\frac1N\sum_{i=1}^{N}\mathbf p_i$$

记：

$$\Delta x_i=x_i-\bar x,\qquad \Delta y_i=y_i-\bar y$$

本版代码直接累加（**没有除以 N**）：

$$
S=\begin{bmatrix}
\sum_i\Delta x_i^2 & \sum_i\Delta x_i\Delta y_i\\
\sum_i\Delta x_i\Delta y_i & \sum_i\Delta y_i^2
\end{bmatrix}
=\begin{bmatrix}s_{xx}&s_{xy}\\s_{xy}&s_{yy}\end{bmatrix}
$$

对应代码：

```cpp
s_xx += dx * dx;
s_xy += dx * dy;
s_yy += dy * dy;

Eigen::Matrix2d scatter;
scatter << s_xx, s_xy,
           s_xy, s_yy;
```

本日例题：点集 `(0,1)`、`(1,1)`、`(2,1)` 的质心为 `(1,1)`，散布矩阵为：

$$S=\begin{bmatrix}2&0\\0&0\end{bmatrix}$$

因此 X 方向分散程度大，Y 方向分散程度为零。

### 4.3 为什么选择最小特征值对应的法向量？——今天重点推导

设拟合直线经过质心、单位法向量为 `n=(a,b)^T`。第 `i` 个点到直线的有符号垂直距离为：

$$d_i=\mathbf n^T(\mathbf p_i-\bar{\mathbf p})=a\Delta x_i+b\Delta y_i$$

TLS 最小化垂直残差平方和：

$$
\begin{aligned}
E(\mathbf n)
&=\sum_i d_i^2\\
&=\sum_i(a\Delta x_i+b\Delta y_i)^2\\
&=a^2\sum_i\Delta x_i^2
 +2ab\sum_i\Delta x_i\Delta y_i
 +b^2\sum_i\Delta y_i^2\\
&=a^2s_{xx}+2abs_{xy}+b^2s_{yy}\\
&=\boxed{\mathbf n^TS\mathbf n}.
\end{aligned}
$$

这是普通平方展开、交换求和与矩阵乘法的结果，并非额外引入了一个未经说明的公式。

**必须有单位长度约束** `||n||=1`：否则取零向量 `n=(0,0)` 就使 `E=0`，但零向量不能表示有效直线法向，更不能构成有意义的几何距离。

在单位向量约束下，`nᵀSn` 的最小值等于 `S` 的最小特征值，对应的特征向量就是最佳**单位法向量**；最大特征值对应的特征向量是沿点云主要分布的**单位切向量**：

```cpp
Eigen::SelfAdjointEigenSolver<Eigen::Matrix2d> solver(scatter);
fit.lambda_min = solver.eigenvalues()(0);
fit.lambda_max = solver.eigenvalues()(1);
fit.normal    = solver.eigenvectors().col(0);
fit.direction = solver.eigenvectors().col(1);
```

`SelfAdjointEigenSolver` 用于实对称矩阵；Eigen 按特征值从小到大排列。特征向量**正负号可变**，`(1,0)` 和 `(-1,0)` 表示同一无向直线的相反切向。

上面例子可得到 `lambda_min=0`、`lambda_max=2`，`normal` 沿 Y 轴、`direction` 沿 X 轴（正负号不唯一）。

### 4.4 从法向量与质心得到直线方程

设 `normal=(a,b)`、`centroid=(x̄,ȳ)`：

$$a(x-\bar x)+b(y-\bar y)=0$$

即：

$$ax+by+c=0,\qquad \boxed{c=-(a\bar x+b\bar y)}$$

单位法向条件下，雷达原点 `(0,0)` 到该直线的垂直距离为：

$$\boxed{distance=|c|}$$

例：`normal=(1,0)`、`centroid=(2,3)`，则 `c=-2`，直线 `x-2=0`，原点到直线距离 `2`。

注意：该 `distance` **不是箱体中心到雷达的距离**。

### 4.5 沿直线投影，恢复有限可见线段

TLS 求出的是一条无限长的直线，传感器看到的是有限长度的点集。将每个点相对质心的偏移投影到单位切向 `direction=d`：

$$t_i=(\mathbf p_i-\bar{\mathbf p})^T\mathbf d$$

取 `t_min`、`t_max`：

$$
\begin{aligned}
visible\_length &=t_{max}-t_{min}\\
endpoint_{min}&=\bar{\mathbf p}+t_{min}\mathbf d\\
endpoint_{max}&=\bar{\mathbf p}+t_{max}\mathbf d\\
visible\_mid&=\bar{\mathbf p}+\frac{t_{min}+t_{max}}2\mathbf d
\end{aligned}
$$

**本次重点辨析：`centroid` ≠ `visible_mid`。**

- `centroid` 是所有采样点坐标的平均值，会受点采样密度影响。
- `visible_mid` 是可见段投影端点的中点，决定于投影的极值范围。

自测：点 `(2,0)`、`(2,1)`、`(2,5)` 的质心为 `(2,2)`、可见中点 `(2,2.5)`；再添加 `(2,1)`，质心变为 `(2,1.75)`，但可见端点及中点仍不变。

这些端点是**投影到拟合直线上的端点**，不要求恰好是原始扫描点。

### 4.6 RMSE 与直线 yaw

由最小二乘结果有：

$$\lambda_{min}=\sum_i d_i^2$$

所以：

$$\boxed{RMSE=\sqrt{\frac{\lambda_{min}}{N}}}$$

源码直接使用 `sqrt(lambda_min / points.size())`，没有再遍历一次点集。

**重要认识**：RMSE 小仅说明**点集接近直线**，不说明那一定是目标箱边。墙面同样可能有极小 RMSE。

方向角：

$$yaw=\operatorname{atan2}(direction_y,direction_x)$$

无向直线有 180° 等价性，因此 `fit.yaw_rad` 调整到 `[-90°,90°)`；正方形物体几何朝向还需按 90° 周期归一化，二者不能混为一谈。

---

## 5. Single Face：从完整单边恢复箱体位姿

实际源码约第 593～629 行。

### 5.1 从箱边几何中点推算中心

已知箱体边长 `s = 0.350 m`。若可见段是**完整箱边**，可从可见边中点沿**指向箱体内部**的单位法向量走半个边长：

$$\boxed{\mathbf p_{center}=visible\_mid+\frac{s}{2}\mathbf n_{inward}}$$

TLS `fit.normal` 正负不确定，因此先判断其是否朝向雷达：

```cpp
const Eigen::Vector2d sensor_origin(0.0, 0.0);
const Eigen::Vector2d to_sensor = sensor_origin - fit.visible_mid;

Eigen::Vector2d inward_normal = fit.normal;
if (fit.normal.dot(to_sensor) > 0.0) {
    inward_normal = -fit.normal;
}

const Eigen::Vector2d box_center =
    fit.visible_mid + 0.5 * box_side_length * inward_normal;
```

推理依据：点积为正时当前法向和“可见面中点 → 雷达原点”的方向同向，应取反才指向箱内。

自测：`visible_mid=(0,-0.40)`，`normal=(0,1)`，雷达原点 `(0,0)`，法向朝雷达，向内法向 `(0,-1)`，得：

$$box\_center=(0,-0.575)\ \mathrm m$$

隐含前提：观测面在传感器与箱体中心之间；目标确实是指定尺寸的正方形；可见段可以近似代表一条完整箱边。

### 5.2 正方形方向的 90° 等价性

无标记正方形绕自身旋转 90° 后轮廓不变，因此几何朝向满足：

$$\theta\sim\theta+k\frac\pi2\quad(k\in\mathbb Z)$$

`canonicalize_square_yaw()` 通过反复加减 `π/2`，把角度归一化到：

$$\boxed{-45^\circ\leq yaw<45^\circ}$$

自测：`-70° → 20°`，二者对于无标记正方形是等价几何朝向。**这并不代表已经识别出箱体的某个固定“正面”。**

### 5.3 Single Face 有效性

```cpp
const double single_rmse_threshold = 0.005;
const double single_length_ratio_min = 0.95;
const double single_length_ratio_max = 1.05;

const double length_ratio = fit.visible_length / box_side_length;

const bool single_valid =
    fit.rmse <= 0.005 &&
    length_ratio >= 0.95 &&
    length_ratio <= 1.05;
```

两类条件须同时成立：

1. **直线拟合质量**：`rmse <= 0.005 m`。
2. **可见长度近似完整箱边**：`0.95 <= visible_length / 0.350 <= 1.05`。

例：`RMSE=0.0005 m` 很小，但 `visible_length=0.60 m`，长度比约 `1.714`，仍是无效 Single Face。**拟合好 ≠ 是目标箱边。**

---

## 6. Corner：有序两线拟合、角点与中心恢复

实际源码包括分割搜索约第 638～684 行、两线检验及位姿恢复约第 693～815 行。

### 6.1 为什么要寻找分割位置？

角点观测中，`target_points` 通常包含相邻的两段箱边，但不知道分割点对应哪个扫描下标。程序枚举分割位置 `k`：

```cpp
const std::size_t min_points_for_fit = 3;

for (std::size_t k = min_points_for_fit;
     k + min_points_for_fit <= target_points.size(); ++k) {
    // [begin, begin+k)  → points1
    // [begin+k, end)    → points2
    // 每组分别调用 fit_line_tls()
}
```

优化目标：

$$\boxed{J(k)=\lambda_{min,1}(k)+\lambda_{min,2}(k)=SSE_1(k)+SSE_2(k)}$$

选取 `J(k)` 最小的 `best_split`。因为 `lambda_min` 就是各组的垂直残差**平方和**，两者相加对应整体的平方残差；不能简单拿两个 RMSE 相加替代。

`min_points_for_fit=3` 仅保证**每组能够参与搜索**，不是最终 Corner 的最小支持点数条件。

本日自测：三种分割位置的总 SSE 分别为：`k=30 → 0.0050`、`k=45 → 0.0005`、`k=60 → 0.0030`，因此 `best_split=45`。

**关键理解**：最佳分割只是两条直线对当前数据的拟合优化结果，不保证两段就是真实正方形相邻边。

### 6.2 检查两条直线是否垂直

使用单位切向量的点积：

$$\boxed{direction\_dot=|\mathbf d_1\cdot\mathbf d_2|=|\cos\theta|}$$

- 接近 0：接近垂直。
- 接近 1：接近平行。

取绝对值可消除 TLS 特征向量正负符号的不确定性。

### 6.3 由直线一般式求交点

两条直线：

$$a_1x+b_1y+c_1=0,\qquad a_2x+b_2y+c_2=0$$

记：

$$D=a_1b_2-a_2b_1$$

若 `|D| < 1e-9`，当前实现视为无法可靠求唯一交点并返回 `false`；否则：

$$\boxed{x=\frac{b_1c_2-b_2c_1}{D}},\qquad
\boxed{y=\frac{a_2c_1-a_1c_2}{D}}$$

**再次区分**：拟合直线是无限延长的，交点可能远离真正观测到的有限线段；“直线相交”不等于“箱边在角点接触”。

### 6.4 交点到最近端点的距离

源码：

```cpp
double distance_to_nearest_endpoint(
    const LineFit &line, const Eigen::Vector2d &point)
{
    const double d_min = (point - line.endpoint_min).norm();
    const double d_max = (point - line.endpoint_max).norm();
    return std::min(d_min, d_max);
}
```

几何量：

$$d_{end}=\min(\|p_{corner}-endpoint_{min}\|,\|p_{corner}-endpoint_{max}\|)$$

`.norm()` 计算向量的欧氏长度。这里求的是**交点到两个端点中较近那个的距离**，而不是交点到任意线段位置的最近距离。

### 6.5 五项 Corner 有效性判据

前提：已成功求得两条直线的唯一交点 `has_intersection`。

| 判据 | 代码阈值 | 物理/算法意义 |
|---|---:|---|
| `direction_dot` | `<= 0.15` | 两条边近似垂直 |
| `min_support_points` | `>= 10` | **两组中点数较少的那组**也必须有足够点 |
| `min_support_length` | `>= 0.10 m` | **较短的那段**也需有足够可见长度 |
| `max(rmse1, rmse2)` | `<= 0.005 m` | **两条线**拟合误差都要足够小 |
| `max(endpoint_distance1, endpoint_distance2)` | `<= 0.015 m` | **两段**都须在交点附近有端点 |

源码中五项以 `&&` 连接，任意一项不满足则 `corner_valid=false`。

本日自测：若 `endpoint_distance1=0.008 m`、`endpoint_distance2=0.020 m`，因 `max=0.020 m > 0.015 m`，即使其他检查全通过，也必须判为 `false`。

这些阈值目前是针对现有实验所用的算法参数，并非普适保证。

### 6.6 如何从角点确定两条“向内延伸的边方向”

TLS 直线切向的正负号不确定，因此还需要确认角点往哪一侧确实存在该段箱边：

```cpp
Eigen::Vector2d edge_direction = line.direction;
const Eigen::Vector2d corner_to_segment = line.visible_mid - corner;
if (edge_direction.dot(corner_to_segment) < 0.0) {
    edge_direction = -edge_direction;
}
```

这样使 `u1`、`u2` 尽量沿**交点 → 各自实际可见段**的方向。

设交点 `p_corner`、箱体边长 `s`，则中心：

$$\boxed{p_{center}=p_{corner}+\frac s2u_1+\frac s2u_2}$$

Corner 几何朝向根据 `u1` 的 `atan2()` 求得，再用 `canonicalize_square_yaw()` 归一化到 `[-45°,45°)`。

本日自测：

$$p_{corner}=(0.10,-0.50),\quad u_1=(1,0),\quad u_2=(0,-1)$$

$$\Rightarrow p_{center}=(0.10,-0.50)+(0.175,0)+(0,-0.175)=\boxed{(0.275,-0.675)\ \mathrm m}$$

**代码重要设计**：可以先计算 `corner_candidate_center`，但只有 `corner_valid==true` 才把它保存到最终候选 `corner_center`；“算出数值”不等于“数据有效”。

---

## 7. 最终模式选择与 ROS2 输出

### 7.1 模式优先级

代码逻辑：

```cpp
const char *mode = "INVALID";
if (single_valid) {
    mode = "SINGLE_FACE";
} else if (corner_valid) {
    mode = "CORNER";
}
```

| `single_valid` | `corner_valid` | 最终模式 |
|---|---|---|
| true | false | `SINGLE_FACE` |
| false | true | `CORNER` |
| true | true | `SINGLE_FACE` |
| false | false | `INVALID` |

这是**当前代码选择逻辑**，不意味着 Single Face 在所有实际条件下都比 Corner 精度更高。

### 7.2 两个话题的角色

- `/box_mode`：`std_msgs/msg/String`，每个正常处理的扫描帧都会发布一种模式；ROI 点数不足的提前返回路径也发布 `INVALID`。
- `/box_pose`：`geometry_msgs/msg/PoseStamped`，**仅当 `single_valid || corner_valid` 时发布新位姿**。

当本帧为 `INVALID` 时，程序不会发新的 `/box_pose`；但这**不会自动清除**下游节点保存的旧位姿。

本日自测：连续三帧：

| 帧 | Single | Corner | `/box_mode` | 新 `/box_pose` |
|---|---|---|---|---|
| A | true | true | `SINGLE_FACE` | 有 |
| B | false | false | `INVALID` | 无 |
| C | false | true | `CORNER` | 有 |

因此，B 帧到来时不能继续把 A 帧旧位姿当成当前有效观测；与此同时，`INVALID` **不等于**“物理世界中一定没有箱体”。

### 7.3 PoseStamped 的时间戳、坐标系及四元数

```cpp
pose_msg.header = msg->header;
pose_msg.pose.position.x = center.x();
pose_msg.pose.position.y = center.y();
pose_msg.pose.position.z = 0.0;
pose_msg.pose.orientation.x = 0.0;
pose_msg.pose.orientation.y = 0.0;
pose_msg.pose.orientation.z = std::sin(yaw / 2.0);
pose_msg.pose.orientation.w = std::cos(yaw / 2.0);
```

- `header.stamp` 继承输入 `LaserScan` 的时间戳，不是发布时重新取的系统时间。
- `header.frame_id` 继承输入扫描坐标系，例如 `laser`；**没有**在本节点内进行到 `map` 或 `base_link` 的坐标变换。
- 当前只估计扫描平面内的 `(x,y,yaw)`，输出 `z=0`。
- 平面 yaw 转 ROS2 四元数：

  $$q=(q_x,q_y,q_z,q_w)=\left(0,0,\sin\frac{yaw}{2},\cos\frac{yaw}{2}\right)$$

本日自测：若扫描坐标系为 `laser`、时间为 `T`、中心 `(0.20,-0.60)`、yaw 为 0，则：

- 输出 `header.frame_id="laser"`、`header.stamp=T`。
- 位置 `(0.20,-0.60,0)`。
- 四元数 `(0,0,0,1)`。
- **不能**认为它直接处于 `map` 系。原因首先是**尚未转换参考坐标系**；“二维/三维”和“laser/map 坐标系”属于两个独立问题。

---

## 8. 已有实验结果回顾与证据边界

> 以下为此前 Phase 4 交流/实验记录中的典型数值或预期附近结果。本次学习记录没有重新播放 rosbag、读取新实测日志，也没有验证绝对地面真值精度。它们用于提醒已完成的最小闭环及其边界，不能代替正式标定数据。

| 场景 | 已有记录中的典型表现 | 说明 |
|---|---|---|
| P4.1 单面 `p4_1` | 中心约 `(0.021,-0.497) m`，yaw 约 `0.006 rad`，RMSE 约 `0.0008 m` | 单线拟合与 Single Face 输出链路已在既有实验中跑通 |
| P4.2 角点 `p4_corner_2` | 中心约 `(-0.071,-0.804) m`，yaw 约 `0.57 rad`（约 32.7°）；两线方向点积绝对值约 `0.03~0.09` | Corner 分割、交点与中心恢复能工作 |
| 部分可见单面负例 | RMSE 可能很小，但 `length_ratio≈0.554` | 仅 RMSE 不能决定目标有效性；长度检查可拒绝不完整箱边 |
| 无有效点 / 不满足条件 | `INVALID` | 不应输出新的有效箱体位姿 |
| P4.4 话题发布 | 既有流程验证了 `/box_mode`、有效时 `/box_pose` 的发布行为 | 实现了 ROS2 消息输入输出闭环 |

**不能由这些数据直接推出的结论**：

- 没有独立测量地面真值，就不能仅靠结果稳定或 RMSE 小来证明绝对位置精度。
- 这些测试不能证明多目标、强干扰、不同距离、不同传感器下的鲁棒性。
- 单个袋数据通过不等于有完整现场部署能力。

如未来重启项目，可用 `ros2 topic echo /box_mode`、`ros2 topic echo /box_pose`、相关日志和实际 rosbag 重新检查，但这些命令**不是今日新增的测试结果**。

---

## 9. 今天的自测与理解变化

按学习过程记录，而不只是罗列公式：

| 复盘知识点 | 本次回答/理解 | 核心收获 |
|---|---|---|
| ROS2 执行流程 | 创建订阅不等于主动执行回调；执行器调度回调 | 从真正的程序入口理解数据处理何时触发 |
| `scan_index` | 过滤后 vector 下标不必等于原始扫描下标 | 数据结构字段含义不能凭名字猜 |
| 连续点段 | 相邻原始下标差很大不一定断段，代码按空间点距判断 | 区分日志指标与实际分支条件 |
| TLS 质心/散布矩阵 | `(0,1),(1,1),(2,1)` → 质心 `(1,1)`、`sxx=2,sxy=0,syy=0` | 点云沿 X 更分散 |
| TLS 极小化推导 | 追问 `Σ[nᵀ(p-centroid)]²` 如何整理为 `nᵀSn` | 通过完全平方和矩阵乘法理解公式来源 |
| 单位法向条件 | 零向量没有有效方向，且会无意义地使目标值为 0 | 明确约束是数学必要条件 |
| 直线方程 | `normal=(1,0),centroid=(2,3)` → `c=-2`、`x=2`、距离 2 | 能由向量恢复可解释几何量 |
| 质心与可见中点 | 增加内部重复采样点时质心变、投影端点中点不变 | 理解为何恢复箱体中心用 `visible_mid` |
| Single 判据 | `RMSE` 很小但 `visible_length=0.60m`，仍无效 | 拟合质量不等于物体模型匹配 |
| Single 中心恢复 | 法向朝雷达需取反，例题得到 `(0,-0.575)` | 理解点积符号的工程含义 |
| 正方形角度归一化 | `-70° → 20°`，周期 90° | 几何朝向不等于有标记物体朝向 |
| 两线分割 | `best_split=45`，但不能仅靠目标函数确认 Corner | 最小拟合误差不等于身份判定 |
| 两线几何 | 垂直直线点积 0、交点可算出，但要接近实测端点 | 区分无限直线交点与真实角点 |
| Corner 判据 | `endpoint_distance2=0.020m`，超过 0.015m，即 `false` | 多项有效性条件是 AND 关系 |
| Corner 中心 | `(0.10,-0.50)+0.175(1,0)+0.175(0,-1)=(0.275,-0.675)` | 理解交点、边方向和半边长的几何关系 |
| 模式与输出 | A/B/C 模式与有效位姿发布准确；旧位姿不能自动认为有效 | 认识下游状态有效期问题 |
| 时间戳和参考系 | 位姿头信息来自 LaserScan；二维估计并非 map 估计 | 把“维度”与“坐标系”严格区分 |
| 全局场景 | 60 点墙面胜过 35 点箱体，被错误选作 target | **发现当前最大点数策略的结构性缺陷** |

**本次学习最大的变化**：不仅能说出代码在做什么，还能说明公式为什么成立、参数在阻止何种失败、以及算法何时可能输出看似合理但不可靠的结果。

---

## 10. 工程参数与配置的来源、意义及限制

按参数性质区分：

| 参数/约定 | 当前值 | 类别 | 含义/局限 |
|---|---|---|---|
| `box_side_length` | `0.350 m` | 物理参数 | 已知正方形目标边长；中心偏移 `0.175 m` 由此确定 |
| ROI x 范围 | `[-0.25,0.29] m` | 场景/系统配置 | 当前实验里目标的横向搜索区 |
| ROI y 范围 | `[-1.10,-0.24] m` | 场景/系统配置 | 当前实验里目标的纵向搜索区 |
| `gap_threshold` | `0.02 m` | 算法参数 | 相邻点在二维空间的断段阈值 |
| `min_points_for_fit` | `3` | 算法参数 | 双线候选分割时每条至少 3 点 |
| Single RMSE 阈值 | `0.005 m` | 算法参数 | 单线拟合质量限制 |
| Single 长度比 | `[0.95,1.05]` | 算法参数 | 近似完整边的要求 |
| Corner 点积阈值 | `0.15` | 算法参数 | 两条拟合边接近垂直 |
| Corner 最小支持点数 | `10` | 算法参数 | 每一条边均至少 10 点 |
| Corner 最小支持长度 | `0.10 m` | 算法参数 | 每一条边均有一定实际可见长度 |
| Corner RMSE 阈值 | `0.005 m` | 算法参数 | 两条线均须拟合良好 |
| Corner 交点端距阈值 | `0.015 m` | 算法参数 | 交点应接近两条实测线段的端点 |
| 求交点奇异性阈值 | `1e-9` | 数值判断参数 | 排除行列式近零的情况 |
| 传感器原点 | `(0,0)` | 坐标约定 | `LaserScan` 参考系原点；不是 `map` 原点 |
| 输入 QoS | `KeepLast(10) + best_effort` | ROS2 通信配置 | 配合当前 LiDAR 消息输入方式 |

除已知箱体物理边长之外，这些阈值和 ROI 均应视为**当前版本与当前条件下的选择**；未来若真实任务发生改变，先记录输入数据、判断失败环节、单变量验证，再调整参数。

---

## 11. 本次识别的主要缺陷与暂缓决策

### 11.1 重要缺陷：最大点数段不一定是目标箱体

综合自测场景：ROI 内有两个被正确分离的点段：

- **点段 A**：墙面或干扰直线，60 个点，连续且接近直线。
- **点段 B**：真实正方形角点，35 个点，两条边都很清晰。

当前代码始终先选 A，因为 `segments[A].size() > segments[B].size()`，然后只对 A 运行 Single Face / Corner 判据；即使 A 被判 `INVALID`，也**不会回头检查 B**。

由此有两种不同风险：

**风险一：漏检（false negative）。** A 不符合箱体几何 → `INVALID`，但真实箱体 B 事实上已被 LiDAR 观测到。  
**风险二：误检（false positive）。** A 如果恰巧具有合适的可见长度、较小 RMSE 等，可能作为 `SINGLE_FACE` 通过，从而输出错误目标的位置。

**根本原因**：当前实现解决了“给定一组候选点，是否符合正方形边/角点的几何模型”，但没有可靠解决“在多个候选点段中，哪个才是真正目标”的身份选择问题。

### 11.2 今天的工程决策：记录，但不立即优化

用户明确决定：**当前不准备处理这个缺陷；Phase 4 最初学习要求已达到。等后续出现真实比赛或机器人任务需求，再返回优化。**

理由：

1. 该失败机制由代码逻辑可以推出，值得保留，但尚未针对实际比赛环境量化发生概率。
2. 当前目标是掌握 ROS2、LiDAR 数据流、TLS、Single/Corner 几何估计和实验闭环；这些学习目标已经完成。
3. 现在贸然引入多候选评分、复杂目标跟踪或其他算法，会增加复杂度且偏离主线。

**未来若确有需求，再做的最小实验**（备忘，不是当前待办）：固定算法和已知箱体位置，仅改变 ROI 内干扰点段的存在及点数；查看 `segment_sizes`、`target_segment_index`、`single_check`、`corner_check`、`pose_mode` 和位姿。确认真实失败后，再评估是否需要对多个 segment 分别验证。

还要注意：**尝试多个 segment 可能改善漏检，但如果存在多个几何上都像箱体的候选，并不能自动保证目标身份正确。** 真正的目标选择标准应由未来的比赛任务约束决定。

### 11.3 其他已经知道、但当前不扩展的边界

- **依赖 ROI 与实验布置**：场景改变可能导致目标落到 ROI 外或混入干扰物。
- **几何拟合不能保证身份识别**：RMSE 小或长度符合不能独立证明是目标箱体。
- **观测不完整时可能 `INVALID`**：属于目前判据的结果，不等于物理世界没有箱体。
- **缺少绝对精度证据**：现有日志和重复输出还不能替代独立真值评估。
- **坐标系限定**：当前为 `LaserScan` 参考系，未实现 `laser → base_link/map` 的 TF 转换。
- **下游过期位姿**：无效帧没有新 `/box_pose`，但旧位姿需要下游主动按时间戳、模式与任务逻辑管理。
- **二维平面假设**：当前仅处理二维扫描截面、已知正方形边长和指定可见面几何，不覆盖任意立体物体检测。

这些内容记作**工程限制与可恢复的未来问题**，不是要求本阶段全部解决的任务清单。

---

## 12. 项目恢复指南：未来需要继续时先做什么？

仅在有**具体比赛或机器人场景需求**时重启本模块，建议依序：

1. **写清任务约束**：箱体尺寸和标记、LiDAR 安装位置、检测距离、预期环境干扰、允许的误检/漏检、下游控制需要的坐标系和频率。
2. **恢复现有基线**：定位本文引用的代码修订，复查 `/scan → /box_mode + /box_pose` 与已有单面/角点/负例测试。
3. **收集真实失败数据**：保留原始 `LaserScan` 或 rosbag、模式日志、分段点数、当前位姿和独立真值（如可得）。
4. **分层定位**：雷达数据是否有效 → 时间戳/坐标系是否正确 → ROI 是否包含目标 → 连续点段是否选对 → TLS 拟合 → 模式与有效性。
5. **仅针对确认的问题改动**：如果是最大点数段误选，再设计并验证多候选选择；如果是坐标系/通信错误，不能直接通过改 TLS 阈值解决。
6. **控制变量复测**：至少对单面、角点、无目标、干扰段这几类场景比较改动前后的结果。

### 阶段结论（写给未来的自己）

本次 Phase 4 的价值不只是“写出一个识别正方形箱体的 ROS2 节点”。更重要的是，已经能够：

- 沿真实执行路径解释回调、点云过滤、分段、TLS、两种位姿恢复、几何检查和 ROS2 发布；
- 从最小二乘目标推导散布矩阵与特征值的关系，解释 `normal`、`direction`、RMSE、投影与中点为何这样计算；
- 用已知几何推导 Single Face 和 Corner 中心，而不是机械照搬代码；
- 区分算法拟合、几何有效性、目标身份与下游控制的不同责任；
- 发现最大点数点段策略在多候选情况下的缺陷，并作出**暂缓优化、待真实任务驱动**的工程决定。

**当前阶段状态：已完成，停止在可理解、可验证的最小闭环。**

---

## 附：快速复习索引

| 想复习的问题 | 直接查看 |
|---|---|
| 谁触发 `scan_callback()`？ | 第 2 节；Node 构造函数 / `main()` |
| 过滤后原始扫描序号在哪里？ | 第 3 节；`Point2D.scan_index` |
| 连续点段到底按什么分段？ | 第 3.3 节；`point_gap > 0.02` |
| 为何最小特征值对应法向量？ | 第 4.3 节；`min nᵀSn` 且 `||n||=1` |
| `centroid` 与 `visible_mid` 区别？ | 第 4.5 节 |
| 为什么 RMSE 不能单独认定箱体？ | 第 4.6、5.3 节 |
| 单面中心怎样算？ | 第 5.1 节 |
| 为什么正方形角度按 90° 归一化？ | 第 5.2 节 |
| 两线为何最小化 `lambda_min1+lambda_min2`？ | 第 6.1 节 |
| Corner 为什么要测交点到最近端点距离？ | 第 6.4、6.5 节 |
| Corner 中心公式是什么？ | 第 6.6 节 |
| `INVALID` 时会发布什么？ | 第 7.2 节 |
| 为什么不能把位姿当成 `map` 坐标？ | 第 7.3 节 |
| 最大点数干扰线问题是什么？ | 第 11.1 节 |
| 将来何时继续优化？ | 第 11.2、12 节 |

**核心公式速查**：

```text
极坐标转点：      x = r cosθ,  y = r sinθ
TLS 质心：       p̄ = (1/N) Σpᵢ
TLS 散布矩阵：   S = Σ(pᵢ-p̄)(pᵢ-p̄)ᵀ
TLS 目标：       min_{||n||=1} nᵀSn
直线方程：       n=(a,b), c=-n·p̄, ax+by+c=0
沿线投影：       tᵢ=(pᵢ-p̄)·direction
可见长度：       L=tmax-tmin
可见中点：       visible_mid=p̄+((tmin+tmax)/2)direction
拟合 RMSE：      sqrt(lambda_min/N)
Single 中心：    visible_mid + (s/2) inward_normal
Two-Line 搜索：  k*=argmin_k (SSE₁(k)+SSE₂(k))
Corner 中心：    intersection + (s/2)u₁ + (s/2)u₂
点积垂直性：     |direction₁·direction₂|≈0
Yaw 四元数：     (0,0,sin(yaw/2),cos(yaw/2))
```
