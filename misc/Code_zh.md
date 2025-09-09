# 代码笔记（中文翻译版）

## 数据集

* `NeRFDataset` 是一个通用的 PyTorch `Dataset` 对象，你可以继承它来实现自己的数据集，用于训练/可视化 mip-NeRF。
* 基本思想是数据集应输出一个 `Ray`（大致对应 `self.rays`）对象，如果是训练则还要输出该射线对应的 RGB 像素（大致对应 `self.images`）标签。
  * `Ray` 是在 `ray_utils.py` 中定义的 namedtuple，包含以下内容：
    * `origins`：射线起点的三维笛卡尔坐标
    * `directions`：射线方向的三维笛卡尔向量
    * `viewdirs`：每条射线的观察方向三维向量。仅在使用 NDC 射线时使用，否则 `directions` 等于 `viewdirs`
    * `radii`：射线的半径（圆锥的底半径）
    * `lossmult`：用于实现射线正则化损失的掩码（本质上是加权 MSE）
      * 仅 Multicam 数据集使用，其他数据集该掩码全为 1，相当于取均值
    * `near`：近裁剪面
    * `far`：远裁剪面
* 该 Ray 对象是模型的输入。当用训练好的模型渲染场景时，会从任意姿态生成这些射线。
* 数据集可以分为 3 个子集：
  * `"train"`：用于训练，输出（rays, pixels）供模型训练
  * `"test"`：用于评估/测试，输出模型未直接训练过的（rays, pixels）
  * `"render"`：用于训练好的模型渲染场景，生成任意姿态的（rays），在 `visualize.py` 中用于生成场景视频
* 数据集在构造时会调用以下函数之一：
  * 两个函数都会随后调用特定的姿态函数生成姿态，最后调用 `generate_rays()` 从姿态生成射线。姿态函数会初始化 `self.h`、`self.w`、`self.focal`、`self.cam_to_world`，如果是训练还会初始化 `self.images`。
  * `generate_training_rays()` 用于 `['train', 'test']` 子集
    * `generate_training_poses`：生成训练姿态，通常从磁盘读取文件并设置相关实例变量；每个子类都必须实现此函数。
  * `generate_training_rays()` 用于 `['render']` 子集
    * `generate_render_poses`：生成用于渲染场景的任意姿态
* 如果你继承该对象，理论上只需实现 `generate_training_rays()`，但其他函数的默认实现可能不适合你的数据集。

### 默认数据集

有 3 个默认数据集：

