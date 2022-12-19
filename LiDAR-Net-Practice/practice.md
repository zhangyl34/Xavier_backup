
# 组会

## 12.19 组会

__1. 体素地图美化__

体素大小 0.05x0.05x0.05m。
<img src="img/12_19_voxelmap.png" width=90%>

__2. RANSAC 方法点云粗配准__

==前提假设：VoteNet 已经估计出了“大致准确”的台车 b-box。换句话说，已知“大致准确”的台车位姿。接下来分两步优化台车位姿：RANSAC 粗调；ICP 精调。==
<img src="img/12_19_bbox.png" width=40%>

* source model:
b-box 内部点云 -> 剔除具有自由度的点云
<img src="img/12_19_source1.png" width=46%> <img src="img/12_19_source2.png" width=34%>
剔除台车大立柱以上部分和控制面板。使用五个面（底座的四个侧面 + 地面）进行点云配准。

* target model:
SOLIDWORKS(.stl) -> blender(.obj) -> pcl(.ply)
<img src="img/12_19_target1.png" width=34%> <img src="img/12_19_target2.png" width=46%>
在原始模型的基础上增加地面信息。

* 初值
<img src="img/12_19_match1.png" width=40%>

* 预处理
    * 点云降采样   
    * 估计点云法向量
    * 计算点云 FPFH 特征（描述邻域几何特征的 33 维向量）。_Fast Point Feature Histograms (FPFH) for 3D registration, ICRA, 2009._

* RANSAC
    * 每次迭代，在 source model 里随机选择 n 个点。
    * 在 target model 中找到对应的 FPFH 特征最相似点。
    * 筛除误匹配点对：匹配点对 transform 后的空间距离应相近；匹配线段的长度应相近。
    * 使用正确匹配的点对计算 T 矩阵。

* 粗匹配结果
<img src="img/12_19_match2.png" width=50%>

__3. ICP 方法点云精配准__

* Point-to-plane ICP
    _Object modelling by registration of multiple range images, Image and Vision Computing, 1992._
    <img src="img/12_19_icp.png" width=30%>
    * 基于当前迭代的 T，查找匹配点对 $\mathcal{K}$：$\mathbf{p} \in Target, \mathbf{q} \in Source$。
    * 基于优化方程的雅可比矩阵，更新 T。

* 精匹配结果：
<img src="img/12_19_match3.png" width=50%>

__4. VoteNet__





# VoteNet 实战



