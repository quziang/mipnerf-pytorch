# mip-NeRF

<span style="font-weight: bold;font-size:14px;">Benjamin Noah Beal</span>

---

要理解 mip-NeRF <a class="citation" href="#r1"><sup id="citation-r1">1</sup></a>，首先需要理解 NeRF <a class="citation" href="#r1"><sup id="citation-r1">2</sup></a>。

## NeRF

NeRF 是一种通过使用稀疏的输入视图来优化底层连续体积场景函数，从而合成复杂场景新视角的方法。该算法使用全连接深度神经网络，其输入是单个连续的5D坐标（空间位置 $(x, y, z)$ 和观察方向 $(\theta, \phi)$），输出是该空间位置的体积密度和视角相关的发射辐射度。

<center>
<figure>
    <img src="https://user-images.githubusercontent.com/42706447/173478044-32c19409-ce43-4582-b487-a089019cf750.png" alt="NeRF Pipeline">
    <figcaption><b>图 1：NeRF 流水线</b></figcaption>
</figure>
</center>

该方法训练神经网络来参数化场景的神经辐射场，这实际上意味着网络学会在其权重中编码场景的3D空间信息，然后可以用于从任意视点渲染场景。

### NeRF 流水线

NeRF 模型的输入是前面提到的5D坐标，但是，这些坐标是通过沿着相机射线采样点生成的，因此实际上 NeRF 模型的输入有效地是一批射线。射线定义为 $\mathcal{R}(t) = o + td$，其中 $o$ 是射线在世界空间中的原点，$d$ 是射线的方向，$t$ 是射线上的某个点。

模型对沿射线的采样点进行两次迭代，其理由是通过分层采样获得的射线初始样本 $N_c$ 代表模型的"粗粒度"评估，然后可以用来产生更有信息的采样 $N_f$，偏向于体积的相关部分，代表"细粒度"评估。两组样本都以相同的方式通过模型，第一个输出只是用于在第二次迭代中获得更好的样本。第一组和第二组样本 $N_c + N_f$ 都用于计算射线的最终渲染颜色。

沿射线的样本使用位置编码进行编码，类似于 transformer 模型的编码。

#### 位置编码 (PE)

位置编码获取射线样本中包含的每个 $t$ 值 $t_k \in t$，计算其沿射线对应的3D位置 $\bold{x} = \mathcal{R}(t)$，并按以下方式编码：

$$
\gamma(\bold{x}) = \left[ \sin(\bold{x}), \cos(\bold{x}), \dots, \sin(2^{L-1}\bold{x}), \cos(2^{L-1}\bold{x}) \right]^{T}
$$

这只是将3D位置 $\bold{x}$ 的每个维度的正弦和余弦连接起来，按从 $1$ 到 $2^{L−1}$ 的 $2$ 的幂次缩放，其中 $L$ 是超参数。

编码后，特征被输入到 MLP 模型中，产生两个不同的预测：体积密度预测 $\sigma$ 和颜色预测 $c = (r, g, b)$。

最后使用体积渲染技术将这些值合成为图像。

### NeRF 架构

<center>
<figure>
    <img src="https://user-images.githubusercontent.com/42706447/173478103-a23aec25-ade3-4dfd-b320-b53da221eaad.png" alt="NeRF Model Architecture">
    <figcaption><b>图 2：NeRF 模型架构</b></figcaption>
</figure>
</center>

* 输入向量用绿色显示
  * 位置编码层用 $\gamma(\bold{x})$ 表示
* 隐藏向量用蓝色显示
* 输出向量用红色显示
  * 最终的 RGB 输出向量在 sigmoid 激活后产生

## mip-NeRF

mip-NeRF 从计算机图形渲染管线中用于防止锯齿的 mipmapping 方法中获得灵感，并扩展 NeRF 以同时表示连续尺度空间的预过滤辐射场。它通过有效地向场景发射圆锥或圆柱体而不是射线，并使用高斯分布来近似与像素对应的圆锥台来实现这一点。这种改进解决了 NeRF 模型在锯齿和尺度处理方面的缺陷。

<center>
<figure>
    <img src="https://user-images.githubusercontent.com/42706447/173478183-0a627268-b1b0-45e8-a8ad-d34bcd84294e.png" alt="mip-NeRF">
    <figcaption><b>图 3：mip-NeRF</b></figcaption>
</figure>
</center>

### mip-NeRF 流水线

mip-NeRF 流水线与 NeRF 流水线相同，但有以下例外：

#### 圆锥台的高斯近似

NeRF 向场景发射射线：

<center>
<figure>
    <img src="https://user-images.githubusercontent.com/42706447/173478219-3fcf4dfb-ce2b-49bf-b35a-8d6170bf8065.png" alt="NeRF Ray">
    <figcaption><b>图 4：NeRF 射线</b></figcaption>
</figure>
</center>

mip-NeRF 向场景发射圆锥，然后将圆锥切片成圆锥台：

<center>
<figure>
    <img src="https://user-images.githubusercontent.com/42706447/173478242-88264643-8b9b-473a-84ad-dfbdcbc1e38b.png" alt="mip-NeRF frustum">
    <figcaption><b>图 5：mip-NeRF 抗锯齿圆锥台</b></figcaption>