* `LLFF`：真实图像数据集，最初由 [Local Light Field Fusion](https://github.com/fyusion/llff) 作者整理，NeRF/mip-NeRF 论文作者也整理了一些场景。假设场景为前向。
* `Blender`：由 NeRF/mip-NeRF 论文作者用 Blender 模型合成的数据集。
* `Multicam`：Blender 数据集的多尺度变体。

## 姿态（Poses）

* 姿态以 $3 \times 5$ 的 numpy 数组存储，由 $3 \times 4$ 的相机到世界变换矩阵和 $3 \times 1$ 的针孔相机内参向量组成。
  * `cam_to_world` 变换矩阵：
$$
\begin{bmatrix}
r_{0} & r_{3} & r_{6} & t_{0}\\
r_{1} & r_{4} & r_{7} & t_{1}\\
r_{2} & r_{5} & r_{8} & t_{2}
\end{bmatrix}
$$
其中 $[r_{0}, r_{1}, r_{2}]$ 为 x 轴旋转，$[r_{3}, r_{4}, r_{5}]$ 为 y 轴旋转，$[r_{6}, r_{7}, r_{8}]$ 为 z 轴旋转，$[t_{0}, t_{1}, t_{2}]$ 为平移。
  * 相机内参：
$$
\begin{bmatrix}
\text{height} \\
\text{width}  \\
\text{focal}
\end{bmatrix}
$$

## 配置（Config）

### 基本超参数

* `--log_dir`：
  * 默认：`"log_dir"`
  * 说明：保存模型/优化器权重、tensorboard、可视化视频的位置
* `--dataset_name`
  * 默认：`"blender"`
  * 选项：`["llff", "blender", "multicam"]`
  * 说明：使用哪个数据集
* `--scene`
  * 默认：`"lego"`
  * 说明：数据集中的哪个场景

### 模型超参数

* `--use_viewdirs`
  * 默认：`True`
  * 说明：是否使用视角方向作为条件
* `--randomized`
  * 默认：`True`
  * 说明：是否使用随机分层采样
* `--ray_shape`
  * 默认：`"cone"`
  * 选项：`["cone", "cylinder"]`
  * 说明：射线形状。llff 数据集用 "cylinder"，其他用 "cone"
* `--white_bkgd`
  * 默认：`True`
  * 说明：背景是否为白色。llff 用 False，其他用 True
* `--override_defaults`
  * 默认：`False`
  * 说明：是否覆盖默认数据集配置。默认情况下，选择 llff 数据集会自动调整如 `--white_bkgd`、`--ray_shape` 等配置，除非指定本参数
* `--num_levels`
  * 默认：`2`
  * 说明：采样层数
* `--num_samples`
  * 默认：`128`
  * 说明：每层采样点数
* `--hidden`
  * 默认：`256`
  * 说明：MLP 宽度
* `--density_noise`
  * 默认：`0.0`
  * 说明：加到原始密度上的噪声标准差。llff 用 1.0，其他用 0.0
* `--density_bias`
  * 默认：`-1.0`
  * 说明：原始密度激活前的偏移
* `--rgb_padding`
  * 默认：`0.001`
  * 说明：RGB 输出的填充值
* `--resample_padding`
  * 默认：`0.01`
  * 说明：直方图的 Dirichlet/alpha 填充值
* `--min_deg`
  * 默认：`0`
  * 说明：3D 点位置编码的最小阶数
* `--max_deg`
  * 默认：`16`
  * 说明：3D 点位置编码的最大阶数
* `--viewdirs_min_deg`
  * 默认：`0`
  * 说明：视角方向位置编码的最小阶数
* `--viewdirs_max_deg`
  * 默认：`4`
  * 说明：视角方向位置编码的最大阶数

### 损失与优化器超参数

* `--coarse_weight_decay`
  * 默认：`0.1`
  * 说明：粗网络损失的权重衰减
* `--lr_init`
  * 默认：`1e-3`
  * 说明：初始学习率
* `--lr_final`
  * 默认：`5e-5`
  * 说明：最终学习率
* `--lr_delay_steps`
  * 默认：`2500`
  * 说明：学习率预热步数
* `--lr_delay_mult`
  * 默认：`0.1`
  * 说明：预热幅度
* `--weight_decay`
  * 默认：`1e-5`
  * 说明：优化器（AdamW）权重衰减

### 训练超参数

* `--factor`
  * 默认：`1`
  * 说明：图像下采样因子。1 表示不下采样
* `--max_steps`
  * 默认：`200_000`
  * 说明：训练步数
* `--batch_size`
  * 默认：`2048`
  * 说明：每个训练批次的射线/像素数
* `--do_eval`
  * 默认：`True`
  * 说明：是否每隔 `--save_every` 步进行一次验证/评估
* `--continue_training`
  * 默认：`False`
  * 说明：是否继续训练
* `--save_every`
  * 默认：`1000`
  * 说明：每隔多少步保存一次检查点，并在 `--do_eval` 时进行评估
* `--device`
  * 默认：`"cuda"`
  * 说明：模型和数据加载到的设备。每次加载一个 batch。

### 可视化超参数
* `--chunks`
  * 默认：`8192`
  * 说明：可视化渲染时将射线分块的大小。不会影响视频质量。尽量大但别 OOM。8192 约等于 10GB
* `--model_weight_path`
  * 默认：`"log/model.pt"`
  * 说明：调用 `visualize.py` 时加载模型权重的位置
* `--visualize_depth`
  * 默认：`False`
  * 说明：调用 `visualize.py` 时是否渲染深度视频
* `--visualize_normals`
  * 默认：`False`
  * 说明：调用 `visualize.py` 时是否渲染法线视频

## 有用资源
* [针孔相机模型](https://www.scratchapixel.com/lessons/3d-basic-rendering/3d-viewing-pinhole-camera)
* [生成相机射线](https://www.scratchapixel.com/lessons/3d-basic-rendering/ray-tracing-generating-camera-rays/generating-camera-rays)
* [标准坐标系](https://www.scratchapixel.com/lessons/3d-basic-rendering/ray-tracing-generating-camera-rays/standard-coordinate-systems)
* [计算 3D 点的像素坐标](https://www.scratchapixel.com/lessons/3d-basic-rendering/computing-pixel-coordinates-of-3d-point/mathematics-computing-2d-coordinates-of-3d-points)
* [投影矩阵](http://www.songho.ca/opengl/gl_projectionmatrix.html)
