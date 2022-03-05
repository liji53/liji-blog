---
title: LLVM(1)
date: 2022-02-23 02:11:25
tags:
---
# Clang Static Analyzer 安装和运行
最近在利用业余时间学习和研究代码检查，经过比对了几种代码检查工具之后，决定把Clang Static Analyzer作为学习的对象，原因是开源+扩展性。本文我将重点围绕llvm环境搭建，以及实现第一个static analyzer checker
## 文档资料
学习CSA绕不开LLVM、Clang，好在LLVM的文档非常全面，以下全是官方文档链接：
LLVM主页面(有提到三者的关系)：https://llvm.org/
LLVM介绍(有跟gcc相比的优势)：https://llvm.org/pubs/2008-10-04-ACAT-LLVM-Intro.html
LLVM向导(有通过LLVM编写新语言的官方教程)：https://llvm.org/docs/tutorial/index.html
LLVM开发手册(项目结构等资料)：https://llvm.org/doxygen/
Clang主页面：https://clang.llvm.org/
Clang文档列表：https://clang.llvm.org/docs/index.html
Clang开发手册：https://clang.llvm.org/doxygen/
CSA已有检查项：https://clang.llvm.org/docs/analyzer/checkers.html
CSA开发教程：https://clang-analyzer.llvm.org/checker_dev_manual.html

除了上面的官方文档，还有这些优秀的文章：
CSA开发教程(16年写的，可与上面结合看)：https://github.com/haoNoQ/clang-analyzer-guide/releases/download/v0.1/clang-analyzer-guide-v0.1.pdf
符号执行(符号执行的原理、面临的挑战)：https://arxiv.org/pdf/1610.00502.pdf
## VS中构建LLVM
参考资料：https://llvm.org/docs/GettingStartedVS.html
#### 环境准备
 Visual Studio 2019+，我用2017的试过，反正构建失败了，于是下载了最新的2022版本
 Cmake
 python3.6+
 pip install psutil
#### vs打开LLVM项目
 下载LLVM项目：
 ```shell
  git clone https://github.com/llvm/llvm-project.git
 ```
 用cmake生成vs文件，再用vs代开LLVM.sln文件
```shell
cd llvm-project
cmake -S llvm -B build -DLLVM_ENABLE_PROJECTS="clang" -DLLVM_TARGETS_TO_BUILD=X86 -DCMAKE_INSTALL_PREFIX="D:\Program Files (x86)\llvm" -Thost=x64
```
 项目内容如下：
![](Images\LLVM_vs.png)
#### 编译Clang
默认是debug模式，编译clang需要很长时间，建议改成release模式
编译完之后会在build目录下生成Debug目录，记得加入到环境变量中
![](Images\LLVM_compile.png)

## 添加第一个检查器
参考资料：https://github.com/haoNoQ/clang-analyzer-guide/releases/download/v0.1/clang-analyzer-guide-v0.1.pdf
由于参考资料是16年的，里面的例子需要修改才能使用
#### 检查目标
c++标准中有这么一条：main函数不应该在程序中被调用。
目标代码如下：
```c++
typedef int (*main_t)(int, char**);
int main(int argc, char** argv){
    main_t foo = main;
    int exit_code = foo(argc, argv); // actually calls main()!
    return exit_code;
}
```
常规的方法可能只能查找显式的调用main，而这个例子中通过函数指针隐式调用mian。
#### 添加checker定义
打开llvm-project\clang\include\clang\StaticAnalyzer\Checkers\Checkers.td，在alpha.core包中添加下面内容：
```C++
def FixedAddressChecker : Checker<"FixedAddr">,
  HelpText<"Check for assignment of a fixed address to a pointer">,
  Documentation<HasAlphaDocumentation>;

/// added
def MainCallChecker : Checker<"MainCall">,
  HelpText<"Check for calls to main">,
  Documentation<HasAlphaDocumentation>;

def PointerArithChecker : Checker<"PointerArithm">,
  HelpText<"Check for pointer arithmetic on locations other than array "
           "elements">,
  Documentation<HasAlphaDocumentation>;
```
#### 添加源文件
在llvm-project\clang\lib\StaticAnalyzer\Checkers目录下新增文件MainCallChecker.cpp
```c++
#include "clang/StaticAnalyzer/Checkers/BuiltinCheckerRegistration.h"
#include "clang/StaticAnalyzer/Core/BugReporter/BugType.h"
#include "clang/StaticAnalyzer/Core/Checker.h"
#include "clang/StaticAnalyzer/Core/PathSensitive/CallEvent.h"
#include "clang/StaticAnalyzer/Core/PathSensitive/CheckerContext.h"

using namespace clang;
using namespace clang::ento;

namespace {
class MainCallChecker : public Checker<check::PreCall> {
  mutable std::unique_ptr<BugType> BT;

public:
  void checkPreCall(const CallEvent &Call, CheckerContext &C) const;
};
}

void MainCallChecker::checkPreCall(const CallEvent &Call,
                                   CheckerContext &C) const {
  if (const IdentifierInfo *II = Call.getCalleeIdentifier())
    if (II->isStr("main")) {
      if (!BT) 
        BT.reset(new BugType(this, "Call to main", "Example checker"));
      ExplodedNode *N = C.generateErrorNode();
      auto Report = std::make_unique<PathSensitiveBugReport>(
          *BT, BT->getDescription(), N);
      C.emitReport(std::move(Report));

    }
}

void ento::registerMainCallChecker(CheckerManager &Mgr) {
  Mgr.registerChecker<MainCallChecker>();
}

bool ento::shouldRegisterMainCallChecker(const CheckerManager &mgr) {
  return true;
}
```
#### CMakeList中添加编译目标
在llvm-project\clang\lib\StaticAnalyzer\Checkers\CMakeLists.txt 中添加
```shell
  VirtualCallChecker.cpp
  MainCallChecker.cpp
  WebKit/NoUncountedMembersChecker.cpp
```
#### 测试
重新编译之后obj.clangStaticAnalyzerCheckers和clang项目之后，测试
```shell
PS E:\code> clang -cc1 -analyze -analyzer-checker="alpha.core" test.cpp
.\test.cpp:4:18: warning: Call to main [alpha.core.MainCall]
        int exit_code = foo(argc, argv); // actually calls main ()!
                        ^~~~~~~~~~~~~~~
1 warning generated.
```
可以看到结果，确实正确找到了调用main函数的地方