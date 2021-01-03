---
title: getting-started-with-llvm-core-libraries
date: 2021-01-03 22:08:50
tags: compilor
---

## Getting Started with LLVM Core Libraries

《Getting Started with LLVM Core Libraries》的中文版《LLVM编译器实践教程》读书笔记

<!--more-->

### LLVM设计和工具

LLVM的设计核心是它的IR，它使用的静态单赋值形式（SSA）具有两个特征

1. 代码被组织为三地址指令
2. 它有数目不受限制的寄存器

LLVM IR的基本原则

1. SSA表示和允许快速优化的无限寄存器
2. 通过将整个程序存储在磁盘IR表示中以实现便捷的链接时优化

LLVM整个编译过程中，其他中间的数据结构有

1. C到LLVM IR时，首先将程序转化为AST
2. IR转换为特定机器的汇编时，LLVM首先将程序转化为DAG以便选择指令，然后将其转会三地址以进行指令调度
3. 为了实现汇编器和链接器，LLVM使用MCModule在对象文件的上下文中保存程序表示



LLVM设计通常强制组件在高度抽象层次上交互，将不同组件分割为不同的库



### Clang前端

1. 词法分析，将语言结构拆分为一组单词和记号（token
2. 语法分析，把记号组合在一起形成表达式，语句和函数体等，并判断这个组合是否合理，但不关心句子的语义，输入为token流，输出为AST
   1. AST的节点表示声明，语句或类型，因此有三个核心类：Decl，Stmt和Type
   2. 每个C或C++语言的结构都被Clang用一个C++类来表示，这个C++类继承自核心类
   3. 这里是一个Stmt的继承关系[Doxygen页面](http://clang.llvm.org/doxygen/classclang_1_1Stmt.html)
   4. AST的根节点是TranslationUnitDecl类，代表整个翻译单元
3. 语义分析，借助符号表来确保代码是否违反编程语言的类型系统
4. 生成LLVM IR代码

```c
//min.c
int min(int a, int b){
    if( a < b )
        return a;
    return b;
}

```



词法分析举例

```c
➜  llvm-learn clang -cc1 -dump-tokens min.c 
int 'int'        [StartOfLine]  Loc=<min.c:3:1>
identifier 'min'         [LeadingSpace] Loc=<min.c:3:5>
l_paren '('             Loc=<min.c:3:8>
int 'int'               Loc=<min.c:3:9>
identifier 'a'   [LeadingSpace] Loc=<min.c:3:13>
comma ','               Loc=<min.c:3:14>
int 'int'               Loc=<min.c:3:15>
identifier 'b'   [LeadingSpace] Loc=<min.c:3:19>
r_paren ')'             Loc=<min.c:3:20>
l_brace '{'             Loc=<min.c:3:21>
if 'if'  [StartOfLine] [LeadingSpace]   Loc=<min.c:4:2>
l_paren '('             Loc=<min.c:4:4>
identifier 'a'          Loc=<min.c:4:5>
less '<'                Loc=<min.c:4:6>
identifier 'b'          Loc=<min.c:4:7>
r_paren ')'             Loc=<min.c:4:8>
return 'return'  [StartOfLine] [LeadingSpace]   Loc=<min.c:5:3>
identifier 'a'   [LeadingSpace] Loc=<min.c:5:10>
semi ';'                Loc=<min.c:5:11>
return 'return'  [StartOfLine] [LeadingSpace]   Loc=<min.c:6:2>
identifier 'b'   [LeadingSpace] Loc=<min.c:6:9>
semi ';'                Loc=<min.c:6:10>
r_brace '}'      [StartOfLine]  Loc=<min.c:7:1>
eof ''          Loc=<min.c:7:2>
```



语法分析举例

```c
➜  llvm-learn clang -fsyntax-only -Xclang -ast-dump min.c
TranslationUnitDecl 0x55f7ee731258 <<invalid sloc>> <invalid sloc>
|-TypedefDecl 0x55f7ee7317d0 <<invalid sloc>> <invalid sloc> implicit __int128_t '__int128'
| `-BuiltinType 0x55f7ee7314f0 '__int128'
|-TypedefDecl 0x55f7ee731840 <<invalid sloc>> <invalid sloc> implicit __uint128_t 'unsigned __int128'
| `-BuiltinType 0x55f7ee731510 'unsigned __int128'
|-TypedefDecl 0x55f7ee731b18 <<invalid sloc>> <invalid sloc> implicit __NSConstantString 'struct __NSConstantString_tag'
| `-RecordType 0x55f7ee731920 'struct __NSConstantString_tag'
|   `-Record 0x55f7ee731898 '__NSConstantString_tag'
|-TypedefDecl 0x55f7ee731bb0 <<invalid sloc>> <invalid sloc> implicit __builtin_ms_va_list 'char *'
| `-PointerType 0x55f7ee731b70 'char *'
|   `-BuiltinType 0x55f7ee7312f0 'char'
|-TypedefDecl 0x55f7ee731e78 <<invalid sloc>> <invalid sloc> implicit __builtin_va_list 'struct __va_list_tag [1]'
| `-ConstantArrayType 0x55f7ee731e20 'struct __va_list_tag [1]' 1 
|   `-RecordType 0x55f7ee731c90 'struct __va_list_tag'
|     `-Record 0x55f7ee731c08 '__va_list_tag'
`-FunctionDecl 0x55f7ee78af88 <min.c:3:1, line:7:1> line:3:5 min 'int (int, int)'
  |-ParmVarDecl 0x55f7ee731ee8 <col:9, col:13> col:13 used a 'int'
  |-ParmVarDecl 0x55f7ee78aeb0 <col:15, col:19> col:19 used b 'int'
  `-CompoundStmt 0x55f7ee78b208 <col:21, line:7:1>
    |-IfStmt 0x55f7ee78b178 <line:4:2, line:5:10>
    | |-<<<NULL>>>
    | |-<<<NULL>>>
    | |-BinaryOperator 0x55f7ee78b0f8 <line:4:5, col:7> 'int' '<'
    | | |-ImplicitCastExpr 0x55f7ee78b0c8 <col:5> 'int' <LValueToRValue>
    | | | `-DeclRefExpr 0x55f7ee78b078 <col:5> 'int' lvalue ParmVar 0x55f7ee731ee8 'a' 'int'
    | | `-ImplicitCastExpr 0x55f7ee78b0e0 <col:7> 'int' <LValueToRValue>
    | |   `-DeclRefExpr 0x55f7ee78b0a0 <col:7> 'int' lvalue ParmVar 0x55f7ee78aeb0 'b' 'int'
    | |-ReturnStmt 0x55f7ee78b160 <line:5:3, col:10>
    | | `-ImplicitCastExpr 0x55f7ee78b148 <col:10> 'int' <LValueToRValue>
    | |   `-DeclRefExpr 0x55f7ee78b120 <col:10> 'int' lvalue ParmVar 0x55f7ee731ee8 'a' 'int'
    | `-<<<NULL>>>
    `-ReturnStmt 0x55f7ee78b1f0 <line:6:2, col:9>
      `-ImplicitCastExpr 0x55f7ee78b1d8 <col:9> 'int' <LValueToRValue>
        `-DeclRefExpr 0x55f7ee78b1b0 <col:9> 'int' lvalue ParmVar 0x55f7ee78aeb0 'b' 'int'
```

### LLVM IR

IR是前端和后端的桥梁

* 高层级的IR允许优化器更轻松地提取源代码的相关信息
* 低层级的IR更加贴近目标机器，更容易为特定硬件生成相应diamagnetic

LLVM开始于一个比Java字节码更低抽象级的IR，这也是其缩写Low Level Virtual Machine的由来，现在LLVM既不是Java的竞争对手，也不是一个虚拟机

LLVM 有多种IR表示，这里LLVM IR专指Instruction类所代表的不同后端共享的中间表示，LLVM项目是从一系列围绕LLVM IR的工具展开，用于本层的优化，IR有三种等价表达形式

1. 内存表示（Instruction类等）
2. 被压缩的磁盘表示（位码文件）
3. 人工可读文本的磁盘表示（LLVM汇编码文件）

#### IR语法

这里以sum函数的汇编码表示来解释LLVM IR

```c
//sum.c
//clang sum.c -emit-llvm -S -c -o sun.ll
int sum( int a, int b){
    return a+b;
}
```

汇编码

```assembly
; ModuleID = 'sum.c'
source_filename = "sum.c"
target datalayout = "e-m:e-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-pc-linux-gnu"

; Function Attrs: noinline nounwind optnone uwtable
define i32 @sum(i32, i32) #0 {
entry:
  %3 = alloca i32, align 4
  %4 = alloca i32, align 4
  store i32 %0, i32* %3, align 4
  store i32 %1, i32* %4, align 4
  %5 = load i32, i32* %3, align 4
  %6 = load i32, i32* %4, align 4
  %7 = add nsw i32 %5, %6
  ret i32 %7
}
```

整个LLVM汇编码文件的内容被视为一个LLVM模块，模块是LLVM IR顶层数据结构，每个模块包含一系列函数，每个函数由包含一系列指令的一系列**基本块**组成

> 在电脑编译器架构中，基本块（basic block）是一段线性的程式码，只能从这段程式码开始处进入这段程式，没有其他程式码会跳跃进入这段程式，只能从这段程式码最后一行离开这段程式，中间没有其他程式码会跳跃离开这段程式 ——wiki

模块还包含用于支持该模型的外围实体，如全局变量，目标数据布局，外部函数原型以及数据结构声明

通过这个例子可以看到LLVM如何表达其基本属性

1. 使用静态单赋值（SSA），该形式下，每个变量都不会被重新赋值，每个变量都只有唯一一条定义它的赋值语句。每次使用一个变量都可以立即回溯到负责其定义的唯一指令。使用SSA形式导致“使用定义链“（use-def链，即可以达到使用处的所有定义/赋值语句的集合）的生成变得非常简单，这个简化操作有很大的价值。使用定义链是经典优化（如常量传播喝冗余表达式消除）的前提条件，而SSA使得定义链的获得非常简单
2. 代码被组织成三地址指令，数据处理指令有两个源操作数，并将结果放在不同的目标操作数中
3. 有无穷多的寄存器，LLVM局部变量可使用以%开头的任意名称，没有数量限制

本地标识符以%开头，全局标识符以@开头（如@sum，%7）函数名最后的#0记号映射到一组函数属性

```json
attributes #0 = { noinline nounwind optnone uwtable "correctly-rounded-divide-sqrt-fp-math"="false" "disable-tail-calls"="false" "less-precise-fpmad"="false" "no-frame-pointer-elim"="true" "no-frame-pointer-elim-non-leaf" "no-infs-fp-math"="false" "no-jump-tables"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="false" "stack-protector-buffer-size"="8" "target-cpu"="x86-64" "target-features"="+fxsr,+mmx,+sse,+sse2,+x87" "unsafe-fp-math"="false" "use-soft-float"="false" }
```

函数的主体部分明确划分为多个基本块，每一个基本块开头处都会有一个标签。每一个基本块都包含一系列指令，其中第一条指令有一个入口点，最后一条指令有一个出口点，这样，当代码执行到对应于基本块的标签时，我们可以知道他将执行这个基本块中的所有指令（回顾一下基本块的定义），直到最后一条指令，之后跳转到另一个基本块来改变控制流，基本块及其关联的标签符合以下条件

1. 每个基本块都需要一个结束符指令结尾，结束符一般为跳转到另一个基本块或者从该函数返回
2. 第一个基本块称为（entry），它的地位在LLVM函数中比较特殊，不能作为任何分支指令的跳转目标



alloca指令在当前函数的堆栈帧中保留一定空间，空间的大小由元素类型的大小决定，并符合一定的对齐方式，通常用于给本地（自动）变量申请地址，本例中，%3为一个i32类型，4字节对齐的元素**的地址**

store指令用于把%0存储在%3地址处，（%0和%1是函数的两个参数）load指令用于把%3处的元素%0的值加载到%5里，如果使用-O1优化，那么clang会直接使用函数参数，删去这些不必要的存储和加载。

add指令求和，其中nsw指”no signed wrap“，表示已知指令时无溢出的，从而允许一些优化



#### LLVM IR内存模型

include/llvm/IR中有以下的C++类

1. Module类，负责聚合编译过程中使用的数据，包含了模块中的所有函数，可以通过迭代器遍历
2. Function类，包含函数定义或者声明有关的所有对象，包含函数列表，函数原型，如果是声明，那么只有函数原型，如果是函数定义，那么还有函数内容，可以通过迭代器遍历，将跨其基本块执行
3. BasicBlock类，封装一系列LLVM指令，可以通过begin()/end()访问，可以访问其CFG（控制流程图）
4. Instruction类，表示LLVM IR中的最小基本单元，有一些快速获取高层次信息的方法，如isTerminator()，getOpcode()等。通过op_begin()/op_end()可以遍历其操作数，这两个方法从User类继承而来
5. Value类和User类，通过它们可以访问use-def和def-use，继承Value类的类型意味着该类定义了可以被其他指令使用的结果。而User的子类意味着该类使用了一个或者多个Value接口。Function和Instruction同时是这两个类的子类，而BasicBlock仅仅是Value的子类
   * Value类定义了use_begin()和use_end()，用于遍历User对象，从而可以访问其def-use。而且对于每个Value类，可以通过getName()访问其名称。这也符合任何LLVM变量都具有与其相关的明确指标的定义。例如add1定义一个加法的结果，myfunc定义一个函数。Value类的replaceAllusesWith(Value *)方法可以遍历该值的所有使用者，并且用参数的值代替，这是一个很好的SSA表示形式的例子，使得可以轻松替换指令，编写优化
   * User类有op_begin()和op_end()方法，允许快速访问它使用的所有Value接口。这个关系对应use-def链，用户也可以使用replaceUsesOfWith(Value* From, Value* To)来替换它使用的任何值



### 后端

LLVM后端由一组代码生成分析器和变换流程（pass）组成，这些流程将LLVM IR转换为目标代码。

//to do