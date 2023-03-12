# 计算机体系结构复习笔记

# 计算机体系结构复习笔记

## 1 计算机系统结构基础

冯·诺依曼结构：运算器、控制器、存储器、输入、输出=>核心思想：存储程序

### 性能、成本、功耗

- 性能：

$$
MIPS=\frac{Instructions}{Seconds}\times10^{-6}
$$

$$
CPUTime=\frac{Seconds}{Program}=\frac{Instructions}{Program}\times\frac{Cycles}{Instruction}\times\frac{Seconds}{Cycle}
$$

$$
ExTime_{new}=Extime_{old}\times(1-Fraction_{enhanced}+\frac{Fraction_{enhanced}}{Speedup_{enhanced}})
$$

$$
Speedup_{overall}=\frac{Extime_{old}}{ExTime_{new}}
$$

- 成本：略

- 功耗：
  $$
  P_{total}=P_{switch}+P_{short}+P_{leakage}
  $$
  又
  $$
  P_{总}=P_{静}+P_{动}=UI
  $$
  时钟不翻转时，动态功耗为零，芯片相当于电阻，可求静态功耗：
  $$
  R=\frac{U}{I}\\
  P_{静}=P_{总}-0=UI=\frac{U^2}{R}
  $$
  动态功耗与时钟频率成正比。可以先求出总功耗和静态功耗，动=总-静，再利用动态功耗与时钟频率的正比关系求其他数值条件下的功耗。

## 2 二进制与逻辑电路

### 数的表示

- 定点数

  原码与补码互换，都是符号位外按位取反+1

  原码有+0、-0，补码的最小值1000...000没有对应的原码

  原码和补码的范围：

    |            原码            |           补码           |
    | :------------------------: | :----------------------: |
    | $-2^{n-1}+1\sim 2^{n-1}-1$ | $-2^{n-1}\sim 2^{n-1}-1$ |

- 浮点数

  浮点数的组成：

  | 总位数 | 符号位 | 阶码 | 尾数 |
  | :----: | :----: | :--: | :--: |
  |   32   |   1    |  8   |  23  |
  |   64   |   1    |  11  |  52  |

  阶码以移码的方式表示。字长为n位，则偏移值为$2^{n-1}$。移码与补码只差一个符号位。

  尾数规格化表示为1.*的形式，因小数点前面的位数肯定为1，故不存储该位，从而提升一位精度。

  浮点数阶码（32位为例）的特殊情况：

  1. $0<e<255$，表示规格化数，此时阶码的移码表示范围为$-126\sim 127$
  2. $e=0$，若尾数全为零，则表示正负零；否则为非规格化数
  3. $e=255$，若尾数为0，符号位为0或1则表示正\负无穷大；否则为非数

### 逻辑与电路

CMOS管：

![2.1](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/comArch/2.1.PNG)



VDD是电源，PMOS管是反逻辑，开关取反；MMOS管是正逻辑，逻辑取反。$V_{in}$决定的是电路里的开关关断情况。如上为非门$\neg A$，当输入为高电平，PMOS管取反关断，NMOS管接通，此时$V_{out}$接地，为低电平；当输入为低电平，PMOS管取反接通，NMOS管关断，此时$V_{out}$接电源，为高电平。

PMOS管在上面，NMOS管在下面。PMOS管连通电源和输出，NMOS管连通地线和输出。

![2.2](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/comArch/2.2.jpg)

观察与非情况。$\neg A\land B=\neg A \vee \neg B$。上方PMOS管的开关取反，因此按$\neg A \vee \neg B$的逻辑构建，即并联；下方NMOS管逻辑取反，按$A\land B$的逻辑构建，即串联。

或非同理，PMOS管串联，NMOS管并联。可概括为P与并联，N与串联，P或串联，N或并联。可简单手推。

![2.3](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/comArch/2.3.PNG)

以复杂电路为例，PMOS管逻辑：$\neg (A\land B) \vee (C\land D)=(\neg A \vee \neg B)\land (\neg C\vee \neg D)$，NMOS管逻辑：$\neg\neg (A\land B) \vee (C\land D)=(A\land B) \vee (C\land D)$

FO4延迟：
$$
FO4延迟=本征延迟+负载延迟=翻转延迟+(延迟/电容)\times(输入电容+连线电容)\times4
$$

## 3 指令系统结构

### MIPS一般指令选集

