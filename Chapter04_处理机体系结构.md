# Chapter04 处理机体系结构

:hammer: 为什么要学习处理机体系机构呢

- 帮助理解整个计算机系统如何工作
- 你可能以后会设计 CPU 

:happy: 本章将要学习

- 定义一个简单的指令集
- 介绍数字硬件介绍、HCL硬件控制语言
- 给出一个机遇顺序操作、功能正确但不实用的Y86-64处理器
- 流水线化处理器、解决冒险与冲突

:fire: 重点

## 4.1 Y86-64 指令集结构

我们将定义一个指令集体系结构，包含定义各种状态单元、指令集和他们的编码、一组编程规范和异常事件处理。

### 4.1.1 程序员可见状态

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201161127332.png" alt="image-20220116112753218" style="zoom:80%;" />

### 4.1.2 Y86-64 指令

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201161130546.png" alt="image-20220116113029438" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201161132605.png" alt="image-20220116113258543" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201161134444.png" alt="image-20220116113410390" style="zoom:80%;" />

- 我们不支持第二变址寄存器和任何寄存器值的伸缩；不允许从一个内存地址直接传送到另一个内存地址，不允许将立即数传送到内存
- 4 个整数操作指令，只对寄存器数据进行操作(x86-64允许对内存数据进行这些操作)；这些指令会设置3个条件码ZF SF OF（零、符号、溢出）
- 每条指令的第一个字节表明指令的类型：这个字节分为两部分，每部分 4 位，高 4 位是代码(icode)部分，低 4 位是功能(function)部分，代码值为 0~0xB，功能值只有在一组相关指令共用一个代码时才有用
- 15 个寄存器的标识符为 0~0xE，当指令不需要指定任何寄存器时，用 0xF 表示
- 这里我们设计的指令都使用绝对寻址方式
- 小端法编码
- 前缀编码，无二义性

例如：`rmmovq %rsp, 0x123456789abcd(%rdx)`编码为`4042cdab896745230100`



在更完整的设计中，处理器通常会调用一个**异常处理程序**，例如终止程序或调用一个用户自定义的信号处理程序

![image-20220117105619726](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171056810.png)

对于有歧义的命令`push/pop %rsp`，我们定义为先压入/弹出数字，然后再变化栈顶指针

## 4.2 硬件控制语言 HCL

### 4.2.1 HCL 及其 常见表达式

HCL 与 C 不一样，我们不把它看成执行了一次计算并将结果放入内存中的某个位置

相反，它只是给表达式一个名字



常见逻辑设计及其表达式：

1. 

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171102468.png" alt="image-20220117110200404" style="zoom:80%;" />

2. 

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171103768.png" alt="image-20220117110337650" style="zoom:80%;" />



2. 

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171104224.png" alt="image-20220117110434103" style="zoom:80%;" />



4. 

<img src="C:/Users/Lenovo/AppData/Roaming/Typora/typora-user-images/image-20220117110537574.png" alt="image-20220117110537574" style="zoom: 67%;" />



ALU

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171107607.png" alt="image-20220117110711550" style="zoom:67%;" />

### 4.2.2 集合关系

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171108041.png" alt="image-20220117110847956" style="zoom:80%;" />



### 4.2.5 存储器和时钟

两类存储器设备

- 时钟寄存器（简称寄存器）：存储单个位或字

  <img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171110544.png" alt="image-20220117111046488" style="zoom:80%;" />

- 随机访问寄存器（简称内存）：存储多个字，用地址来选择读或写某个字

  - 虚拟内存系统
  - 寄存器文件
    - 寄存器标识符作为地址
    - 有两个读端口（A 和 B）与一个写端口（W）
    - 多端口随机访问，运行同时进行多个读和写操作
    - 写操作由时钟信号控制，上升沿触发；当 dstW 设为特殊的 ID 值 0xF 时，不会写任何程序寄存器
    - 同时读写同一个寄存器：我们会看到一个从旧值变到新值的变化

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171110525.png" alt="image-20220117111052486" style="zoom:80%;" />



