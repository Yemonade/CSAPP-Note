# Chapter03 程序的机器级表示

## 3.0 综述

高级语言与汇编代码的最大区别/优点是：高级语言编写的程序可以在不同的机器上编译和执行，而汇编代码则是与特定机器相关的

:hammer: ​为什么学习机器代码呢

- 理解编译器的优化能力，分析代码中隐含的低效率
- 了解程序的运行时行为，比如线程如何共享数据
- 了解安全漏洞是如何出现的

:happy: 本章将要介绍

- 汇编语言（基于x86-64，**默认是 64 位机器，地址是 32 位的，一个内存地址存一个字节**）
- C 语言与汇编语言的相互转化（包括数组、结构、联合、控制语句、栈、内存访问越界）
- 学会阅读编译器产生的汇编代码

:fire: 重点

- 寄存器的职责与使用
- C 语言与汇编语言的转化
- **栈帧**的分配

## 3.1 历史观点



## 3.2 程序编码

```bash
linux> gcc -Og -o p p1.c p2.c
```

名词解释：

- `gcc`: 指 gcc 编译器，是 Linux 默认的编译器，也可以简单使用 cc 来启动它
- `-Og`: 优化等级，告诉编译器使用生成符合原始 C 代码整体结构的机器代码的优化等级。等级越高越难读懂。实际中，从得到的程序性能考虑，教高级别的优化（`-O1`或`-O2`）被认为是较好的选择

流程：

- C 预处理器扩展源代码，插入所有用`#include`命令指定的文件，扩展`#define`
- 编译器产生两个源文件的汇编代码，命名为 p1.s 与 p2.s
- 汇编器将汇编代码转化成二进制目标代码文件 p1.o 和 p2.o（目标代码是机器代码的一种形式，包含所有资料的二进制表示，**但还没有填入全局变量的地址**）
- 链接器将两个目标代码文件与实现库函数（例如`printf`）的代码合并，并产生最终的可执行文件 p（由命令行中的`-o p`指定）

### 3.2.1 机器级代码

机器级编程的两种抽象

1. 指令集体系结构或指令集架构（Instruction Set Architecture, ISA）来定义机器级程序的格式与行为。简单来说定义了指令及指令的功能
2. 程序的内存地址使用虚拟地址，提供的内存模型看上去是一个非常大的字节数组

常见的处理器状态

- 程序计数器（PC）：给出**将要**执行的下一条指令在内存中的地址
- 整数寄存器：16个，分别存储 64 位的值。有的寄存器用来记录重要的程序状态，其他寄存器用来保存临时信息，例如过程的参数，局部变量以及函数的返回值
- 条件码寄存器：算术或逻辑指令的状态信息，实现控制或数据流
- 向量寄存器：存放一个或多个整数或浮点数值

汇编语言不区分任何有无符号数，不区分各种数据类型。

程序内存包含：程序的可执行机器代码，OS 需要的一些信息，用来管理过程调用和返回的运行时栈，用户分配的内存块（例如利用`malloc`库函数分配的）

### 3.2.2 代码示例

假设我们写了一个`mstore.c`文件

```c
long mult2(long, long);

void multstore(long x, long y, lonmg *dest) {
    long t = mult2(x, y);
    *dest = t;
}
```

使用下面命令行编译

```bash
linux> gcc -Og -S mstore.c
```

将会产生汇编文件`mstore.s`

```
multstore:
	pushq	%rbx
	movq	%rdx, %rbx
	call 	mult2
	movq	%rax, (%rbx)
	popq	%rbx
	ret
```

反汇编器

```bash
linux> objdump -d mstore.o
```

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201132236984.png" alt="image-20220113223638863" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201132245690.png" alt="image-20220113224529586" style="zoom:80%;" />



生成实际可执行代码需要对一组目标文件代码运行链接器，而这一组目标文件代码文件必须包含一个`main`函数。假设 main.c 如下

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201132303740.png" alt="image-20220113230337695" style="zoom:80%;" />

用如下命令编译产生可执行文件 prog

```bash
linux> gcc -Og -o prog main.c mstore.c
```

通过下面命令反汇编 prog 文件

```bash
linux> objdump -d prog
```

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201132305541.png" alt="image-20220113230527464" style="zoom:80%;" />



