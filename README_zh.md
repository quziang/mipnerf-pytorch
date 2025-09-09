# PyTorch mip-NeRF 

mip-NeRF 的 PyTorch 重新实现。

![nerfTomipnerf](https://user-images.githubusercontent.com/42706447/173477320-06b7705c-d061-4933-a8be-8c1c272ee101.png)

虽然与官方仓库不完全一致，但我们按照自己的喜好重新组织了代码结构（主要是数据集的结构和为了在消费级显卡上运行而调整的超参数），使其更加模块化，并删除了一些重复的代码，但仍能达到相同的效果。

## 特性

* 可以使用球形或螺旋姿态为所有3个数据集生成视频
* 球形：

https://user-images.githubusercontent.com/42706447/171090423-2cf37b0d-44c9-4394-8c4a-46f19b0eb304.mp4

* 螺旋：

https://user-images.githubusercontent.com/42706447/171099856-f7340263-1d65-4fbe-81e7-3b2dfa9e93b8.mp4

* 深度和法线视频渲染：
* 深度：

https://user-images.githubusercontent.com/42706447/171091394-ce73822c-689f-496b-8821-78883e8b90d4.mp4

* 法线：

https://user-images.githubusercontent.com/42706447/171091457-c795855e-f8f8-4515-ae62-7eeb707d17bc.mp4

* 可以提取网格模型

https://user-images.githubusercontent.com/42706447/171100048-8f57fc9a-4be5-44c2-93dd-ee5f6b54dd6e.mp4

https://user-images.githubusercontent.com/42706447/171092108-b60130b5-297d-4e72-8d3a-3e5a29c83036.mp4

## 未来计划

将来我们计划实现/改进：

* 进一步消除重复/冗余代码，优化GPU内存和渲染速度
* 清理和扩展网格提取代码
* 为multicam数据集添加缩放姿态
* [Mip-NeRF 360: 无界抗锯齿神经辐射场](https://jonbarron.info/mipnerf360/) 支持
* [NeRV: 用于重光照和视图合成的神经反射率和可见性场](https://pratulsrinivasan.github.io/nerv/) 支持

## 安装/运行

1. 使用 `mipNeRF.yml` 创建conda环境
2. 获取训练数据
   1. 运行 `bash scripts/download_data.sh` 下载所有3个数据集：LLFF、Blender和Multicam。
   2. 单独运行对应的bash脚本下载个别数据集
         * `bash scripts/download_llff.sh` 下载LLFF数据集
         * `bash scripts/download_blender.sh` 下载Blender数据集
         * `bash scripts/download_multicam.sh` 下载Multicam数据集（注意这也会下载blender数据集，因为multicam基于它派生）
3. 可选择更改配置参数：可以在 `config.py` 中更改默认参数或通过命令行参数指定
    * 默认配置设置适用于高端消费级显卡（约8-12GB显存）
4. 运行 `python train.py` 开始训练
   * `python -m tensorboard.main --logdir=log` 启动tensorboard
5. 运行 `python visualize.py` 从训练好的模型渲染视频
6. 运行 `python extract_mesh.py` 从训练好的模型提取网格

## 代码结构

我在[这里](misc/Code_zh.md)更详细地解释了代码的具体细节，但这里是一个基本概述。

* `config.py`：指定超参数。
* `datasets.py`：基础通用 `Dataset` 类 + 3个默认数据集实现。
  * `NeRFDataset`：所有数据集都应该继承的基类。
  * `Multicam`：用于原始mip-NeRF论文中的multicam数据。
  * `Blender`：用于原始NeRF中的合成数据集。
  * `LLFF`：用于原始NeRF中的llff数据集。
* `loss.py`：mip-NeRF损失函数，主要就是MSE，但也计算psnr。
* `model.py`：mip-NeRF模型，不像原作者写的那样模块化，但按这种逐字方式展示时更容易理解其结构。
* `pose_utils.py`：用于生成姿态的各种函数。
* `ray_utils.py`：与模型输入光线相关的各种函数，大部分在模型的前向函数中使用。
* `scheduler.py`：mip-NeRF学习率调度器。
* `train.py`：训练mip-NeRF模型。
* `visualize.py`：使用训练好的mip-NeRF创建视频。

## mip-NeRF 总结

这里是我在编写这个项目时写的关于NeRF和mip-NeRF工作原理的总结。

* [总结](misc/Summary_zh.md)

## 结果

<sub><sup>所有PSNR都是平均PSNR（粗糙+精细）。</sub></sup>

### LLFF - Trex

<div>
   <img src="https://user-images.githubusercontent.com/42706447/173477393-8b93a3f8-3624-4826-a67c-82923d03ea34.png" alt="pic0" width="49%">
   <img src="https://user-images.githubusercontent.com/42706447/173477391-1f932ca3-6456-4af5-b041-bf63dbbed68a.png" alt="pic1" width="49%">
</div>
<div>
   <img src="https://user-images.githubusercontent.com/42706447/173477394-9ab07f60-58b9-4311-8aba-c052412b4f68.png" alt="pic2" width="49%">
   <img src="https://user-images.githubusercontent.com/42706447/173477395-d69bdb34-ea6e-43de-8315-88c6f5e251e7.png" alt="pic3" width="49%">
</div>

<br>
视频：
<br>

https://user-images.githubusercontent.com/42706447/171100120-0a0c9785-8ee7-4905-a6f6-190269fb24c6.mp4

<br>
深度：
<br>

https://user-images.githubusercontent.com/42706447/171100098-9735d79a-c22f-4873-bb4b-005eef3bc35a.mp4

<br>
法线：
<br>

https://user-images.githubusercontent.com/42706447/171100112-4245abd8-bf69-4655-b14c-9703c13c38fb.mp4

### Blender - Lego

<div>
   <img src="https://user-images.githubusercontent.com/42706447/173477588-a4d0034d-b8e5-4ea2-9459-5fff3e6b1cde.png" alt="pic0" width="49%">
   <img src="https://user-images.githubusercontent.com/42706447/173477593-d23a9603-b6b5-4d4f-9a2b-dcfd0d646dbc.png" alt="pic1" width="49%">
</div>
<div>
   <img src="https://user-images.githubusercontent.com/42706447/173477594-ee6e5dda-b704-4403-9433-ee93bf2a8154.png" alt="pic2" width="49%">
   <img src="https://user-images.githubusercontent.com/42706447/173477595-2f0e2d88-e241-4ddc-809d-927c6e01c881.png" alt="pic3" width="49%">
</div>

视频：
<br>

https://user-images.githubusercontent.com/42706447/171090423-2cf37b0d-44c9-4394-8c4a-46f19b0eb304.mp4

<br>
深度：
<br>

https://user-images.githubusercontent.com/42706447/171091394-ce73822c-689f-496b-8821-78883e8b90d4.mp4

<br>
法线：
<br>

https://user-images.githubusercontent.com/42706447/171091457-c795855e-f8f8-4515-ae62-7eeb707d17bc.mp4

### Multicam - Mic

<div>
   <img src="https://user-images.githubusercontent.com/42706447/173477781-2c48d8e0-b0e5-4cd4-9599-cc0336333b30.png" alt="pic0" width="49%">
   <img src="https://user-images.githubusercontent.com/42706447/173477778-9fd4c802-e0b2-4e0b-bc31-6f27abc92c87.png" alt="pic1" width="49%">
</div>
<div>
   <img src="https://user-images.githubusercontent.com/42706447/173477782-ec40bc91-1da7-49d2-b65b-b3250f34a8fc.png" alt="pic2" width="49%">
   <img src="https://user-images.githubusercontent.com/42706447/173477784-8dfa7bc7-7122-40ed-855a-0081a593f1ce.png" alt="pic3" width="49%">
</div>

视频：
<br>

https://user-images.githubusercontent.com/42706447/171100600-7f3307c7-0ca4-4677-b9b7-180cf27fd175.mp4

<br>
深度：
<br>

https://user-images.githubusercontent.com/42706447/171100593-e0139375-1ae6-4235-8961-ba3c45f88ead.mp4

<br>
法线：
<br>

https://user-images.githubusercontent.com/42706447/171092348-9315a897-a6a3-4c49-aedf-3f3331fdfe52.mp4

## 参考资料/贡献

* 感谢 [Nina](https://github.com/ninaahmed) 协助编写代码
* [原始NeRF Tensorflow代码](https://github.com/bmild/nerf)
* [NeRF项目页面](https://www.matthewtancik.com/nerf)
* [NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis](https://arxiv.org/abs/2003.08934)
* [原始mip-NeRF JAX代码](https://github.com/google/mipnerf)
* [mip-NeRF项目页面](https://jonbarron.info/mipnerf/)
* [Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields](https://arxiv.org/abs/2103.13415)
* [nerf_pl](https://github.com/kwea123/nerf_pl)