## 4.3 Y86-64 的顺序实现

我们描述一个称为 SEQ（顺序的）的处理器

### 4.3.1 五大阶段

- 取指：从内存读取指令字节，地址为 PC 的值，它还可能取出 rA 与 rB，还可能取出一个四字节常数valC 。按顺序的方式计算下一条指令地址 valP = PC + 已取出的指令的长度
- 译码：从寄存器文件最多读取两个操作数，得到 valA 和/或 valB；通常读入 rA 和 rB 字段指明的寄存器，有时读 %rsp
- 执行：ALU 要么执行指令指明的操作，要么增加或减少栈指针。得到的值称 valE。可能设置条件码。这个阶段会决定是不是应该选择分支
- 访存：读/写内存
- 写回：最多可以写两个结果到寄存器文件
- 更新 PC

处理器无限循环，发生任何异常时，执行 halt 指令或非法指令，或试图读/写非法地址

:raised_hands:

![image-20220117112519772](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171125121.png)

![image-20220117115245907](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171152246.png)

![image-20220117115013475](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171150645.png)



PS

- valP：下一条指令的地址，`val <- PC + x`，这里的`x`指的就是本条指令的长度
- valC：从内存读取的跳转地址
- valM：从内存读取的值或返回地址
- valE：写操作的地址
- rA rB 指的是寄存器的标识符，R[rA] 指的是该标识符指定的寄存器内存的值

### 4.3.2 硬件结构

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171136579.png" alt="image-20220117113635076" style="zoom:80%;" />

- 取指：指令内存通过 PC 读取指令的内容，PC 增加器计算 valP

- 译码：从寄存器文件的两个读端口 A 和 B 读取寄存器值 valA 和 valB

- 执行：送入 ALU

  - 若是整数操作，执行相应的运算
  - 对于其他指令，作为一个加法器调整栈指针，或计算有效地址，或简单加0

  ALU 还负责计算条件码的新值

- 访存：数据内存读或写一个内存字

- 写回：通过寄存器文件的两个写端口写入寄存器

  - 端口 E：写 ALU 计算出来的值
  - 端口 M：用来写从数据内存读出的值

- PC 更新：PC 的新值选择自

  - valP：下一条指令的地址
  - valC：调用指令或跳转指令指定的目标地址
  - valM：从内存读取的返回地址



下面是更细致的硬件图

![image-20220117115652695](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171156286.png)

- 白色方框：时钟寄存器。PC 是 SEQ 唯一的时钟寄存器
- 浅蓝色方框：硬件单元。内存、ALU
- 灰色圆角矩形：控制逻辑块。这些块用来从一组信号源中选择，或者用来计算一些布尔函数。（想想之前介绍的 HCL）。后面我们会详细分析这些块，包括 HCL 描述
- 粗线：字数据
- 细线：字节数据或更短
- 虚线：一个 bit

之前那描述的表格的计算都有这样的性质：每一行都代表某个值的计算或激活某个硬件单元，我们在下表给出一些示例，列出这些计算和动作设计的元件

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171210583.png" alt="image-20220117121050066" style="zoom:80%;" />

### 4.3.3 SEQ 的时序

SEQ 的实现包括组合逻辑和两种存储器设备

- 时钟寄存器：PC、条件码寄存器
  - PC: 每个时钟周期，PC都会装载新的指令地址
  - 条件码寄存器：只有执行整数运算指令时，才会装载
- 随机访问存储器：寄存器文件、指令内存、数据内存
  - 寄存器文件的两个写端口允许每个时钟周期更新两个寄存器

原则：**从不回读**

- 处理器从来不需要为了完成一条指令的执行而去读由该指令更新了的状态
- 假设我们对 pushq 指令的时序是先将 %rsp-8，在将更新后的 %rsp 值作为写操作的地址，这样就违背了该原则

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171350083.png" alt="image-20220117135044245" style="zoom:80%;" />

