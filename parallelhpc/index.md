# 高性能并行计算笔记

## 并行计算基础

粒度：在并行执行过程中，二次通讯之间每个处理机计算工作量大小的一个粗略描述。分为粗粒度、细粒度。

复杂性：在不考虑通讯开销的前提下，每个处理机上的计算量最大者，即为并行计算复杂性。

并行度：算法可以并行的程度。

加速比：
$$
S_p(q)=\frac{T_s}{T_p(q)}
$$
效率：
$$
E_p(q)=\frac{S_p(q)}{q}
$$
Amdahl定律：假设串行计算所需要的时间$T_s=1$，$\alpha$是执行该计算所必须的串行部分所占的百分比，则有
$$
S_p(q)=\frac{1}{\alpha + \frac{1-\alpha}{q}}
$$
Gustafson定律：假设并行计算所需要的时间$T_p=1$，$\alpha$是执行该并行计算所需的串行部分所占的百分比，则有
$$
S_p(q)=\frac{\alpha+(1-\alpha)\times q}{1}
$$

## 矩阵乘并行计算

**矩阵卷帘存储方式**：对于一般$m\times n$分块矩阵和一般的处理机阵列$p\times q$，小块$A_{ij}$存放在处理机$P_{kl}(k=i\mod p,l=j\mod q)$中。

**串行矩阵乘法**：略，注意循环顺序。

**行列分块算法**：

<img src="https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/comArch/image-20230317163002700.png"/>

$A_i,B_i,C_{i,j},j=0,1,...,p-1$存放在$P_i$中，这种存放方式使数据在处理机中不重复。由于使用$p$个处理机，每次每个处理机计算出一个$C_{i,j}$，计算$C$需要$p$次来完成。$C_{i,j}$的计算是按对角线进行的。

其中，C按行分块，处理机间传输B的分块：

<img src="https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/hpc/image-20230318191313300.png"/>

**行行分块算法**：

![image-20230318192422680](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/hpc/image-20230318192422680.png)

$$
C_{i,\*}=A_{i,\*}B=\sum_{j=0}^{p-1}A_{i,j}B_{j,\*}
$$

其中，C按行分块，处理机间传送B的分块：

![image-20230318194425469](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/hpc/image-20230318194425469.png)

**列行分块算法**：

![image-20230318194916638](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/hpc/image-20230318194916638.png)
$$
C_{\*,j}=\sum_{i=0}^{p-1}A_{\*,i}B_{i,j}
$$
C按列分块，处理机间通讯所传输的是计算的部分积：

![image-20230318195825319](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/hpc/image-20230318195825319.png)

**列列分块算法**：

![image-20230318195017716](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/hpc/image-20230318195017716.png)
$$
C_{\*,j}=\sum_{i=0}^{p-1}A_{\*,i}B_{i,j}
$$
C按列分块，计算过程传送矩阵A的分块：

![image-20230318195934807](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/hpc/image-20230318195934807.png)

**Cannon算法**：

[MPI学习笔记（四）：矩阵相乘的Cannon卡农算法 - 惊小呆520 - 博客园 (cnblogs.com)](https://www.cnblogs.com/babyclass/p/16633535.html)

主要的思路是A子矩阵横向循环传送，B子矩阵纵向循环传送。

![image-20230318213337524](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/hpc/image-20230318213337524.png)

子矩阵预重排：

![image-20230318213457299](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/hpc/image-20230318213457299.png)

## LU分解

矩阵以一维卷帘方式存储，矩阵的第$i$列存放在$P_{i\mod p}$中。

![image-20230319141840687](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/hpc/image-20230319141840687.png)
