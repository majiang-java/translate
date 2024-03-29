# 自行翻译文献内存屏障
## 抽象内存访问模型
- 设备操作
- 保证可靠性

## 什么是内存屏障
- 不同种类的内存屏障
- 内存屏障有哪些没有的假设
- 数据依赖内存屏障（层级数据）
- 控制依赖关系
- SMP屏障配对
- 内存屏障序列的例子
- 读内存屏障 vs 加载内存推测
- 多重原子性

## 明确内核屏障
- 编译屏障
- CPU 内存屏障
- MMIO写屏障

## 隐式内核内存障碍 
- 锁定采集功能
- 中断禁用功能
- 睡眠和唤醒功能
- 杂项功能

## CPU间获取屏障效应
- 获取vs内存访问
- 获取与I / O访问

## 需要哪些记忆障碍
- 处理器间交互
- 原子操作
- 访问设备
- 中断

## 内核I / O屏障效果
## 假设最小执行排序模型

## cpu缓存的影响
- 缓存一致性
- 缓存一致性与DMA
- 缓存一致性与MMIO

## CPU能够做到的事情


### 抽象内存访问模型
考虑以下抽象模型系统：


```
		            :                :
		+-------+   :   +--------+   :   +-------+
		|       |   :   |        |   :   |       |
		|       |   :   |        |   :   |       |
		| CPU 1 |<----->| Memory |<----->| CPU 2 |
		|       |   :   |        |   :   |       |
		|       |   :   |        |   :   |       |
		+-------+   :   +--------+   :   +-------+
		    ^       :       ^        :       ^
		    |       :       |        :       |
		    |       :       |        :       |
		    |       :       v        :       |
		    |       :   +--------+   :       |
		    |       :   |        |   :       |
		    |       :   |        |   :       |
		    +---------->| Device |<----------+
		            :   |        |   :
		            :   |        |   :
		            :   +--------+   :
		            :                :
```
每个cpu执行访问内存的程序， 在抽象cpu中，访问内存的顺序非常随意的，并且一个cpu可能使用任何顺序访问内存，似乎这种事情得到了保持，同样，编译器可以以任何顺序发出指令，
只要它该改动明显不影响最终计划的顺利运行。


因此，在上图中，当CPU通过CPU与系统其余部分（虚线）之间的接口时，系统的其余部分会感知CPU执行的存储器操作的影响。

例如，请考虑以下事件序列：


```
	CPU 1		CPU 2
	===============	===============
	{ A == 1; B == 2 }
	A = 3;		x = B;
	B = 4;		y = A;
```

在执行途中，会产生24中不同的组合

```
	STORE A=3,	STORE B=4,	y=LOAD A->3,	x=LOAD B->4
	STORE A=3,	STORE B=4,	x=LOAD B->4,	y=LOAD A->3
	STORE A=3,	y=LOAD A->3,	STORE B=4,	x=LOAD B->4
	STORE A=3,	y=LOAD A->3,	x=LOAD B->2,	STORE B=4
	STORE A=3,	x=LOAD B->2,	STORE B=4,	y=LOAD A->3
	STORE A=3,	x=LOAD B->2,	y=LOAD A->3,	STORE B=4
	STORE B=4,	STORE A=3,	y=LOAD A->3,	x=LOAD B->4
	STORE B=4, ...
	...
```
并且能够导致4种不同的组合结果:

```
	x == 2, y == 1
	x == 2, y == 3
	x == 4, y == 1
	x == 4, y == 3
```

此外，对于一个CPU对一个store的写可能不被另一个CPU的读感应到。

例如，请考虑以下事件序列：

```
	CPU 1		CPU 2
	===============	===============
	{ A == 1, B == 2, C == 3, P == &A, Q == &C }
	B = 4;		Q = P;
	P = &B		D = *Q;
```
这里存在明显的数据依赖性，因为加载到D中的值取决于
由CPU 2从P检索的地址。在序列的最后，任何一个
以下结果是可能的：


```
	（Q ==＆A）和（D == 1）
	（Q ==＆B）和（D == 2）
	（Q ==＆B）和（D == 4）
```
需要注意的是CPU永远不会将加载C到D中，因为CPUj将在执行*Q之前加载P到Q

