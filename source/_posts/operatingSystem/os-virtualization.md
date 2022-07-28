---
title: OS-虚拟CPU
date: 2021-06-15 21:54:20
tags:
categories: 操作系统
---
# 虚拟CPU
面试的时候，我们经常会被问道线程与进程的区别(进程是操作系统进行资源分配(除了cpu)的基本单位，线程是进行cpu调度的基本单位)。
但这次我不是唠嗑线程与进程的区别，而是聚焦到进程能够实现并发（迷惑我们以为拥有完整cpu）的相关技术-即虚拟CPU。
实现虚拟CPU的技术，主要靠几种底层机制(时钟中断、软中断、上下文切换)以及一种上层策略(进程调度策略)来实现。
首先我们看下进程的状态转化图(实际不止3种)：
![](Images\proc_statue.png)
这张图里红色的内容就是我们要唠叨的重点了。

### 进程切换
对于进程切换，初学者可能会理所当然的认为是操作系统在管理、切换进程，这当然没错。但是否有过这样的疑问：一个执行中的进程，如果占据了CPU，那操作系统是如何夺回CPU的，要知道CPU只会干“PC=PC+1”

##### 中断
计算机靠中断来实现从用户进程到内核的切换，中断在计算机里的重要性无可替代，简单来说中断就是打断运行中的cpu，并告诉cpu先去处理其他紧急的事情。
如果继续研究下去，就又产生2个问题，中断是如何打断cpu的，cpu怎么知道要去处理该中断的紧急事件？
中断分为：
1. 内中断：系统调用(trap)、各种异常(如缺页、地址越界)等内部事件。
2. 外中断：由外部设备产生的事件，如磁盘完成数据准备，时钟设备的一个时钟周期已到，等等。



##### 1.内中断-系统调用
为了能够顺利进入到内核，主要有2种方式(软中断和硬中断)，我们先介绍主动进入内核的办法-系统调用(通过各种异常也能进入)，通过系统调用也解决了上面权限的问题。