这段代码与 mstore.c 反汇编产生的代码不同之处在于

- 左侧列出的地址不同
- 链接器填上了`callq`指令调用函数`mult2`需要使用的地址
- 多了最后两行代码，只是为了使函数代码变成 16 字节，有利于存储，其余无影响

## 3.3 数据格式

![image-20220113230855524](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201132308590.png)

## 3.4 访问信息

### 3.4.1 寄存器

寄存器是 64 bit 的

![image-20220113231017249](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201132310490.png)

| 寄存器名称 | 作用                                                         |
| ---------- | ------------------------------------------------------------ |
| eax        | 累加(Accumulator)寄存器，常用于函数返回值                    |
| ebx        | 基址(Base)寄存器，以它为基址访问内存                         |
| ecx        | 计数器(Counter)寄存器，常用作字符串和循环操作中的计数器      |
| edx        | 数据(Data)寄存器，常用于乘除法和I/O指针                      |
| esi        | 源变址寄存器                                                 |
| dsi        | 目的变址寄存器                                               |
| esp        | 栈指针寄存器(extended stack pointer)，其内存放着一个指针，该指针永远指向系统栈最上面一个栈帧的栈顶 |
| ebp        | 基址指针寄存器(extended base pointer)，其内存放着一个指针，该指针永远指向系统栈最上面一个栈帧的底部 |
| eip        | 指令寄存器(extended instruction pointer)， 其内存放着一个指针，该指针永远指向下一条待执行的指令地址注： 可以说如果控制了EIP寄存器的内容，就控制了进程——我们让EIP指向哪里，CPU就会去执行哪里的指令 |

- ebp 在未受改变之前始终指向栈帧的开始，也就是栈底，所以**ebp 的用途是在堆栈中寻址用的**。
- esp是会随着数据的入栈和出栈移动的，也就是说，**esp 始终指向栈顶。**

### 3.4.2 寻址

![image-20220113231105594](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201132311691.png)



:raised_hands: 第一张表给出状态信息，第二章表给出寻址方式

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201132313644.png" alt="image-20220113231309540" style="zoom:80%;" />

### 3.4.3 数据传送指令

![image-20220113231702961](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201132317016.png)

:heavy_exclamation_mark: PS

- 两个操作数不能同时指向内存

- `movl`指令以寄存器为目的时，会将寄存器的高位 4 字节置为 0。这一技术利用的属性是：**生成 4 字节值并以寄存器作为目的的指令会把高 4 字节置为 0**

- 常规的`movq`指令只能以表示为 32 位补码数字的立即数作为源操作数，将符号扩充到 64 位的值，然后放在目的位置；而`movabsq`能够以任意 64 位立即数作为源操作数，并且只能以寄存器作为目的

- 五种可能的组合

  ![image-20220113232111424](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201132321482.png)

下面两张表给出将小容器的数据移动到大容器时使用的命令

![image-20220113231717465](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201132317568.png)

:heavy_exclamation_mark: PS

- 没有一条指令可以把 4 字节扩展到 8 字节（逻辑上应该命名为 movzlq），但可以通过`movl`指令实现
- `cltq`等价于`movslq %eax, %rax`

:raised_hands:

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201132326824.png" alt="image-20220113232659660" style="zoom:80%;" />



:raised_hands:

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201132333144.png" alt="image-20220113233310066" style="zoom:80%;" />



<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201132339194.png" alt="image-20220113233956044" style="zoom:80%;" />

### 3.4.4 入栈与出栈

![image-20220113234153686](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201132341750.png)

![image-20220113234239427](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201132342516.png)



:heavy_exclamation_mark: PS: 栈可以像标准的内存寻址方式访问栈内的任意位置

## 3.5 算术与逻辑操作

![image-20220113234440693](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201132344761.png)



:heavy_exclamation_mark: PS:

- 上面的命令除`leaq`外，其余都省略 size 信息，如`subq` `subw`
- 只有`leaq`不改变标志位，其余指令都会改变
- 逻辑操作的 CF 与 OF 会置为 0
- 移位操作的 CF 将被设置为最后一个被移出的位，OF 置为 0
- `INC`和`DEC`会设置 OF 和 ZF，但不会改变进位标志

### 3.5.1 加载有效地址