每次时钟由低变高时，更新所有值，并且处理器开始执行一条新指令

### 4.3.4 硬件结构的描述

**1. 取指阶段**

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171354914.png" alt="image-20220117135452612" style="zoom:80%;" />



```c
bool need_regids = 
    icode in {IRRMOVQ, IOPQ, IPUSHQ, IPOPQ,
             IIRMOVQ, IRMMOVQ, IMRMOVQ};

bool need_valC = 
    icode in {IIRMOVQ, IRMMOVQ, IRMMOVQ, IMRMOVQ, IJXX, ICALL};
```



这个单元一次从内存读取 10 个 Byte

- 第一个字节：指令字节，分为 2 个 4 位的数，分别标号为 icode ifun
  - `instr_valid` 指令合法吗
  - `need_regids` 指令包括一个寄存器标识符吗
  - `need_valC` 指令包括常数字吗
  - 当指令地址越界时，会产生信号`instr_valid` `imem_error`
- 其余 9 个字节：寄存器标识符和常数字

PC 增加器产生值 p+1+r+8i

- p: PC 值
- r: need_regids 值
- i: need_valC 值

**2. 译码和写回阶段**

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171403845.png" alt="image-20220117140350482" style="zoom:80%;" />



```c
word srcA = [
    icode in {IRRMOVQ, IRMMOVQ, IPOQ, IPUSHQ} : rA;
    icode in {IPOPQ, IRET} : RRSP;
    1: RNONE;
];

word srcB = [
    icode in {IOPQ, IRMMOVQ, IMRMOVQ} : rB;
    icode in {IPUSHQ, IPOPQ, ICALL, IRET} : RRSP;
    1: RNONE;
];
    
word dstE = [
    // ...
];
  
word dstM = [
    // ...
]
```



把这两个阶段练习在一起是因为它们都要访问寄存器文件

- 两个读端口 A 和 B，读出来的值送往 valA valB
- 两个写端口 M 和 E，写入 dstE dstM
- 若某个地址端口上的值为0xF，表示不需要访问寄存器

**3. 执行阶段**

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171412562.png" alt="image-20220117141234230" style="zoom:80%;" />

```c
word aluA = [
    // ...
];

word aluB = [
    // ...
];

word alufun = [
    icode == IOPQ : ifun;
    1: ALUADD;
];

bool set_cc = icode in {IOPQ};
```

- cond 硬件单元会根据条件码和功能码来确定是否进行条件分支或条件传送，它产生信号 Cnd，用于设置条件传送的 dstE，也用在条件分支的下一个 PC 逻辑中。对于其他指令，取决于指令的功能码和条件码的设置，Cnd 信号可以被设置为 1 或 0，但控制逻辑会忽略它。我们省略这个单元的详细设计

**4. 访存阶段**

![image-20220117141855005](https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171428984.png)

```c
word mem_addr = [
    icode in {IRMMOVQ, IPUSHQ, ICALL, IMRMOVQ}: valE; // 表名这些指令需要对地址为valE的内存进行写
    icode in {IPOPQ, IRET}: valA; // 表明这些指令需要对valA所指示的寄存器进行写
];

word mem_data = [
    icode in {IRMMOVQ, IPUSHQ}: valA; // 表明这些指令需要读取valA所指示的寄存器的数据
    icode == ICALL : valP; // 表明这些指令需要读取valP所指示的内存地址的数据
];

bool mem_read = icode in {IMRMOVQ, IPOPQ, IRET};
bool mem_write = icode in {IRMMOVQ, IPUSHQ, ICALL};

word State = [
    imem_error || dmem_error: SADR;
    !instr_valid: SINS;
    icode == IHALT: SHLT;
    1: SAOK;
];
```

可以看到内存读或写的地址总是 valE 或 valA

**5. 更新 PC**

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171427071.png" alt="image-20220117142504880" style="zoom:80%;" />