### 设备操作
一些设备将其控制接口呈现为存储器位置的集合,但访问控制寄存器的顺序非常重要。例如，设想一个带有一组内部寄存器的以太网卡通过地址端口寄存器（A）和数据访问端口寄存器（D）被访问。为了访问内部寄存器5，以下代码可能被用到。

```
    *A = 5;
	x = *D;
```

但这可能会显示为以下两个序列之一：


```
	STORE *A = 5, x = LOAD *D
	x = LOAD *D, STORE *A = 5
```
第二条直接导致一个错误，因为它在读之后设置了地址。

### 担保
可以预期CPU有一些最小的保证。

- 在任何给定的CPU上，相关的内存访问将按顺序发出。这意味着：
    ```
    Q = READ_ONCE(P); D = READ_ONCE(*Q);
    ```
    CPU将会发出以下内存操作
    
    ```
    Q = LOAD P, D = LOAD *Q
    ```
    并且总是使用这个顺序，但是，在DEC Alpha上，READ_ONCE（）也是发出内存屏障指令，
    这样DEC Alpha CPU将发出以下内存操作：
    
    ```
    Q = LOAD P, MEMORY_BARRIER, D = LOAD *Q, MEMORY_BARRIER
    ```
    无论是否在DEC Alpha，READ_ONCE（）总是阻止编译恶作剧（指令重排）


- 在特定CPU中重叠的加载和存储似乎是在该CPU中排序的，这意味着：
    ```
    a = READ_ONCE(*X); WRITE_ONCE(*X, b);
    ```
    CPU仅仅会发布以下内存操作的序列。
    
    ```
    a = LOAD *X, STORE *X = b
    ```
    并且
    
    ```
    WRITE_ONCE(*X, c); d = READ_ONCE(*X);
    ```
    CPU只会发出：
    
    
    ```
    STORE *X = c, d = LOAD *X
    ```
    （如果它们针对重叠的内存块，则加载和存储重叠）
- 并且有许多事情需要_must_或_must_not_假设：

 （*）它_must_not_假设编译器将执行您想要的操作
     内存引用不受READ_ONCE（）和
     WRITE_ONCE（）。没有它们，编译器就有权使用它们
     做各种各样的“创造性”转换，包括在内
     COMPILER BARRIER部分。

 （*）_must_not_假设将发布独立的装载和存储
     按顺序给出。这意味着：

	X = * A; Y = * B; * D = Z;

     我们可能会得到以下任何一个序列：


```
    X = LOAD * A，Y = LOAD * B，STORE * D = Z.
	X = LOAD * A，STORE * D = Z，Y = LOAD * B.
	Y = LOAD * B，X = LOAD * A，STORE * D = Z.
	Y = LOAD * B，STORE * D = Z，X = LOAD * A.
	STORE * D = Z，X = LOAD * A，Y = LOAD * B.
	STORE * D = Z，Y = LOAD * B，X = LOAD * A.
```

 （*）_must_假设重叠的存储器访问可以合并或
     丢弃。这意味着：


```
X = * A; Y = *（A + 4）;
```


     我们可能会得到以下任何一个序列：


```
X = LOAD * A; Y = LOAD *（A + 4）;
	Y = LOAD *（A + 4）; X = LOAD * A;
	{X，Y} = LOAD {* A，*（A + 4）};
```


     并为：

	* A = X; *（A + 4）= Y;

     我们可以得到任何：

	
```
	STORE *A = X; STORE *(A + 4) = Y;
	STORE *(A + 4) = Y; STORE *A = X;
	STORE {*A, *(A + 4) } = {X, Y};
```

- 还有反担保:
  1. 这些保证不适用于bitfield，因为编译器经常使用
     使用非原子 <读-修改-写> 生成代码来修改它们
     序列。不要尝试使用bitfield来同步并行
     算法。

   2. 即使在bitfield受锁，所有字段保护的情况下也此
     在给定的位域中必须由一个锁保护。如果两个字段
     在给定的位域中，由编译器的不同锁保护
     非原子 <读 - 修改 - 写> 序列可以导致更新一个
     字段来破坏相邻字段的值。

   3. 这些保证仅适用于正确对齐和大小的标量
     变量。“大小合适”目前意味着变量
     与“char”，“short”，“int”和“long”相同的大小。“正确“对齐”意味着自然对齐，因此没有约束“char”，双字节对齐，用于“short”，四字节对
     “int”，以及“long”的四字节或八字节对齐，
     分别在32位和64位系统上。请注意这些
     保证被引入C11标准，所以要小心
     使用较旧的C11前编译器（例如，gcc 4.6）。部分
     包含此保证的标准是第3.14节，其中
     定义“内存位置”如下：

     >	记忆位置
		要么是标量类型的对象，要么是最大序列
		相邻位域的全部具有非零宽度

		注1：两个执行线程可以更新和访问
		分开的存储位置而不会干扰
		彼此。

		注2：位域和相邻的非位域成员
		在不同的内存位置。这同样适用
		两个位字段，如果在嵌套内声明了一个
		结构声明和另一个不是，或两者
		由零长度位字段声明分隔，
		或者它们是否由非位字段成员分隔
		宣言。同时更新两个是不安全的
		如果声明了所有成员，则相同结构中的位字段
		它们之间也是位域，无论是什么
		这些中间位域的大小恰好是。   
