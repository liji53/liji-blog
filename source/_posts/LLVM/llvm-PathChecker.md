---
title: 【Clang Static Analyzer】基于Exploded Graph的检查
date: 2022-06-11 20:45:01
tags:
categories: LLVM
---
- [基于Exploded Graph的检查](#%E5%9F%BA%E4%BA%8EExploded-Graph%E7%9A%84%E6%A3%80%E6%9F%A5)
    - [对Program state的基本操作](#%E5%AF%B9Program-state%E7%9A%84%E5%9F%BA%E6%9C%AC%E6%93%8D%E4%BD%9C)
        - [1.获取SVal](#1-%E8%8E%B7%E5%8F%96SVal)
        - [2.遍历Region Store](#2-%E9%81%8D%E5%8E%86Region-Store)
        - [3.假设符号值](#3-%E5%81%87%E8%AE%BE%E7%AC%A6%E5%8F%B7%E5%80%BC)
        - [4.生成新的符号值](#4-%E7%94%9F%E6%88%90%E6%96%B0%E7%9A%84%E7%AC%A6%E5%8F%B7%E5%80%BC)
    - [参与Exploded Graph的构建](#%E5%8F%82%E4%B8%8EExploded-Graph%E7%9A%84%E6%9E%84%E5%BB%BA)
        - [1.添加Exploded Node](#1-%E6%B7%BB%E5%8A%A0Exploded-Node)
        - [2.添加自定义状态](#2-%E6%B7%BB%E5%8A%A0%E8%87%AA%E5%AE%9A%E4%B9%89%E7%8A%B6%E6%80%81)
    - [遍历Exploded Graph](#%E9%81%8D%E5%8E%86Exploded-Graph)

# 基于Exploded Graph的检查

### 对Program state的基本操作
在[Exploded Graph和内存模型](https://liji53.github.io/2022/05/21/LLVM/llvm-explodedGraph)中我们简单介绍了Program state,它存储了程序上下文变量的符号值(region)、表达式的符号值(Environment)、变量的限制范围(Constraints)等等，下面我们介绍下对Program state的常用操作。

##### 1.获取SVal
获取符号值，主要分为2种情况：1.从表达式中获取符号值；2.从内存区域中获取符号值
它的常规用法如下：
```c++
// 1.从表达式获取符号值
const Expr *E = /* AST表达式 */;
SVal Val = C.getSVal(E);  /*C是CheckerContext*/
// 2.从内存区域获取符号值（理解为解引用）
const Expr *E = /* 指针表达式 */;
const MemRegion *Reg = State->getSVal(E, LC).getAsRegion ();
SVal Val = State->getSVal(Reg);
```

##### 2.遍历Region Store
Region Store存储了MemRegion与SVal的映射关系，对它的遍历需要注册回调类来实现：
```c++
class CallBack : public StoreManager::BindingsHandler {
  using store = llvm::SmallVector<std::pair<const MemRegion *, SVal>, 8>;

public:
  store V;
  // Region Store会回调这个方法，返回false表示不继续遍历，返回true表示继续遍历
  bool HandleBinding(StoreManager &SMgr, Store Store,
                     const MemRegion *Region, SVal Val) override {
    V.emplace_back(Region, Val);
    return true;
  }
};
CallBack Handler;
ProgramStateRef State = C.getState();
// 对Region Store遍历，在iterBindings接口中会回调HandleBinding
State->getStateManager().getStoreManager().iterBindings(State->getStore(), Handler);
for (const auto i : Handler.V) {
  i.first->dump();
  llvm::errs() << "\n";
  i.second.dump();
  llvm::errs() << "\n";
}
```

##### 3.假设符号值
在执行条件语句之后，我们可能需要对变量的取值范围进行判断，通过ConstraintManager/state的assume系列函数，可以判断表达式值的范围。
其实之前在[CFG和Path-sensitive-checker回调函数](https://liji53.github.io/2022/05/14/LLVM/llvm-CFG/)的check::PreCall一节中已经给过一个的例子了。
用法如下：
```c++
SVal Val = /* 符号值 */;
// 必须是DefinedSVal才行，不能是未初始化变量(UndefinedVal)和无法计算的值(UnknownVal)
Optional<DefinedSVal> DVal = Val.getAs<DefinedSVal>();
if (! DVal )
  return ;
// 1. 通过ConstraintManager来进行判断
ConstraintManager &CM = C.getConstraintManager();
// 判断DV这个符号值是否为非0
ProgramStateRef trueState = CM.assume(C.getState(), *DV, true);
// 判断DV这个符号值是否在[0,1]的区间
ProgramStateRef trueState = CM.assumeInclusiveRange(C.getState(), *DV, One, Two, true);
// 2. 通过state来判断，其内部实现也是通过ConstraintManager
ProgramStateRef TrueState, FalseState;
// DVal可以是UnknownVal的
std::tie(TrueState, FalseState) = State->assume(DVal);
```

##### 4.生成新的符号值
在比较符号值的时候，我们可能需要构建一些新的符号值，通过SValBuilder可以构建。
比方：
```c++
// 摘录自MallocChecker，用于计算内存buffer的大小
SVal MallocChecker::evalMulForBufferSize(CheckerContext &C, const Expr *Blocks,
                                         const Expr *BlockBytes) {
  SValBuilder &SB = C.getSValBuilder();
  // 内存块的个数
  SVal BlocksVal = C.getSVal(Blocks);
  // 一个内存块的大小
  SVal BlockBytesVal = C.getSVal(BlockBytes);
  ProgramStateRef State = C.getState();
  // 通过eval计算内存块的大小
  SVal TotalSize = SB.evalBinOp(State, BO_Mul, BlocksVal, BlockBytesVal,
                                SB.getContext().getSizeType());
  return TotalSize;
}
```

### 参与Exploded Graph的构建
Exploded Graph是符号执行时的路径图，只要可能达到的路径，都将分析执行，因此存在路径爆炸(函数堆栈比较深，条件分支比较多)的问题。
在符号执行的过程中，我们可以往这个路径图中添加我们自己的节点、符号状态等等，甚至通过添加节点改变Exploded Graph的路径方向。
Exploded Graph的一个重要原则就是：节点都是不可变的，只能新增。

##### 1.添加Exploded Node
有时候我们可能对函数的异常返回不关心，因此不需要对异常分支进行符号执行。
这时候我们可以这么做：
```c++
// 测试程序
int func(int i){
	if (!test()) { /*don't care*/ }
	else { /*需要符号执行，检查程序错误*/ }
}
// CSA checker
void checkPostCall(const CallEvent &Call, CheckerContext &C) const {
  // 获取函数返回值的符号值
  SVal SVfunc = Call.getState()->getSVal(Call.getOriginExpr(), C.getLocationContext());
  const Optional<DefinedOrUnknownSVal> DS = SVfunc.getAs<DefinedOrUnknownSVal>();
  ProgramStateRef trueState, falseState;
  // 假设函数返回值，有没有可能真、假
  std::tie(trueState, falseState) = Call.getState()->assume(*DS);
  // 只关心返回值为True的情况
  if (trueState) {  
    // 对于False的分支，后面不会再符号执行了
    C.addTransition(trueState); 
  }
}
```

##### 2.添加自定义状态
前面讲到Program State的GDM用于存储用户自定义的状态，一般需要检查有关联的一系列的事件时，就会用到。如内存泄漏的检查。
下面截取[SimpleStreamChecker](https://code.woboq.org/llvm/clang/lib/StaticAnalyzer/Checkers/SimpleStreamChecker.cpp.html)的代码片段，看下如何使用：
```c++
/// 先自定义文件指针的状态
struct StreamState {
private:
  enum Kind { Opened, Closed } K;
  StreamState(Kind InK) : K(InK) {}
public:
  ...
};
// 对于open系列的函数，保存open返回的符号，为什么是checkPostCall，我们前面已经讲过了
void SimpleStreamChecker::checkPostCall(const CallEvent &Call,
                                        CheckerContext &C) const {
  // ... 省略
  // Get the symbolic value corresponding to the file handle.
  SymbolRef FileDesc = Call.getReturnValue().getAsSymbol();
  if (!FileDesc)  return;
  // Generate the next transition (an edge in the exploded graph).
  ProgramStateRef State = C.getState();
  State = State->set<StreamMap>(FileDesc, StreamState::getOpened());
  C.addTransition(State);
}
// 对于close系列的函数，根据close时的参数，查找对应的符号，并检查
void SimpleStreamChecker::checkPreCall(const CallEvent &Call,
                                       CheckerContext &C) const {
  // ... 省略
  // Get the symbolic value corresponding to the file handle.
  SymbolRef FileDesc = Call.getArgSVal(0).getAsSymbol();
  if (!FileDesc)  return;
  // Check if the stream has already been closed.
  ProgramStateRef State = C.getState();
  // 查找之前open时的符号
  const StreamState *SS = State->get<StreamMap>(FileDesc);
  if (SS && SS->isClosed()) {
    // 重复关闭
  }
  // Generate the next transition, in which the stream is closed.
  State = State->set<StreamMap>(FileDesc, StreamState::getClosed());
  C.addTransition(State);
}
// 类似的还有Set、LIST、TRAIT
REGISTER_MAP_WITH_PROGRAMSTATE(StreamMap, SymbolRef, StreamState)
```

### 遍历Exploded Graph
这种场景的使用，等实际用到的时候，可以参考[UnreachableCodeChecker](https://code.woboq.org/llvm/clang/lib/StaticAnalyzer/Checkers/UnreachableCodeChecker.cpp.html)