```c
word new_pc = [
    icode == ICALL: valC;
    icode == IJXX && Cnd : valC;
    icode == IRET: valM;
	1: valP;
];
```

**小结**：唯一缺点是太慢了。这种方法每个单元在整个时钟周期中只有一部分时间才被使用。所有我们引入**流水线**

## 4.4 流水线

### 4.4.1 简单流水线

可以提高系统的吞吐量，但会轻微地增加延迟

吞吐量=每秒执行的指令条数

:raised_hands:

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171435861.png" alt="image-20220117143535550" style="zoom:80%;" />



上图中 3.12 = 1/(20+300)

现将组合逻辑拆分为三块（将原来的 300ps 分解为三个模块，每个模块 100ps），中间用寄存器中转

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171435964.png" alt="image-20220117143559455" style="zoom: 67%;" />

​	

上图中相当于 360ps 执行了 3 条，所以，吞吐量 8.33=1/120

详细说明如下

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171439684.png" alt="image-20220117143916106" style="zoom:80%;" />



- 颜色与甘特图的颜色是对应的
- 只有当时钟上升时，寄存器的值才会被更新
- 吞吐量提高的代价是增加了一些硬件，以及延迟的少量增加（寄存器的时间开销）
- 为什么要增加寄存器？因为不同模块之间执行时间无法统一，可能模块A执行完但模块B还没执行完，所以需要寄存器这个中转站暂存模块A的结果。可以看出，模块划分的越细，延迟就越大

### 4.4.2 局限性

**1. 不一致的划分：运行时钟的速率由最慢的阶段的延迟限制的**

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171609703.png" alt="image-20220117160927292" style="zoom:80%;" />

**2. 流水线过深，收益反而下降：将计算时钟的时间缩短k倍，由于流水线寄存器的延迟，吞吐量不会恰好增加k倍**

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171610782.png" alt="image-20220117161014403" style="zoom:80%;" />

### 4.4.3 带反馈的流水线潜在的危险

到目前为止的流水线设计，我们忘记考虑相邻指令之间可能是相关的情况

- 数据流相关
- 控制流相关

潜在危险

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171612135.png" alt="image-20220117161246507" style="zoom:80%;" />

上图的危险是指：原来 I1 的输出作为 I2 的输入，加入带反馈的流水线后 I1 的输出变成 I4 的输入

## 4.5 Y86-64 流水线实现

### 4.5.1 SEQ+: 重新安排计算阶段

我们将原本最后一步的更新 PC 阶段挪到时钟周期的开始时执行

当一个新的时钟周期开始时，这些信号值通过同样的逻辑来计算当前指令的 PC，我们将这些寄存器标号为如下图，用来指明在任一给定的周期，它们保存的是前一个周期中产生的控制信号

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171627934.png" alt="image-20220117162726567" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171629045.png" alt="image-20220117162907341" style="zoom:80%;" />

### 4.5.2 插入流水线寄存器

在 SEQ+ 的各阶段插入流水线寄存器，并对信号重新排列，得到 PIPE-处理器

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171630513.png" alt="image-20220117163034830" style="zoom:80%;" />



符号说明

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171631721.png" alt="image-20220117163117424" style="zoom:80%;" />

:raised_hands:

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171633804.png" alt="image-20220117163338189" style="zoom:80%;" />

### 4.5.3 预测下一个 PC

- 当分支地址比下一条地址低时就预测选择分支，因为循环是由后向分支结束的，而循环通常会执行多次。前向分支用于条件操作，这样的可能性较小
- 使用栈的返回地址预测：对于大多数程序，预测返回值很容易，因为过程调用和返回总是成对出现的。高性能处理器运用了这个属性，在取指单元中放入一个硬件栈，保存过程调用指令产生的返回地址。通常这种预测很可靠，这个硬件栈对程序员来说是不可见的

### 4.5.4 流水线冒险

**1. 数据冒险**

