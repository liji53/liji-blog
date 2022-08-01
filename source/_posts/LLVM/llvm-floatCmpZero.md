---
title: 【Clang Static Analyzer】浮点变量与零比较检查(原创)
date: 2022-07-02 09:19:31
tags:
categories: LLVM
---

- [浮点变量与零比较检查](#%E6%B5%AE%E7%82%B9%E5%8F%98%E9%87%8F%E4%B8%8E%E9%9B%B6%E6%AF%94%E8%BE%83%E6%A3%80%E6%9F%A5)
    - [分析可行性](#%E5%88%86%E6%9E%90%E5%8F%AF%E8%A1%8C%E6%80%A7)
        - [1.测试文件](#1-%E6%B5%8B%E8%AF%95%E6%96%87%E4%BB%B6)
        - [2.确定回调函数](#2-%E7%A1%AE%E5%AE%9A%E5%9B%9E%E8%B0%83%E5%87%BD%E6%95%B0)
    - [代码实现](#%E4%BB%A3%E7%A0%81%E5%AE%9E%E7%8E%B0)
        - [1.回调函数实现](#1-%E5%9B%9E%E8%B0%83%E5%87%BD%E6%95%B0%E5%AE%9E%E7%8E%B0)
        - [2.获取左值右值的符号值](#2-%E8%8E%B7%E5%8F%96%E5%B7%A6%E5%80%BC%E5%8F%B3%E5%80%BC%E7%9A%84%E7%AC%A6%E5%8F%B7%E5%80%BC)
        - [3.检查符号值是否为0](#3-%E6%A3%80%E6%9F%A5%E7%AC%A6%E5%8F%B7%E5%80%BC%E6%98%AF%E5%90%A6%E4%B8%BA0)
        - [4.提交错误报告](#4-%E6%8F%90%E4%BA%A4%E9%94%99%E8%AF%AF%E6%8A%A5%E5%91%8A)

# 浮点变量与零比较检查
在C/C++中，不可将浮点变量用"=="或"!="与数字直接比较，而应该转化成">="或"<="此形式进行比较。
这条检查在CSA的已有检查规则中并不存在，因此我们自己来实现下。这里我们只给出实现类，但checker的注册、CMakelist的添加等，可以回看[CSA安装和运行](https://liji53.github.io/2022/02/23/LLVM/llvm-environment/)

### 分析可行性
一般来说，要实现一个CSA checker，我们要分析这么一个过程：
1. 确定检查的场景，即待check的文件，明确代码可能会以什么样的形式存在
2. 根据上面的场景，确定checker什么时候进行检查，即checker的回调函数
3. 确定是否存在上下文依赖，如资源类型的检查，就需要检查上下文状态
4. 实现代码，在实现过程中，需要结合AST、Exploded graph来不断确认程序状态
5. 当检查出问题之后，确定是否需要继续检查，还是停止检查，确定错误报告

##### 1.测试文件
下面是自测时用的代码，列举了常用的场景，但像"if(浮点变量)"和"if(0.0 == 浮点变量)"这种场景, 这篇文章暂时不考虑在内。
```c++
// 为了代码展示的简洁性，尽量把能一行展示的放在一起
extern double fabs(double x);
const double CNST_DOUBLE_ZERO = 0.00000001;
struct compound { double a; int b; };

int func(float float_var, struct compound C)
{
	int i = 2, j = 0;
	int* x = &j;
	int& y = j;
	int** z = &x;
	double k = 0.0;
	struct compound D = {3.14, 0};
	// 需要检查出来的场景
	if (float_var == 0) {}      // == 操作符场景
	if ((float_var) != (0)) {}  // != 操作符场景
	if (0 == float_var - 1) {}  // float为BinaryOperator场景
	if (j == float_var) {}      // 变量场景
	if (*x == float_var) {}     // 指针场景
	if (**z == float_var) {}    // 二级指针场景
	if (y == float_var) {}		// 引用场景
	if ((0 == k && i - 2 == float_var) || fabs(float_var - 1) == 0) {}  // 多层表达式的场景
	if (0 == C.a || 0 == C.a - 1) {}  // float是struct成员变量的场景
	if (D.b == C.a) {}
	// 不要误报的场景
	if (i == 0) {}
	if (CNST_DOUBLE_ZERO == float_var) {}
	if (-CNST_DOUBLE_ZERO < float_var < CNST_DOUBLE_ZERO) {}
	if (fabs(float_var - 4) < CNST_DOUBLE_ZERO) {}
	return 1;
}
```

##### 2.确定回调函数
本次检查比较简单，我们只需要考虑"=="和"!="这两种表达式，同时由于我们不关心表达式计算后的结果，而只关心表达式的左值和右值是否为0，因此我们的回调函数可以确定为check::PreStmt\<BinaryOperator\>

### 代码实现

##### 1.回调函数实现
在回调函数中，我们首先要缩小检查范围，如本例我们只检查：
1. "=="和"!="的操作符
2. 左值或右值是浮点数的。
3. 左值或右值等于0的

```c++
class FloatCmpZeroChecker : public Checker<check::PreStmt<BinaryOperator>> {
public:
  // 注册回调BinaryOperator
  void checkPreStmt(const BinaryOperator *B, CheckerContext &C) const {
    // 过滤 非"等于"的操作符
    BinaryOperator::Opcode Op = B->getOpcode();
    if (Op != BO_EQ && Op != BO_NE) {return;}
    // 左值或右值必须有一项是float或double类型
    const auto *Rhs = B->getRHS();
    const auto *Lhs = B->getLHS();
    const ASTContext &Ctx = C.getASTContext();
    CanQualType CanRhsTy = Ctx.getCanonicalType(Rhs->getType()); 
    CanQualType CanLhsTy = Ctx.getCanonicalType(Lhs->getType());
    // 其实由于类型转化的存在，只有有一项是浮点类型，2者都将是浮点类型
    if (!isFloatType(CanRhsTy, Ctx) && !isFloatType(CanLhsTy, Ctx)) {
      return;
    }
    // 检查左值是否为0
    if (Optional<DefinedSVal> L_DV = get_definedSVal(Lhs, C)) {
      checkZero(L_DV, C);
    }    
    // 检查右值是否为0
    if (Optional<DefinedSVal> R_DV = get_definedSVal(Rhs, C)) {
      checkZero(R_DV, C);
    }
  }
  bool isFloatType(CanQualType CanTy, const ASTContext &Ctx) const {
    return Ctx.FloatTy == CanTy || Ctx.DoubleTy == CanTy ||
           Ctx.LongDoubleTy == CanTy;
  }
}
```

##### 2.获取左值右值的符号值
先给出代码：
```c++
void checkZero(const Expr *expr, CheckerContext &C) const {
  // 忽略'()' 和 类型转化
  const Expr *org_expr = expr->IgnoreParenCasts();
  SVal val_org = C.getSVal(org_expr);
  // 本身是float类型(UnKnownVal)，则过滤
  Optional<DefinedSVal> DV = val_org.getAs<DefinedSVal>();
  if (!DV) { return;}
  // 变量、指针、引用
  if (Optional<Loc> LocVal = val_org.getAs<Loc>()) {
    // 从memregion中获取SVal
    val_org = C.getState()->getSVal(LocVal.getValue());
    DV = val_org.getAs<DefinedSVal>();
    if (!DV) {return;}
  }
}
```
这部分涉及到内存模型的使用，因此我们多啰嗦一点，结合前面的测试文件具体看下对应哪些内容
```c++
// float_var是DeclRefExpr(AST)，它的SVal其实是Loc类型, 同理的还有局部变量j，引用变量y
if (float_var == 0)  
if (j == float_var)        // 局部变量场景
if (y == float_var) {}		// 引用场景
// float_var-1是BinaryOperator(AST), 它的SVal是float_var-1这个表达式的返回值，因此为UnKnownVal类型
if (0 == float_var - 1)
// 一级指针和二级指针都是Loc类型，需要从memregion中获取SVal
if (*x == float_var) {}     // 指针场景
if (**z == float_var) {}    // 二级指针场景
```

##### 3.检查符号值是否为0
```c++
void checkZero(Optional<DefinedSVal> DV, CheckerContext &C) const {
  ConstraintManager &CM = C.getConstraintManager();
  // 假设当前表达式的符号值，返回(可能为真，可能为假)2个状态
  ProgramStateRef stateNotZero, stateZero;
  std::tie(stateNotZero, stateZero) = CM.assumeDual(C.getState(), *DV);
  // 不可能为”非0“，即肯定为0
  if (!stateNotZero) {
    assert(stateZero);
    reportBug("float compared with zero", stateZero, C);
    return;
  }
}
```

##### 4.提交错误报告
```c++
// 获取错误内容的表达式
static const Expr *getDenomExpr(const ExplodedNode *N) {
  const Stmt *S = N->getLocationAs<PreStmt>()->getStmt();
  if (const auto *BE = dyn_cast<BinaryOperator>(S))
    return BE->getRHS();
  return nullptr;
}

class FloatCmpZeroChecker : public Checker<check::PreStmt<BinaryOperator>> {
private:
  mutable std::unique_ptr<BuiltinBug> BT;
public:
  void reportBug(const char *Msg, ProgramStateRef StateZero, CheckerContext &C,
                 std::unique_ptr<BugReporterVisitor> Visitor = nullptr) const {
    // 生成不sink的节点，程序继续分析
    if (ExplodedNode *N = C.generateNonFatalErrorNode(StateZero)) {
      if (!BT)
        BT.reset(new BuiltinBug(this, "float compared with zero"));

      // 提交错误报告
      auto R = std::make_unique<PathSensitiveBugReport>(*BT, Msg, N);
      R->addVisitor(std::move(Visitor));
      bugreporter::trackExpressionValue(N, getDenomExpr(N), *R);
      C.emitReport(std::move(R));
    }
  }
};
```