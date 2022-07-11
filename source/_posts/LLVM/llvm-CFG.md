---
title: 【Clang Static Analyzer】CFG和Path-sensitive checker回调函数
date: 2022-05-14 17:12:43
tags:
categories: LLVM
---
# 【Clang Static Analyzer】 CFG和Path-sensitive checker回调函数
前面我们讲到CSA的检查可以基于CFG，但这种检查的实际应用场景是极少的。
但关于CFG的概念，以及对于CFG图的理解是我们需要知道的。
CFG由AST构建出来，用于Path-sensitive的执行(即CSA是基于CFG的节点进行模拟执行的)，一般用于配合path-sensitive checker来检查生命周期。

而基于Exploded graph的检查是CSA的核心，涉及到相关的概念，我们需要先捋一下：
1. Exploded graph: 可以理解为CSA分析过程中产生的数据结构，我们写Path-sensitive checker的核心工作就是往这个数据结构中添加信息。
2. Path-sensitive: 会根据不同条件分支，跟踪程序控制流的每个分支并记录每个分支的状态，比如有2个条件分支，则会记录这两个分支的程序状态。
3. 符号执行: 跟程序执行类似，但会探索所有可能的分支，并收集每个分支的符号变量的限制范围。

### CFG
如何使用CFG呢？其实我们只需要会遍历CFG，并能拿到相关信息即可。(毕竟很少用到)
使用-cc1 -analyze -analyzer-checker="debug.ViewCFG"可以查看CFG图。

