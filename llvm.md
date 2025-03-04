# LLVM从入门到入土
## 写在前面
从这一部分开始，我们将正式开始进入**代码生成**阶段。课程组给出了三种目标码，分别是生成到 **`PCode`** ，**`LLVM IR`** ，以及 **`MIPS`** 。写MIPS需要额外进行**代码优化**的操作。这里建议有时间的同学们可以去提前调研一下这几种代码再进行选择。由于理论课上，以及编译原理的教材上主要介绍了**四元式**，且在课本最后也介绍了PCode，所以采用PCode作为目标码的同学可以主要参考**编译原理教材**。

**`LLVM`** 可能看上去上手比较困难，毕竟我相信大部分同学是第一次接触，而在往年的编译原理课程中，LLVM的代码生成是软件学院的课程要求，指导书也是针对往届软件学院的编译原理实验。在2022年与计算机学院合并之后，课程组虽然也添加了LLVM的代码生成通道，但是由于课程合并后，例如**文法，实验过程，实现要求**等的不同，课程组同学在去年实验中收到了许许多多同届同学关于LLVM的问题，包括**看不懂指导书，无从下手**等问题，所以在今年的指导书中，我们将作出以下改进：
- 根据今年的实验顺序，**重新编排**每一个小实验部分的顺序，使得同学们在实验过程中更加顺畅。
- 对每一个部分进行**相关的说明**，帮助同学们更好地理解LLVM的代码生成过程。
- 会在指导书的每一个章节结束给出一些相对**较强的测试样例**，方便同学们做一个部分就测试一个部分。
- 由于LLVM本身就是一种很优秀的中间代码，所以对于想最终生成到MIPS的同学，今年的指导书中将新增LLVM的**中端优化部分**，帮助同学们更方便地从LLVM生成MIPS代码。

## 0. 简单介绍
### LLVM是什么

**`LLVM`** 最早叫底层虚拟机 (Low Level Virtual Machine) ，最初是伊利诺伊大学的一个研究项目，目的是提供一种现代的、基于SSA的编译策略，能够支持任意编程语言的静态和动态编译。从那时起，LLVM已经发展成为一个由多个子项目组成的伞式项目，其中许多子项目被各种各样的商业和开源项目用于生产，并被广泛用于学术研究。

现在，LLVM被用作实现各种静态和运行时编译语言的通用基础设施（例如，GCC、Java、.NET、Python、Ruby、Scheme、Haskell、D以及无数鲜为人知的语言所支持的语言族）。它还取代了各种特殊用途的编译器，如苹果OpenGL堆栈中的运行时专用化引擎和Adobe After Effects产品中的图像处理库。最后，LLVM还被用于创建各种各样的新产品，其中最著名的可能是OpenCL GPU编程语言。

> ~~看不懂不要紧，反正我也是官网上找的介绍~~~
> 
> 一些参考资料：
> - https://aosabook.org/en/v1/llvm.html#footnote-1
> - https://llvm.org/

### 三端设计
传统静态编译器，例如大多数C语言的编译器，最主流的设计是**三端设计**，其主要组件是前端、优化器和后端。前端解析源代码，检查其错误，并构建特定语言的**抽象语法树 `AST`**(Abstract Syntax Tree)来表示输入代码。AST可以选择转换为新的目标码进行优化，优化器和后端在代码上运行。

**`优化器`** 的作用是**增加代码的运行效率**，例如消除冗余计算。 **`后端`** ，也即代码生成器，负责将代码**映射到目标指令集**，其常见部分包括指令选择、寄存器分配和指令调度。

当编译器需要支持**多种源语言或目标体系结构**时，使用这种设计最重要的优点就是，如果编译器在其优化器中使用公共代码表示，那么可以为任何可以编译到它的语言编写前端，也可以为任何能够从它编译的目标编写后端，如图 0-1 所示。

![](image/0-1.png)

##### <p align="center">图 0-1 三端设计示意图</p>

官网对这张设计图的描述只有一个单词 **Retargetablity**，译为**可重定向性**或**可移植性**。通俗理解，如果需要移植编译器以支持新的源语言，只需要实现一个新的前端，但现有的优化器和后端可以重用。如果不将这些部分分开，实现一种新的源语言将需要从头开始，因此，不难发现，支持 $N$ 个目标和 $M$ 种源语言需要 $N×M$ 个编译器，而采用三端设计后，中端优化器可以复用，所以只需要 $N+M$ 个编译器。例如我们熟悉的Java中的 **`JIT`** ， **`GCC`** 都是采用这种设计。

**`IR`** (Intermediate Representation) 的翻译即为中间表示，在基于LLVM的编译器中，前端负责解析、验证和诊断输入代码中的错误，然后将解析的代码转换为LLVM IR，中端（此处即LLVM优化器）对LLVM IR进行优化，后端则负责将LLVM IR转换为目标语言。

### 工具介绍
> 讲道理这一节应该配合下一节一起食用，但是下一节需要用到这些工具，就先放前面写了。

我们的实验目标是将C语言的程序生成为LLVM IR的中间代码，尽管我们会给指导书，但不可避免地，同学们还会遇到很多不会的情况。所以这里给出一个能够自己进行代码生成测试的工具介绍，帮助大家更方便地测试和完成实验。

这里着重介绍 **`Ubuntu`** (20.04或更新) 的下载与操作，一是方便，二是感觉大家或多或少有该系统的Vmware或者云服务器等等。
> 如果真的没有，腾讯云学生优惠有Ubuntu 20.04的云服务器，9.9RMB一年，如果实在不想花钱，也可以在Windows或MacOS上直接装。MacOS和Windows安装Clang和LLVM的方法请自行搜索。

首先安装 **`LLVM`** 和 **`Clang`** 
```bash
$ sudo apt-get install llvm
$ sudo apt-get install clang
```
安装完成后，输入指令查看版本。如果出现版本信息则说明安装成功。
```bash
$ clang -v 
$ lli --version 
```
> **注意：** 请务必保证llvm版本至少是**10.0.0 及以上**，否则**会影响正确性！**

如果使用apt无法安装，则将下列代码加入到 `/etc/apt/sources.list` 文件中
```bash
deb http://apt.llvm.org/focal/ llvm-toolchain-focal-10 main
deb-src http://apt.llvm.org/focal/ llvm-toolchain-focal-10 main
```
然后在终端执行
```bash
wget -O - https://apt.llvm.org/llvm-snapshot.gpg.key|sudo apt-key add -
apt-get install clang-10 lldb-10 lld-10
```
**`MacOS`** 上的安装也稍微提一嘴，需要安装XCode或XCode Command Line Tools, 其默认自带Clang
```bash
xcode-select --install
brew install llvm
```
安装完成后，需要添加LLVM到$PATH
```bash
echo 'export PATH="/usr/local/opt/llvm/bin:$PATH"' >> ~/.bash_profile
```
这时候可以仿照之前查看版本的方法，如果显示版本号则证明安装成功。

我们安装的 **`Clang`** 是 LLVM 项目中 C/C++ 语言的前端，其用法与 GCC 基本相同。
 **`lli`** 会解释.bc 和 .ll 程序。