下一条指令的操作数与上一条指令有关，但上一条指令未将结果写入寄存器

:raised_hands: eg（假设寄存器初始值为0）

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171649623.png" alt="image-20220117164955378" style="zoom:80%;" />



上图执行到第 6 个周期时，`addq`指令正在读取寄存器的值，但此时 rdx 的值是正确的（已写入）， rax 的值却是错误的（正在处于写入阶段），所以结果是错误的

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171701838.png" alt="image-20220117170109375" style="zoom:80%;" />

数据冒险的类型

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171701377.png" alt="image-20220117170155542" style="zoom:80%;" />



数据冒险解决方案

1. 暂停：暂停技术就是让一组指令阻塞在它们所处的阶段，而允许其他指令继续通过流水线。

   那么针对上述例子，我们该怎么做？

   处理方法是：每次要把一条指令阻塞在译码阶段，就在执行阶段插入一个气泡。气泡就像一个自动产生的`nop`指令，不会改变任何状态

   处理器不会让`addq`指令带着不正确的结果通过这个阶段，而是会暂停指令，将它阻塞在译码阶段；此外我们还要必须将紧随其后的`halt`指令阻塞在取指阶段

   具体机制我们在后面再谈

   <img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171722726.png" alt="image-20220117172254358" style="zoom:80%;" />

   虽然实现这一机制相当容易，但降低系统的吞吐量

2. 转发

   思想：与其暂停直到完成，不如简单地将要写的值传送到目的流水线寄存器

   在D读，检查E中的dst是不是我要读的寄存器，若是，直接给我

   <img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171725970.png" alt="image-20220117172522464" style="zoom:80%;" />

   <img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171727469.png" alt="image-20220117172725157" style="zoom:80%;" />

   <img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171728629.png" alt="image-20220117172828311" style="zoom:80%;" />



最终 PIPE 硬件结构如下

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171726614.png" alt="image-20220117172641733" style="zoom:80%;" />



3. 加载/使用数据冒险

   有一类数据冒险不能单纯用转发来解决，因为内存读在流水线发生的比较晚

   <img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171731061.png" alt="image-20220117172911662" style="zoom:80%;" />

   此时我们可以将暂停和转发结合起来

   <img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171731061.png" style="zoom:80%;" />

​	这种方法称加载互锁（load interlock）；加载互锁与转发技术结合起来足以处理所有可能类型的数据冒险。因为只有加载互锁会降低流水线的吞吐量，我们几乎可以实现每个时间周期发射一条新指令的吞吐量目标



**控制冒险**

- 在我们的流水线化处理器中，控制冒险只会发生在`ret`和跳转指令，而且后一种情况只有在跳转方向预测错误时才会造成麻烦

- 对于`ret`我们应该这样做

  <img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171732088.png" alt="image-20220117173233714" style="zoom:80%;" />

  但效率太低，我们需要预测，并引入恢复机制

- 对于`jne`

  <img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171736305.png" alt="image-20220117173631099" style="zoom:80%;" />

  <img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171736310.png" alt="image-20220117173642890" style="zoom:80%;" />

  AD：**任何情况下都一定是两条，因为`jne`刚完成E时前面只有 2 个周期，只会进入两条指令；且这两条一定一条刚完成 D，一条刚完成 F，均未进入执行阶段，故可简单采用插入气泡解决**

### 4.5.5 异常处理

- 种类：`halt`指令；非法指令；非法地址
- 在一个流水线化的系统中，异常处理包括一些细节问题

- - 可能同时有多条指令引起异常：基本原则是由流水线中最深的指令引起的异常优先级最高
  - 当首先取出一条指令，开始执行时，导致了一个异常，而后来由于分支预测错误取消了该指令：流水线控制逻辑会取消该指令，但是我们想要避免出现异常
  - 一条指令导致了一个异常，它后面的指令在异常指令完成之前改变了部分状态

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171745659.png" alt="image-20220117174520446" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171744336.png" alt="image-20220117174430914" style="zoom:80%;" />



