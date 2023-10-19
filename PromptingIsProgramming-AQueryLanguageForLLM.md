---
title: Prompting Is Programming:A Query Language For LLM
date: 2023-10-17 15:42:37
tags:
    - LLM
    - promptEngineering
---
PLDI2023的文章，提出LMP(Language Model Programming, 语言模型编程)思想，设计了LMQL(Language Model Query Language, 语言模型查询语言)，它利用来自 LMP prompt的约束和控制流，以生成有效的推理过程，最大限度地减少对底层语言模型的昂贵调用的数量。思想有借鉴意义，但方法已经out of date了。

<!-- more -->
# Background

LLM很牛逼，但是使用过程中面临这些问题

1. 程序是task-specific model-specific的（无法泛化）
2. 需要多次交互（不够自动化）
3. 输出太长（没有约束）
4. 效率和性能（其实就是输出太长了，传递的东西太多）

因此提出了一种新理论，LMP(Language Model Programming, 语言模型编程)，并给出其实践LMQL(Language Model Query Language, 语言模型查询语言)。

# LMP(Language Model Programming, 语言模型编程)
LMP思想是：使用少量的**脚本**和**输出限制**，实现前后端分离的LLM提示词工程。例如，允许用户定义复杂的交互、控制流和输出限制。
* **脚本**：自动化
* **输出限制**：省钱+提高效率

个人理解：这个思想就是将LLM的纯无代码模式过度到低代码模式，这个想法在当时是很好的，但是想象力不够丰富。估计作者没想到一年不到langchain发展能这么快。

附：langchain项目起源于2022年10月，最早一版在今年的1月16日，目前发版频率是1版/天-2版/天，这个工作在去年12月放在arxiv上。
# LMQL(Language Model Query Language, 语言模型查询语言)

## 语法定义

![1697544698103](image/PromptingIsProgramming-AQueryLanguageForLLM/1697544698103.png)

## 例子

![1697544664866](image/PromptingIsProgramming-AQueryLanguageForLLM/1697544664866.png)
# 评价
今年软件工程的A里跟大模型靠边的也不算特别多，跟工具链靠边的就更少了，想来应该是两个原因，一是学术界发表文章存在周期，而大模型发展过快，学术界的论文难以跟上（arxiv上大佬的文章也许可以，但是找大佬是个问题），二是学术界偏好理论性的工作，而工具链则更偏向工程。所以，我们未来的学习导向应该还是要调研工程性项目，学术界目前能扒到对我们的工作有价值的东西不多。
虽然这篇是今年的文章，但是工作是在去年年底完成的。站在当下看这篇文章，方法还是比较naive的，思想和langchain相近，却只考虑了langchain中的一点点内容。但是把目光放回到作者在做这个工作的那个时代，大模型刚刚扬帆启航，在那个年代能够提出这么一个工具链相关的东西，也算是目光比较高远了。接下来我们重点看看LangChain的工作。