`leaq`实际上是`movq`的变形，目的操作数必须是一个寄存器，指令形式是从内存读数据到寄存器，但根本就没有引用内存

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201132349977.png" alt="image-20220113234938909" style="zoom:80%;" />



### 3.5.2 特殊的算术运算

![image-20220114115001185](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141150625.png)



PS: 上图中`idivq`有误

- `R[%rdx]...` 存余数
- `R[%rax]...` 存商

对于乘法操作，假如超过 64 位，例如下例

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141156422.png" alt="image-20220114115608742" style="zoom:80%;" />

## 3.6 控制

### 3.6.1 条件码

![image-20220114120332589](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141203835.png)

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141203952.png" alt="image-20220114120341822" style="zoom:80%;" />

![image-20220114120402408](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141204847.png)

- 上表的命令只设置条件码不改变任何其他寄存器
- `cmp`与`sub`指令行为一致
- `test`与`and`行为一致

### 3.6.2 访问条件码

![image-20220114140230501](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141402613.png)



### 3.6.3 跳转指令

![image-20220114141134593](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141411701.png)



`jmp *%rax`: 跳到%rax存的值的那个地址

`jmp *(%rax)`：跳到M[%rax]存的值的那个地址

### 3.6.4 跳转指令的编码

有两种方式

- 最常用的是 PC 相对的，即会将目标指令的地址与紧跟在跳转指令后面那条指令的地址之间的差编码
  - 当执行 PC 相对寻址时，PC 的值是跳转指令后面那条指令的地址，而不是跳转指令本身的地址
- 给出绝对地址

下面这个例子是由编译文件 branch.c 产生的，是第一种相对地址的方式

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141423762.png" alt="image-20220114142332703" style="zoom:80%;" />

下面是链接后的反汇编版本

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141418221.png" alt="image-20220114141806177" style="zoom:80%;" />

可以看出，这些指令被定位到不同的地址，但是关于跳转指令的编码并没有变。这使得指令编码很简洁，目标代码可以不用做改变就可以移到内存中不同的位置

:raised_hands:

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141426211.png" alt="image-20220114142653099" style="zoom:80%;" />



:raised_hands:

红线是要求要填的内容

![image-20220114143329677](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141433781.png)



### 3.6.5 实现分支语句 1 ——使用条件控制

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141435622.png" alt="image-20220114143550485" style="zoom:80%;" />

**通用形式**

```c
if (test_expr)
    then-expr
else
    else-expr

等价于
        
	t = test_expr;
	if (!t)
        goto false;
	then-statement
    goto done;
false:
	else-statement
done:
	...
```

这种机制简单而通用，但是在现代处理器上，可能会非常低效，所以一种替代的策略是使用数据的条件传送

### 3.6.6 实现分支语句 2 —— 使用条件传送

![image-20220114144326844](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141443940.png)



<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141445624.png" alt="image-20220114144527528" style="zoom:80%;" />

**通用形式**

```c
v = test_expr ? then-expr : else-expr;

等价于
    
v = then-expr;
ve = else-expr;
t = test-expr;
if (!t) v = ve;

等价于
    
	if (!test-expr)
        goto false;
	v = then-expr;
	goto done;
false:
	v = else-expr;
done：
    ...
```



:raised_hands:

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141448065.png" alt="image-20220114144849982" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141449088.png" alt="image-20220114144923027" style="zoom:80%;" />



- 不是所有的条件表达式都可以用条件传送来编译：例如`return (xp ? *xp : 0)`这样还是有可能发生空指针异常
- 使用条件传送不是总是会提高代码效率，编译器必须考虑浪费的计算和由于分支预测错误所造成的性能之间的平衡
- 总的来说使用非常受限

### 3.6.7 循环

```c
do
    body-statement
    while (test-expr);

等价于
    
loop
    body-statement
    t = test-expr;
	if (t)
        goto loop;
```



<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141517376.png" alt="image-20220114151722273" style="zoom:80%;" />

```c
while (test-expr)
    body-statement
    
等价
    
	goto test;
loop:
	body-statement
test:
	t = test-expr;
	if (t)
        goto loop;

等价于
    
t = test-expr;
if (!t)
    goto done;
loop
    body-statement
    t = test-expr;
    if (t)
        goto loop;
done;
```

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141520897.png" alt="image-20220114152033786" style="zoom:80%;" />