### 4.5.7 PIPE 各阶段的实现

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171747546.png" alt="image-20220117174753064" style="zoom:80%;" />



**1. PC 选择和取指阶段**

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171748265.png" alt="image-20220117174829790" style="zoom:80%;" />



<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171753098.png" alt="image-20220117175356660" style="zoom:80%;" />



<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171754469.png" alt="image-20220117175444228" style="zoom: 80%;" />

**2. 译码与写回阶段**

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171755626.png" alt="image-20220117175522012" style="zoom:80%;" />

```c
word Stat = [
    W_stat == SBUB: SAOK;
    1: W_stat;
];
```

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171756331.png" alt="image-20220117175611735" style="zoom:80%;" />

这 5 个转发源的优先级是非常重要的：流水线阶段越早的转发源优先级越高。原因如下

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171756422.png" alt="image-20220117175657796" style="zoom:80%;" />

**3. 执行阶段逻辑**

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171757969.png" alt="image-20220117175723611" style="zoom:80%;" />

**4. 访存阶段**

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171757339.png" alt="image-20220117175747949" style="zoom:80%;" />



### 4.5.8 流水线控制逻辑

必须考虑与处理下面 4 种情况

- 加载使用冒险：在一条从内存中读取一个值的指令和一条使用该值的指令之间，流水线必须暂停一个周期。解决方法：二者之间插入气泡（图4-54）

- 处理`ret`：流水线必须暂停直到`ret`指令到达写回阶段。解决方法：后面插入 3 个气泡（图4-55）.但实际上我们是这么处理的，二者是等效的

  <img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171812300.png" alt="image-20220117181253892" style="zoom:80%;" />

- 预测错误的分支：在分支逻辑发现不应该选择分支之前，分支目标处的几条指令已经进入流水线了。必须取消这些指令，并从跳转指令后面的那条指令开始取指。解决方法：在后面两条已经发生的指令插入气泡（图4-56）

- 异常：当一条指令导致异常，我们想要禁止后面的指令更新程序员可见的状态，并且在异常指令到达写回阶段时，停止执行

  <img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171813941.png" alt="image-20220117181348849" style="zoom:80%;" />

  eg

  <img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201171814060.png" alt="image-20220117181426812" style="zoom:80%;" />



触发条件

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201172003307.png" alt="image-20220117200325185" style="zoom:80%;" />



**流水线控制机制**

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201172007992.png" alt="image-20220117200752885" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201172008193.png" alt="image-20220117200846078" style="zoom:80%;" />



<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201172030987.png" alt="image-20220117203005909" style="zoom:80%;" />



**控制条件的组合**

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201172035641.png" alt="image-20220117203519489" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201172035331.png" alt="image-20220117203530236" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201172036132.png" alt="image-20220117203637948" style="zoom:80%;" />

**控制逻辑的实现**

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201172037696.png" alt="image-20220117203714529" style="zoom:80%;" />



我们通过计算PIPE执行一条指令所需的平均时钟周期来衡量系统的性能，该指标为**CPI（Cycles Per Instruction）**。而影响该指标的因素是流水线中插入bubble的数目，因为插入bubble时就会损失一个流水线周期。

我们可以在处理器中运行某个基准程序，然后统计执行阶段中运行的指令数$C_i$和bubble数$C_b$，就得到对应的CPI指标
$$
CPI=\frac{C_i+C_p}{C_i}=1+\frac{C_p}{C_i}
$$


由于只有三种特殊请款（加载/使用冒险、预测错误、`ret`指令）会插入bubble，所以我们可以将惩罚项$\frac{C_b}{C_i}$分为三部分，`lp`表示加载/使用冒险插入bubble的平均数，`mp`表示预测错误插入bubble的平均数，`rp`表示`ret`指令插入bubble的平均数，则CPI可变为
$$
CPI=1+lp+mp+rp
$$


我们可以根据指令出现的频率以及出现特殊情况的频率对 CPI 进行计算

