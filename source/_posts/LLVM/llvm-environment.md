
---
title: 【Clang Static Analyzer】环境 
date: 2022-02-23 02:11:25
tags:
categories: LLVM
---

- [CSA安装和运行](#CSA%E5%AE%89%E8%A3%85%E5%92%8C%E8%BF%90%E8%A1%8C)
    - [VS中构建LLVM](#VS%E4%B8%AD%E6%9E%84%E5%BB%BALLVM)
        - [1-环境预准备](#1-%E7%8E%AF%E5%A2%83%E9%A2%84%E5%87%86%E5%A4%87)
        - [2-vs打开LLVM项目](#2-vs%E6%89%93%E5%BC%80LLVM%E9%A1%B9%E7%9B%AE)
        - [3-编译并添加命令参数](#3-%E7%BC%96%E8%AF%91%E5%B9%B6%E6%B7%BB%E5%8A%A0%E5%91%BD%E4%BB%A4%E5%8F%82%E6%95%B0)
    - [添加第一个检查器](#%E6%B7%BB%E5%8A%A0%E7%AC%AC%E4%B8%80%E4%B8%AA%E6%A3%80%E6%9F%A5%E5%99%A8)
        - [1-测试文件](#1-%E6%B5%8B%E8%AF%95%E6%96%87%E4%BB%B6)
        - [2-添加checker定义](#2-%E6%B7%BB%E5%8A%A0checker%E5%AE%9A%E4%B9%89)
        - [3-添加源文件](#3-%E6%B7%BB%E5%8A%A0%E6%BA%90%E6%96%87%E4%BB%B6)
        - [4-CMakeList中添加编译目标](#4-CMakeList%E4%B8%AD%E6%B7%BB%E5%8A%A0%E7%BC%96%E8%AF%91%E7%9B%AE%E6%A0%87)
        - [5-测试](#5-%E6%B5%8B%E8%AF%95)
    - [文档资料](#%E6%96%87%E6%A1%A3%E8%B5%84%E6%96%99)
      
# CSA安装和运行
最近在利用业余时间学习和研究代码检查工具，比对了几种代码检查工具之后，决定把Clang Static Analyzer（开源+可扩展性+丰富的文档）作为学习的对象，并尝试运用到我们的项目中去。本文我们将重点围绕llvm环境搭建，并实现一个简单的static analyzer checker。

### VS中构建LLVM
参考资料：https://llvm.org/docs/GettingStartedVS.html
##### 1-环境预准备
 Visual Studio 2019+，用2017试过，但构建失败了
 Cmake
 python3.6+
 pip install psutil
##### 2-vs打开LLVM项目
 下载LLVM项目：
 ```shell
  git clone https://github.com/llvm/llvm-project.git
 ```
 用cmake生成vs项目文件
```shell
cd llvm-project
cmake -S llvm -B build -DLLVM_ENABLE_PROJECTS="clang" -DLLVM_TARGETS_TO_BUILD=X86 -DCMAKE_INSTALL_PREFIX="D:\Program Files (x86)\llvm" -Thost=x64
```
再用vs代开build目录下的LLVM.sln文件，打开内容如下：
![](Images\LLVM_vs.png)

##### 3-编译并添加命令参数
默认是debug模式，编译clang.exe需要很长时间；改成release模式，可以快很多。
编译完之后会在build目录下生成Debug/bin目录，再设置调试的命令参数，如下图所示：
![](Images\llvm_compile.png)

### 添加第一个检查器
参考资料：https://github.com/haoNoQ/clang-analyzer-guide/releases/download/v0.1/clang-analyzer-guide-v0.1.pdf
下面的例子仅仅是为了让我们能够跑起来Clang Static Analyzer，对添加checker知道流程步骤即可，如果对其中的源码存在疑问，可以看后续文章。

##### 1-测试文件
目标代码如下：
```c++
typedef int (*main_t)(int, char**);
int main(int argc, char** argv){
    main_t foo = main;
    return foo(argc, argv); // actually calls main()
}
```
我们要检查的内容就是: main函数是否在程序中被调用。

##### 2-添加checker定义
打开llvm-project\clang\include\clang\StaticAnalyzer\Checkers\Checkers.td，在alpha.core包中添加下面内容：
```C++
def FixedAddressChecker : Checker<"FixedAddr">,
  HelpText<"Check for assignment of a fixed address to a pointer">,
  Documentation<HasAlphaDocumentation>;

/// added
def MainCallChecker : Checker<"MainCall">,
  HelpText<"Check for calls to main">,
  Documentation<HasAlphaDocumentation>;
```
这个td文件，会被TableGen工具翻译，翻译成.inc头文件，用于checker的注册。
翻译之后的内容(部分)如下：
```c++
#ifdef GET_CHECKERS
......
CHECKER("osx.API", MacOSXAPIChecker, "Check for proper uses of various Apple APIs", "https://clang-analyzer.llvm.org/available_checks.html#osx.API", false)
CHECKER("alpha.core.MainCall", MainCallChecker, "Check for calls to main", "https://clang-analyzer.llvm.org/alpha_checks.html#alpha.core.MainCall", false)
```
其中CHECKER的**宏实现在BuiltinCheckerRegistration.h**中，这里不再展开。

##### 3-添加源文件
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
// PreCall是一个ProgramPoint，StaticAnalyzer在分析过程中会进行回调
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
// 注册checker
void ento::registerMainCallChecker(CheckerManager &Mgr) {
  Mgr.registerChecker<MainCallChecker>();
}

bool ento::shouldRegisterMainCallChecker(const CheckerManager &mgr) {
  return true;
}
```
##### 4-CMakeList中添加编译目标
在llvm-project\clang\lib\StaticAnalyzer\Checkers\CMakeLists.txt 中添加
```shell
  VirtualCallChecker.cpp
  MainCallChecker.cpp
  WebKit/NoUncountedMembersChecker.cpp
```
##### 5-测试
重新编译obj.clangStaticAnalyzerCheckers和clang项目之后，测试
```shell
PS E:\code> clang -cc1 -analyze -analyzer-checker="alpha.core" test.cpp
.\test.cpp:4:18: warning: Call to main [alpha.core.MainCall]
        int exit_code = foo(argc, argv); // actually calls main ()!
                        ^~~~~~~~~~~~~~~
1 warning generated.
```
可以看到，clangStaticAnalyzer确实找到了调用main函数的地方。
同样的，如果仅仅需要检查（禁用）某个函数，仿照上面的过程即可。

### 文档资料
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
Tablegen的介绍：https://llvm.org/docs/TableGen/

除了上面的官方文档，还有一篇很好用的入门文章：
CSA开发教程(16年写的)：https://github.com/haoNoQ/clang-analyzer-guide/releases/download/v0.1/clang-analyzer-guide-v0.1.pdf
优秀的博客：
https://www.zhihu.com/column/clang-static-analyzer
https://csstormq.github.io/