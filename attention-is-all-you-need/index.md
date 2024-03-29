# 【论文笔记】Attention Is All You Need


主要的序列转导模型是基于复杂的循环或卷积神经网络，包括一个编码器和一个解码器。表现最好的模型还通过注意机制连接编码器和解码器。我们提出了一个新的简单的网络架构，变压器，完全基于注意力机制，完全摒弃递归和卷积。在两个机器翻译任务上的实验表明，这些模型在质量上更优越，同时更具并行性，并且需要更少的训练时间。我们的模型在WMT 2014英语-德语翻译任务上实现了28.4 BLEU，比现有的最佳结果(包括集合)提高了2个BLEU以上。在WMT 2014英法翻译任务中，我们的模型在8个gpu上训练3.5天后，建立了一个新的单模型最先进的BLEU分数41.8，这是文献中最佳模型训练成本的一小部分。我们通过将Transformer成功地应用于具有大量和有限训练数据的英语选区解析，证明了它可以很好地推广到其他任务。

Vaswani A, Shazeer N, Parmar N, et al. Attention is all you need[J]. Advances in neural information processing systems, 2017, 30.

## 1 Intro

- RNN、LSTM、门RNN已经是state of the art的序列模型
- encoder-decoder架构
- 循环模型因为其顺序性质，阻碍了并行化
- 注意力机制允许不考虑序列中的距离建模依赖关系，大部分与RNN结合使用
- Transformer：完全依赖注意力机制

## 2 Background

- 已有的避免序列依赖计算的方法采用卷积神经网络，然而计算任意位置关联信息的算法复杂度大（线性、常数）。**Transformer采用常数复杂度的计算，用多头注意力机制弥补因此带来的分辨能力下降的问题**
- **自注意力机制**：将单个序列不同位置联系起来的注意机制
- **端到端**记忆网络基于循环注意力机制，而不是序列对齐的循环

## 3 Model Arch

### 3.1 Encoder-Decoder

$$
(x_1,x_2,...,x_n)\rightarrow(z_1,z_2,...,z_n)\rightarrow(y_1,y_2....,y_m)
$$

从一个序列映射到另一个序列，每步都是自回归的，所生成的标志作为下一步的额外输入。

![img](https://pic3.zhimg.com/80/v2-e09a169399782babea626d87282bf14e_720w.webp)

**Encoder**: 

- 编码器由N = 6个相同层的堆栈组成。每一层有两个子层。
- 第一个是一个多头自注意机制，
- 第二个是一个简单的、按位置完全连接的前馈网络。
- 我们在每两个子层周围使用一个残差，然后进行层归一化。也就是说，每个子层的输出是`LayerNorm(x + subblayer (x))`，其中`subblayer (x)`是子层本身实现的函数。为了方便这些残差连接，模型中的所有子层以及嵌入层都会产生维度为`dmodel = 512`的输出。

**Decoder**: 

- 解码器也由N = 6个相同层的堆栈组成。除了每个编码器层中的两个子层外，解码器还插入第三个子层，该子层对编码器堆栈的输出执行多头注意。
- 与编码器类似，我们在每个子层周围使用残差连接，然后进行层归一化。
- 我们还修改了解码器堆栈中的自注意力子层，以防止注意到后续的位置。这种掩码，结合输出嵌入被一个位置抵消的事实，确保对位置$i$的预测只能依赖于小于$i$位置的已知输出。

### 3.2 Attention

![image-20230407181529596](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/hpc/image-20230407181529596.png)

$$
Attention(Q,K,V)=softmax(\frac{QK^T}{\sqrt{d_k}})V
$$


- 加性注意力理论复杂度与点乘注意力相似，但点乘可依靠高度优化的矩阵乘法加速
- $d_k$较大时点乘注意力表现不佳，因此予以放缩

多头注意力：
$$
MultiHead(Q,K,V)=Concat(head_1,...,head_h)W^O \\
where\ head_i=Attention(QW_i^Q,KW_i^K,VW_i^V)
$$

- 允许模型在不同位置共同注意来自不同表示子空间的信息

## 4 为什么使用自注意力：

- 易于并行化
- 与RNN相比，易于学习远程依赖关系
- 与CNN相比，一般大小的卷积层不能连接所有输入和输出的位置，若扩展卷积则计算复杂度巨大
- 自注意力机制提供可解释性

## 5 训练

- 标准 WMT 2014 英语-德语数据集
- 8块 NVIDIA P100 GPU
- Adam 优化器：$\beta _1=0.9,\beta _2=0.98,\epsilon=10 ^{-9}$

$$
lrate=d_{model}^{-0.5}·\min (step\_num ^{-0.5},step\_num·warmup\_steps ^{-1.5})
$$

- 正则化：残差dropout，标签平滑化

## 6 结果：新的 state-of-the-art

## 7 结论：

提出了Transformer，在翻译任务取得了state-of-the-art，下一步工作是对注意力机制进行扩展。

