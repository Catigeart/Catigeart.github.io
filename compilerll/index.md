# 【编译原理】简明自顶向下分析算法总结：递归下降，LL(1)分析算法


# 语法分析概念
从编译器前端的流程上说，语法分析对词法分析得到的记号流进行分析，识别其中的语法错误，并将正确的记号流转化为语法树，交给编译器的后续步骤进行进一步处理。
## 上下文无关语法
上下文无关语法是一个四元组：$G=(T,N,P,S)$，其中
- $T$是终结符集合
- $N$是非终结符集合
- $P$是一组产生式规则：
	- 形式：$X\rightarrow \beta _1,\beta _2,...,\beta _n$
	- 其中$X\in N,\beta _i\in (T\cup N)$
- $S$是唯一的开始符号，$S\in N$
## 推导
给定文法$G$，从$S$开始，用产生式的右部替换左部的非终结符（把非终结符“展开”），直到不出现非终结符为止。推导结果称为**句子**。
- 最左推导：每次总选择最左侧符号替换
- 最右推导：每次总选择最右侧符号替换
- 语法分析的任务：给定$G$和句子$s$，回答是否存在对句子$s$的推导
## 分析树和二义性文法
### 分析树
推导可以表示成树状结构，树中的每个内部结点代表非终结符，每个叶子结点代表终结符，每一步推导代表如何从双亲结点生成直接孩子结点。
- 分析树的含义取决于树的后序遍历。
### 二义性文法
给定文法$G$，如果存在句子$s$对应不止一颗分析树，称$G$是二义性文法。
- 解决方案：文法重写
# 语法分析算法
显然，语法分析从两个思路去做：
- 思路1：根据文法$G$，从唯一的开始符号$S$开始，对非终结符，用产生式的右部替换左部（“展开”），观察是否能产生对应的句子$s$；
- 思路2：根据句子$s$，对其不断归约（“合并”），看是否能归约成开始符号$S$的形态。

思路1称为**自顶向下分析**，对应分析树的自顶向下的构造顺序；思路2称为**自底向上分析**。
## 自顶向下分析
从开始符号$S$出发去推导句子称为自顶向下分析。后文将从最原始的回溯展开动机出发，逐步讨论如何优化算法（然后更加秃头）。
### 朴素的回溯思路
最朴素的自顶向下分析思想是：从开始符号$S$出发，随意地推导出句子$t$，比较$t$和$s$。
以下是华保健老师里的网课伪代码：
```c
tokens[]; // holding all tokens
i = 0; // 指向第i个token
stack = [S] // S是开始符号
while (stack != [])
	if (stack[top] is a terminal t)
		if (t == tokens[i++]) // 如果匹配成功
			pop();
		else
			backtrack();
	else if (stack[top] is a nonterminal T)
		pop();
		push(the next right hand side of T) // 不符合，尝试下一个右部式
```
简单地说，就是不断地去试探展开式如何匹配每个句子，如果匹配不成功就回溯，试探下一种可能，由此引出递归下降分析算法和LL(1)分析算法。
### 递归下降分析算法
递归下降分析算法（需要回溯的方法）的基本思想如下：
- 每个非终结符构造一个分析函数
- 用前看符号指导产生式规则的选择

示例，对如下产生式规则：
```
S -> N V N
N -> s
   | t
   | g
   | w
V -> e
   | d
```
构造每条规则的算法伪代码：
```c
parse_S()
	parse(N)
	parse(V)
	parse(N)

parse_N()
	token = token[i++]
	if (token == s || token == t || token == g || token == w)
		return;
	error("...")

parse_V()
	// TODO
```
一般的递归下降分析算法框架如下：
```c
parse_X()
	token = nextToken()
	switch(token)
	case ...:
	case ...:
	......
```
可以看出，通用的递归下降分析技术仍然可能需要回溯。据此进一步讨论LL(1)分析算法。
### LL(1)分析
- 概念：从左(L)到右推导符号，最左(L)推导，采用一个(1)前看符号
- 基本思想：表驱动的分析方法

仍然是很直观的思路。对于需要回溯的情况，我们可以根据已有的条件和规律对其剪枝，从而避免无谓的搜索。而在语法分析当中，显然token与token之间的相对位置是存在关系的。据此，我们可以利用这些相对位置关系去构造分析表，记录遇到下一个符号时应该跳转到哪个状态，从而避免回溯。
具体地说，我们需要构造两个集合：$FIRST$集和$FOLLOW$集。$FIRST$集的动机是引导表达式展开的跳转方向，$FOLLOW$集的动机是避免空符$\epsilon$带来的错误。当$FIRST$可能为空时，就应当加入$FOLLOW$集，后文算法将会体现这一点。
**注：此处约定英文大写符号默认为非终结符，英文小写符号默认为终结符，希腊字母暗示为一般文法符号，无法确认是终结符还是非终结符。**
- $FIRST(\alpha)$：从任意文法符号串$\alpha$开始推导得出的所有可能的终结符集合。通俗地说，就是$\alpha$可能以什么开头。
- $FOLLOW(A)$：可能在某些句型中紧跟$A$后边的终结符号的集合。