### 什么是内存屏障
如上所述，没有依赖关系的内存操作实际会以随机的顺序执行，但对CPU-CPU的交互和I / O来说却是个问题。我们需要某种方式来指导编译器和CPU以约束执行顺序。

内存屏障就是这样一种干预手段。它们会给屏障两侧的内存操作强加一个顺序关系。

这种强制措施是很重要的，因为一个系统中，CPU和其它硬件可以使用各种技巧来提高性能，包括内存操作的重排、延迟和合并；预取；推测执行分支以及各种类型的缓存。内存屏障是用来禁用或抑制这些技巧的，使代码稳健地控制多个CPU和(或)设备的交互
。
### 内存屏障的种类
内存屏障有四种基本类型：

1. write（或store）内存屏障。
    write内存屏障保证：所有该屏障之前的store操作，看起来一定在所有该屏障之后的store操作之前执行。
    write屏障仅保证store指令上的偏序关系，不要求对load指令有什么影响。
    随着时间推移，可以视CPU提交了一系列store操作到内存系统。在该一系列store操作中，write屏障之前的所有store操作将在该屏障后面的store操作之前执行。

    [!]注意，write屏障一般与read屏障或数据依赖障碍成对出现；请参阅“SMP屏障配对”小节。

2. 数据依赖屏障。
数据依赖屏障是read屏障的一种较弱形式。在执行两个load指令，第二个依赖于第一个的执行结果（例如：第一个load执行获取某个地址，第二个load指令取该地址的值）时，可能就需要一个数据依赖屏障，来确保第二个load指令在获取目标地址值的时候，第一个load指令已经更新过该地址。
数据依赖屏障仅保证相互依赖的load指令上的偏序关系，不要求对store指令，无关联的load指令以及重叠的load指令有什么影响。
如write（或store）内存屏障中提到的，可以视系统中的其它CPU提交了一些列store指令到内存系统，然后the CPU being considered就能感知到。由该CPU发出的数据依赖屏障可以确保任何在该屏障之前的load指令，如果该load指令的目标被另一个CPU的存储（store）指令修改，在屏障执行完成之后，所有在该load指令对应的store指令之前的store指令的更新都会被所有在数据依赖屏障之后的load指令感知。
参考”内存屏障顺序实例”小节图中的顺序约束。
[!]注意：第一个load指令确实必须有一个数据依赖，而不是控制依赖。如果第二个load指令的目标地址依赖于第一个load，但是这个依赖是通过一个条件语句，而不是实际加载的地址本身，那么它是一个控制依赖，需要一个完整的read屏障或更强的屏障。查看”控制依赖”小节，了解更多信息。

    [！]注意：数据依赖屏障一般与写障碍成对出现；看到“SMP屏障配对”章节。

3.  read（或load）内存屏障。
read屏障是数据依赖屏障外加一个保证，保证所有该屏障之前的load操作，看起来一定在所有该屏障之后的load操作之前执行。

    read屏障仅保证load指令上的偏序关系，不要求对store指令有什么影响。
    
    read屏障包含了数据依赖屏障的功能，因此可以替代数据依赖屏障。
    
    [!]注意：read屏障通常与write屏障成对出现；请参阅“SMP屏障配对”小节。
    
    通用内存屏障。
    通用屏障确保所有该屏障之前的load和store操作，看起来一定在所有屏障之后的load和store操作之前执行。
    
    通用屏障能保证load和store指令上的偏序关系。
    
    通用屏障包含了read屏障和write屏障，因此可以替代它们两者。
- 
