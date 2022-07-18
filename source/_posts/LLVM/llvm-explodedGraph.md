---
title: 【Clang Static Analyzer】Exploded Graph和内存模型
date: 2022-05-21 10:45:01
tags:
categories: LLVM
---

- [Exploded-Graph和内存模型](#Exploded-Graph%E5%92%8C%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B)
    - [Exploded-graph](#Exploded-graph)
        - [Program-Point](#Program-Point)
        - [Program-State](#Program-State)
        - [Exploded-graph图](#Exploded-graph%E5%9B%BE)
    - [内存模型](#%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B)
        - [SVal](#SVal)
        - [MemRegion](#MemRegion)
        - [SymExpr](#SymExpr)
        - [三者与Exploded-graph的联系](#%E4%B8%89%E8%80%85%E4%B8%8EExploded-graph%E7%9A%84%E8%81%94%E7%B3%BB)
    - [参考资料与例子](#%E5%8F%82%E8%80%83%E8%B5%84%E6%96%99%E4%B8%8E%E4%BE%8B%E5%AD%90)

# Exploded-Graph和内存模型
基于Exploded Graph检查指的是结合代码执行过程中的上下文来进行检查。打个比方，我们用GDB调试的时候打断点，然后在这个断点处查看上下文，能知道当前程序的变量值、调用堆栈等信息。CSA也是类似的检查过程。
前面我们讲的Path-sensitive checker回调函数就可以理解成是程序断点，而Exploded Graph就是断点处的上下文信息，包括当前变量的值，表达式的值，符号的约束信息等。

### Exploded-graph
Path-sensitive checker的核心数据结构是Exploded graph，Exploded graph由Exploded graph node组成，Exploded graph node包含Program Point和Program State
如图所示：
![](Images/Exploded_graph_node.png)

##### Program-Point
表示当前程序执行的位置。一般来说，我们写checker用到它的机会不多，往往在checkEndAnalysis(还记得一系列的回调函数把)遍历Exploded graph才用。
**一个CFGElement往往对应一个Program Point或者多个Program Point**，下面是Program Point的部分信息：
```js
// 部分关键信息
{ 
    "kind": "Statement",   # Program Point的类型
    "block_id": 3,         # 对应CFG的block ID， BlockEntrance类型的才有
    "stmt_kind": "DeclStmt", # stmt的类型
    "stmt_id": 670,          # 对应stmt的id，stmt和Program Point的对应关系是1对1或1对多
    "stmt_point_kind": "PostStmt", # 当前程序执行的位置
    "node_id": 7,          # Exploded graph node号，注意与block_id的区别
    "is_sink": 0           # 1的话表示后面不需要再执行了
}
```
详细资料[ProgramPoint](https://clang.llvm.org/doxygen/classclang_1_1ProgramPoint.html)

##### Program-State
Program State包含我们要检查的核心信息，包含以下内容：
 - **Environment: 表达式的符号值（注意这些符号值生命短暂）**
 - **Region Store: 表示上下文变量的符号值（如果变量后面不再用到了，会DeadSymbol，即会删除Region Store中对应的变量）**
 - **Constraints：表示变量的限制范围**
 - Taint: 存储受到“污染”的变量
 - GDM：用于存储checker的自定义的状态信息

关于Region Store、Environment、Constraints里面存的是什么，后面还会继续讲，我们先清楚大概的内容即可。

##### Exploded-graph图
通过-cc1 -analyze -analyzer-checker="debug.ViewExplodedGraph" 可以查看相关的图，但这个图非常的大，我只能截取部分内容：

![](Images/Exploded_graph.png)
这个图其实有一定的迷惑性，我一开始的时候，以为一个方框代表一个Exploded graph node。但其实CSA为了简化展示，把**具有相同的Program State放在了一起**。
建议自己动手生成下，仔细观察下不同的代码对应的Exploded graph.

### 内存模型
我们在写代码时，会涉及到变量的名字、内存地址、内存地址的值等相关概念, 同样的在CSA中同样也有类似的概念, 如"int a = 0"
 - a是一个符号(名字)，在CSA中用SymExpr表示符号，
 - 符号a的值是0，在CSA中用SVal表示符号值，是符号值的抽象
 - 符号a的地址&a，在CSA中用MemRegion表示内存区域，是内存地址的抽象

##### SVal
在符号执行中，使用SVal(符号值)来参与模拟执行，因此**SVal可以理解成各种类型的变量的值**，包括具体的值（如int）、表达式的临时值、不确定的变量值、内存地址值等。
其中SVal比较重要的子类有以下几种：
 - [UndefinedVal](https://clang.llvm.org/doxygen/classclang_1_1ento_1_1UndefinedVal.html) 表示未初始化的局部变量
 - [UnknownVal](https://clang.llvm.org/doxygen/classclang_1_1ento_1_1UnknownVal.html) 表示CSA无法计算的值，如float、switch中default选项
 - [NonLoc](https://clang.llvm.org/doxygen/classclang_1_1ento_1_1NonLoc.html) 表示非地址类型的值
 - [Loc](https://clang.llvm.org/doxygen/classclang_1_1ento_1_1Loc.html) 表示地址类型的值
 - [MemRegionVal](https://clang.llvm.org/doxygen/classclang_1_1ento_1_1loc_1_1MemRegionVal.html) 表示内存地址的值

下面我们给出一段示例：
```c++
struct S{
    int member;
};
int global;           // NonLoc - SymbolVal
int func(){
    float pi = 3.14;  // UnknownVal
    int i;            // UndefinedVal
    int j = 0;        // NonLoc - ConcreteInt
    int* p = &j;      // Loc - MemRegionVal
    struct S s;       // NonLoc - LazyCompoundVal
}
```
详情：[SVal](https://clang.llvm.org/doxygen/classclang_1_1ento_1_1SVal.html)

##### MemRegion
MemRegion是所有Region的基类，不仅**表示存储SVal的内存区域，同时也表示内存段的信息**(如代码段、堆栈段等)
因此MemRegion分为2大类：
- MemSpaceRegion表示内存段的信息，共有5种类型，分别对应实际程序运行时的代码段、数据段、堆、栈
- SubRegion表示内存区域，比方变量的内存区域VarRegion。SubRegion是有内存大小的，而MemSpaceRegion是没有大小的。

比较常用的是SubRegion，下面列举部分常见的：
- [VarRegion](https://clang.llvm.org/doxygen/classclang_1_1ento_1_1VarRegion.html) 表示有内存地址的变量
- [ElementRegion](https://clang.llvm.org/doxygen/classclang_1_1ento_1_1ElementRegion.html) 专门表示数组的子元素
- [FieldRegion](https://clang.llvm.org/doxygen/classclang_1_1ento_1_1FieldRegion.html) 表示对象的成员

SubRegion内部都存在一个SuperRegion,表示父内存区域，通过这两者的关系，就可以构建出内存的层次。
比方（引用自[《Clang-analyzer-guild-v0.1》](https://github.com/haoNoQ/clang-analyzer-guide/releases/download/v0.1/clang-analyzer-guide-v0.1.pdf)）：
```c++
struct A {
  int x, y;
};
struct B: A {
  int u, v;
};
struct C {
  int t;
  B *b;
};
void foo(C c) {
  c.b[5].y; // <-- that
}
```
上面代码的内存模型如下：
![](Images/MemRegion.png)

详情：[MemRegion](https://clang.llvm.org/doxygen/classclang_1_1ento_1_1MemRegion.html)

##### SymExpr
SymExpr表示的是变量的符号(名字)，**在符号执行过程中并不是每个变量的值都能确定的，因此对于那些无法确定值的变量，就会用SymExpr来表示。**
涉及符号执行的表达式大概分为4种：CallExpr(函数调用)、CaseExpr(类型转化)、BinaryOperator(二目运算)、UnaryOperator(一目运算)
因此SymExpr也分为对应的4种类型：
 - [SymbolData](https://clang.llvm.org/doxygen/classclang_1_1ento_1_1SymbolData.html) 它的子类SymbolConjured表示无法对函数调用符号执行的符号值
 - [BinarySymExpr](https://clang.llvm.org/doxygen/classclang_1_1ento_1_1BinarySymExpr.html) 表示无法对二目运算符号执行的符号值
 - [UnarySymExpr](https://clang.llvm.org/doxygen/classclang_1_1ento_1_1UnarySymExpr.html) 表示无法对一目运算符号执行的符号值
 - [SymbolCast](https://clang.llvm.org/doxygen/classclang_1_1ento_1_1SymbolCast.html) 表示无法对类型转化符号执行的符号值

详情：[SymExpr](https://clang.llvm.org/doxygen/classclang_1_1ento_1_1SymExpr.html)

##### 三者与Exploded-graph的联系
对于SVal、MemRegion、SymExpr的概念我们已经有初步的了解了，下面我们结合Exploded graph的Program State来看下他们的联系：

|作用|SVal|MemRegion|SymExpr|
|---|---|---|---|
|作为Constraints的key|||✔|
|作为Region Store的key||✔||
|作为Region Store的value|✔|||
|作为Environment的value|✔|||

### 参考资料与例子
用到Program Point来检查可参考下面案例：
推荐阅读[UnreachableCodeChecker](https://code.woboq.org/llvm/clang/lib/StaticAnalyzer/Checkers/UnreachableCodeChecker.cpp.html)

相关的博客：
https://www.zhihu.com/people/movietravelcode/posts
这篇文章以及[《Clang-analyzer-guild-v0.1》](https://github.com/haoNoQ/clang-analyzer-guide/releases/download/v0.1/clang-analyzer-guide-v0.1.pdf)让我对内存模型的理解省去大量的时间