对$FIRST$集可以采用不动点算法计算。计算各文法符号的$FIRST(\alpha)$时，不断应用下列规则刷表，直到所有FIRST()都不再更新：
1. 如果$\alpha=a$是终结符，则$FIRST(a)=\{a\}$；
2. 如果$\alpha=N$不是终结符，则对$N$的每个产生式$\beta_1\beta_2...\beta_n$，对其中的$\beta_i$，若$\beta_1\beta_2...\beta_{i-1}$的$FIRST$集中都有$\epsilon$空符，则$FIRST(N) \cup = FIRST(\beta_i)$。通俗地说，对于产生式的第$i$个符号，如果它前面的符号都可能取空，那么它的$FIRST$元素当然也可能是$N$的$FIRST$元素，因此要将它的集合加入到$N$的集合中去。当然，对于$i=1$的情况，总是会$FIRST(N)\cup FIRST(\beta_1)$。

给出伪代码如下：
```cpp
while (some set is changing)
	for (symbol alpha: symbols)
		if (alpha is terminal)
			FIRST[alpha] = {alpha}
		else if (alpha is nonterminal)
			for (production p: alpha->beta_1,...beta_n)
				for (i = 1; i <= n; i++)
					if (beta_i is terminal a)
						FIRST[alpha] += {a}
						break
					else if (beta_i is nonterminal M)
						FIRST[N] += FIRST[M]
						if (M cannot be null)
							break
```
类似地，可以给出$FOLLOW$集的不动点算法：
对所有非终结符$A$的FOLLOW(A)集合时，不断应用以下规则刷表，不再有集合被更新：
3. 如果存在产生式$A\rightarrow \alpha B\beta$，则$\beta$除空符$\epsilon$以外的所有符号都在$FOLLOW(B)$中；
4. 如果存在产生式$A\rightarrow \alpha B$，或$A\rightarrow \alpha B\beta$且$FIRST(\beta)$包含空符$\epsilon$，则$FOLLOW(B) \cup=FOLLOW(A)$。

通俗地说，就是找到每个“紧跟其后”的终结符号。类比$FIRST$集的求法，把“紧跟其后”的符号的FIRST集加入所求的FOLLOW集中。如果“紧跟其后”的$FIRST$集有机会取空符，那么下一个集合的$FIRST$集合也有机会补上来“紧跟其后”，以此类推。
伪代码如下：
```cpp
while (some set is changing)
	for (nonterminal A : nonterminals)
		for (production p: A->beta_1,...,beta_n)
			tmp = FOLLOW[N]
			for (i = n; i > 0; i--)
				if (beta_i == terminal a)
					tmp = {a}
				else if (beta_i == nonterminal M)
					FOLLOW[M] += tmp
					if (M cannot be null)
						tmp = FIRST[M]
					else
						tmp += FIRST[M]
```
已有$FIRST集$，$FOLLOW$集，可按如下规则构造二维预测分析表$M[][]$，该预测分析表的行头是非终结符号，列头是输入的下一个符号（终结符号）。对文法$G$的每个产生式$A\rightarrow\alpha$，进行如下处理：
1. 对于$FIRST(\alpha)$的每个终结符号$\alpha$，将$A\rightarrow\alpha$加入到$M[A,a]$中
2. 如果$\epsilon\in FIRST(\alpha)$，则对于$FOLLOW(A)$的每个终结符号$b$，也将$A\rightarrow\alpha$加入到$M[A,b]$中。

由上可以很清楚地看出$FOLLOW$集的作用：前一个$FIRST$为空的时候的“替补”。通过同时求解$FIRST$集和$FOLLOW$集，可以保证LL(1)分析总能前看到“正确”的符号。
由上，可以书写LL(1)分析伪代码：
```cpp
tokens[]; // all tokens
i = 0;
stack = [S] // S是开始符号
while (stack != [])
	if (stack[top] is a terminal t)
		if (t == token[i++])
			pop();
		else
			error(...); // 朴素自顶向下的回溯改成直接报错
	else if (stack[top] is a nonterminal T)
		pop();
		push(table[T, tokens[i]]); // 朴素自顶向下的尝试展开任意表达式改成按表展开
```
尽管LL(1)对朴素自顶向下做了优化，但LL(1)在语法上仍然存在发生冲突的可能。下面简单讨论两种解决冲突的方法：消除左递归和提取左公因子。
#### 消除左递归
有例子如下：
```
E -> E + T
   | T
```
此时由于LL算法总是从左到右读入，从左到右展开，那么计算的时候将会无限展开E->E+T而进入死循环。对于左递归的情况，有一般解法可以转换成右递归，如上式可改写如下：
```
E -> T E'
E'-> + T E'
```
#### 提取左公因子
有例子如下：
```
X -> a Y
   | a Z
```
显然，此时同样的a可能会导向不同的表达式，存在冲突，直观的解决方法时将公共的a提取出来，如下所示：
```
X -> a X'
X'-> Y
   | Z
```
****
先整理自顶向下到这里，有空再整理自底向上……
> 参考资料：
> 《编译原理》课程，华保健，中国科学院大学（网易云课堂或b站免费观看）
> 《编译原理》（紫龙书），机械工业出版社
> 《编译原理》，清华大学出版社