具体如何使用上述工具链，我们将在下一章介绍。
### LLVM IR示例
LLVM IR 具有三种表示形式，一种是在**内存中**的数据结构格式，一种是在磁盘二进制 **位码 (bitcode)** 格式 **`.bc`** ，一种是**文本格式** **`.ll`** 。生成目标代码为LLVM IR的同学要求输出的是 **`.ll`** 形式的 LLVM IR。

作为一门全新的语言，与其讲过于理论的语法，不如直接看一个实例来得直观，也方便大家快速入门。

例如，我们的源程序 `main.c` 如下
```c
int a=1;
int add(int x,int y){
    return x+y;
}
int main(){
    int b=2;
    return add(a,b);
}
```
现在，我们想知道其对应的LLVM IR长什么样。这时候我们就可以用到Clang工具。下面是一些常用指令
```bash
$ clang main.c -o main # 生成可执行文件
$ clang -ccc-print-phases main.c # 查看编译的过程
$ clang -E -Xclang -dump-tokens main.c # 生成 tokens
$ clang -fsyntax-only -Xclang -ast-dump main.c # 生成语法树
$ clang -S -emit-llvm main.c -o main.ll -O0 # 生成 llvm ir (不开优化)
$ clang -S main.c -o main.s # 生成汇编
$ clang -c main.c -o main.o # 生成目标文件
```
输入 `clang -S -emit-llvm main.c -o main.ll` 后，会在同目录下生成一个 `main.ll` 的文件。在LLVM中，注释以';'打头。
```llvm
; ModuleID = 'main.c'     
source_filename = "main.c"  
target datalayout = "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-pc-linux-gnu"


; 从下一行开始，是实验需要生成的部分，注释不要求生成。
@a = dso_local global i32 1, align 4 

; Function Attrs: noinline nounwind optnone uwtable 
define dso_local i32 @add(i32 %0, i32 %1) #0 {
  %3 = alloca i32, align 4
  %4 = alloca i32, align 4
  store i32 %0, i32* %3, align 4
  store i32 %1, i32* %4, align 4
  %5 = load i32, i32* %3, align 4
  %6 = load i32, i32* %4, align 4
  %7 = add i32 %5, %6
  ret i32 %7
}

; Function Attrs: noinline nounwind optnone uwtable 
define dso_local i32 @main() #0 {
  %1 = alloca i32, align 4
  %2 = alloca i32, align 4
  store i32 0, i32* %1, align 4
  store i32 2, i32* %2, align 4
  %3 = load i32, i32* @a, align 4
  %4 = load i32, i32* %2, align 4
  %5 = call i32 @add(i32 %3, i32 %4)
  ret i32 %5
}

; 实验要求生成的代码到上一行即可

attributes #0 = { noinline nounwind optnone uwtable ...}
; ...是我自己手动改的，因为后面一串太长了

!llvm.module.flags = !{!0}
!llvm.ident = !{!1}

!0 = !{i32 1, !"wchar_size", i32 4}
!1 = !{!"clang version 10.0.0-4ubuntu1 "}
```
用`lli main.ll`解释执行生成的 .ll 文件。如果一切正常，输入`echo $?`查看上一条指令的返回值。

在本次实验中，我们会用到一些库函数。使用的时候请将 <a href="https://github.com/echo17666/BUAA-Compiler2023-llvm-pro/tree/master/files/libsysy.c">`libsysy.c`</a> 和 <a href="https://github.com/echo17666/BUAA-Compiler2023-llvm-pro/tree/master/files/libsysy.h">`libsysy.h`</a>放在同一目录下，对使用到了库函数的源程序进行编译时，需要用到如下指令：

```bash
# 1. 分别导出 libsysy 和 main.c 对应的的 .ll 文件
$ clang -emit-llvm -S libsysy.c -o lib.ll
$ clang -emit-llvm -S main.c -o main.ll

# 2. 使用 llvm-link 将两个文件链接，生成新的 IR 文件
$ llvm-link main.ll lib.ll -S -o out.ll

# 3. 用 lli 解释运行
$ lli out.ll
```

粗略一看，LLVM IR很长很麻烦，但仔细一看，在我们需要生成的代码部分，像是一种特殊的三元式。事实上，LLVM IR使用的是**三地址码**。我们对上述代码进行简要注释。

- Module ID：指明 **`Module`** 的标识
- source_filename：表明该Module是从什么文件编译得到的。如果是通过链接得到的，此处会显示 `llvm-link`
- target datalayout 和 target triple 是程序标签属性说明，和硬件/系统有关。其本身也有一套相应的文法，各个部分说明如下图所示，感兴趣的同学可以自行查阅资料。

![](image/0-2.png)
##### <p align="center">图 0-2 target解释</p>

- `@a = dso_local global i32 1, align 4`：全局变量，名称是a，类型是i32，初始值是1，对齐方式是4字节。dso_local 表明该变量会在同一个链接单元内解析符号。
- `define dso_local i32 @add(i32 %0, i32 %1) #0`：函数定义。其中第一个i32是返回值类型，%add是函数名；第二个和第三个i32是形参类型，%0，%1是形参名。
> llvm中的标识符分为两种类型：全局的和局部的。全局的标识符包括函数名和全局变量，会加一个`@`前缀，局部的标识符会加一个`%`前缀。
- #0指出了函数的`attribute group`。在文件的最后，也能找到对应的attributes #0。因为attribute group可能很包含很多attribute且复用到多个函数，所以我们IR使用attribute group ID(即#0)的形式指明函数的attribute，这样既简洁又清晰。
- 而在大括号中间的函数体，是由一系列 **`BasicBlock`** 组成的。每个BasicBlock都有一个**label**，label使得该BasicBlock有一个符号表的入口点，其以terminator instruction(ret、br等)结尾的。每个BasicBlock由一系列 **`Instruction`** 组成。Instruction是LLVM IR的基本指令。
- %7 = add i32 %5, %6：随便拿上面一条指令来说，%7是Instruction的实例，它的操作数里面有两个值，一个是%5，一个是%6。%5和%6也是Instruction的实例。

下面给出一个Module的主要架构，可以发现，LLVM中几乎所有的结构都可以认为是一个 **`Value`** ，结构与结构之间的Value传递可以简单理解为继承属性和综合属性。而 **`User`** 类和 **`Use`** 类则是LLVM中的重要概念，简单理解就是，User类存储使用Value的列表，而Use类存储Value和User的使用关系，这可以让User和Value快速找到对方。
![](image/0-3.png)
##### <p align="center">图 0-3 LLVM架构简图</p>
这其中架构的设计，是为了方便LLVM的优化和分析，然后根据优化过后的Module生成后端代码。同学们可以根据自己的需要自行设计数据类型。但如果只是想生成到LLVM的同学，这部分内容其实没有那么重要，可以直接面向AST生成代码。

