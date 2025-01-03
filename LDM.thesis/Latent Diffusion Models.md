# Latent Diffusion Models

High-Resolution Image Synthesis with Latent Diffusion Models

## 摘要

动机：在有限的计算资源下进行扩散模型训练，同时保持质量和灵活性

引入跨注意力层，以卷积方式实现对一般条件输入（如文本或边界框）的响应以及高分辨率合成

## 1：引言

> 贡献

1：与纯粹基于 Transformer 的方法相比，在**高维数据上的扩展**更优雅

1.1 ==> 在压缩级别上工作，提供比之前工作更真实、更细致的重建

1.2 ==> 高效地应用于高分辨率的百万像素图像合成

2：与基于像素的扩散方法相比，在多种任务上（无条件图像生成、修复、随机超分辨率）取得了具有竞争力的**性能**，显著降低了计算成本和推理成本

3：与之前需要同时学习编码器/解码器架构和基于分数的先验的工作相比，无需对重建能力和生成能力进行复杂的**权衡**，确保了极高的重建忠实度，对潜在空间的正则化需求极低

4：对于密集条件约束任务（超分辨率、修复、语义合成），可以以**卷积**方式应用，并生成一致的超大图像

5：设计了基于**跨注意力**的通用条件机制，支持多模态训练

6：发布了**预训练**的潜在扩散模型和自编码模型

## 2：相关工作

1：generative models for image synthesis

2：diffusion probabilistic models（DM）

3：two-stage image synthesis

ARM：自回归模型

## 3：方法

autoencoding model（自编码模型） ==> learn a space that is perceptually equivalent to the image space

自编码模型的优点：

- **低维**空间采样
- 利用从UNet继承的inductive bias，使得在处理**具有空间结构的数据**时**有效，无需激进的压缩
- **通用压缩模型**，其潜在空间可以用于训练多种生成模型

### 3.1：Perceptual Image Compression

autoencoder（自编码器）==> 通过 感知损失 + patch-based对抗目标 训练

- 给定RGB空间的图像 x，编码器 e 把 x 编码到潜在表示 z，**z = e(x)**

- 解码器 D 从潜在表示中重建图像 x^~，**x^~ = D(z) = D(e(x))**

  x的维度：![73365919187](Latent%20Diffusion%20Models.assets/1733659191878.png)

  z的维度：![73365920008](Latent%20Diffusion%20Models.assets/1733659200082.png)

- 编码器下采样因子 f = H/h = W/w，讨论不同的下采样因子（2的指数倍）

避免潜在空间具有任意的高方差，采用了2种不同的正则化：

- KL正则化：对学习到的潜在表示施加轻微的 KL 惩罚，使其趋向于标准正态分布（类似VAE）
- VQ正则化：在解码器中使用向量量化层

### 3.2：Latent Diffusion Models

> Diffusion Models

扩散模型：通过逐步对正态分布变量去噪，学习数据分布 p(x)，对应学习固定长度为 T 的马尔可夫链的反向过程

图像合成模型，依赖于变分下界的重新加权变体

目标函数：

![73366015218](Latent%20Diffusion%20Models.assets/1733660152180.png)

> Generative Modeling of Latent Representations

通过训练的感知压缩模型（由 e 和 D 组成），可以访问一个高效的、低维的潜在空间

与高维像素空间相比，这个潜在空间更适合基于似然的生成模型，因为：

- 专注于数据中重要的语义信息
- 在一个更低维、计算上更高效的空间中进行训练

利用模型提供的与图像相关的归纳偏置：包括构建主要基于 2D 卷积层的 U-Net 的能力，并进一步将目标集中在感知上最相关的信息位上，使用重新加权的目标函数

目标函数修改为：

![73366069481](Latent%20Diffusion%20Models.assets/1733660694814.png)

神经网络的主干：**time-conditional UNet**

![73366084776](Latent%20Diffusion%20Models.assets/1733660847767.png)

zt 可以在训练期间通过 e 高效地获取

从 p(z) 的采样，可以通过 D 的一次前向传递，解码到图像空间

### 3.3：Conditioning Mechanisms

底层 U-Net 主干中加入跨注意力机制

