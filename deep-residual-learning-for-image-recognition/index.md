# 【论文笔记】Deep Residual Learning for Image Recognition


深度神经网络更难训练。我们提出了一个残差学习框架，以简化比以前使用的网络深度大得多的网络的训练。我们明确地将层重新表述为参考层输入的学习残差函数，而不是学习未参考的函数。我们提供了全面的经验证据，表明这些残差网络更容易优化，并且可以从相当大的深度中获得精度。在ImageNet数据集上，我们评估了深度高达152层的残差网络——比VGG网络深度8倍[40]，但仍然具有较低的复杂性。这些残差网络的集合在ImageNet测试集上的误差达到3.57%。该结果在ILSVRC 2015分类任务中获得第一名。我们还介绍了100层和1000层的CIFAR-10分析。

表征的深度对于许多视觉识别任务至关重要。仅由于我们的极深表示，我们在COCO对象检测数据集上获得了28%的相对改进。深度残差网络是我们参加ILSVRC & COCO 2015竞赛的基础1，我们还在ImageNet检测、ImageNet定位、COCO检测和COCO分割的任务中获得了第一名。

He K, Zhang X, Ren S, et al. Deep residual learning for image recognition[C]//Proceedings of the IEEE conference on computer vision and pattern recognition. 2016: 770-778.

## 1 深度网络的训练问题

多个模型证明了更深的模型有取得更好表现的潜力。然而，随着网络的加深，训练却出现了退化现象。即网络加深了，精度却下降了，而且不是由过拟合，也不是梯度消失/爆炸引起的。

理论上，更深的网络不应该比更浅的网络差。出现这种现象是因为深层次的网络优化越来越困难。

![image-20230510004048272](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/dl/image-20230510004048272.png)

![image-20230510004242494](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/dl/image-20230510004242494.png)

## 2 深度残差网络

- 核心假设：通过一个堆叠的非线性层去拟合（残差）更容易

- 该假设的动机：更深的网络不应该表现比更浅的网络差，因为在极端情况下，更浅的网络继续叠加恒等映射使其变成更深的网络，也不会降低效果。因此顺着恒等映射的思路出发，如果恒等映射是最优的，那么通过残差块，可以轻易地将其跳过的模型块的参数逼近0，从而逼近恒等映射

$$
\bold{y}=\mathcal{F}(\bold{x},\{W_i\})+\bold{x}
$$

- 没有引入额外的参数和计算复杂度

## 3 ResNet

模型实验：

- VGG

- 以VGG为基础网络，减少了每层的参数，增加了层的数量；
- 上述的基础上增加短路连接

观察到退化现象，增加短路连接效果更好 => ResNet