之前在[《c/c++反汇编》](https://liji53.github.io/2021/06/10/compileAssemblyLink/disassembly/)已经对应用层的系统调用汇编代码做过一波分析了。
现在我们再简单讲下后面的流程：通过syscall指令使执行逻辑从用户态切换到内核态，在进入到内核态之后，cpu会从指定的寄存器中读取内核代码的入口地址，进入内核代码之后，自然就可以通过rax读取系统调用号，再从system call table转到对应的处理函数中。

到这里我们就会产生一些困惑，因为从用户态到内核态，硬件到底做了什么呢？在《Operating Systems: Three Easy Pieces》写道：硬件会把cpu的模式从用户模式切换到内核模式，同时会保存寄存器到内核栈，但实际呢？
其实最好的答案还是亲自到官方上找相应cpu架构的指令集。比方[intel](https://www.intel.cn/content/www/cn/zh/developer/articles/technical/intel-sdm.html)，从Intel® 64 and IA-32 Architectures Software Developer's Manual Volume 2B: Instruction Set Reference, M-U这一章节就有syscall的描述

最后我们再来理解下这个流程：
![](Images\set_trap_table.png)

##### 2.时钟中断
而另一种被动的进入内核的方法就是时钟中断，一旦时钟中断产生，cpu就会自动运行到内核态，怎么知道去哪里运行内核呢，跟前面的做法一样，在启动时给硬件设置对应的处理时钟中断地址。
依靠这个中断，我们就能把进程按照时间段进行切分(进程运行的基本时间单位)，一个进程运行一段时间，然后硬件定时器发生一个时钟中断，于是进程就被动的进入内核，内核再根据调度策略决定下面运行哪个程序，如此反复。
简单看下来自xv6的源码：
```c
timerinit()
{
  ...
  // prepare information in scratch[] for timervec.
  // scratch[0..2] : space for timervec to save registers.
  // scratch[3] : address of CLINT MTIMECMP register.
  // scratch[4] : desired interval (in cycles) between timer interrupts.
  uint64 *scratch = &timer_scratch[id][0];
  scratch[3] = CLINT_MTIMECMP(id);
  scratch[4] = interval;
  // 把定时器的参数写入到mscratch寄存器，timervec中需要读取对应的参数，并设置定时器。
  w_mscratch((uint64)scratch);

  // set the machine-mode trap handler.
  //在启动时给硬件设置对应的处理时钟中断的地址。
  w_mtvec((uint64)timervec);

  // enable machine-mode interrupts.
  w_mstatus(r_mstatus() | MSTATUS_MIE);

  // enable machine-mode timer interrupts.
  w_mie(r_mie() | MIE_MTIE);
}
```
进程的切换主要靠时钟中断、系统调用，流程图：
![](Images\interrupt_table.png)

### 上下文切换
从上面的进程切换中，我们已经了解底层实现的机制，但我们知道寄存器只有这么一些，切换后进程如何回到一开始的状态呢？
上下文切换主要分为2部分(其实还应该包括页表的切换等)：
1. 硬件自动保存的寄存器
2. 当OS决定切换时，手动保存的寄存器

其中通过软件保存的寄存器，我们看下xv6的代码：
```c
.globl swtch
swtch:
    /// a0寄存器保存了c->context的地址
    sd ra, 0(a0)
    sd sp, 8(a0)
    sd s0, 16(a0)
    ...
    /// a1寄存器保存了p->context的地址
    ld ra, 0(a1)
    ld sp, 8(a1)
    ld s0, 16(a1)
    ...  
    ret

// Saved registers for kernel context switches.
// 上下文的结构体
struct context {
  uint64 ra;
  uint64 sp;
  // callee-saved
  uint64 s0;
  uint64 s1;
  ...
};
/// 参数1: 当前进程的上下文
/// 参数2: 切换进程的上下文
swtch(&c->context, &p->context);
```

### 进程调度策略(MLFQ)
在讲调度策略之前，我们需要先了解下OS进行进程切换时根据什么来选择哪个进程：
1. 周转时间： Tturnaround = Tcompletion − Tarrival
2. 响应时间： Tresponse = Tfirstrun − Tarrival

##### 2种基本算法
最短作业优先(Shortest Job First):
这是一种周转时间最好的算法，它的策略就是根据程序运行时间(最短优先)来决定运行哪个程序，但这种算法有几个问题：
1. OS并不知道进程的运行时长是多少
2. 响应时间长，需要等最短作业的程序运行完，才响应，可能存在饿死现象

轮询调度算法(Round Robin):
这种算法就是通过定时器进行时间切片，轮换的切换进程，进程都是平等的。这种算法拥有最好的响应时间与公平性。
但它最大的问题就是周转时间太长，每个进程都是最长运行时间

##### 多级反馈队列(现代OS的主流)
这种算法就是结合了SJF和RR算法的优点，使程序有较好的周转时间和响应时间。

首先看下这个算法怎么结合SJF算法的：通过对进程进行优先级分类，把进程放到不同的优先级的队列中。
再看它怎么结合RR算法的：对于同一优先级的程序，使用RR算法，轮换运行进程，就可保证它们的响应时间
因此它的快照长这样：
![](Images\multi_level_feedback_queue.png)
根据这张图，我们得到MLFQ的2条基本规则：
一. 如果Ａ的优先级大于Ｂ的优先级，运行Ａ，不运行Ｂ
二. 如果Ａ的优先级等于Ｂ的优先级，论转运行Ａ和Ｂ

但之前在讲SJF算法时，它还存在一个问题即OS并不知道进程的运行时间是多长,如果不知道他的运行时间，我们怎么对进程进行分类呢？
其实OS会根据观察进程的行为，来动态调整它的优先级．比方如果进程一直占用CPU，OS就认为它的优先级低，而如果一个进程在用完时间片之前就放弃cpu则认为它是高优先级。

因此我们再得到MLFQ的2条基本规则：
三. 进程第一次加载，把该进程放在最高优先级
四. 如果进程用完时间片则降低优先级；如果没用完就放弃cpu则保持当前优先级

##### 再优化下MLFQ
基于上面的MLFQ算法，我们思考下MLFQ这个算法还存在什么问题？
1. 存在饥饿现象，即如果有很多的高优先级进程，那么低优先级的进程很可能长时间得不到运行
2. 用户可以操控调度器，简单来说就是进程可以在时间片结束之前，假装放弃cpu，从而让进程一直保持在高优先级

我们先来看怎么解决第一个问题，即MLFQ的又一条规则：
五. 一段时间之后，将所有进程都放到最高优先级（重置）

例子如图所示：
![](Images\reboot_prior.png)

下面我们再看如何解决第二个问题，这个问题的根源是时间片是固定的，因此进程只要计算出计算片的时长就能利用了，解决办法就是优化第四条规则：
四. 给进程分配不固定的时间，一旦进程用完时间，就降低优先级。
例子如图所示：
![](Images\time_allotment.png)

##### MLFQ的进阶
如何让MLFQ工作更高效，更公平，这就需要根据环境调整它的参数，主要涉及到以下的参数：
1. 优先级队列的个数
2. 每个队列，分到的时间片长度
3. 多久时间重置进程优先级

这里给出书中提到的solaris系统的参数
1. 60个队列
2. 高先级的进程分到的时间最短20ms，低优先级的进程时间片最长几百毫秒
3. 每隔1秒重置一次进程优先级

### 参考资料
书籍：
《Operating Systems: Three Easy Pieces》[线上书籍](https://pages.cs.wisc.edu/~remzi/OSTEP/)
《Modern Operating Systems》（第四版）
xv6代码：
https://github.com/mit-pdos/xv6-riscv.git
PS：
对于新手特别推荐阅读《Operating Systems: Three Easy Pieces》英文原版