为处理来自各种模态的 y，引入了一个特定领域的编码器 Tθ， 把 y 映射到一个中间表示 Tθ(y)，维度为![73366408712](Latent%20Diffusion%20Models.assets/1733664087121.png)

跨注意力层的实现：

![73366403840](Latent%20Diffusion%20Models.assets/1733664038408.png)

对于参数的解释：

![73366413861](Latent%20Diffusion%20Models.assets/1733664138617.png)

> framework
>
> 通过拼接（concatenation）或更通用的跨注意力机制（cross-attention mechanism）对潜在扩散模型 (LDMs) 进行条件化

![73366423965](Latent%20Diffusion%20Models.assets/1733664239657.png)

基于图像条件对，目标函数修改为：

![73366432250](Latent%20Diffusion%20Models.assets/1733664322505.png)

## 4：实验

### 4.1：感知压缩的权衡分析

**实验内容**：比较不同下采样因子 f（如 1, 2, 4, 8, 16, 32）对 LDM 模型性能的影响。下采样因子越大，压缩越强。

**结果与分析**：

- 小的下采样因子（如 f=1,2）导致训练进展缓慢，因为未能充分利用低维潜在空间的优势。
- 过大的下采样因子（如 f=32）会导致信息损失，限制最终生成质量。
- 最优权衡出现在 f=4 到 f=8 之间，既保证了高效的训练和推理，又提供了感知上忠实的生成结果。

**结论**：**中等强度的压缩**（如 f=4 和 f=8）在效率和质量之间提供了最佳平衡。

### 4.2：无条件图像生成

**实验内容**：在多个数据集（CelebA-HQ, FFHQ, LSUN-Churches, LSUN-Bedrooms）上评估 LDM 的无条件生成能力，并通过 FID、Precision 和 Recall 指标与其他方法（如 GAN, DDPM）进行比较。

**结果与分析**：

- LDM 在大多数数据集上的 FID 指标优于现有扩散模型（例如 ADM）和 GAN 方法，尤其在 CelebA-HQ 数据集上达到 SOTA 性能。
- 与现有基于像素空间的扩散方法相比，LDM 显著降低了推理和训练的计算成本。

**结论**：LDM 在无条件图像生成任务中表现出色，能够在更低的计算资源下实现更好的质量。

### 4.3：条件图像生成

**实验内容**：

- 通过引入交叉注意力机制（cross-attention），LDM 被扩展到条件生成任务（例如文本到图像生成）。
- 使用 MS-COCO 数据集评估文本生成性能，并在语义地图条件下进行语义合成。

**结果与分析**：

- 在文本到图像生成上，LDM 超越了 DALL-E 和 CogView 等方法，FID 指标显著降低。
- 在语义合成任务中，LDM 能够在低分辨率训练的基础上生成更高分辨率的图像（如 512×1024）。

**结论**：LDM 的交叉注意力机制极大地增强了条件生成的灵活性，尤其适用于文本到图像等复杂条件。

### 4.4：超分辨率任务

**实验内容**：在 ImageNet 数据集上进行 64×64→256×256 超分辨率任务，与 SR3 模型进行比较。

**结果与分析**：

- LDM 在 FID 指标上优于 SR3，但 IS 指标稍逊。
- 用户研究表明，在感知一致性上，LDM 生成的高分辨率图像更受欢迎。

**结论**：LDM 能有效进行超分辨率生成，且具有更高的生成质量。

### 4.5：图像修复

**实验内容**：在 Places 数据集上进行图像修复，与 LaMa 等方法比较，评估填补遮挡区域的效果。

**结果与分析**：

- LDM 修复质量（FID）优于大多数现有方法，并通过用户研究证明更受人类偏好。
- 高分辨率的修复任务（如 512×512）得益于潜在空间的特性。

**结论**：LDM 提供了一种通用的条件生成方法，在高质量修复任务中表现突出。

> 总结

**性能提升**：LDM 在多个任务上展现出较传统扩散模型显著的性能提升，尤其是在计算效率和感知质量之间实现了良好平衡。

**通用性与灵活性**：LDM 的架构设计（如交叉注意力机制）使其适应多种条件生成任务，例如文本、语义地图到图像生成。

**计算优势**：相较于像素空间的扩散模型，LDM 大幅减少了训练时间和推理计算需求，降低了硬件门槛。