</figure>
</center>

然后我们拟合多元高斯分布来近似圆锥台。这允许有效地近似位于*台体内*的点，因为实际使用圆锥台方程需要计算没有闭式解的积分。

<center>
<figure>
    <img src="https://user-images.githubusercontent.com/42706447/173478267-d96de152-5639-4b04-bac4-6a6086805fed.png" alt="mip-NeRF gaussian">
    <figcaption><b>图 6：mip-NeRF 高斯近似</b></figcaption>
</figure>
</center>

#### 积分位置编码 (IPE)

最后，两者之间的另一个主要区别是 mip-NeRF 实现了积分位置编码来编码根据前述高斯分布分布的坐标。这是 NeRF 位置编码的推广。为了理解它，将 PE 重写为傅立叶特征是有帮助的：

$$
\bold{P} = 
\begin{bmatrix}
1 & 0 & 0 & 2 & 0 & 0 & & 2^{L-1} & 0 & 0\\
0 & 1 & 0 & 0 & 2 & 0 &  \dots & 0 & 2^{L-1} & 0\\
0 & 0 & 1 & 0 & 0 & 2 & & 0 & 0 & 2^{L-1}
\end{bmatrix}^{T}, \ \gamma(\bold{x}) = 
\begin{bmatrix}
\sin(\bold{Px}) \\
\cos(\bold{Px}) \\
\end{bmatrix}
$$

这种重新参数化使我们能够推导出 IPE 的闭式形式。使用变量线性变换的协方差是变量协方差的线性变换这一事实 $(\text{Cov}\left[\bold{Ax}, \bold{By}\right] = \bold{A}\text{Cov}\left[\bold{x}, \bold{y}\right]\bold{B}^T)$，我们可以识别圆锥台高斯在被提升到 PE 基 $\bold{P}$ 后的均值和协方差：

$$
\bold{\mu}_{\gamma} = \bold{P\mu}, \ \bold{\Sigma}_{\gamma} = \bold{P\Sigma P}^T
$$

产生 IPE 特征的最后一步是计算在这个提升的多元高斯上的期望，由位置的正弦和余弦调制。这些期望有简单的闭式表达式：

$$
E_{x \sim \mathcal{N}(\mu, \sigma^2)} \left[\sin(\mu) \text{exp}\left(-\left(\frac{1}{2}\right)\sigma^2\right)\right], \\
E_{x \sim \mathcal{N}(\mu, \sigma^2)} \left[\cos(\mu) \text{exp}\left(-\left(\frac{1}{2}\right)\sigma^2\right)\right],
$$

我们看到这个期望的正弦或余弦只是均值的正弦或余弦被方差的高斯函数衰减。有了这个，我们可以计算最终的 IPE 特征作为均值和协方差矩阵对角线的期望正弦和余弦：

$$
\gamma(\bold{\mu, \Sigma}) = E_{x \sim \mathcal{N}(\mu_{\gamma}, \Sigma_{\gamma})}[\gamma(\bold{x})] \\
= \begin{bmatrix}
\sin(\bold{\mu}_{\gamma}) \circ \text{exp}\left(-\left(\frac{1}{2}\right) \text{diag}( \ \Sigma_{\gamma})\right) \\
\cos(\bold{\mu}_{\gamma}) \circ \text{exp}\left(-\left(\frac{1}{2}\right) \text{diag}( \ \Sigma_{\gamma})\right)
\end{bmatrix}
$$

### mip-NeRF 架构

mip-NeRF 的架构很大程度上遵循 NeRF 的架构：

<center>
<figure>
    <img src="https://user-images.githubusercontent.com/42706447/173478353-33a84b93-7d6e-4945-9dd2-4f0d42e812c4.png" alt="mip-NeRF Model Architecture">
    <figcaption><b>图 7：mip-NeRF 模型架构</b></figcaption>
</figure>
</center>

* 最终的体积密度输出向量在 softplus 激活后产生

## 技术细节

* 原始 NeRF 代码实际上使用两个全连接深度神经网络，一个被称为"粗粒度"网络，另一个被称为"细粒度"网络。
  * 粗粒度网络在射线采样的第一次迭代中使用
  * 细粒度网络用于所有其他样本
* mip-NeRF 使用单个网络，更准确地遵循本总结中描述的过程

## 参考文献

许多公式、推导和解释都直接取自下面引用的论文/项目页面。

<table>

<tr valign="top">
<td align="right">
[<a name="r1">1</a>]
</td>
<td>
 <a href="https://arxiv.org/pdf/2003.08934.pdf">NeRF: Representing Scenes as
Neural Radiance Fields for View Synthesis</a>,
<a href="https://www.matthewtancik.com/nerf">NeRF 项目页面</a>
</td>

</tr>

<tr valign="top">
<td align="right">
[<a name="r2">2</a>]
</td>
<td>
 <a href="https://arxiv.org/pdf/2103.13415.pdf">Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields</a>,
<a href="https://jonbarron.info/mipnerf/">mip-NeRF 项目页面</a>
</td>
</tr>

</table>
