<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [物体追踪](#物体追踪)
  - [BundleSDF: Neural 6-DoF Tracking and 3D Reconstruction of Unknown Objects](#bundlesdf-neural-6-dof-tracking-and-3d-reconstruction-of-unknown-objects)

<!-- /code_chunk_output -->

## 物体追踪

--- 

### BundleSDF: Neural 6-DoF Tracking and 3D Reconstruction of Unknown Objects

__NVIDIA | CVPR 2023__

<img src="img/bundleSDF_2.png">

<font color=OrangeRed>Abstract:</font>

* 单目 RGBD 相机；
* 接近实时（10 Hz）；
* 6-DoF pose tracking + 3D reconstruction of an unknown object；
* 能够应对“遮挡、反射、缺乏纹理、缺乏几何特征、突然抖动”等情况。

<font color=OrangeRed>3.1 Coarse Pose Initialization:</font>

__由上一帧的物体位姿粗略估计当前帧的物体位姿。__
首先用 video segmentation network 获取 mask；再用 keypoint matching network 提取特征点的描述子；最后用 RANSAC 估计位姿。

<font color=OrangeRed>3.2 Memory Pool:</font>

__记忆池用于保存关键帧信息；关键帧可以用来优化 3.1 节得到的粗略位姿。__
* 当某一帧与记忆池中的关键帧姿态差异较大时，该帧被加入到记忆池中。
* 每个关键帧都关联一个 bool 值。一开始 bool 值为 false，代表该帧的位姿需要在 pose graph 中被优化；当该位姿在 neural object field 中第一次被优化，则 bool 值置为 true，该位姿从此只在 neural object field 中被优化。

<font color=OrangeRed>3.3 Online Pose Graph Optimization:</font>

__每次从记忆池中选择姿态最接近的 K 个关键帧，优化当前帧的物体位姿。__
* 用 Gauss-Newton 法最小化优化方程。
* Lf 项代表匹配的特征点之间的位置误差。
* Lp 项代表特征点重投影后的点面误差。
* Ls 项代表特征点到 implicit object shape 的误差。

<font color=OrangeRed>3.4 Neural Object Field:</font>

__用记忆池中的关键帧学习 neural object field。__
* object field 用两个映射来表示：geometry function + appearance function。
* 这两个映射分别用两个神经网络在线学习。