- Inst R1, R2, R3：R2和R3 do sth.，结果放到R1
- Inst R1, #num(R2)：存取到地址R2+num
- 对于条件判断指令，最后一个是offset（标号）
- ADD, ADDI, SUB, SUBI, MUL, DIV, LW, SW, BEQ（3操作数）, BNE（3操作数）, BLEZ（2操作数）, BGTZ（2操作数）

- SLT	R1, R2, R3	;R1=bool(R2>=R3)
- R0永远为0

### 指令结构题选

- 指令与周期

  假设在指令系统设计中考虑两种条件转移指令的设计方法：

  1. CPU A：先通过一条比较指令设置条件码A，再用一条分支指令检测条件码
  2. CPU B：比较指令包含在分支指令中

  两种CPU的条件转移指令都需要2个时钟周期，其他指令需要1个时钟周期。CPU A中全部指令的25%是条件转移指令，对应地比较指令战所有指令的25%。

  设CPU A共有x个指令，可求得：

  $$
  Cycle_A=2\times0.25x+0.25x+0.5x=1.25x\\
  Cycle_B=2\times0.25x+0.5x=x
  $$

  则当 CPU A 频率为 1.2 倍时，性能是 CPU B 的 1.2/1.25=0.96 倍；当 CPU A 频率为 1.1 倍时，性能是 CPU B 的 1.1/1.25=0.88 倍。因此 CPU A 两种情况下都差。

- MIPS的原子交换

  LL (Load Linked) 和 SC (Store Conditional)是特殊的Load/Store指令。LL 指令的功能是从内存中读取一个字，以实现接下来的 RMW（Read-Modify-Write） 操作；SC 指令的功能是向内存中写入一个字，以完成前面的 RMW 操作。LL/SC 指令的独特之处在于，它们不是一个简单的内存读取/写入的函数，当使用 LL 指令从内存中读取一个字之后，比如 LL d, off(b)，处理器会记住 LL 指令的这次操作（会在 CPU 的寄存器中设置一个不可见的 bit 位），同时 LL 指令读取的地址 off(b) 也会保存在处理器的寄存器中。接下来的 SC 指令，比如 SC t, off(b)，会检查上次 LL 指令执行后的 RMW 操作是否是原子操作（即不存在其它对这个地址的操作），如果是原子操作，则 t 的值将会被更新至内存中，同时 t 的值也会变为1，表示操作成功；反之，如果 RMW 的操作不是原子操作（即存在其它对这个地址的访问冲突），则 t 的值不会被更新至内存中，且 t 的值也会变为0，表示操作失败。

  ```asm
  L1: LL		R1,	(R3)
  	ADDIU	R2,	R1,	1
  	SC		R2,	(R3)
  	BEQ		R2,	0,	L1
  	NOP
  ```

  从(R3)中读取到R1，R2=R1+1，把R2存放到(R3)，如果发现(R3)被修改过则R2被置零，用BEQ判断。

## 4 静态流水线

五级流水线：

| IF(Instruction Fetch) | ID(Instruction Decode) | EX(Instruction Execute) | MEM(Memory Access) | WB(Write-Back) |
| :-------------------: | :--------------------: | :---------------------: | :----------------: | :------------: |
|         取指          |          译码          |          执行           |        访存        |      写回      |

数据相关：

- 写后读（RAW）（真相关）
- 写后写（WAW）
- 读后写（WAR）
- 转移指令在ID执行，减少等待拍数。如转移指令需要的数据未计算得出，则它会在ID堵塞

前递/旁路：前面的运算输出直接作为后面指令的输入，是特殊结构

指令重排：使相关隔得足够远，如存取和计算分开。

分支预测：在流水线时空图上表示为无视转移指令继续执行，如果预测错误则下一拍重来

### ALU Verilog

书中简单CPU指令含义：

| 名称 |        含义        |
| :--: | :----------------: |
| ADDU |      无符号加      |
| SUBU |      无符号减      |
| SLT  |    左小右大为真    |
| SLTU | 无符号左小右大为真 |
| AND  |         与         |
|  OR  |         或         |
| XOR  |        异或        |
| NOR  |        或非        |
| SLLV |      逻辑左移      |
| SRLV |      逻辑右移      |
| SRAV |      算数右移      |

