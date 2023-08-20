# 【论文笔记】Non-local Neural Networks


Wang X, Girshick R, Gupta A, et al. Non-local neural networks[C]//Proceedings of the IEEE conference on computer vision and pattern recognition. 2018: 7794-7803.

卷积运算和循环运算都是每次处理一个局部邻域的构建块。在本文中，我们将非局部操作作为捕获远程依赖关系的通用构建块。受计算机视觉中经典的非局部均值方法[4]的启发，我们的非局部运算将一个位置的响应计算为所有位置特征的加权和。这个构建块可以插入到许多计算机视觉体系结构中。在视频分类任务上，即使没有任何花哨的东西，我们的非局部模型也可以在Kinetics和Charades数据集上竞争或优于当前的竞争获胜者。在静态图像识别中，我们的非局部模型改进了COCO任务套件上的目标检测/分割和姿态估计。代码将提供。

注意力机制的一般化总结：

- 在本文中，我们提出了非局部操作作为一种高效、简单和通用的组件，用于捕获深度神经网络的远程依赖关系。我们提出的非局部运算是对计算机视觉中经典的非局部均值运算的推广。直观地说，非局部操作计算的是某个位置的响应作为输入特征映射中所有位置的特征的加权和。位置集可以是空间、时间或时空，这意味着我们的操作适用于图像、序列和视频问题。

![image-20230510202847582](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/dl/image-20230510202847582.png)

## NLNN

$$
\bold{y}_i=\frac{1}{\mathcal{C}(\bold{x})}\sum_{\forall j}f(\bold{x}_i,\bold{x}_j)g(\bold{x}_j)
$$

其中，$i$是输出位置的索引（空间，时间或者时空），$j$是所有可能位置的枚举，$\bold{x}$是输入信号（图像，序列，影像，通常是它们的特征），$\bold{y}$是和$\bold{x}$相同尺寸的输出。$f$在$i$和所有$j$之间计算一个标量（表示他们之间的关系，比如联系的密切程度），$g$计算输入在$j$上的表示。输出由因子$\mathcal{C}(\bold{x})$标准化。

非局部操作是一个灵活的构建块，可以很容易地与卷积/循环层一起使用。它可以添加到深度神经网络的早期部分，不像最后经常使用的fc层。这允许我们构建一个结合了非本地和本地信息的更丰富的层次结构。

### NLNN函数的实例化

**高斯函数**：
$$
f(\bold{x}_i,\bold{x}_j)=e^{\bold{x}_i^T\bold{x}_j}
$$
**嵌入高斯函数**：
$$
f(\bold{x}_i,\bold{x}_j)=e^{\theta (\bold{x}_i)^T\phi(\bold{x}_j)}
$$
**自注意力**也是NLNN的一种特殊实例化：
$$
\bold{y}=softmax(\bold{x}^TW_\theta^T W_\phi \bold{x})g(\bold{x})
$$
**点乘**：
$$
f(\bold{x}_i,\bold{x}_j)=\theta(\bold{x}_i)^T\phi(\bold{x}_j)
$$


**拼接**：
$$
f(\bold{x}_i,\bold{x}_j)=ReLU(\bold{w}_f^T[\theta(\bold{x}_i),\phi(\bold{x}_j)])
$$
其中(...,...)表示两者的Concat。

### 非局部块形式

$$
\bold{z}_i=W_z\bold{y}_i+\bold{x}_i
$$

其中$+\bold{x}_i$表示残差形式的连接。通过这种块构建，NLNN块可以很容易地加入到现有的网络。

### 实验

![image-20230510210126230](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/dl/image-20230510210126230.png)

![image-20230510210200567](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/dl/image-20230510210200567.png)

实验比较了几种NLNN函数实现的效果，基于现有模型进行消融实验，证明得到了提升。