##### 1.CFG图说明
CFG的图还是很容易理解的：
![](Images/CFG.png)
官方[CFGBlock](https://clang.llvm.org/doxygen/classclang_1_1CFGBlock.html)详情
官方[CFGElement](https://clang.llvm.org/doxygen/classclang_1_1CFGElement.html)详情

##### 2.CFG的遍历
CFG的遍历也比较简单，参考代码如下：
```c++
void checkASTCodeBody(const Decl *D, AnalysisManager &mgr, BugReporter &BR) const {
    CFG *cfg = mgr.getCFG(D);
    // 遍历CFGBlock
    for (const auto& cfgNode : *cfg) {
      // 遍历当前节点的上(父)层节点
      for (CFGBlock::const_pred_iterator I = cfgNode->pred_begin(); I != cfgNode->pred_end(); ++I) {
        llvm::errs() << "CFG pred Node ID:" << (*I)->getBlockID() << '\n';
      }
      // 遍历当前节点的下(子)层节点
      for (CFGBlock::const_succ_iterator I = cfgNode->succ_begin(); I != cfgNode->succ_end(); I++) {
        llvm::errs() << "CFG succ Node ID:" << (*I)->getBlockID() << '\n';
      }
      // 遍历CFGElement
      for (const auto& cfgElemet : *cfgNode) {
        if (Optional<CFGStmt> stmt = cfgElemet.getAs<CFGStmt>()) {
          const Stmt *astStmt = const_cast<Stmt *>(stmt->getStmt());
          if (const CallExpr *CE = dyn_cast<CallExpr>(astStmt)) {
            // 做一些检查！
          }
        }
      }
    }
}
```

### Path-sensitive checker回调函数
Path-sensitive checker的核心数据结构是Exploded graph，但这个数据结构我们先缓缓，先看相关的回调函数。
前面我们我们讲了基于AST的注册回调函数，但这些回调函数并不会参与到exploded graph的构建，一般只能做一些语法、函数禁用的检查。
下面是一些常用的回调函数，还有一些并没有列举（因为没用过）

##### 1. check\:\:PreStmt\<T\> 和 check\:\:PostStmt\<T\>
模板T可以是任意的AST Stmt类型，但实际上**条件控制语句（if等）、返回语句(return)并不会发生回调**
PreStmt的回调发生在“符号执行”之前，PostStmt发生在执行之后。
因此对于PreStmt来说，如果订阅的是表达式，那么这个表达式的符号值还没计算出来。如：
```c++
/// 比方回调 "i == 0;"这个表达式
void checkPreStmt(const BinaryOperator *S, CheckerContext &C) const {
  SVal rhs = C.getSVal(S->getRHS());  // 右值已经符号执行完成了
  SVal exp = C.getSVal(S);            // 整个表达式的符号值还没有执行
  Optional<DefinedSVal> DV_rhs = rhs.getAs<DefinedSVal>();
  Optional<DefinedSVal> DV_exp = exp.getAs<DefinedSVal>();
  if (!DV_rhs) {
    llvm::errs() << "checkPreStmt not rhs DefinedSVal" << '\n';
  }
  if (!DV_exp) {
    // 会打印，因为整个表达式的符号值还没计算出来
    llvm::errs() << "checkPreStmt not exp DefinedSVal" << '\n';   
  }
}
```
同理，对于PostStmt来说，整个表达式的符号值已经执行可以获取，但子表达式**可能在环境中已经不存在了**无法获取
关于代码中的DefinedSVal代表的是什么，我们后面再讲。

##### 2. check\:\:PreCall 和 check\:\:PostCall
check\:\:PreCall的触发时机等价于check\:\:PreStmt\<CallExpr\>，
check\:\:PostCalll的触发时机等价于check\:\:PostStmt\<CallExpr\>
但相比PreStmt和PostStmt，PreCall、PostCalll的参数CallEvent封装了函数的相关操作，可以方便我们使用。
同样地，PreCall发生在函数“符号执行”之前，因此只能获取参数的符号值，而PostCalll在执行函数（可能）之后，因此可以对函数返回值进行判断。
一般来说，**PreCall往往用在判断函数参数(如fclose判断哪些fd被关闭)，PostCalll判断函数返回值（如fopen判断哪些fd被打开）**。
```c++
// 比方回调 "if (i == 0) F1111(i);" 这个情况下，函数(F1111)的参数i必然是0，可以检查F1111的参数是否为0
void checkPreCall(const CallEvent &Call, CheckerContext &C) const {
  // 从符号表中取符号名
  IdentifierInfo* IIFunc = &C.getASTContext().Idents.get("F1111");
  if (Call.getCalleeIdentifier() != IIFunc)  // 检查函数名
    return;
  if (Call.getNumArgs() != 1)  // 检查函数参数个数是否为1
    return;
  SVal args = Call.getArgSVal(0);  // 获取参数1的符号值
  Optional<NonLoc> NL = args.getAs<NonLoc>();
  if (!NL) {
    return;
  }
  // 构造符号值为0的符号
  BasicValueFactory &BVF = C.getSValBuilder().getBasicValueFactory();
  llvm::APSInt Zero = BVF.getIntValue(0, false);
  ConstraintManager &CM = C.getConstraintManager();
  // 检查函数参数的值的范围
  if (CM.assumeInclusiveRange(C.getState(), *NL, Zero, Zero, true)) {
    llvm::errs() << "F1111(args = 0)" << '\n';  // 打印
  }
  // checkPreCall无法对函数返回值进行判断，但在PostCalll中可以
  SVal ret = Call.getReturnValue();
  Optional<DefinedSVal> DV = ret.getAs<DefinedSVal>();
  if (!DV) {
    llvm::errs() << "checkPreCall not excute func" << '\n';  //打印
  }
```

##### 3. check\:\:BranchCondition
check\:\:BranchCondition发生在出现条件分支的情况，如if、while、for等，**还包括||和&&**
一般用于检查条件值：
```c++
// 比方回调"int j; if (j){}", j属于未初始化变量
void checkBranchCondition(const Stmt *Condition, CheckerContext &C) const {
  SVal X = C.getSVal(Condition);
  // 如果条件的符号值是未初始化的变量
  if (X.isUndef()) {
    // 直接生成sink节点，不需要继续执行（其他检查也会停止）
    C.generateErrorNode();
  }
}
```

##### 4. check\:\:Location和check\:\:Bind
check\:\:Location 发生在变量发生写（不包括变量初始化的赋值）和读的时候
check\:\:Bind 发生在变量发生写（包括变量初始化）的时候
```c++
int func(int i)
{
    int z = 1;   // 会触发Bind
    int j = i;   // 会触发Bind，Location（i变量被读取，参数is_load=true）
    j = z;       // 会触发Bind和2次Location(z变量被读取，参数is_load=true；j变量被写入，参数is_load=false)
    if (j){      // 会触发Location(j被读取，参数is_load = ture)
        return j; // 会触发Location(j被读取，参数is_load = ture)
    }
    return j;    // 程序不可达，不会触发
}
```

##### 5. check\:\:EndAnalysis 和 check\:\:BeginFunction 和 check\:\:EndFunction
check\:\:EndAnalysis 发生在CSA完成分析一个函数体的时候，一般我们在这个时间节点遍历完整的Exploded Graph。
check\:\:BeginFunction 发生在函数的开始，一个函数只会回调一次。
而check\:\:EndFunction 发生在每个可能会return的分支
```c++
int func(int i)
{
    // 这种情况，我理解的是会回调3次，但实际情况是根据return的值不同，
    // 会产生不一样的结果，目前我还不知道为什么会这样
    if (i == 0) { return -1; }  // 如果是负数，会回调EndFunction
    if (i == 2) { return 3; }  // 如果是非负数，不会回调EndFunction
    return 4;  // 会回调EndFunction
}
```

##### 6. check\:\:LiveSymbols 和 check\:\:DeadSymbols
check\:\:DeadSymbols 发生在变量不再被使用的时候（并不是变量的生命周期结束）
check\:\:LiveSymbols 发生在check\:\:DeadSymbols之前，一般用于把感兴趣的变量标记为InUse，这样就不会垃圾回收
```c++
int func(int a){
    int i = 0;  // 这种其实也会触发LiveSymbols和DeadSymbols
    int j = 0;  
    int k = 0;
    if (a) {
        i = 1;  // 这个分支里，j是DeadSymbols
    }
    else {
        j = 1;  // i是DeadSymbols
    }
    return k;  // i 和 j 都是DeadSymbols了
}
```

##### 7. eval::Assume
跟check\:\:BranchCondition的触发场景类似。一般用于修改ProgramState,比方删除不需要再关注的“符号变量”。如：
```c++
// 来自MallocChecker的代码片段
ProgramStateRef evalAssume(ProgramStateRef state, SVal Cond, bool Assumption) const {
  RegionStateTy RS = state->get<RegionState>();
  for (RegionStateTy::iterator I = RS.begin(), E = RS.end(); I != E; ++I) {
    // If the symbol is assumed to be NULL, remove it from consideration.
    ConstraintManager &CMgr = state->getConstraintManager();
    ConditionTruthVal AllocFailed = CMgr.isNull(state, I.getKey());
    if (AllocFailed.isConstrainedTrue())
      state = state->remove<RegionState>(I.getKey());
  }
  // ...
  return state;
}
```

##### 8. eval::Call
这个回调函数可以让checker参与到CSA的分析过程中，但checker的开发人员需要对函数进行模拟。

### 参考资料&例子
只使用CFG来检查的例子很少，可参考下面案例：
推荐阅读[DeadStoresChecker](https://code.woboq.org/llvm/clang/lib/StaticAnalyzer/Checkers/DeadStoresChecker.cpp.html)来熟悉CFG的使用

Path-sensitive checker的实例很多，但初学者可优先参考下面案例：
推荐阅读[SimpleStreamChecker](https://code.woboq.org/llvm/clang/lib/StaticAnalyzer/Checkers/SimpleStreamChecker.cpp.html)来熟悉Path-sensitive checker

以及下面这个文档说明：
推荐阅读[CheckerDocumentation](https://code.woboq.org/llvm/clang/lib/StaticAnalyzer/Checkers/CheckerDocumentation.cpp.html)来熟悉各种回调函数
