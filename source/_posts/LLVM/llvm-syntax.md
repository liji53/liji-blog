
---
title: 【Clang Static Analyzer】基于AST检查
date: 2022-04-21 07:42:33
tags:
categories: LLVM
---

- [基于AST检查](#%E5%9F%BA%E4%BA%8EAST%E6%A3%80%E6%9F%A5)
    - [AST的结构](#AST%E7%9A%84%E7%BB%93%E6%9E%84)
        - [1.Decl](#1-Decl)
        - [2.Stmt](#2-Stmt)
        - [3.Expr](#3-Expr)
    - [AST的遍历](#AST%E7%9A%84%E9%81%8D%E5%8E%86)
        - [1. 通过ConstStmtVisitor遍历Stmt](#1-%E9%80%9A%E8%BF%87ConstStmtVisitor%E9%81%8D%E5%8E%86Stmt)
        - [2. 通过RecursiveASTVisitor遍历](#2-%E9%80%9A%E8%BF%87RecursiveASTVisitor%E9%81%8D%E5%8E%86)
    - [Checkers的注册回调函数(AST)](#Checkers%E7%9A%84%E6%B3%A8%E5%86%8C%E5%9B%9E%E8%B0%83%E5%87%BD%E6%95%B0-AST)
        - [检查器](#%E6%A3%80%E6%9F%A5%E5%99%A8)
    - [AST matchers](#AST-matchers)
        - [1.通过Callback实现](#1-%E9%80%9A%E8%BF%87Callback%E5%AE%9E%E7%8E%B0)
        - [2.更简洁的实现](#2-%E6%9B%B4%E7%AE%80%E6%B4%81%E7%9A%84%E5%AE%9E%E7%8E%B0)
    - [参考资料与例子](#%E5%8F%82%E8%80%83%E8%B5%84%E6%96%99%E4%B8%8E%E4%BE%8B%E5%AD%90)

# 基于AST检查
Clang Static Analyzer 的检查可以分为三种：
1. 基于AST(abstract syntax tree)的检查
2. 基于CFG(Control flow graph)的检查
3. 基于Exploded graph的检查（核心）

但不管哪种方法，在实现一个checker之前，我们需要对编译过程以及AST有一个直观的了解：
1. 编译过程：（clang前端）词法->语法->语义->IR->优化->CodeGen (llvm后端)
2. AST：语法解析之后的产物，最简单最直观的学习方法就是用clang -Xclang -ast-dump -fsyntax-only

### AST的结构
虽然用-ast-dump能清楚看到每句源码对应的AST节点，但这里还是要强调下AST的2种最基本的类型Decl和Stmt

##### 1.Decl
Decl是声明，比方:

|语句|类型|
|---|---|
|typedef int (\* main_t )( int , char \*\*); | TypedefDecl|
|int foo(int argc, char\*\* argv); | FunctionDecl |
|int i; | VarDecl|
|AST的入口节点 |TranslationUnitDecl|

Decl系列的类继承图以及常用方法可以看[Decl](https://clang.llvm.org/doxygen/classclang_1_1Decl.html)

##### 2.Stmt
Stmt是定义（有内存分配的）、函数调用、表达式等，比方：

|语句|类型|
|---|---|
|{...}| CompoundStmt|
|if (...)|IfStmt|
|return -1;|ReturnStmt|

Stmt系列的类继承图以及常用方法可以看[Stmt](https://clang.llvm.org/doxygen/classclang_1_1Stmt.html)

##### 3.Expr
Expr是表达式，属于Stmt的子类，比方：

|语句|类型|
|---|---|
|foo();|CallExpr|
|char buf[1] = {0};|InitListExpr|
|new CXXNewExpr(foo)|CXXNewExpr|
|整数|IntegerLiteral|
|二目运算|BinaryOperator|
|一目操作(除了sizeof和alignof)|UnaryOperator |

Expr系列的类继承图以及常用方法可以看[Expr](https://clang.llvm.org/doxygen/classclang_1_1Expr.html)

### AST的遍历
AST的遍历是通过继承（include/clang/AST/*Visitor.h）遍历类来实现的，常用的有：
1. ConstStmtVisitor
2. ConstDeclVistor
3. RecursiveASTVisitor

##### 1. 通过ConstStmtVisitor遍历Stmt
通过ConstStmtVisitor 可以遍历属于Stmt类及其子类的节点，包括Expr。
```c++
class WalkAST : public ConstStmtVisitor<WalkAST> {
  // 递归遍历子节点，这个是必须实现的
  void VisitChildren(const Stmt *S) {
    for (const Stmt *Child : S->children())
      if (Child)
        this->Visit(Child);
  }
public:
  // 需要检查什么类型的Stmt，则重写对应的方法，如下面的这三个方法
  void VisitStmt(const Stmt *S) { 
    // do some checking
    VisitChildren(S); 
  }
  void VisitCallExpr(const CallExpr *CE) {
    // do some checking 
    VisitChildren(CE);
  }
  void VisitWhileStmt(const WhileStmt *WS) { 
    // do some checking 
    VisitChildren(WS);
  }
};
```
需要注意的是：如while(true) 这种语句即属于Stmt也属于WhileStmt，但回调的时候只会调用WhileStmt，即**优先回调子集**。
ConstDeclVistor与ConstStmtVisitor的用法相似，不再重复。

##### 2. 通过RecursiveASTVisitor遍历
```c++
class MyClass : public RecursiveASTVisitor<MyClass> {
public:
  // 需要检查什么类型的Stmt，则重写对应的方法，如下面的这三个方法，返回true表示继续遍历。
  bool VisitWhileStmt(const WhileStmt *S) {
    llvm::errs() << "VisitWhileStmt!!!!" << '\n';
    return true;
  }
  bool VisitStmt(const Stmt *S) {
    llvm::errs() << "VisitStmt!!!!" << '\n';
    return true;
  }
  bool VisitCallExpr(const CallExpr *CE){
    llvm::errs() << "VisitCallExpr!!!!" << '\n';
    return true;
  }
};
```
RecursiveASTVisitor 有三种类型的接口：
1. Traverse* 这种接口是遍历AST的入口，如TraverseDecl、TraverseStmt
2. WalkUpFrom* 这种接口会向前调用父类的WalkUpFrom
3. Visit* 这种接口会回调实际的访问节点

如果遇到像while即属于Stmt也属于WhileStmt的，则RecursiveASTVisitor不像ConstStmtVisitor只会调用子集，而是会先调用VisitStmt，再调用一遍VisitWhileStmt

### Checkers的注册回调函数(AST)
这里我们暂时只关注三个跟AST相关的回调函数（Path-sensitive的后面唠叨）:
1. ASTCodeBody               会回调函数体
2. ASTDecl<\*\*\*\*>         回调\*\*\*\*Decl类型
3. checkEndOfTranslationUnit 回调整个AST

##### 检查器
这里我们简单搭一个检查器的实现框架：
```c++
class TestChecker : public Checker
    <check::ASTCodeBody, check::ASTDecl<FunctionDecl>, 
    check::ASTDecl<VarDecl>, check::EndOfTranslationUnit> {
public:
    // Clang Static Analyzer core会回调这些注册函数，从而实现错误诊断代码逻辑
  void checkASTCodeBody(const Decl *D, AnalysisManager &mgr,
                        BugReporter &BR) const {
    MyClass Walker = MyClass();  // 参考上小节AST的遍历
    Walker.TraverseStmt(D->getBody());
  }
  void checkASTDecl(const FunctionDecl *D, AnalysisManager &Mgr,
                    BugReporter &BR) const {}
  void checkASTDecl(const VarDecl *D, AnalysisManager &Mgr,
                    BugReporter &BR) const {}
  void checkEndOfTranslationUnit(const TranslationUnitDecl *TU,
                    AnalysisManager &Mgr, BugReporter &BR) const {}
```

### AST matchers
如果你觉得上面的遍历AST的实现比较啰嗦的话，还可以通过AST matchers来实现。
AST Matchers内部帮你实现了遍历的流程，你只需要注册你关注的代码模式即可。

##### 1.通过Callback实现
```c++
// 注册id，回调中通过这个ID可以找到你想要检查的代码
constexpr llvm::StringLiteral WarnAtNode = "id";
class HandleMatch : public MatchFinder::MatchCallback {
public:
  virtual void Run(const MatchFinder::MatchResult &Result) {
    const CallExpr *Class = Result.Nodes.getStmtAs<CallExpr>(WarnAtNode);    
    // 如果注册的时候，用的是decl(XXXX),则参考这么这种
    // const CXXRecordDecl *Class = Result.Nodes.GetDeclAs<CXXRecordDecl>(WarnAtNode);
    // 可以做业务，比方提交错误报告
    ...
  }
};
MatchFinder finder;
finder.AddMatcher(stmt(hasDescendant(callExpr(
    callee(functionDecl(hasName("main")))).bind("id"))), new HandleMatch);
// 开始匹配，会遍历AST
finder.matchAST(ASTContext);
```
##### 2.更简洁的实现
通过ASTMatchFinder.h提供的match函数，可以不需要实现上面的Callback，原理其实是一样的，只不过ASTMatchFinder.h把Callback封装了。
```c++
constexpr llvm::StringLiteral WarnAtNode = "id";
// Matches返回所有已经匹配到的代码
auto Matches = match(stmt(hasDescendant(callExpr(
        callee(functionDecl(hasName("main")))).bind("id"))), new HandleMatch);
for (const auto &Match : Matches){
  // 在emitDiagnostics中可以做一些业务逻辑，比方提交日志。
  emitDiagnostics(Match, D, BR, AM, this);
}
```


### 参考资料与例子
找了2个简单的例子，供大家阅读：
推荐阅读[CStringSyntaxChecker](https://code.woboq.org/llvm/clang/lib/StaticAnalyzer/Checkers/CStringSyntaxChecker.cpp.html)来熟悉基于AST遍历的检查
推荐阅读[PointerIterationChecker](https://code.woboq.org/llvm/clang/lib/StaticAnalyzer/Checkers/PointerIterationChecker.cpp.html)来熟悉AST Matchers的检查