```verilog
module alu_module
(
    input [ 3:0] aluop, // 4位宽的alu操作符
    input [31:0] in1, // 32位宽的输入数1
    input [31:0] in2, // 32位宽的输入数2
    output [31:0] out // 32位宽的输出数
);
wire slt_res;
// slt为比较的意思，在汇编中，若前者小于后者则为1，否则为0
// 用或连接，以下任一条件成立即为1
// 这段代码用于比较有符号数的大小，讨论四种情况：前负后正，前负后负，前正后正，前正后负
assign slt_res = (in1[31] && !in2[31]) & 1’b1 | // in1[31]为1且in2[31]为0为真时
    (in1[31] && in2[31] ) & !(in1<in2) | // in1[31]为1且in2[31]为1且in1>=in2时
    (!in1[31] && !in2[31]) & (in1<in2) | // in1[31]为0且in2[31]为0且in1<in2时
    (!in1[31] && in2[31]) & 1’b0;  // in1[31]为0且in2[31]为1为假时
// 操作码和书上有些对不上，理解意思即可。
// slt逻辑需要手写，注意区分有符号和无符号的做法。
assign out = 
{32{aluop==4’b0000}} & (in1+in2) | // ADDU 
{32{aluop==4’b0001}} & (in1-in2) | // SUBU 
{32{aluop==4’b0010}} & {31’b0, slt_res} | // SLT  
{32{aluop==4’b0011}} & {31’b0, in1<in2} | // < 
{32{aluop==4’b0100}} & (in1&in2) | // & 
{32{aluop==4’b0101}} & (in1|in2) | // | 
{32{aluop==4’b0110}} & (in1^in2) | // ^ 
{32{aluop==4’b0111}} & ~(in1|in2) | // ~^ 
{32{aluop==4’b1000}} & (in1<<in2) | // << (逻辑移位) 
{32{aluop==4’b1010}} & (in1>>in2) | // >> 
{32{aluop==4’b1011}} & (in1>>>in2); // verilog2001 中算术右移的新表达方式 
endmodule
```



## 5 动态流水线

### 静态调度

循环展开：

- 一个循环展开多做几次，通过修改每次的偏移量实现
- 寄存器重命名消除假相关
- 指令重排序

软流水：

