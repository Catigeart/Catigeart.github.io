# 【论文笔记】Modeling Relational Data with Graph Convolutional Networks


Schlichtkrull M, Kipf T N, Bloem P, et al. Modeling relational data with graph convolutional networks[C]//The Semantic Web: 15th International Conference, ESWC 2018, Heraklion, Crete, Greece, June 3–7, 2018, Proceedings 15. Springer International Publishing, 2018: 593-607.

知识图支持各种各样的应用，包括问题回答和信息检索。尽管在它们的创建和维护上投入了巨大的努力，但即使是最大的(例如，Yago、DBPedia或Wikidata)也仍然不完整。我们引入了关系图卷积网络(R-GCNs)，并将其应用于两个标准的知识库补全任务:链接预测(缺失事实的恢复，即主-谓词-对象三元组)和实体分类(缺失实体属性的恢复)。RGCNs与最近一类在图上操作的神经网络相关，并且专门用于处理现实知识库的高度多关系数据特征。我们证明了R-GCNs作为实体分类的独立模型的有效性。我们进一步证明，用于链路预测的分解模型(如DistMult)可以通过使用编码器模型来丰富它们，从而在关系图中的多个推理步骤中积累证据，从而得到显著改进，证明在FB15k-237上比仅解码器基线提高了29.8%。

## 1 Intro

知识库组织和存储知识，但存在知识完整性的问题，从而产生了预测知识库中的缺失信息的统计关系学习（SRL）任务。假设知识库存储三元组：主语、谓词、宾语，那么可以建立有向二元关系。考虑两个SRL任务：链路预测（恢复丢失的三元组）和实体分类（为实体分配类型或分类属性）。

本文贡献：

- 第一个证明图卷积网络（GCN）框架可应用于关系数据建模，特别是链路预测和实体分类

- 引入参数共享和强制稀疏约束的技术，并利用他们将R-GCNs应用于具有大量关系的多图
- 展示因式分解模型的性能，以DistMult为例

## 2 神经关系建模

本文符号：$G=(\mathcal{V,E,R}), v_i\in V, (v_i, r,v_j)\in \mathcal{E},r\in\mathcal{R}$

### 2.1 关系图卷积网络

我们的模型是GCNs模型的扩展，可以理解为可微消息传递框架的特例：
$$
h_i^{l+1}=\sigma(\sum_{m\in\mathcal{M}_i}g_m(h_i^{(l)},h_j^{(l)}))
$$
其中$h_i^{(l)}\in\mathbb{R}^{d^{(l)}}$是节点$v_i$在第i层隐藏层的状态，$d^{(l)}$是该层的维度。$g_m$是传递的消息，$\sigma$是激活函数，$\mathcal{M}_i$是该结点的输入消息集。

受其启发，我们在带标签有向图之上的前向传播模型如下：
$$
h_i^{(l+1)}=\sigma(\sum_{r\in\mathcal{R}}\sum_{j\in\mathcal{N}_i^r}\frac{1}{c_{i,r}}W_r^{(l)}h_j^{(l)}+W_0^{(l)}h_i^{(l)})
$$
$\mathcal{N}_i^r$表示关系R中节点i的邻居索引集，$c_{i,r}$是特定问题的归一化系数，可以学习得到或提前指定。

直观上，上式通过归一化和对相邻节点变换后的特征向量进行累加。与常规GCNs不同，我们引入了特定于关系的转换，即依赖于边缘的类型和方向。

为了确保l + 1层节点的表示也可以被l层对应的表示所告知，我们为数据中的每个节点添加了一个特殊关系类型的单个自连接。请注意，可以选择更灵活的函数，如多层神经网络(以牺牲计算效率为代价)，而不是简单的线性消息转换。我们把这个留给以后的工作。

通过堆叠上述块构建R-GCNs。

上式可以用稀疏矩阵有效实现，可以通过层的堆叠学习多个关系步骤的依赖。

![image-20230511144601123](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/dl/image-20230511144601123.png)

### 2.2 正则化

将R-GCNs应用于高度多关系数据的一个中心问题是，随着图中关系数量的增加，参数数量也会迅速增加。在实践中，这很容易导致对稀有关系的过度拟合和非常大的模型。

文章引入了两种不同的正则化方法：基分解(basis-decomposition)和块对角分解(block-diagonal-decomposition)。