对于一些常用的Instructions，下面给出示例。对于一些没有给出的，可以参考[LLVM IR指令集](https://llvm.org/docs/LangRef.html#instruction-reference)。

| llvm ir | 使用方法 | 简介 |
| ---| --- | --- |
| add | ` <result> = add <ty> <op1>, <op2>` | / |
| sub     | `<result> = sub <ty> <op1>, <op2>` | / |
| mul     | `<result> = mul <ty> <op1>, <op2> ` | / |
| sdiv    | `<result> = sdiv <ty> <op1>, <op2>  ` | 有符号除法 |
| icmp    | `<result> = icmp <cond> <ty> <op1>, <op2>   ` | 比较指令 |
| and     | `<result> = and <ty> <op1>, <op2>  ` | 与 |
| or      | `<result> = or <ty> <op1>, <op2>   `  | 或 |
| call    | `<result> =  call  [ret attrs]  <ty> <fnptrval>(<function args>)` | 函数调用 |
| alloca  | `  <result> = alloca <type> ` | 分配内存 |
| load    | `<result> = load  <ty>, <ty>* <pointer>` | 读取内存 |
| store   | `store  <ty> <value>, <ty>* <pointer>` | 写内存 |
| getelementptr | `<result> = getelementptr <ty>, * {, [inrange] <ty> <idx>}*` <br>  `<result> = getelementptr inbounds <ty>, <ty>* <ptrval>{, [inrange] <ty> <idx>}*` | 计算目标元素的位置（这一章会单独详细说明） |
| phi | `<result> = phi [fast-math-flags] <ty> [ <val0>, <label0>], ...` |/|
| zext..to | `<result> = zext <ty> <value> to <ty2>  ` | 将 `ty`的`value`的type扩充为`ty2` |
| trunc..to | `<result> = trunc <ty> <value> to <ty2>  ` | 将 `ty`的`value`的type缩减为`ty2` |
| br      | `br i1 <cond>, label <iftrue>, label <iffalse>` <br> `br label <dest>  ` | 改变控制流  |
| ret     | `ret <type> <value> `  ,`ret void  ` | 退出当前函数，并返回值 |

### 一些说明
- 这一部分的代码生成的目标，即输入 **`AST`**（或者**四元式**） ，输出一个 **`Module`**（便于继续生成到MIPS） 或直接输出 **`LLVM IR`** 代码。想通过LLVM IR生成到MIPS的同学可以根据对应的 **`Module`** 结构自行存储数据，然后根据Module生成对应的LLVM IR代码自测正确性。
- 目标生成到LLVM语言的同学请注意，clang 默认生成的虚拟寄存器是**按数字顺序**命名的，LLVM 限制了所有数字命名的虚拟寄存器必须严格地**从 0 开始递增**，且每个函数**参数和基本块**都会占用一个编号。如果不能确定怎样用数字命名虚拟寄存器，请使用**字符串命名**虚拟寄存器。
- 本章主要讲述通过 **`AST`** 生成对应的代码，这也是LLVM官网的默认推荐逻辑，当然同学们也可以通过自行设计的**四元式**生成LLVM。本质上LLVM其实是个**三地址码**，其指令其实就是一个**四元组**（结果，运算符，操作数1，操作数2，例如%3=add i32 %1, %2）不难发现，这其实就是**四元式**。所以从四元式生成LLVM可以直接一步到位，本章也就不作过多赘述。
## 1. 主函数与常量表达式
### 主函数
首先我们从最基本的开始，即只包含return语句的主函数（或没有参数的函数）。可能用到的文法包括
```c
CompUnit    → MainFuncDef
MainFuncDef → 'int' 'main' '(' ')' Block
Block       → '{' { BlockItem } '}' 
BlockItem   → Stmt
Stmt        → 'return' Exp ';' 
Exp         → AddExp
AddExp      → MulExp 
MulExp      → UnaryExp 
UnaryExp    → PrimaryExp
PrimaryExp  → Number
```
对于一个无参的函数，首先需要从AST获取函数的名称，返回值类型。然后分析函数体的Block。Block中的Stmt可以是return语句，也可以是其他语句，但是这里只考虑return语句。return语句中的Number在现在默认是**常数**。
所以对于一个代码生成器，我们需要实现的功能有：
- 遍历**AST**，遍历到函数时，获取函数的**名称**、**返回值类型**
- 遍历到**Block**内的**Stmt**时，如果是**return**语句，生成对应的**指令**
### 常量表达式
新增内容有
```c
Stmt       → 'return' [Exp] ';' 
Exp        → AddExp
AddExp     → MulExp | AddExp ('+' | '−') MulExp 
MulExp     → UnaryExp | MulExp ('*' | '/' | '%') UnaryExp
UnaryExp   → PrimaryExp | UnaryOp UnaryExp 
PrimaryExp → '(' Exp ')' | Number
UnaryOp    → '+' | '−'
``` 
对于常量表达式，这里只包含常数的四则运算，正负号操作。这时候我们就需要用到之前的Value思想。举个例子，对于 `1+2+3*4`，我们生成的AST样式如下
![](image/1-1.png)
##### <p align="center">图 1-1 简单四则运算AST参考图</p>
那么在生成的时候，我们的顺序是从左到右，从上到下。所以我们可以先生成 `1`，然后生成 `2`，然后生成 `1+2`，然后生成 `3`，然后生成 `4`，然后生成 `3*4`，最后生成 `1+2+3*4`。那对于1+2的**AddExp**，在其生成的指令中，1和2的值就类似于综合属性，即从AddExp的实例的值（3）由产生式右边的值（1和2）推导出来。而对于3\*4的**MulExp**，其生成的指令中3和4的值就类似于继承属性，即从MulExp的实例的值（12）由产生式左边的值（3和4）推导出来。最后，对于1+2+3\*4的**AddExp**，生成指令的实例的值就由产生式右边的AddExp的值（3）和MulExp的值（12）推导出来。

同理，对于数字前的正负，我们可以看做是**0和其做一次AddExp**，即+1其实就是0+1 （其实正号甚至都不用去管） ，-1其实就是0-1。所以在生成代码的时候，可以当作一个特殊的AddExp来处理。

> 特别注意：`MulExp → UnaryExp | MulExp ('*' | '/' | '%') UnaryExp`运算中，`%`模运算的代码生成不是mod哦
### 测试样例
源程序
```c
int main() {
    return --+---+1 * +2 * ---++3 + 4 + --+++5 + 6 + ---+--7 * 8 + ----++--9 * ++++++10 * -----11;
}
```
生成代码参考（因为最后的代码是扔到评测机上重新编译去跑的，所以生成的代码不一定要一样，但是要确保输出结果一致）
```llvm
define dso_local i32 @main() {
    %1 = sub i32 0, 1
    %2 = sub i32 0, %1
    %3 = sub i32 0, %2
    %4 = sub i32 0, %3
    %5 = sub i32 0, %4
    %6 = mul i32 %5, 2
    %7 = sub i32 0, 3
    %8 = sub i32 0, %7
    %9 = sub i32 0, %8
    %10 = mul i32 %6, %9
    %11 = add i32 %10, 4
    %12 = sub i32 0, 5
    %13 = sub i32 0, %12
    %14 = add i32 %11, %13
    %15 = add i32 %14, 6
    %16 = sub i32 0, 7
    %17 = sub i32 0, %16
    %18 = sub i32 0, %17
    %19 = sub i32 0, %18
    %20 = sub i32 0, %19
    %21 = mul i32 %20, 8
    %22 = add i32 %15, %21
    %23 = sub i32 0, 9
    %24 = sub i32 0, %23
    %25 = sub i32 0, %24
    %26 = sub i32 0, %25
    %27 = sub i32 0, %26
    %28 = sub i32 0, %27
    %29 = mul i32 %28, 10
    %30 = sub i32 0, 11
    %31 = sub i32 0, %30
    %32 = sub i32 0, %31
    %33 = sub i32 0, %32
    %34 = sub i32 0, %33
    %35 = mul i32 %29, %34
    %36 = add i32 %22, %35
    ret i32 %36
}
```
## 2. 全局变量与局部变量

### 全局变量
本章实验涉及的文法包括：
```c
CompUnit     → {Decl} MainFuncDef
Decl         → ConstDecl | VarDecl
ConstDecl    → 'const' BType ConstDef { ',' ConstDef } ';' 
BType        → 'int' 
ConstDef     → Ident  '=' ConstInitVal
ConstInitVal → ConstExp
ConstExp     → AddExp
VarDecl      → BType VarDef { ',' VarDef } ';'
VarDef       → Ident | Ident '=' InitVal
InitVal      → Exp
```
在llvm中，全局变量使用的是和函数一样的全局标识符 `@` ，所以全局变量的写法其实和函数的定义几乎一样。在我们的实验中，全局变/常量声明中指定的初值表达式必须是**常量表达式**。不妨举几个例子：
```c
//以下都是全局变量
int a=5;
int b=2+3;
```
生成的llvm如下所示
```llvm
@a = dso_local global i32 5
@b = dso_local global i32 5
```
可以看到，对于全局变量中的常量表达式，在生成的llvm中我们需要算出其**具体的值**。

### 局部变量
本章内容涉及文法包括：
```c
BlockItem → Decl | Stmt
```
局部变量使用的标识符是 `%` 。与全局变量不同，局部变量在赋值前需要申请一块内存。在对局部变量操作的时候，我们也需要采用**load/store**来对内存进行操作。
同样的，我们举个例子来说明一下：
```c
//以下都是局部变量
int a=1+2;
```
生成的llvm如下所示
```llvm
%1 = alloca i32
%2 = add i32 1, 2
store i32 %2, i32* %1
```

### 符号表设计与作用域
这一章我们将主要考虑变量，包括全局变量和局部变量以及作用域的说明。不可避免地，我们需要进行符号表的设计。

涉及到的文法如下：
```c
Stmt       → LVal '=' Exp ';' 
           | [Exp] ';'
           | 'return' Exp ';'
LVal       → Ident
PrimaryExp → '(' Exp ')' | LVal | Number
```
我们举个最简单的例子：
```c
int a=1;
int b=2+a;
int main(){
  int c=b+4;
  return a+b+c;
}
```

如果我们需要将上述代码转换为llvm，我们应当怎么考虑呢？直观来看，a和b是**全局变量**，c是**局部变量**。我们首先将全局变量a和b进行赋值，然后到main函数内部，我们对c进行赋值。那么在 `return a+b+c;` 的时候，根据上个实验，llvm最后几行应该是
```llvm
%sumab = add i32 %a, %b
%sumabc = add i32 %sumab, %c
ret i32 %sumabc
```
问题就是，我们如何获取标识符 `%a，%b，%c`，这时候我们的符号表的作用就体现出来了。简单来说，符号表类似于一个索引。通过符号表，我们可以很快速的找到变量对应的标识符。

对于上面的c语言程序，llvm生成如下：
```llvm
@a = dso_local global i32 1
@b = dso_local global i32 3
define dso_local i32 @main() {
    %1 = alloca i32          ;分配c内存
    %2 = load i32, i32* @b   ;读取全局变量b
    %3 = add i32 %2, 4       ;计算b+4
    store i32 %3, i32* %1    ;把b+4的值存入c
    %4 = load i32, i32* @a   ;读取全局变量a
    %5 = load i32, i32* @b   ;读取全局变量b
    %6 = add i32 %4, %5      ;计算a+b;
    %7 = load i32, i32* %1   ;读取c
    %8 = add i32 %6, %7      ;计算(a+b)+c
    ret i32 %8               ;return
}
```
不难发现，对于全局变量的使用，可以直接使用全局变量的全局标识符（例如@a），而对于局部变量，我们则需要使用分配内存的标识符。由于标识符是自增的数字，所以快速找到对应变量的标识符就是符号表最重要的作用。同学们可以选择遍历一遍AST后造出一张统一的符号表，然后根据完整的符号表进行代码生成，也可以在遍历AST的同时造出一张栈式符号表，根据实时的栈式符号表生成相应代码。符号表存储的东西同学们可以自己设计，下面给出符号表的简略示例，同学们在实验中可以根据自己需要自行设计。

同时，同学们需要注意变量的作用域，即语句块内声明的变量的生命周期在该语句块内，且内层代码块覆盖外层代码块。
```c
int a=1;
int b=2;
int c=3;
int main(){
  int d=4;
  int e=5;
  {//blockA
    int a=7;
    int e=8;
    int f=9;
  }
  int f=10;
}
```
在上面的程序中，在**blockA**中，a的值为7，覆盖了全局变量a=1，e覆盖了main中的e=5，而在main的最后一行，f并不存在覆盖，因为main外层不存在其他f的定义。

同样的，下面给出上述程序的llvm代码：
```llvm
@a = dso_local global i32 1
@b = dso_local global i32 2
@c = dso_local global i32 3

define dso_local i32 @main() {
    %1 = alloca i32
    store i32 4, i32* %1
    %2 = alloca i32
    store i32 5, i32* %2
    %3 = alloca i32
    store i32 7, i32* %3
    %4 = alloca i32
    store i32 8, i32* %4
    %5 = alloca i32
    store i32 9, i32* %5
    %6 = alloca i32
    store i32 10, i32* %6
}
```
上述程序的符号表简略示意如下：

![](image/2-1.png)
##### <p align="center">图 2-1 完整符号表与栈式符号表示意图</p>

### 测试样例
源程序
```c
int a=1;
int b=2+a;
int c=3*(b+------10);
int main(){
  int d=4+c;
  int e=5*d;
  {
    a=a+5;
    int b=a*2;
    a=b;
    int f=20;
    e=e+a*20;
  }
  int f=10;
  return e*f;
}
```
llvm参考如下：
```llvm
@a = dso_local global i32 1
@b = dso_local global i32 3
@c = dso_local global i32 39
define dso_local i32 @main() {
    %1 = alloca i32
    %2 = load i32, i32* @c
    %3 = add i32 4, %2
    store i32 %3, i32* %1
    %4 = alloca i32
    %5 = load i32, i32* %1
    %6 = mul i32 5, %5
    store i32 %6, i32* %4
    %7 = load i32, i32* @a
    %8 = load i32, i32* @a
    %9 = add i32 %8, 5
    store i32 %9, i32* @a
    %10 = alloca i32
    %11 = load i32, i32* @a
    %12 = mul i32 %11, 2
    store i32 %12, i32* %10
    %13 = load i32, i32* @a
    %14 = load i32, i32* %10
    store i32 %14, i32* @a
    %15 = alloca i32
    store i32 20, i32* %15
    %16 = load i32, i32* %4
    %17 = load i32, i32* %4
    %18 = load i32, i32* @a
    %19 = mul i32 %18, 20
    %20 = add i32 %17, %19
    store i32 %20, i32* %4
    %21 = alloca i32
    store i32 10, i32* %21
    %22 = load i32, i32* %4
    %23 = load i32, i32* %21
    %24 = mul i32 %22, %23
    ret i32 %24
}

```
echo $?的结果为**198**

> 我相信各位如果去手动计算的话，会算出来结果是4550。然而由于echo $?的返回值只截取最后一个字节，也就是8位，所以 `4550 mod 256 = 198`

###### 废话，这种例子当然是随便编的awa
## 3. 函数的定义及调用
> 本章主要涉及**不含数组**的函数的定义，调用等。
### 库函数
涉及文法有：
```c
Stmt → LVal '=' 'getint''('')'';'
     | 'printf''('FormatString{','Exp}')'';'
```


首先我们添加**库函数**的调用。在实验中，我们的llvm代码中，库函数的声明如下：
```llvm
declare i32 @getint()
declare void @putint(i32)
declare void @putch(i32)
declare void @putstr(i8*)
```
只要在llvm代码开头加上这些声明，就可以在后续代码中使用这些库函数。同时对于用到库函数的llvm代码，我们在编译时也需要使用llvm-link命令将库函数链接到我们的代码中。

对于库函数的使用，在我们的文法中其实就包含两句，即`getint`和`printf`。其中，`printf`包含了有Exp和没有Exp的情况。同样的，我们给出一个简单的例子：
```c
int main(){
  int a;
  a=getint();
  printf("hello:%d",a);
  return 0;
}
```
llvm代码如下：
```llvm
declare i32 @getint()
declare void @putint(i32)
declare void @putch(i32)
declare void @putstr(i8*)

define dso_local i32 @main() {
    %1 = alloca i32
    %2 = load i32, i32* %1
    %3 = call i32 @getint() 
    store i32 %3, i32* %1
    %4 = load i32, i32* %1
    call void @putch(i32 104)
    call void @putch(i32 101)
    call void @putch(i32 108)
    call void @putch(i32 108)
    call void @putch(i32 111)
    call void @putch(i32 58)
    call void @putint(i32 %4)
    ret i32 0
}
```
不难看出，`call i32 @getint()` 即为调用getint的语句，对于其他的任何函数的调用也是像这样去写。而对于`printf`，我们需要将其转化为多条`putch`和`putint`的调用，或者使用`putstr`以字符串输出。这里需要注意的是，`putch`和`putint`的参数都是 `i32` 类型，所以我们需要将字符串中的字符转化为对应的ascii码。

### 函数定义与调用
涉及文法如下：
```c
CompUnit    → {Decl} {FuncDef} MainFuncDef
FuncDef     → FuncType Ident '(' [FuncFParams] ')' Block 
FuncType    → 'void' | 'int'
FuncFParams → FuncFParam { ',' FuncFParam }
FuncFParam  → BType Ident
UnaryExp    → PrimaryExp | Ident '(' [FuncRParams] ')' | UnaryOp UnaryExp 
```
其实之前的main函数也是一个函数，即主函数。这里我们将其拓广到一般函数。对于一个函数，其特征包括**函数名**，**函数返回类型**和**参数**。在本实验中，函数返回类型只有 **`int`** 和 **`void`** 两种。由于目前只有零维整数作为参数，所以参数的类型统一都是`i32`。FuncFParams之后的Block则与之前主函数内处理方法一样。值得一提的是，由于每个**临时寄存器**和**基本块**占用一个编号，所以没有参数的函数的第一个临时寄存器的编号应该从**1**开始，因为函数体入口占用了一个编号0。而有参数的函数，参数编号从**0**开始，进入Block后需要跳过一个基本块入口的编号（可以参考测试样例）。

当然，如果全部采用字符串编号寄存器，上述问题都不会存在。

对于函数的调用，参考之前库函数的处理，不难发现，函数的调用其实和**全局变量**的调用基本是一样的，即用`@函数名`表示。所以函数部分和**符号表**有着密切关联。同学们需要在函数定义和函数调用的时候对符号表进行操作。对于有参数的函数调用，则在调用的函数内传入参数。对于没有返回值的函数，则直接`call`即可，不用为语句赋一个实例。

### 测试样例
源代码：
```c
int a=1000;
int aaa(int a,int b){
        return a+b;
}
void ab(){
        a=1200;
        return;
}
int main(){
        ab();
        int b=a,a;
        a=getint();
        printf("%d",aaa(a,b));
        return 0;
}
```
llvm输出参考：
```llvm
declare i32 @getint()
declare void @putint(i32)
declare void @putch(i32)
declare void @putstr(i8*)
@a = dso_local global i32 1000
define dso_local i32 @aaa(i32 %0, i32 %1){
  %3 = alloca i32
  %4 = alloca i32
  store i32 %0, i32* %3
  store i32 %1, i32* %4
  %5 = load i32, i32* %3
  %6 = load i32, i32* %4
  %7 = add nsw i32 %5, %6
  ret i32 %7
}
define dso_local void @ab(){
  store i32 1200, i32* @a
  ret void
}

define dso_local i32 @main(){
  %1 = alloca i32
  %2 = alloca i32
  call void @ab()
  %3 = load i32, i32* @a
  store i32 %3, i32* %1
  %4 = call i32 @getint()
  store i32 %4, i32* %2
  %5 = load i32, i32* %2
  %6 = load i32, i32* %1
  %7 = call i32 @aaa(i32 %5, i32 %6)
  call void @putint(i32 %7)
  ret i32 0
}
```
- 输入：1000
- 输出：2200
## 4. 条件语句与短路求值
### 条件语句
涉及文法如下
```c
Stmt    → 'if' '(' Cond ')' Stmt [ 'else' Stmt ]
Cond    → LOrExp
RelExp  → AddExp | RelExp ('<' | '>' | '<=' | '>=') AddExp
EqExp   → RelExp | EqExp ('==' | '!=') RelExp
LAndExp → EqExp | LAndExp '&&' EqExp
LOrExp  → LAndExp | LOrExp '||' LAndExp 
```
在条件语法中，我们需要进行条件的判断与选择。这时候就涉及到基本块的标号。在llvm中，每个**临时寄存器**和**基本块**占用一个编号。所以对于纯数字编号的llvm，这里就需要进行**回填**操作。对于在代码生成前已经完整生成符号表的同学，这里就会显得十分容易。对于在代码生成同时生成符号表的同学，也可以采用**栈**的方式去回填编号，对于采用字符串编号的则没有任何要求。

要写出条件语句，首先要理清楚逻辑。在上述文法中，最重要的莫过于下面这一条语法
```c
Stmt    → 'if' '(' Cond ')' Stmt1 [ 'else' Stmt2 ]  (BasicBlock3)
```
为了方便说明，对上述文法的两个Stmt编号为Stmt1和2。在这条语句之后基本块假设叫BasicBlock3。不难发现，条件判断的逻辑如左下图。
![](image/4-1.png)
##### <p align="center">图 4-1 条件判断流程示意图与基本块流图</p>

首先进行Cond结果的判断，如果结果为**1**则进入**Stmt1**，如果Cond结果为**0**，若文法有else则将进入**Stmt2**，否则进入下一条文法的基本块**BasicBlock3**。在Stmt1或Stmt2执行完成后都需要跳转到BasicBlock3。对于一个llvm程序来说，对一个含else的条件分支，其基本块构造可以如右上图所示。

如果能够理清楚基本块跳转的逻辑，那么在写代码的时候就会变得十分简单。

这时候我们再回过头去看Cond里面的代码，即LOr和Land，Eq和Rel。不难发现，其处理方式和加减乘除非常像，除了运算结果都是1位(i1)而非32位(i32)。同学们可能需要用到 `trunc`或者 `zext` 指令进行类型转换。

### 短路求值
可能有的同学会认为，反正对于llvm来说，跳转与否只看Cond的值，所以我只要把Cond算完结果就行，不会影响正确性。不妨看一下下面这个例子：
```c
int a=5;
int change(){
  a=6;
  return a;
}
int main(){
  if(1||change()){
    printf("%d",a);
  }
  return 0;
}
```
如果要将上面这段代码翻译为llvm，同学们会怎么做？如果按照传统方法，即先**统一计算Cond**，则一定会执行一次 **`change()`** 函数，把全局变量的值变为**6**。但事实上，由于短路求值的存在，在读完1后，整个Cond的值就**已经被确定**了，即无论`1||`后面跟的是什么，都不影响Cond的结果，那么根据短路求值，后面的东西就不应该执行。所以上述代码的输出应当为**5**而不是6，也就是说，我们的llvm不能够单纯的把Cond计算完后再进行跳转。这时候我们就需要对Cond的跳转逻辑进行改写。

改写之前我们不妨思考一个问题，即什么时候跳转。根据短路求值，只要条件判断出现“短路”，即不需要考虑后续与或参数的情况下就已经能确定值的时候，就可以进行跳转。或者更简单的来说，当**LOrExp值为1**或者**LAndExp值为0**的时候，就已经没有必要再进行计算了。
```c
Cond    → LOrExp
LAndExp → LAndExp '&&' EqExp
LOrExp  → LOrExp '||' LAndExp 
```
- 对于连或来说，只要其中一个LOrExp或最后一个LAndExp为1，即可直接跳转Stmt1。
- 对于连与来说，只要其中一个LAndExp或最后一个EqExp为0，则直接进入下一个LOrExp。如果当前为连或的最后一项，则直接跳转Stmt2（有else）或BasicBlock3（没else）

上述两条规则即为短路求值的最核心算法，示意图如下。
![](image/4-2.png)
##### <p align="center">图 4-2 短路求值算法示意图</p>

### 测试样例
```c
int a=1;
int func(){
    a=2;return 1;
}

int func2(){
    a=4;return 10;
}
int func3(){
    a=3;return 0;
}
int main(){
    if(0||func()&&func3()||func2()){printf("%d--1",a);}
    if(1||func3()){printf("%d--2",a);}
    if(0||func3()||func()<func2()){printf("%d--3",a);}
    return 0;
}
```
```llvm
declare i32 @getint()
declare void @putint(i32)
declare void @putch(i32)
declare void @putstr(i8*)
@a = dso_local global i32 1
define dso_local i32 @func() {
    %1 = load i32, i32* @a
    store i32 2, i32* @a
    ret i32 1
}
define dso_local i32 @func2() {
    %1 = load i32, i32* @a
    store i32 4, i32* @a
    ret i32 10
}
define dso_local i32 @func3() {
    %1 = load i32, i32* @a
    store i32 3, i32* @a
    ret i32 0
}

define dso_local i32 @main() {
    br label %1

1:
    %2 = icmp ne i32 0, 0
    br i1 %2, label %12, label %3

3:
    %4 = call i32 @func()
    %5 = icmp ne i32 0, %4
    br i1 %5, label %6, label %9

6:
    %7 = call i32 @func3()
    %8 = icmp ne i32 0, %7
    br i1 %8, label %12, label %9

9:
    %10 = call i32 @func2()
    %11 = icmp ne i32 0, %10
    br i1 %11, label %12, label %14

12:
    %13 = load i32, i32* @a
    call void @putint(i32 %13)
    call void @putch(i32 45)
    call void @putch(i32 45)
    call void @putch(i32 49)
    br label %14

14:
    br label %15

15:
    %16 = icmp ne i32 0, 1
    br i1 %16, label %20, label %17

17:
    %18 = call i32 @func3()
    %19 = icmp ne i32 0, %18
    br i1 %19, label %20, label %22

20:
    %21 = load i32, i32* @a
    call void @putint(i32 %21)
    call void @putch(i32 45)
    call void @putch(i32 45)
    call void @putch(i32 50)
    br label %22

22:
    br label %23

23:
    %24 = icmp ne i32 0, 0
    br i1 %24, label %34, label %25

25:
    %26 = call i32 @func3()
    %27 = icmp ne i32 0, %26
    br i1 %27, label %34, label %28

28:
    %29 = call i32 @func()
    %30 = call i32 @func2()
    %31 = icmp slt i32 %29, %30
    %32 = zext i1 %31 to i32
    %33 = icmp ne i32 0, %32
    br i1 %33, label %34, label %36

34:
    %35 = load i32, i32* @a
    call void @putint(i32 %35)
    call void @putch(i32 45)
    call void @putch(i32 45)
    call void @putch(i32 51)
    br label %36

36:
    ret i32 0
}

```
> 注：由于历史遗留问题，跳转参考代码较乱，参考价值不大，同学们可以自行设计，仅需要保证短路求值正确性即可。
- 输出：4--14--24--3
## 5. 循环与中断
### 循环
涉及文法如下：
```c
Stmt    → 'for' '(' [ForStmt] ';' [Cond] ';' [ForStmt] ')' Stmt 
        | 'break' ';'
        | 'continue' ';'
ForStmt → LVal '=' Exp
```
如果经过了上一章的学习，这一章其实难度就小了不少。对于这条文法，同样可以改写为
```c
Stmt    → 'for' '(' [ForStmt1] ';' [Cond] ';' [ForStmt2] ')' Stmt (BasicBlock)
```
如果查询C语言的for循环，其中对for循环的描述为：
```c
for(initialization;condition;incr/decr){  
   //code to be executed  
}
```
不难发现，实验文法中的ForStmt1，Cond，ForStmt2分别表示了上述for循环中的**初始化(initialization)**，**条件(condition)**和**增量/减量(increment/decrement)**。同学们去搜索C语言的for循环逻辑的话也会发现，for循环的逻辑可以表述为
- 1.执行初始化表达式ForStmt1
- 2.执行条件表达式Cond，如果为1执行循环体Stmt，否则结束循环执行BasicBlock
- 3.执行完循环体Stmt后执行增量/减量表达式ForStmt2
- 4.重复执行步骤2和步骤3

![](image/5-1.png)

##### <p align="center">图 5-1 for循环流程图</p>
### break/continue
对于`break`和`continue`，直观理解为，break**跳出循环**，continue**跳过本次循环**。再通俗点说就是，break跳转到的是**BasicBlock**，而continue跳转到的是**ForStmt2**。这样就能达到目的了。所以，对于循环而言，跳转的位置很重要。这也是同学们在编码的时候需要着重注意的点。

同样的，针对这两条指令，对上图作出一定的修改，就是整个循环的流程图了。

![](image/5-2.png)

##### <p align="center">图 5-2 for循环完整流程图</p>
### 测试样例
```c
int main(){    
    int a1=1,a2;    
    a2=a1;    
    int temp; 
    int n,i;
    n=getint();   
    for(i=a1*a1;i<n+1;i=i+1){     
        temp=a2;
        a2=a1+a2;
        a1=temp;
        if(i%2==1){
            continue;
        }
        printf("round %d: %d\n",i,a1);
        if(i>19){
            break;
        }
    }   
    return 0;    
}
```
```llvm
declare i32 @getint()
declare void @putint(i32)
declare void @putch(i32)
declare void @putstr(i8*)
define dso_local i32 @main() {
    %1 = alloca i32
    store i32 1, i32* %1
    %2 = alloca i32
    %3 = load i32, i32* %1
    store i32 %3, i32* %2
    %4 = alloca i32
    %5 = alloca i32
    %6 = alloca i32
    %7 = call i32 @getint()
    store i32 %7, i32* %5
    %8 = load i32, i32* %1
    %9 = load i32, i32* %1
    %10 = mul i32 %8, %9
    store i32 %10, i32* %6
    br label %11

11:
    %12 = load i32, i32* %6
    %13 = load i32, i32* %5
    %14 = add i32 %13, 1
    %15 = icmp slt i32 %12, %14
    %16 = zext i1 %15 to i32
    %17 = icmp ne i32 0, %16
    br i1 %17, label %18, label %44

18:
    %19 = load i32, i32* %2
    store i32 %19, i32* %4
    %20 = load i32, i32* %2
    %21 = load i32, i32* %1
    %22 = load i32, i32* %2
    %23 = add i32 %21, %22
    store i32 %23, i32* %2
    %24 = load i32, i32* %4
    store i32 %24, i32* %1
    br label %25

25:
    %26 = load i32, i32* %6
    %27 = srem i32 %26, 2
    %28 = icmp eq i32 %27, 1
    %29 = zext i1 %28 to i32
    %30 = icmp ne i32 0, %29
    br i1 %30, label %31, label %32

31:
    br label %41

32:
    %33 = load i32, i32* %6
    %34 = load i32, i32* %1
    call void @putch(i32 114)
    call void @putch(i32 111)
    call void @putch(i32 117)
    call void @putch(i32 110)
    call void @putch(i32 100)
    call void @putch(i32 32)
    call void @putint(i32 %33)
    call void @putch(i32 58)
    call void @putch(i32 32)
    call void @putint(i32 %34)
    call void @putch(i32 10)
    br label %35

35:
    %36 = load i32, i32* %6
    %37 = icmp sgt i32 %36, 19
    %38 = zext i1 %37 to i32
    %39 = icmp ne i32 0, %38
    br i1 %39, label %40, label %41

40:
    br label %44

41:
    %42 = load i32, i32* %6
    %43 = add i32 %42, 1
    store i32 %43, i32* %6
    br label %11

44:
    ret i32 0
}
```
- 输入：10
- 输出：
                
        round 2: 2
        round 4: 5
        round 6: 13
        round 8: 34
        round 10: 89
- 输入：40
- 输出：
- 
        round 2: 2
        round 4: 5
        round 6: 13
        round 8: 34
        round 10: 89
        round 12: 233
        round 14: 610
        round 16: 1597
        round 18: 4181
        round 20: 10946
> 本质为一个只输出20以内偶数项的斐波那契数列
## 6. 数组与函数
### 数组
数组涉及的文法相当多，包括以下几条：
```c
ConstDef     → Ident { '[' ConstExp ']' } '=' ConstInitVal
ConstInitVal → ConstExp | '{' [ ConstInitVal { ',' ConstInitVal } ] '}'
VarDef       → Ident { '[' ConstExp ']' } | Ident { '[' ConstExp ']' } '=' InitVal
InitVal      → Exp | '{' [ InitVal { ',' InitVal } ] '}'
FuncFParam   → BType Ident ['[' ']' { '[' ConstExp ']' }]
LVal         → Ident {'[' Exp ']'}
```
在数组的编写中，同学们会频繁用到 **`getElementPtr`** 指令，故先系统介绍一下这个指令的用法。

getElementPtr指令的工作是计算地址。其本身不对数据做任何访问与修改。其语法如下：
```llvm
<result> = getelementptr <ty>, <ty>* <ptrval>, {<ty> <index>}*
```
现在我们来理解一下上面这一条指令。第一个 `<ty>` 表示的是第一个索引所指向的类型，有时也是**返回值的类型**。第二个 `<ty>` 表示的是后面的指针基地址 `<ptrval>` 的类型， `<ty> <index>` 表示的是一组索引的类型和值，在本实验中索引的类型为i32。索引指向的基本类型确定的是增加索引值时指针的偏移量。

说完理论，我们结合一个实例来讲解。考虑数组 **`a[5][7]`**，需要获取 **`a[3][4]`** 的地址我们有如下写法：
```llvm
%1 = getelementptr [5 x [7 x i32]], [5 x [7 x i32]]* @a, i32 0, i32 3
%2 = getelementptr [7 x i32], [7 x i32]* %1, i32 0, i32 4

%3 = getelementptr [5 x [7 x i32]], [5 x [7 x i32]]* @a, i32 0, i32 3, i32 4

%4 = getelementptr [5 x [7 x i32]], [5 x [7 x i32]]* @a, i32 0, i32 0
%5 = getelementptr [7 x i32], [7 x i32]* @4, i32 3, i32 0,
%6 = getelementptr i32, i32* %5, i32 4
```

![](image/6-1.png)

##### <p align="center">图 6-1 getElementPtr示意图</p>

对于 **`%6`** ，其只有一组索引**i32 4**，所以索引使用基本类型为**i32**，基地址为 **`%5`** ，索引值为4，所以指针相对于%5（21号格子）前进了**4个i32**类型，即指向了25号格子，返回类型为i32*。

而当存在多组索引值的时候，每多一组索引值，索引使用基本类型就要去掉一层。再拿上面举个例子，对于 **`%1`** ，基地址为@a，类型为 **[5 x [7 x i32]]** 。第一组索引值为**0**，所以指针前进0个0x5x7个i32，第二组索引值为**3**，这时候就要**去掉一层**，即把[5 x [7 x i32]]中的5去掉，即索引使用的基本类型为 **[7 x i32]** ，指针向前移动3x7个i32。返回类型为 **[7 x i32]\***，而对于 **`%2`** ，第一组索引值为**0**，其首先前进了0x7个i32，第二组索引值为**4**，去掉一层，索引基本类型为i32，指针向前移动4个i32，指向25号格子

当然，可以一步到位，如 **`%3`** ，后面跟了三组索引值，第一组让指针前进0x5x7个i32，第二组让指针前进3x7个i32，第三组让指针前进4个i32，索引使用基本类型去掉两层，为**i32\***。

对于 **`%4`** ，虽然其两组索引值都是0，但是其索引使用基本类型去掉了一层，变为了 **[7 x i32]** 。在 **`%5`** 的时候，第一组索引值为3，即指针前进3x7个i32，第二组索引值为0，即指针前进0个i32，索引使用基本类型变为i32，返回指针类型为 **i32\***。

当然，同学们也可以直接将所有高维数组模拟为1维数组，例如对于**a[5][7]**中取**a[3][4]**，可以直接将a转换为一个**a[35]**，然后指针偏移7x3+4=25，直接取**a[25]**。
### 数组定义与调用
这一章我们将主要讲述数组定义和调用，包括全局数组，局部数组的定义，以及函数中的数组调用。对于全局数组定义，与全局变量一样，我们需要将所有量**全部计算到特定的值**。同时，对于全局数组，对数组中空缺的值，需要**置0**。对于全是0的地方，可以采用 **`zeroinitializer`** 来统一置0。
```c
int a[1+2+3+4]={1,1+1,1+3-1};
int b[10][20];
int c[5][5]={{1,2,3},{1,2,3,4,5}};
```
```llvm
@a = dso_local global [10 x i32] [i32 1, i32 2, i32 3, i32 0, i32 0, i32 0, i32 0, i32 0, i32 0, i32 0]
@b = dso_local global [10 x [20 x i32]] zeroinitializer
@c = dso_local global [5 x [5 x i32]] [[5 x i32] [i32 1, i32 2, i32 3, i32 0, i32 0], [5 x i32] [i32 1, i32 2, i32 3, i32 4, i32 5], [5 x i32] zeroinitializer, [5 x i32] zeroinitializer, [5 x i32] zeroinitializer]
```
当然，zeroinitializer不是必须的，同学们完全可以一个个**i32 0**写进去，但对于一些很阴间的样例点，不用zeroinitializer可能会导致 **`TLE`** ，例如全局数组 `int a[1000];` ，不使用该指令就需要输出**1000次i32 0**，必然导致TLE，所以还是推荐同学们使用zeroinitializer。

对于局部数组，在定义的时候同样需要使用`alloca`指令，其存取指令同样采用**load和store**，只是在此之前需要采用`getelementptr`获取数组内应位置的地址。

对于数组传参，其中涉及到维数的变化问题，例如，对于参数中**含维度的数组**，同学们可以参考上述`getelementptr`指令自行设计，因为该指令很灵活，所以下面的测试样例仅仅当一个参考。同学们可以将自己生成的llvm使用**lli**编译后自行查看输出比对。
### 测试样例
```c
int a[3+3]={1,2,3,4,5,6};
int b[3][3]={{3,6+2,5},{1,2}};
void a1(int x){
    if(x>1){
        a1(x-1);
    }
    return;
}
int a2(int x,int y[]){
    return x+y[2];
}
int a3(int x,int y[],int z[][3]){
    return x*y[1]-z[2][1];
}
int main(){
    int c[2][3]={{1,2,3}};
    a1(c[0][2]);
    int x=a2(a[4],a);
    int y=a3(b[0][1],b[1],b);
    printf("%d",x+y);
    return 0;
}
```
```llvm
declare i32 @getint()
declare void @putint(i32)
declare void @putch(i32)
declare void @putstr(i8*)
@a = dso_local global [6 x i32] [i32 1, i32 2, i32 3, i32 4, i32 5, i32 6]
@b = dso_local global [3 x [3 x i32]] [[3 x i32] [i32 3, i32 8, i32 5], [3 x i32] [i32 1, i32 2, i32 0], [3 x i32] zeroinitializer]
define dso_local void @a1(i32 %0) {
    %2 = alloca i32
    store i32 %0, i32 * %2
    br label %3

3:
    %4 = load i32, i32* %2
    %5 = icmp sgt i32 %4, 1
    %6 = zext i1 %5 to i32
    %7 = icmp ne i32 0, %6
    br i1 %7, label %8, label %11

8:
    %9 = load i32, i32* %2
    %10 = sub i32 %9, 1
    call void @a1(i32 %10)
    br label %11

11:
    ret void
}
define dso_local i32 @a2(i32 %0, i32* %1) {
    %3 = alloca i32*
    store i32* %1, i32* * %3
    %4 = alloca i32
    store i32 %0, i32 * %4
    %5 = load i32, i32* %4
    %6 = load i32*, i32* * %3
    %7 = getelementptr i32, i32* %6, i32 2
    %8 = load i32, i32* %7
    %9 = add i32 %5, %8
    ret i32 %9
}
define dso_local i32 @a3(i32 %0, i32* %1, [3 x i32] *%2) {
    %4 = alloca [3 x i32]*
    store [3 x i32]* %2, [3 x i32]* * %4
    %5 = alloca i32*
    store i32* %1, i32* * %5
    %6 = alloca i32
    store i32 %0, i32 * %6
    %7 = load i32, i32* %6
    %8 = load i32*, i32* * %5
    %9 = getelementptr i32, i32* %8, i32 1
    %10 = load i32, i32* %9
    %11 = mul i32 %7, %10
    %12 = load [3 x i32] *, [3 x i32]* * %4
    %13 = getelementptr [3 x i32], [3 x i32]* %12, i32 2
    %14 = getelementptr [3 x i32], [3 x i32]* %13, i32 0, i32 1
    %15 = load i32, i32 *%14
    %16 = sub i32 %11, %15
    ret i32 %16
}

define dso_local i32 @main() {
    %1 = alloca [2 x [ 3 x i32]]
    %2 = getelementptr [2 x [3 x i32]], [2 x [3 x i32]]*%1, i32 0, i32 0, i32 0
    store i32 1, i32* %2
    %3 = getelementptr [2 x [3 x i32]], [2 x [3 x i32]]*%1, i32 0, i32 0, i32 1
    store i32 2, i32* %3
    %4 = getelementptr [2 x [3 x i32]], [2 x [3 x i32]]*%1, i32 0, i32 0, i32 2
    store i32 3, i32* %4
    %5 = getelementptr [2 x [3 x i32]], [2 x [3 x i32]]*%1, i32 0, i32 0, i32 2
    %6 = load i32, i32* %5
    call void @a1(i32 %6)
    %7 = alloca i32
    %8 = getelementptr [6 x i32], [6 x i32]* @a, i32 0, i32 4
    %9 = load i32, i32* %8
    %10 = getelementptr [6 x i32], [6 x i32]* @a, i32 0, i32 0
    %11 = call i32 @a2(i32 %9, i32* %10)
    store i32 %11, i32* %7
    %12 = alloca i32
    %13 = getelementptr [3 x [3 x i32]], [3 x [3 x i32]]* @b, i32 0, i32 0, i32 1
    %14 = load i32, i32* %13
    %15 = mul i32 1, 3
    %16 = getelementptr [3 x [3 x i32]], [3 x [3 x i32]]* @b, i32 0, i32 0
    %17 = getelementptr [3 x i32], [3 x i32]* %16, i32 0, i32 %15
    %18 = getelementptr [3 x [3 x i32]], [3 x [3 x i32]]* @b, i32 0, i32 0
    %19 = call i32 @a3(i32 %14, i32* %17, [3 x i32]* %18)
    store i32 %19, i32* %12
    %20 = load i32, i32* %7
    %21 = load i32, i32* %12
    %22 = add i32 %20, %21
    call void @putint(i32 %22)
    ret i32 0
}
```
- 输出：24