<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141524382.png" alt="image-20220114152441265" style="zoom:80%;" />



### 3.6.8 switch 语句

GCC 根据开关情况的数量和开关情况值的私塾程度来翻译开关语句。当开关数量较多（例如 4 个以上），并且值的范围夸大比较小时，就会使用跳转表

![image-20220114153401357](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141534509.png)

- n 先减去 100，把取值范围挪到 0~6，作为 index
- 对 4 和 6 使用同样的代号（loc_D）
- 对缺失的 1 和 5 使用默认情况的标号（loc_def）
- `jmp`前面有`*`，表明这是一个间接跳转

![image-20220114153427407](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141534500.png)

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141539531.png" alt="image-20220114153917459" style="zoom:80%;" />

:raised_hands:

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141540974.png" alt="image-20220114154055894" style="zoom:80%;" />

## 3.7 过程

### 3.7.1 运行时栈

:fire:

- **栈是运行时分配的**
- **栈、数据、代码是存放在不同段的**

| 区域           | 作用                                                         |
| -------------- | ------------------------------------------------------------ |
| 栈(stack)      | 由编译器自动分配和释放，存放函数的参数值，局部变量的值等。操作方式类似与数据结构中的栈 |
| 堆(heap)       | 一般由程序员分配和释放，若程序员不释放，程序结束时可能由操作系统回收。与数据结构中的堆是两码事，分配方式类似于链表 |
| 静态区(static) | 全局变量和静态变量存放于此                                   |
| 文字常量区     | 常量字符串放在此，程序结束后由系统释放                       |
| 程序代码区     | 存放函数体的二进制代码                                       |

过程 P 调用 Q

![image-20220114154505047](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141545141.png)

大于 6 个的部分：从 rsp （栈顶指针）开始类似数组的`*(A+i)`取第 i 个参数

为了提高空间和时间效率，x86-64 过程只分配自己所需要的栈帧部分

例如当传递的参数 <=6 个时，那么所有的参数就可以通过寄存器传递，此时栈帧的某些部分就可以忽略

实际上，许多函数甚至根本不需要栈帧。当所有局部变量都可以保存在寄存器，且该函数不会调用其他函数时，就可以这样处理

### 3.7.2 转移控制

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201132303740.png" alt="image-20220113230337695" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201132305541.png" alt="image-20220113230527464" style="zoom:80%;" />

![image-20220114160254694](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141602912.png)

![image-20220114155659015](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141556324.png)

- rsp 是栈顶指针；rip 存的是 PC 的值

### 3.7.3 形参的存储

![image-20220114160553030](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141605391.png)

超过 6 个的部分就要通过栈来传递，即参数构造区部分

通过栈传递参数时，所有的数据大小都向 8 的倍数对齐

![image-20220114160638849](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141606329.png)



![image-20220114161252123](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141612378.png)

### 3.7.4 局部变量的存储

寄存器也可以存储局部变量。

在某些情况下，局部数据必须存放在内存中，包括：

- 寄存器不足够存放所有的本地数据
- 对一个局部变量使用地址运算符`&`，因此必须能够为它产生一个地址
- 某些局部变量是数组或结构，因此必须通过引用访问到

:raised_hands:

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141620540.png" alt="image-20220114162054255" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141621528.png" alt="image-20220114162101115" style="zoom:80%;" />



<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141623257.png" alt="image-20220114162328857" style="zoom:80%;" />



:raised_hands:

![image-20220114163339475](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141633541.png)

![image-20220114163344974](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141633125.png)



解释