- [软流水的方法--国科大体系结构 期末必考题](https://blog.csdn.net/csdn_muxin/article/details/114491152)

### 动态调度

Tomasulo：

- 通过硬件寄存器重命名消除WAR和WAW相关，并乱序执行

![5.1](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/comArch/5.1.PNG)

- Tomasulo过程：
  - 发射：把操作队列的指令根据操作类型送到保留站（如果保留站 有空），发射过程中读寄存器的值和结果状态域 
  - 执行：如果所需的操作数都准备好，则执行，否则侦听结果总线 并接收结果总线的值。 
  - 写回：把结果送到结果总线，释放保留站
- 指令在发射的时候会更新寄存器状态表，如果后序指令和前序指令的目的寄存器重合了，就用后序指令的写信息标志寄存器，**表示只会把后序指令的计算结果写进寄存器**，这样可以解决WAW

重排序缓存ROB（ReOrder Buffer）

- 为实现精确例外，把后面指令对机器状态的修改延迟到前面指令已经执行完

![5.2](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/comArch/5.2.PNG)

- 寄存器和保留站记录的标号改成ROB，每次发射一条指令都FIFO在ROB占坑
- 会依次写回寄存器，但是寄存器的标号仍然是WAW中的最后一次写入的标号

## 多发射数据通路

### 保留站的组织（指令缓存的结构）

独立保留站、分组保留站、全局保留站

### 寄存器与保留站的关系（读取寄存器内容的时间）

保留站前读寄存器（Tomasulo算法）：

![6.1](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/comArch/6.1.PNG)

- 寄存器的输出作为保留站的输入 
- 操作数没有准备好就读寄存器，保留站侦听结果总线获取没写回的值
- 有序读寄存器
- 保留站中值的来源：寄存器、重命名寄存器、进入保留站时侦听、进入保留站后侦听
- 保留站中有值域，因此较复杂
- 寄存器读端口数为发射宽度

保留站后读寄存器：

![6.2](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/comArch/6.2.PNG)



- 保留站的输出作为寄存器的输入
- 保留站确信所有值都已经准备好后再读寄存器，有可能有Forward的情况
- 乱序读寄存器
- 保留站中没有值域，比较简单
- 寄存器读端口数为相应功能部件数

### 寄存器重命名方法

重命名的方法不拘一格。

硬件重命名的分类：

- 重命名寄存器和结构寄存器分开

  - 如重命名到保留站、ROB、专门的重命名寄存器、发射队列

  ![6.3](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/comArch/6.3.PNG)

- 重命名寄存器和结构寄存器不分开

  - 需要映射表，可以有逻辑寄存器那么多项，也可以有物理寄存器那么多项

  ![6.4](https://cdn.jsdelivr.net/gh/Catigeart/imgHost/img/comArch/6.4.PNG)

硬件重命名方法概括

- 一种是独立的重命名寄存器，把运算结果写回到独立的重命名寄存器，提交的时候再从重命名寄存器写到结构寄存器，指令被取消时确认结构寄存器的内容为最新内容并清空重命名寄存器
- 一种使用物理寄存器堆，并通过一个映射表建立逻辑寄存器和物理寄存器的映射关系，给每条指令的目标寄存器分配物理寄存器，指令提交时确认该物理寄存器为结构寄存器并释放老的物理寄存器，指令取消的时候只修改重命名表以取消后面已经建立的重命名关系

### 乱序数据通路总结

三个因素两两搭配，共有$3\times 2\times2=12$个组合：

- 单项、分组、全局保留站？
- 保留站前/后读寄存器？
- 独立重命名寄存器/物理寄存器堆？

## 7 转移预测

### 软件方法解决控制相关

循环展开、软流水

### 硬件转移预测技术

转移模式历史表PHT（Pattern History Table）

- 每项都是两位饱和计算器，T+1，N-1，高位为1预测T，高位为0预测N

转移历史寄存器BHR（Branch History Register）

- 考虑一种情况，分支TNTNTNTN，这样两位饱和计算器准确率可能为0
- BHR是一个记录转移历史的移位寄存器，根据BHR的值，选择不同的PHT项
- 如，BHR为NT，选择预测N的PHT；BHR为TN，选择预测T的PHT
  - 从而捕捉到了TNTNTN的变化规律，PHT不会反复横跳，提升预测准确率

PHT+BHR

- 类似Cache的全相联组相联直接相联，BHR也有3种PC索引的情况：完整PC索引（1对1）、低位PC索引，全局共用
- PHT索引也有3种方式：只用BHR（g）、用PC和BHR（p）、用部分PC和BHR（s）

|                 | Global PHT | Per-address PHTs | per-Set PHTs |
| :-------------: | :--------: | :--------------: | :----------: |
|   Global BHR    |    GAg     |       GAp        |     GAs      |
| Per-address BHR |    PAg     |       PAp        |     PAs      |
|   per-Set BHR   |    SAg     |       SAp        |     SAs      |

## 8 功能部件

### 先行进位加法器

```verilog
module add16(a, b, cin, out, cout);
    input [15:0] a;
    input [15:0] b;
    input cin; // 低位进位
    output [15:0] out; // 计算结果
    output cout; // 高位进位 
    wire [15:0] p = a|b; // 进位传递因子，两个或者为1可能进位，即表示可以传递低位进位
    wire [15:0] g = a&b; // 进位生成因子，两个为1必定进位
    wire [3:0] P, G;
    wire [15:0] c; // 0, 4, 8, 12是低位进位，其他是待求解的高位进位
    assign c[0] = cin;
    C4 C0_3(.p(p[3:0]),.g(g[3:0]),.cin(c[0]),.P(P[0]),.G(G[0]),.cout(c[3:1])); // 注意调用的写法
    C4 C4_7(.p(p[7:4]),.g(g[7:4]),.cin(c[4]),.P(P[1]),.G(G[1]),.cout(c[7:5]));
    C4 C8_11(.p(p[11:8]),.g(g[11:8]),.cin(c[8]),.P(P[2]),.G(G[2]),.cout(c[11:9]));
    C4 C12_15(.p(p[15:12]),.g(g[15:12]),.cin(c[12]),.P(P[3]),.G(G[3]),.cout(c[15:13]));
    // 前4个分块都算好以后，再合起来做一个C4
    C4 C_INTER(.p(P),.G(G),.cin(c[0]),.P(),.G(),.cout({c[12],c[8],c[4]})); // 大括号括起来做一个整体
    assign cout = (a[15]&b[15]) | (a[15]&c[15]) | (b[15]&c[15]); // 三个中有俩是真的
    assign out = (~a&~b&c)|(~a&b&~c)|(a&~b&~c)|(a&b&c); // 三个中至少有一个是真的，该位就可以为1
endmodule

/* 4位并行进位块 */
module C4(p,g,cin,P,G,cout)
    input [3:0] p, g; // 进位传递因子和进位生成因子
    input cin; // 低位进位
    output P,G; // 传递给下一个块的进位生成因子和进位传递因子
    output [2:0] cout;    
    assign cout[0]=g[0]|(p[0]&cin); // c_1
    assign cout[1]=g[1]|(p[1]&g[0])|(p[1]&p[0]&cin); // c_2
    assign cout[2]=g[2]|(p[2]&g[1])|(p[2]&p[1]&g[0])|(p[2]&p[1]&p[0]&cin); // c_3
    assign G=g[3]|(p[3]&g[2])|(p[3]&p[2]&g[1])|(p[3]&p[2]&p[1]&g[0]); // c_4，有进位生成
    assign P=&p; // 如果全部为1，则表示该块可以传递低位进位
endmodule
```

### 定点补码乘法器

**Booth 2位乘**
$$
[X]_补\times [Y]_补=[X]_补\times\\{[Y]_补的最高位取反，其他位不变的数\\}
$$

$$
(-y_{31}\times 2^{31}+y_{30}\times 2^{30}+...+y_1\times 2^1+y_0\times 2^0) \\\\
= (y_{29}+y_{30}-2\times y_{31})\times 2^{30}+(y_{27}+y_{28}-2\times y_{29})\times 2^{28}+... \\\\
+(y_1+y_2-2\times y_3)\times 2^2 + (y_{-1}+y_0-2\times y_1)\times 2^0
$$

其中$y_{-1}$为0。因此可以每次移2位，通过括号内的计算结果求该次移位求和是以下操作之一：$-2[X]_补,-[X]_补, 0,+[X]_补,+2[X]_补$。

**华莱士树**

堆叠加法器。

## 9 高速缓存

### 页着色

**Cache相联**

Cache相联包括全相联、直接相联、组相联。全相联指任意页可以映射到任意内存块，直接相联指特定的页只能映射到特定内存块，组相联指特定的页可以映射到某一组内存块中的任意一个。直接相联其实也是“一路组相联”。

组的编号是由地址的高位决定的。

**虚-实与Index-Tag**

地址分为虚地址和实地址。虚地址是逻辑地址，实地址是物理地址。页上记录的是虚地址，因此映射到Cache的时候需要转换为内存的实地址。Index是页到Cache的映射，Tag是Cache到内存的映射。

考虑各种因素，采用VIPT（虚索引，实标记）的方式进行转换和映射。页映射到Cache时兵分两路，一边直接用虚索引与Cache匹配，另一边传送到TLB，由TLB转换为实标记再匹配。

页和对应Cache块的低位一致，因此虚索引匹配就是低位匹配。低位不包括cache line的块内偏移位。$VI_{len}=log_2(C_{cache}\div C_{cache\;line}\div n_{set})$，其中C指容量。

页地址在被转换为物理Tag后，会映射到对应的Cache line上。虚实地址的低位是相同的。如果页地址低位的长度比VI长，那么尽管这些页会被映射到同一个Cache line中，但他们其实对应的是不同的物理地址。就产生了一个虚地址对应多个实地址的别名问题。为了解决别名问题，我们采用页着色的方法，强制指定这些页映射到不同的Cache line中。页着色标号从cache地址的高位开始标号。

**页着色例子**

例如，现有4KB页大小，L1 cache大小为32KB，四路组相联，cache line大小为32B。对于该页对应的内存，虚实地址的[11:0]12位数都是相同的；cache地址为[14:0]。cache line对应[4:0]5位。此时VI长度为$log_2(32KB\div 32B \div 4)=8$，即[12:5]8位。这时，可能有两个[11:0]相同，但第12位不同的页映射到同一个cache line中，因此需要1位页着色。页着色位为第14位。

### 不命中的损耗

$$
MissPenalty_{L1}=HitTime_{L2} + MissPenalty_{L2} \times MissRate_{L2}
$$

## 10 存储管理

### TLB例外

MIPS中的三种TLB例外：

1. TLB重填例外：TLB中没有相应项
2. TLB失效例外：相应的物理页不在内存中（包括非法存，非法取）
3. TLB修改例外：非法写只读页

TLB例外例题：在一台Linux（页大小为4KB） 系统的MIPS计算机执行如下程序，请问发生了多少次例外？

```c
void cycle(double *a) {
    int i;
    double b[65536];
    for(i = 0; i < 3; i++) {
        memcpy(a, b, sizeof(b));
    }
}
```

答：假设操作系统页大小为4KB，同时系统的物理内存足够大，即页表分配的物理内存在程序执行过程中不会被交换到swap区上。并且在后续的分析中忽略代码和局部变量i所在的页的影响。 a、b两数组大小均为65536×8=512KB。再假设a、b两数组的起始地址恰为一页的起始地址。 那么a、b两数组各自均需要512KB/4KB=128个页表项。 所以a、b的访问造成的TLB invalid 例外次数为128+128=256次； 假设TLB表项为128项，每项映射连续的偶数奇数两个页，采用LRU算法。那么第一次循环 中，a、b两数组每访问相邻偶数奇数两页的首地址时，均会触发一次TLB refill例外。后续各次循环TLB中均命中。那么共触发的TLB refill例外次数为128/2 + 128/2 = 128次。

### TLB Verilog

已知一台计算机的虚地址为 48 位，物理地址为 40 位，页大小为 16KB，TLB 为 64 项全相 联，TLB 的每项包括一个虚页号 vpn，一个物理页号 pfn，以及一个有效位 valid，请根据如下模块接口写出一个 TLB 的地址查找部分的 Verilog 代码。 module tlb_cam(vpn_in, pfn_out, hit,…); 其中 vpn_in 为输入的虚页号，pfn_out 为输出的物理页号，hit 为表示是否找到的输出信号， “…”表示与该 TLB 输入输出有关的其他信号。重复的代码可以用“…”来简化。

```verilog
module tlb_cam(vpn_in, pfn_out, hit, valid_out);
    input [33:0] vpn_in; // 输入的虚页号
    output [25:0] pfn_out; // 输出的物理页号
    output hit; // 是否找到
    output valid_out;
    
    reg[60:0] cam_content [63:0];//[60:60] valid [59:34] pfn [33:0] vpn; 前面是位宽，后面是数组容量
    wire [63:0] entry_hit;
	// 逐个比较
    assign entry_hit[0] = (vpn_in==cam_content[0][33:0]); 
	assign entry_hit[1] = (vpn_in==cam_content[1][33:0]);
	…
	assign entry_hit[63] = (vpn_in==cam_content[63][33:0]);
	
    assign hit = |entry_hit;
	// entry_hit相当于mask，命中则&通过
    assign pfn_out = {26{entry_hit[ 0]}} & cam_content[0][59:34] |
	{26{entry_hit[ 1]}} & cam_content[1][59:34] |
	…
	{26{entry_hit[63]}} & cam_content[63][59:34];
	// 有效位valid_out=命中&&该位有效位
    assign valid_out = entry_hit[ 0] && cam_content[ 0][60] ||
 	entry_hit[ 1] && cam_content[ 1][60] ||
 	…
 	entry_hit[63] && cam_content[63][60];
endmodule
```



## 11 多处理器系统 

**基于目录的cache一致性协议**：为每一存储行维持一个目录项，该目录项记录所有当前持有该行备份的处理器号以及此行是否已被改写等信息。当一个处理器欲往某存储行写数且可能引起数据不一致时，它就根据目录的内容只向持有该行的备份的那些处理器发出写使无效/写更新信号，从而避免广播。

问：在基于目录的cache一致性系统中，目录记载了P1处理器已经有数据块A的备份。

1. 哪些情况下目录会又收到一个P1对A块访问的请求？
2. 如何处理上述情况？

答：

1. 可能 P1 已经把 A 替换出去了，随后 P1 又希望访问 A，因此向目录发出了 A 的 miss 请求。如果网络不能保证 P1 发出的替换 A 消息和 miss 访问 A 消息达到目录的顺序，后发出的A 的 miss 请求越过先发出的 A 的替换请求，先到达目录，就会产生上述现象。
2. 两种可行的处理方式：
   1. 网络保证点到点消息的顺序性；
   2. 目录发现不一致时，暂缓miss 请求的处理，等待替换消息到达后，目录状态正确后，再返回 miss 请求的响应。