- `mrmovq`和`popq`占所有执行指令的 25%，其中 20% 会导致加载/使用冒险
- 条件分支指令栈所有执行指令的 20%，使用AT策略会有 60% 的成功率
- `ret`指令栈所有执行指令的 2%

由此我们可以得到三种特殊情况的惩罚

<img src="https://raw.githubusercontent.com/Yemonade/imgCloud/main/img/202201172041506.png" alt="image-20220117204124435" style="zoom:80%;" />

### 4.5.10 未完成的工作

#### 1. 多周期指令

我们提供的Y86-64指令集只有简单的操作，在执行阶段都能在一个时钟周期内完成，但是如果要实现整数乘法和除法以及浮点数运算，我们首先要增加额外的硬件来执行这些计算，并且这些指令在执行阶段通常都需要多个时钟周期才能完成，所以执行这些指令时，我们需要平衡流水线各个部分之间的关系。

实现多周期指令的简单方法是直接暂停取指阶段和译码阶段，直到执行阶段执行了所需的时钟周期后才恢复，这种方法的性能通常比较差。

常见的方法是使用独立于主流水线的特殊硬件功能单元来处理复杂的操作，通常会有一个功能单位来处理整数乘法和除法，还有一个功能单位来处理浮点数运算。在译码阶段中遇到多周期指令时，就可以将其发射到对应的功能单元进行运算，而主流水线会继续执行其他指令，使得多周期指令和其他指令能在功能单元和主流水线中并发执行。但是如果不同功能单元以及主流水线的指令存在数据相关时，就需要暂停系统的某部分来解决数据冒险。也同样可以使用暂停、转发以及流水线控制。

#### 2. 与存储系统的接口

我们假设了取指单元和数据内存都能在一个时钟周期内读写内存中的任意位置，但是实际上并不是。

1. 处理器的存储系统是由多种硬件存储器和管理虚拟内存的操作系统共同组成的，而存储系统包含层次结构，最靠近处理器的一层是**高速缓存（Cache）存储器**，能够提供对最常使用的存储器位置的快速访问。典型系统中包含一个用于读指令的cache和一个用于读写数据的cache，并且还有一个**翻译后备缓冲器（Translation Look-aside Buffer，TLB）**来提供从虚拟地址到物理地址的快速翻译。将TLB和cache结合起来，大多数时候能再一个时钟周期内读指令并读写数据。
2. 当我们想要的引用位置不在cache中时，则出现高速缓存**不命中（Miss）**，则流水线会将指令暂停在取指阶段或访存阶段，然后从较高层次的cache或处理器的内存中找到不命中的数据，然后将其保存到cache中，就能恢复指令的运行。这通常需要3~20个时钟周期。
3. 如果我们没有从较高层次的cache或处理器的内存中找到不命中的数据，则需要从磁盘存储器中寻找。硬件会先产生一个**缺页（Page Fault）**异常信号，然后调用操作系统的异常处理程序代码，则操作系统会发起一个从磁盘到主存的传送操作，完成后操作系统会返回原来的程序，然后重新执行导致缺页异常的指令。其中访问磁盘需要数百万个时钟周期，操作系统的缺页中断处理程序需要数百个时钟周期。

#### 3. 当前的微处理器设计

我们采用的五阶段流水线设计，吞吐量都现在最多一个时钟周期执行一条指令，CPI测量值不可能小于1.0。较新的处理器支持**超标量（Superscalar）**操作，能够在取值、译码和执行阶段并行处理多条指令，使得CPI测量值小于1.0，通常会使用IPC（每个周期执行的指令数）来测量性能。更新的处理器会支持**乱序（Out-of-order）**技术，对指令执行顺序进行打乱来并行执行多条指令。



## References

[[读书笔记]CSAPP：13[B]处理器体系结构：流水线 - 深度人工智障的文章 - 知乎](https://zhuanlan.zhihu.com/p/107760564)