- line 2: 由于在`call_proc`函数体内部声明了`x1, x2, x3, x4`4 个局部变量，所以首先为这 4 个局部变量分配内存空间。考虑到后面传递形参给`proc`需要额外的两个空间，故共需 32 byte，存储在栈上
- line 3 - line 6: 初始化这 4 个局部变量并赋值
- line 7 - line 15: 调用`proc`前的参数传递准备
  - line 15: 传递第一个形参，即 x1。这里直接传递 x1 的值，即1，传递媒介是 edi。当进入`proc`后，可直接通过 edi 访问 x1（且不会影响`call_proc`内部的 x1 的值，因为传递的是值！）
  - line 14: 传递第二个形参，即 &x1。这里将 x1 的地址，即 %rsp+24 送入 %rsi。这样当进入`proc`后，发现第二个参数是地址，按规则得知第二个参数应存放在 %rsi，所以会读取 %rsi。注意，`proc`内部对M[%rsi]的改变会影响`call_proc`内 x1 的值，因为是直接对内存进行改变
  - line 13 - line 10 同理。这样我们目前已经传递了 6 个参数，还剩最后两个 x4 与 &x4
  - 寄存器以及用完了，我们只能将剩余的两个形参通过栈来传递，即 %rsp+8 与 %rsp+17
    - line 7 - line 8: 将 %rsp+17 （值即为$x4）存入 %rsp+8，用来给`proc`访问，注意到 M[%rsp+17] 存的就是 x4, 而 %rsp+8 = %rsp+17，所以M[%rsp+8]=M[%rsp+17]，即`proc`内部对 x4 的改变会影响`call_proc`。而这也符号代码逻辑，因为我们传递的就是指针
    - line 9: 传递 x4 值给 %rsp，`proc`内部对 x4 的改变不会影响`call_proc`的 x4。因为`call_proc`内部通过 %rsp+17 访问（%rsp+17 存的就是 x4 的值），而`proc`将会通过 %rsp+16 访问（这两个 %rsp 存的值不一样，因为 `call`了）
- line 16: `call`后会将`call`的下一句指令的地址入栈，%rsp += 8（所以原本通过 %rsp，%rsp+8 访问的局部变量区（给`proc`访问的区域）变成通过 %rsp+8, %rsp+16 访问，这可以通过`proc`的汇编代码可以看出）
- 剩下行：`proc`返回后剩下的指令

可以发现，第一个到第六个形参 按 3.7.3 给出的表指定的寄存器的顺序传递，剩余的两个形参依次存在栈顶

注意：返回地址是在存完多余参数，局部变量，被调用者保存寄存器后，由call存入的



![image-20220114163359315](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141633382.png)

下图是函数 proc 的内容以及栈帧

![image-20220114160638849](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141606329.png)

![image-20220114161252123](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201141612378.png)



PS: 上图中 a4p a4 为`call_proc`为传递参数而构造的局部变量区，'返回地址'为`call_proc`中`call`命令的下一个指令的地址



### 3.7.5 关于被调用者保存寄存器与调用者保存寄存器

![image-20220114230134351](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201142301466.png)



换句话说，当 Q 临时使用被调用者保存寄存器时，要保证不用时恢复其旧值；剩下的调用者保存寄存器 Q 可以任意使用与修改，而这其中的 6 个寄存器用来传参



<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201142306935.png" alt="image-20220114230613780" style="zoom:80%;" />

- line 2: 调用 Q 的时候，y 需要通过 rdi 传递（规定的顺序），由于 rdi 是调用者保存寄存器，所以 P 必须自己保存，即需要保存 x 的值。我们选择使用 rbp 来暂存 x 的值，为了保证不破坏 rbp 原有的状态（`main`函数中可能使用到 rbp），所以 P 要自己另存 rbp 的值作为副本
- line 3: 由于下面要存储返回结果到 rbx，所以也要将 rbx 的内容入栈
- line 4: 对齐栈帧？？？
- line 5: 将 x 的值暂存到 rbp
- line 6: 将 y 通过 rdi 传递
- line 7: call Q
- line 8: 调用 Q 结束后，Q 的返回值存在 rax，将其转存到 rbx
- line 9: 将 x（存在 rbp）暂存到 rdi
- ...

PS: 函数的返回值默认存在 %rax



总结

当发生函数调用的时候,栈空间中存放的数据是这样的:
1、调用者函数把被调函数所需要的参数按照与被调函数的形参**顺序相反**的顺序压入栈中,即:从右向左依次把被调函数所需要的参数压入栈;
2、调用者函数使用`call`指令调用被调函数,并把`call`指令的下一条指令的地址当成返回地址压入栈中(这个压栈操作隐含在`call`指令中);
3、在被调函数中,被调函数会先保存调用者函数的栈底地址(`push ebp`),然后再保存调用者函数的栈顶地址,即:当前被调函数的栈底地址(`mov ebp,esp`);
4、在被调函数中,从 ebp 的位置处开始存放被调函数中的局部变量和临时变量,并且这些变量的地址按照定义时的顺序依次减小,即:这些变量的地址是按照栈的延伸方向排列的,先定义的变量先入栈,后定义的变量后入栈;

