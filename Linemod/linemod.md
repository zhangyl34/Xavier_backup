
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [类定义](#类定义)
  - [class shapeInfo_producer](#class-shapeinfo_producer)
  - [class ColorGradient](#class-colorgradient)
  - [class ColorGradientPyramid](#class-colorgradientpyramid)
  - [class Detector](#class-detector)
- [TRAIN](#train)
- [TEST](#test)

<!-- /code_chunk_output -->

---

# 类定义

## class shapeInfo_producer

**存放某一模板，做一系列旋转和尺度缩放的信息**

- produce_infos() 根据 angle_range 和 scale_range，生成一系列模板信息，存入 infos。
- src_of() 根据 info 信息，对 src 做 transform 变换。

## class ColorGradient

**存放阈值信息和特征点个数；提供读写功能；ColorGradientPyramid 的接口**

- process() 根据成员变量和 src&mask，返回一个指向 ColorGradientPyramid 的指针。构造函数内调用了 update() 函数，实际调用的是 quantizedOrientations()。

## class ColorGradientPyramid

**提取特征**

- pyrDown() num_features/2; size/2; update()。update() 实际调用的是 quantizedOrientations()。
- quantizedOrientations() 利用 sobel 算子提取 x 和 y 方向的梯度，三色通道用最大梯度通道作为代表；并计算 magnitude 和 phase；调用 hysteresisGradient() 函数。
- hysteresisGradient() 计算每个像素方格的主方向。将 360 度的方向划分为 16 个区间，并对角线区间合并，最终得到8个方向区间；如果某方格 magnitude 大于阈值，且周围 3×3 个方格的方向有 5 个以上是一致的，则认定该方向为该方格的主方向。
- extractTemplate() 将每个像素方格的 magnitude 与周围 5×5 个方格进行比较，如果最大则保留，并将其余 24 个置为 0；magnitude 大于阈值的方格成为 candidate；调用 selectScatteredFeatures() 筛选 candidates。
- selectScatteredFeatures() 把 candidates 中空间距离过近的剔除，最终筛选约 num_feature 个，存入 features。
- quantize() 把 angle 拷贝出来。


## class Detector

- modality 是指向 ColorGradient 的指针；
- addTemplate() 某一模板未经 transform 的 src&mask 作为输入，对其金字塔调用 extractTemplate()，调用结果存入 TemplatePyramid；根据 TemplatePyramid 特征点的坐标极值，修剪 TemplatePyramid；将 TemplatePyramid 存入 class_templates （这是一个 map\<class_id, template_pyramids\>，其中 class_id="test"）。
- addTemplate_rotate() 根据模板未经 transform 的特征点信息，作 rotate 变换后存入 class_templates。
- match() 结构体 Match 用来存放 class_id, template_id, similarity, x, y 等信息。LinearMemoryPyramid 的结构为 vector\<vector\<linearMemories\>\>，分别代表 <两层金字塔<一张图像<八个方向的线性存储矩阵>>>。首先计算 LinearMemoryPyramid；再调用 matchClass()，得到 match 信息。
- spread() 调用了 orUnaligned8u()，将每个像素方格的方向用周围 T×T 的方格填充，得到方向矩阵，每个方格用 2^8 来表示方向。
- computeResponseMaps() 根据输入的方向矩阵，生成 8 个 response_map （方向矩阵每个方格中的方向，与 8 个标准方向最小夹角的 cos 值）。
- linearize() 将某一 response_map 储存成线性形式，即 (T×T,...) 的矩阵。
- matchClass() 先用图像最小的金字塔和模板最小的金字塔去做匹配，核心在于调用了 similarity_64() 函数，得到 similarity map；得分高于阈值的位置成为候选区，进一步用更大的图像金字塔和更大的模板金字塔去做匹配。
- similarity_64() 根据 feature 大小循环调用 accessLinearMemory()，将得到的每行 offset linear memory 相加。最终每 T×T 的方格，只有一个 similarity 数据，论文中有 T×T 个数据。
- accessLinearMemory() 每个 feature 调用一次此函数。根据 feature 方向，确定 8 个 linear_memories Mat 中的一个；根据 feature 在 T×T 内的位置，确定 Mat T×T 行中的一行；根据 feature 的 T×T 在图像上的位置，确定 offset (lm_index)。最终返回该 feature 对应的一行 offset linear memory。

------------

# TRAIN

生成 detector 实例，num_feature=128，T={4，8}，分别代表两层金字塔的 T。

将 img 和 mask 扩展为 padded_img 和 padded_mask，使图像可以被旋转。

将 padded_img 和 padded_mask 传给 shapeInfo_producer 实例，生成一系列旋转和平移变换后的模板信息。

循环调用 addTemplate() 和 addTemplate_rotate() 将特征信息存入 class_templates。训练时，没有用到 spread()，每个方格只有一个方向。

最后保存数据到文件。

------------

# TEST

读取模板信息，读取图片。核心在于调用了 match() 函数。