基分解将$W_r^{(l)}$分解为B个基变换的线性组合：

$$
W_r{(l)}=\sum_{b=1}^B a_{rb}^{(l)}V_b^{(l)}
$$
块对角分解将$W_r^{(l)}$视为块对角矩阵：
$$
W_r^{(l)}=\oplus_{b=1}^B Q_{br}^{(l)}
$$
基函数分解可以看作是不同关系类型之间有效的权值共享形式，而块分解可以看作是对每个关系类型的权值矩阵的稀疏性约束。块分解结构编码了一种直觉，即潜在特征可以分组为变量集，这些变量集在组内比组间耦合得更紧密。这两种分解都减少了学习高度多关系数据(如现实知识库)所需的参数数量。同时，我们期望基参数化可以缓解稀有关系的过拟合，因为在稀有关系和更频繁的关系之间共享参数更新。

## 3 实体分类

对于节点(实体)的(半)监督分类，我们简单地将形式为(2)的R-GCN层堆叠起来，在最后一层的输出上使用softmax(·)激活(每个节点)。我们最小化所有标记节点上的以下交叉熵损失(忽略未标记节点):
$$
\mathcal{L}=-\sum_{i\in\mathcal{Y}}\sum_{k=1}^K t_{ik}\ln h_{ik} ^{(L)}
$$
$\mathcal{Y}$是有标签的节点索引集，$h_{ik}^{(L)}$是第i个带标签节点的第k个输出，$t_{ik}$是真值标签。使用批梯度下降训练模型。

## 4 链路预测

链接预测处理的是对新事实的预测，如三元组(主语，关系，宾语)。形式上，知识库由一个有向标记图G = (V, E, R)表示。我们得到的不是边的完整集合E，而是不完全子集E。任务是给可能的边(s, r, o)分配分数f(s, r, o)，以确定这些边属于E的可能性有多大。

因此，本文引入图自编码器模型，由实体编码器（R-GCNs）和打分函数（解码器）组成。

本文使用DistMult作为打分函数：
$$
f(s,r,o)=e_s^TR_re_o
$$
其中$e_s$和$e_o$是使用R-GCN学习到的实体嵌入，$R_r$是可学习的关系r对应的对角矩阵。

使用负采样训练模型。对每个正样本采样$\omega$个负样本通过优化交叉熵损失函数使模型给正样本的打分高于负样本：
$$
\mathcal{L}=-\frac{1}{(1+\omega)|\hat{\mathcal{E}}|}\sum_{(s,r,o,y)\in T}y\log l(f(s,r,o))+(1-y)\log (1-l(f(s,r,o)))
$$
其中T是正样本和负样本集合，l是sigmoid函数，y=1表示正样本，y=0表示负样本。

## 5 实验

### 5.1 实体分类

![image-20230511202756145](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/dl/image-20230511202756145.png)

![image-20230511202826316](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/dl/image-20230511202826316.png)

### 5.2 链路预测

![image-20230511202936469](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/dl/image-20230511202936469.png)

![image-20230511203037500](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/dl/image-20230511203037500.png)

## 结论

我们介绍了关系图卷积网络(R-GCNs)，并在两个标准统计关系建模问题的背景下证明了它们的有效性：链路预测和实体分类。对于实体分类问题，我们已经证明了R-GCN模型可以作为一个竞争性的，端到端可训练的基于图的编码器。对于链路预测，以DistMult分解为解码组件的R-GCN模型优于直接优化的分解模型，在标准链路预测基准上取得了具有竞争力的结果。使用RGCN编码器丰富因子分解模型对于具有挑战性的FB15k-237数据集特别有价值，比仅解码器基线提高29.8%。

我们的工作可以通过几种方式得到扩展。例如，图自编码器模型可以与其他分解模型结合使用，例如ComplEx (Trouillon et al . 2016)，它可以更适合于非对称关系的建模。在R-GCNs中集成实体特征也很简单，这将有利于链路预测和实体分类问题。为了解决我们方法的可扩展性，探索子采样技术是值得的，例如Hamilton, Ying和Leskovec(2017)。最后，它有望用数据依赖的注意机制取代当前对相邻节点和关系类型的求和形式。除了建模知识库之外，R-GCNs还可以推广到关系分解模型已经被证明有效的其他应用中。
