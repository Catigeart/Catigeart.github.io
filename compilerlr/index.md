# 【编译原理】简明自底向上分析算法总结：LR(0)，SLR，LR(1)，LALR分析算法




[【编译原理】简明自顶向下分析算法总结：递归下降，LL(1)分析算法](https://blog.csdn.net/weixin_43404388/article/details/113118729)

****
语法分析有两个总的思路，一个是自顶向下分析，一个是自底向上分析。自底向上的分析思路是，对一个句子$s$，不断进行归约（“合并”），看能否归约成开始符号$S$的状态。
## 自底向上分析（LR概述）
自底向上分析通常讨论的是LR分析算法，也叫“移进-归约算法”。仍然是循序渐进的讨论，从比较朴素的归约动机开始，逐步讨论如何对其完善。
LR分析指每次从左(L)读入，从右(R)反向构造出最右推导序列。分析是在从左到右读入的过程中进行的。在读入一定子串以后，如果右端可以进行归约，则向上归约。引入点记号“**·**”以指示句子读入的情况。如下例子所示：
```
有文法如下：
E -> E + T
   | T
T -> T * F
   | F
F -> n

给出算式 2 + 3 * 4，归约过程如下：
				2 + 3 * 4
2				· + 3 * 4
F				· + 3 * 4
T				· + 3 * 4
E				· + 3 * 4
E + 			  · 3 * 4
E + 3			    · * 4
E + F			    · * 4
E + T			    · * 4
E + T *			      · 4
E + T * 4		        ·
E + T * F		        ·
E + T		       		·
E			       		·
S			       		·
```
由此，LR过程实际上生成的是一个**逆序的最右推导**。已处理部分的符号串，实际上是一个栈结构。因此，该算法的框架可以归纳成如下的移进、归约两个步骤。
1. 移进：将一个记号移进到栈顶中；
2. 归约：将栈顶的n个符号归约成一个非终结符。

显然，我们需要一个机制去确定何时应当移进，何时应当归约，否则该算法也会退化成盲目的搜索。
讨论“句柄”的概念。句柄在LR算法中可以理解为**和产生式右部匹配的子串**。显然，找到合适的句柄，我们就能在正确的步骤归约，从而实现对句子$s$的正确验证。
为了正确高效地实现LR算法，我们可以采取类似于LL的**表驱动的算法思路。**通常来说，LR类算法的分析表结构如下：
| 状态                                                         | ACTION              | GOTO              |
| ------------------------------------------------------------ | ------------------- | ----------------- |
|                                                              | ...（非终结符分栏） | ...（终结符分栏） |
| $ACTION$，即“动作”。在第$n$个状态时读入某终结符应当做什么动作。$ACTION$表的元素通常有2种： |                     |                   |
1. $si$，s是“shift”的缩写，指示状态的转换，**i指示转换到的状态编号**；
2. $ri$，r是“reduce”的缩写，指示此时应当对栈顶子串做归约动作，**i指示对应的（右部）表达式的编号**。

此外，$ACTION$表中的“acc”是“accept”的缩写，表示读入到此处的时候可以已可以接受该句子，LR算法结束。
$GOTO$表意思是“跳转”，表示**归约在栈顶得到某非终结符后，应该跳转到什么状态**。
不同的LR算法，对分析表的构造方法并不相同。以下将对LR(0)，SLR，LR(1)，LALR进行讨论。
### LR(0)分析
首先，为求解方便，我们可以将文法$G$转换为增广文法$G'$，即对开始符号$S$增加一条$S'\rightarrow S$，$S'$是新的开始符号，求解从$S'$开始。*具体而言，这样做是为了开始符号只在位于一条式子的左部。*
为讨论方便，补充项目的定义：项目即规约过程中的一个带点的句子，是左边规约部分和右边未读入部分的拼接。
那么，刚刚在讨论分析表时，表的每一横行代表某状态。所以，该如何划分状态呢？我们可以讨论一种“等价”。比如有式子$S\rightarrow E+E,E\rightarrow aBc$，当前的项目状态是$S\rightarrow E+·E$。那么，我“期盼”着将要读入的是$E$，也就等价于我“期盼着”将要读入的是$aBc$。我们将这些等价的项目归为同一个集，称为项目集闭包，对应自动机中的一个状态。
由此，我们可以得到求项目集闭包的通俗描述：若”期盼着“读入某符号，将其归入一个状态；若该符号是非终结符，那么对于它的某右部表达式，也就相当于”期盼“着读入该表达式的第一个符号，那么该符号的相关项目也应当归入该状态。
根据构造闭包的思路，我们有LR(0)分析表构造算法如下：
- 首先，根据$S'\rightarrow S$构造初始闭包C0，该闭包只含一个项目：$S'\rightarrow ·S$
- 若闭包的分析还没有完成，取出某还没有分析的闭包，根据它读入新符号以后的action和goto情况，建立新的状态闭包

LR(0)分析表构造算法伪代码如下：
```cpp
closure(C) // C是某状态对应的闭包
	while (C is still changing)
		for (item i: C) // example: i = A-> beta · B gamma 
			C += {B -> ...}

Goto(C, x) // 求GOTO得到的新状态
	tmp = {}
	for (item i: C) // example: i = A-> beta · B gamma
		tmp += {A-> beta B · gamma}
	return closure(C)

LR0_table()
	C0 = closure(S'->·S$)
	state_set = {C0}
	Q = enqueue(C0)
	while (Q isn't empty)
		C = dequeue(Q)
		for (x: N+T) // N是非终结符集，T是终结符集
			D = Goto(C, x)
			if (x in T)
				ACTION[C][x] = D
			else
				GOTO[C][x] = D
			if (D isn't in state_set)
				state_set.add(D)
				enqueue(D)	
```
LR(0)算法伪代码如下：
```cpp
stack = []
push($) // 终止符
push(1) // 初始状态
while (true)
	token t = nextToken()
	state s = stack[top]
	if (ACTION[s][t] == "si")
		push(t)
		push(i)
	else if (ACTION[s][t] == "rj")
		pop (the right hand of the production "j:X->beta")
		state s = stack[top]
		push(X)
		push(Goto[s][X])
	else
		error...
```
### SLR分析
然而，LR(0)分析仍然可能存在冲突，考虑以下状态：
```
E -> T ·
T -> T · * F
```
此时如果读入*，将无法得知应当采取移进动作还是采取归约动作（移进-归约冲突）。对于这种情况，SLR算法给出了一个依靠$FOLLOW$集解决的方案。对于下一个要读入的符号，它要么被移进，要么被归约成某个非终结符。设某状态下可接受若干符号，只要这些符号要么位于移进集合，要么位于某个FOLLOW集，集合之间两两互不相交，就可以用前看下一个输入的符号来解决。这种方法被称为简单的LR方法，也就是SLR方法。
- SLR和LR的区别只在于对归约的处理：
	- 对于状态i上的项目 X-> ${\alpha}$ ·
		- 仅对 y ${\in}$FOLLOW(X)添加ACTION[i, y]
### LR(1)分析
SLR分析仍然存在问题。SLR只是简单地考察下一个输入符号b是否属于与归约项目$A\rightarrow\alpha$相关联的$FOLLOW(A)$，但$b\in FOLLOW(A)$只是归约$\alpha$的一个必要条件，不是充分条件。LR(1)分析法提出的动机是，在特定位置，某符号$A$的后继符集合是$FOLLOW(A)$的子集。那么如何求出这个在特定位置的后继符集合呢？依靠前面已知状态的推导。观察如下例子：
```
S -> L = R
L -> * R
```
由第一条产生式可以进一步推导出 L -> \* R也应该加入到这个闭包里，但这两个L -> \* R是不一样的。原本的表达式是一个原始的式子，它期望后接的符号是终结符$，而由S推导出来的L式，后接的应该是=。因此推导出来的新闭包可以表示为：
```
S -> L = R, $
L -> * R, =
L -> * R, $
```
根据这样的思路，我们就把LR算法的前看1符号情况进行了细化，称为LR(1)算法。LR(1)的算法大部分和LR(0)相同，仅闭包的构造方法不同：
- 对项目$[X\rightarrow\alpha·Y\beta,a]$
	- 添加$[Y\rightarrow\gamma,b]$到项目集
		- 其中$b$是$X$式中$Y$被期望紧接着的符号（展望符）。 
### LALR分析
LR(1)分析固然分析能力更强，但是也带来了新的问题：状态机中的许多状态，可能仅仅式被期望的后接符号（称为展望符）不一样，而他们的表达式集是完全相同的。LALR分析，即在LR(1)分析得到的状态机图基础上，将表达式完全相同的状态予以合并。这样合并以后，由于忽略了展望符的信息，可能产生归约-归约冲突。
LALR分析形式上与LR(1)相同，大小上与LR(0)/SLR相当，分析能力介于SLR和LR(1)之间。
- 分析能力：SLR(1)<LALR(1)<LR(1)
> 参考资料：
> 《编译原理》课程，陈鄞，哈尔滨工业大学（中国大学MOOC）
> 《编译原理》课程，华保健，中国科学院大学（网易云课堂或b站免费观看）
> 《编译原理》（紫龙书），机械工业出版社
> 《编译原理》，清华大学出版社