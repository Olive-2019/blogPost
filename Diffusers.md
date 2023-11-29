---
title: Diffusers
date: 2023-11-29 15:29:05
tags:
- AIGC
- diffusion
---
简化diffusion模型使用与开发的工具
<!-- more -->
# diffusion模型介绍
扩散模型（Diffusion Models）是一类用于生成数据的生成模型，它们特别擅长在生成过程中处理复杂的数据结构，比如图像、音频、文本等。扩散模型的核心概念是在随机噪声下，逐步提炼出所需的数据。

其中，“稳定扩散”（Stable Diffusion）是一种特定类型的扩散模型，它建立在随机过程和概率理论的基础上，通过逐步升级噪声信号以生成所需的数据。这种模型在生成图像、音频和其他结构化数据方面表现出色，并且能够在生成过程中实现良好的控制和可解释性。

在机器学习和深度学习领域，稳定扩散模型被广泛用于生成高质量的图像、音频和其他数据类型。它们通常采用大规模的预训练模型，并结合不同的训练技巧和优化策略来生成具有高度真实感和多样性的数据。这些模型的应用范围涵盖了艺术创作、数据增强、医学图像生成等多个领域。
## 与LLM的关系
扩散模型和像GPT这样的大型语言模型之间存在一些区别和相似之处。

1. 生成方法：
Diffusion Models： 这些模型利用**随机过程和概率理论**生成数据。它们在生成过程中逐步演化噪声信号以生成所需的数据，常用于图像、音频等多种数据类型的生成。
GPT： GPT等大型语言模型侧重于**自然语言处理**，采用了Transformer架构，通过学习大量的文本数据，预测文本序列中下一个词或字符。这些模型在自然语言生成、对话系统等领域表现出色。
2. 应用领域：
Diffusion Models： 主要用于生成**图像、音频以及其他结构化数据**，例如生成逼真的图像或改善图像质量。
GPT： 主要用于**自然语言处理任务**，例如文本生成、摘要生成、翻译和对话系统等。
3. 技术原理：
Diffusion Models： 这些模型基于**随机过程**，利用**噪声**信号来生成数据。它们通常使用多步骤的推断算法来逐步改进生成的结果。
GPT： 这些模型主要基于**Transformer**等结构，利用**自注意力机制**来捕捉文本序列的长期依赖性，并通过预测下一个词或字符来生成文本。

# Diffusers工具介绍
用于生成图像、音频甚至分子的3D结构的先进**pretrained Diffusion Models**的首选工具。方便用户快速找到可以用的模型，同时也方便用户训练自己的模型

这个库主要包括三个主要组件：
1. diffusion pipelines：先进的扩散管道，仅需几行代码即可进行推理。
2. noise schedulers：可替换的噪声调度器，用于平衡生成速度和质量之间的权衡。
3. models：预训练模型，可用作构建块，并与调度器结合，用于创建自己的端到端扩散系统。

# Diffusion Pipeline
大概可以理解为下载模型
## 普通pipeline推理代码
```python
from diffusers import DiffusionPipeline
# 下载stable-diffusion-v1-5模型
pipeline = DiffusionPipeline.from_pretrained("runwayml/stable-diffusion-v1-5", use_safetensors=True)

# 放到gpu上
pipeline.to("cuda")

# 推理
image = pipeline("An image of a squirrel in Picasso style").images[0]
image.save("image_of_squirrel_painting.png")
```
## Local pipeline
不同的命令
```bash
# Git Large File Storage (LFS) 的命令，用于在 Git 仓库中启用 Git LFS。Git LFS 是 Git 的一个扩展，用于管理大型文件，例如图像、音频文件或其他体积较大的文件，使它们能够更有效地与 Git 仓库一起使用。
# git lfs install 命令会在当前的 Git 仓库中启用 Git LFS。启用后，Git LFS 将跟踪和管理大型文件，而不是直接将它们存储在 Git 仓库中。相反，Git LFS 会将大型文件指针存储在 Git 中，而文件的实际内容则存储在 Git LFS 服务器上，从而使得 Git 仓库的大小得以控制，同时允许有效地管理和协作大型文件。
git lfs install
# 下载权重
git clone https://huggingface.co/runwayml/stable-diffusion-v1-5
```
代码区别
```python
pipeline = DiffusionPipeline.from_pretrained("./stable-diffusion-v1-5", use_safetensors=True)
```

# Models
大多数模型会对噪声样本进行处理，在每个时间步长预测噪声残差（其他模型可能直接学习预测前一个样本、速度或速度预测），即对于不太嘈杂的图像和输入图像之间的差异。你可以混合和匹配不同的模型，来训练一个新的diffusion model。
```python
from diffusers import UNet2DModel

# 加载模型
repo_id = "google/ddpm-cat-256"
model = UNet2DModel.from_pretrained(repo_id, use_safetensors=True)

# 获取模型参数
# 模型参数是冻结状态的，这种设计的目的是为了确保在模型架构初始定义时所使用的参数保持不变，不会被修改，而其他参数仍然可以在推理过程中灵活调整，以满足不同的需求。这种配置方式有助于保持模型结构的稳定性和一致性，同时允许在推理时灵活调整其他参数。
model.config

import torch

torch.manual_seed(0)
# 生成高斯噪声图像
noisy_sample = torch.randn(1, model.config.in_channels, model.config.sample_size, model.config.sample_size)
noisy_sample.shape
torch.Size([1, 3, 256, 256])

# 在进行推理时，将噪声图像和一个时间步传递给模型。时间步表示输入图像的噪声程度，在开始时噪声较多，在结束时噪声较少。这有助于模型确定自身在扩散过程中的位置，是靠近开始还是结束。使用 sample 方法获取模型的输出：
with torch.no_grad():
    noisy_residual = model(sample=noisy_sample, timestep=2).sample

```

# Schedulers
扩散模型中的调度器（Schedulers）在生成过程中起着重要作用。它们控制着去噪（denoising）步骤的进行，影响着生成结果的质量和速度。下面介绍几种常见的扩散模型调度器：

1. Poisson Disc Num Steps Scheduler： 这种调度器按照泊松分布方式确定去噪步骤的数量。随着去噪的进行，图像质量逐渐提升。
2. Euler Discrete Scheduler： 使用欧拉离散方案进行去噪。可能会提供更快的去噪速度，但质量与速度之间存在权衡。
3. Exponential Scheduler： 使用指数函数调整去噪过程中的步长。可能在保证质量的同时提供较快的去噪速度。

这些调度器根据模型和任务的不同可能有所不同。选择合适的调度器对于生成过程的质量和速度非常重要。一些扩散模型库或工具可能允许用户在生成过程中灵活选择不同的调度器，以实现更好的控制和结果。

```python
from diffusers import DDPMScheduler
# 获取schedule
scheduler = DDPMScheduler.from_config(repo_id)

# 一步输出，如果这里是循环，就可以看到每一步都发生了什么
less_noisy_sample = scheduler.step(model_output=noisy_residual, timestep=2, sample=noisy_sample).prev_sample

```