![](https://img-blog.csdn.net/20131124171848671?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd2FuZ3llemkxOTkzMDkyOA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)



首先，将调用者函数的 EBP 入栈(`push ebp`)

然后将调用者函数的栈顶指针 ESP 赋值给被调函数的 EBP (作为被调函数的栈底mov ebp,esp`)

此时，EBP 寄存器处于一个非常重要的位置，该寄存器中存放着一个地址(原 EBP 入栈后的栈顶)

- 以该地址为基准,向上(栈底方向)能获取**返回地址、参数值**
- 向下(栈顶方向)能获取函数的**局部变量值**
- 而该地址处又存放着上一层函数调用时的 **EBP** 值;

一般规律

- SS:[ebp+4] 处为被调函数的返回地址
- SS:[EBP+8] 处为传递给被调函数的第一个参数(最后一个入栈的参数，此处假设其占用4字节内存)的值
- SS:[EBP-4] 处为被调函数中的第一个局部变量
- SS:[EBP]处为上一层EBP值

如此递归，就形成了函数调用栈；

![](https://img-blog.csdn.net/20131125160054015?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd2FuZ3llemkxOTkzMDkyOA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

### 3.7.6 递归调用

![image-20220114232540403](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201142325512.png)

## 3.8 数组

### 3.8.1 一维

![image-20220115113814521](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201151138942.png)

PS: 最后一行表明 index 存成 long 类型

### 3.8.2 多维

![](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201151151231.png)

![image-20220115115135434](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201151151568.png)



![image-20220115115142200](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201151151428.png)



### 3.8.3 定长数组

:raised_hands:

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201151439878.png" alt="image-20220115143946379" style="zoom:80%;" />



### 3.8.4 变长数组

变长数组缩写为 **VLA**(variable-length array)，它是一种在运行时才确定长度的数组（地址空间连续的数组，并不是表现得像数组的多段内存组成的数据结构），而非编译期。

声明：`int A[expr1][expr2]`

可以作为一个局部变量，也可以作为形参

在遇到这个声明时，先求表达式的值来确定维度

注意：**变长数组是分配在堆上吗？**

当然不是，注意这里概念不要搞混淆了，变长数组不是动态数组，虽然是到运行时才确定大小，但说到底它还是局部变量，而局部变量，又没有动态申请内存的动作，当然也只会在栈上分配内存。

:raised_hands:

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201151505812.png" alt="image-20220115150513353" style="zoom:80%;" />



<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201151505896.png" alt="image-20220115150541527" style="zoom:80%;" />



PS: 具体原理需要深入了解

## 3.9 异质的数据结构

1. 对齐原则：任何 K 字节的基本对象的地址必须是 K 的倍数

![image-20220115150829624](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201151508044.png)



2. 结构的末尾可能做一些填充使其数据结构的每个元素都会满足对齐的要求

```c
struct S2 {
    int i;
    int j;
    char c;
};
```

如果我们将这个结构打包成 9 个字节，只要保证结构的起始地址满足 4 字节的对齐要求（这样子我们可以保证满足字段 i 和 j 的对齐要求）

不过考虑以下声明

```c
struct S2 d[4];
```

若分配 9 个字节，不可能满足 d 的每个元素的对齐要求。所以编译器会给结构 S2 分配 12 个字节，最后 3 个字节浪费空间

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201151512944.png" alt="image-20220115151225759" style="zoom:80%;" />



3. 强制对齐的情况

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201151513632.png" alt="image-20220115151328346" style="zoom:80%;" />

### 3.9.1 结构体

结构体的所有组成部分都存放在内存的一段连续区域中，各个字段的选取完全是在编译时处理的，机器代码不包含关于字段声明或字段名字的信息

:raised_hands:

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201151514765.png" alt="image-20220115151436470" style="zoom:80%;" />

### 3.9.2 联合

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201151515334.png" alt="image-20220115151546930" style="zoom:80%;" />

在一些上下文中，联合十分有用，但有时会引发一些错误

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201151516770.png" alt="image-20220115151640399" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201151517584.png" alt="image-20220115151746303" style="zoom:80%;" />



<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201151519046.png" alt="image-20220115151930759" style="zoom:80%;" />

## 3.10 将控制与数据结合起来

### 3.10.1 指针

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201151522389.png" alt="image-20220115152252289" style="zoom:80%;" />

### 3.10.2 GDB 调试器的应用

### 3.10.3 内存越界引用和缓冲区溢出

C 对于数组引用不进行任何边界检查，而且局部变量和状态信息（例如寄存器的值和返回地址）都存放在栈中。这里存在一个隐患：对越界的数组元素的写操作会破坏存储在栈中的状态信息。当程序使用这个被破坏的状态，视图重新加载寄存器或执行`ret`指令时，就会出现很严重的错误。

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201151527950.png" alt="image-20220115152719539" style="zoom:80%;" />



:raised_hands: 缓冲区溢出实例

![](https://img2018.cnblogs.com/blog/1539443/201902/1539443-20190217110743142-907197647.png)

![](https://img2018.cnblogs.com/blog/1539443/201902/1539443-20190217110805893-1535175405.png)



c 代码中我们为 char 数组 buf 设置了 8 个字节的缓冲区，但是我们在汇编代码中为它在栈上分配了 24 个字节。

剩下 16 个字节是不被使用的，而在`gets`中，由于未指定目标缓冲区大小，当输入超过 8 个字符时，显然会破坏后续的栈空间，在 23 个之前不会有太大影响，而一旦超过 23 个，会破坏返回地址的值，若超过 32 个字节，会破坏调用函数(即 echo )里，调用caller之前存的值（还记得吗，返回地址是在存完多余参数，局部变量，被调用者保存寄存器后，由call存入的）

具体情况如下

<img src="https://img2018.cnblogs.com/blog/1539443/201902/1539443-20190217121645193-721081720.png" style="zoom:80%;" />

:raised_hands: 一道例题

![](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201151551058.png)

![](https://img2018.cnblogs.com/blog/1539443/201902/1539443-20190217124853171-1594764518.png)

- 栈帧的第一行是`call get_line()`后的结果，给出了返回地址 0x400076
- 现在进入`get_line`
- line 1: push %rbx
  - 这是因为 rbx 是被调用者保存寄存器，需要由`get_line`保存，因为后面`get_line`可能要用到 rbx
  - 所以栈帧第二行是将 rbx 的值压入后的结果`01 23 45 67 89 AB CD EF`
  - 此时 rsp 指向栈帧的第 2 行末尾
- line 2: 随后给局部变量 buf 数组分配栈空间，分配了 0x10 = 16 个字节（多分配一点防止溢出）。分配后 rsp 指向栈帧的第 4 行末尾
- line 3: 将 buf 的指针存入 rdi
- line 4: `call gets`

输入字符后（输入的字符需要 24 个字节来存），栈帧如下所示

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201151604925.png" alt="image-20220115160446844" style="zoom:80%;" />

要注意的是，调用`gets`后，存入输入的字符串的顺序是先存到 %rsp，再到 1(%rsp)，一直延续的，所以第 4 行对应存前 8 个字符，且从后向前存，结果是: `37 36 35 34 33 32 31 30`，同理，第三行结果是: `35 34 33 32 31 30 39 38`，再向上存，第二行保存的 %rbx 值被覆盖为: `33 32 31 30 39 38 37 36`，最后仍剩 2 个字符，4 和空，所以返回地址对应的第一行会被覆盖为: `00 00 00 00 00 40 00 34`。



E 题，对`malloc`的调用应该是`malloc(strlen(buf)+1)`, 因为`strlen`计算长度时并未包含空字符，而且`malloc`后未检查返回值是否为`NULL`



根据上述内容，可以看出缓冲区溢出攻击的原理：在给程序输入的字符串中包含了一些恶意的攻击代码，同时会用一个指向攻击代码的指针覆盖掉返回地址，这样，函数返回时会“迷路”，转而去执行攻击代码。

### 3.10.4 对抗缓冲区溢出攻击

**1. 栈随机化**

程序开始时，在栈上分配一段 0~n 字节之间的随机大小的空间。程序不使用这段空间，但是它会导致程序每次执行后序的栈位置发生了变化。分配的范围 n 必须足够大，又不能太大，不然会浪费。下面的代码是一种确定“典型的”栈地址的方法：

```c
int main() {
    long local;
    printf("local at %p\n", &local);
    return 0;
}
```

在 Linux 系统中，栈随机化已经变成了标准行为，它是 ASLR（地址空间布局随机化）技术的一种

然而攻击者可以用蛮力克服随机化。

![image-20220115161351804](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201151613896.png)



也就是说，暴力破解的话需要一个一个尝试所有的栈入口地址，即 $2^{23}$ 个地址。但我们加上 256 个字节的空操作，这些字节覆盖了 $2^{8}$ 个地址，那么我们只需要枚举 $2^{15}$​ 个地址即可。而这是可能的



**2. 栈破坏检测**

加入栈保护者机制，其思想是在栈帧中任何局部缓冲区与栈状态之间存储一个特殊的金丝雀值，也称哨兵值。在恢复寄存器状态和`ret`之前，程序检测这个金丝雀值是否改变

![image-20220115162125750](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201151621837.png)



:raised_hands:

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201151622084.png" alt="image-20220115162238981" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201151622249.png" alt="image-20220115162250137" style="zoom:80%;" />



**3. 限制可执行代码区域**

限制哪些内存区域能够存放可执行代码：在典型的程序中，只有保存编译器产生的代码的那部分内存才需要是可执行的，其他部分被限制为只允许读和写

### 3.10.5 变长栈帧

栈是在运行时分配的，有些函数需要的局部存储是变长的，例如`alloca`

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201151635271.png" alt="image-20220115163523172" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201151635488.png" alt="image-20220115163534341" style="zoom:80%;" />

![](https://img2018.cnblogs.com/blog/1539443/201902/1539443-20190217165101938-82676343.png)

程序中声明的 long * 数组，其所占空间大小由传入的 n 决定，所以栈上至少需要 8n 个字节容纳这个数组，另外局部变量 i 由于用到了 &i，所以也要存到栈上。

line 2 - line 3: 首先 %rbp 是被调用者保存寄存器，要把 %rbp 原先的值存入栈，%rbp 用来存初始的栈顶位置，它被称为“帧指针”

为什么要特意用一个被调用者保存寄存器来存初始的栈顶位置呢？是因为，在 for 循环中，i 是不停变化的，因此局部变量 i 在被存到栈中后不能不管它，要在循环过程中修改它的值，而右图中可看到，i 存放在初始栈顶向下的 8 个字节中，因此需要记录下初始栈顶位置以方便定位到 i

line 4: 在栈上分配 16 个字节，其中 8 个存 i，8 个没用到

line 5: 计算得到 8n+22

line 6: `andq $-16,%rax` 注意 -16 补码是0x1111111111111110，即将%rax最后 4 位置 0，也就是把 8n+22 向下舍入最接近的 16 的倍数，n 是奇数时，结果是 8n+8，n 是偶数时，结果是 8n+16

line 7: 开辟相应大小的栈空间，开辟后栈指针 %rsp 的地址由右图中的 s1 变到下方的 s2

line 8 - line 10: 计算 (s2+7)/8 * 8，代表将 s2 向上舍入到最近的8的倍数，即对应右图中的 p。注意左图显示的只是部分汇编代码，它们不是连续的，第 11 行和 12 行间有省略号，这里做了一些事情，比如注释中的 i in %rax，在上面的代码中没有操作过，因此这部分操作是被略过去的。

line 13 - line 19: 完成 for 循环

line 20: `leave`指令，等价于 `movq %rbp, %rsp　　popq %rbp`，即恢复寄存器 %rbp 的值，且栈被释放，恢复回之前的值。



## References

[【操作系统】深入函数调用堆栈----汇编_从底层爬-程序员宝宝](https://www.cxybb.com/article/woalss/106197623)

[CSAPP阅读笔记-变长栈帧，缓冲区溢出攻击-来自第三章3.10的笔记-P192-P204](http://runxinzhi.com/czw52460183-p-10390432.html)

本章节参考答案页码 P262





