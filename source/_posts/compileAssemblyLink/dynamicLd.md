---
title: 动态链接
date: 2021-05-26 20:04:45
tags:
categories: 编译、汇编、链接
---
# 动态链接
静态链接存在2个主要问题：
1.内存、磁盘空间浪费严重，每个进程都有自己的独立内存空间，需要对每个共享库都分配内存空间
2.更新、发布需要重新编译，假如软件中其中一个库是第三方提供的，一旦更新这个库需要将整个程序重新编译链接。

动态库为了解决上面的两个问题，它的核心思想就是将链接过程推迟到运行时，并且将指令共享给多个进程。

### 动态库的文件格式
首先回看下前文静态链接中给出的一张elf格式图：
![](Images\ELF_view.png)
##### 1.program header table
只有可执行文件和动态库才有这个table，链接时会把相同属性的section合并在一起，合并后就叫segment，这么做的好处是减少内存页的碎片。
通过readelf -l [目标文件] 即可查看program header table，如下图：
![](Images\program_header_table.png)
绿色指的是会映射到虚拟内存中

##### 2.interp section 和 INTERP segment
首先看下.interp section 是什么？ -- 动态链接器的路径
```shell
[liji@null dynamic]$ readelf -S a
......
[ 1] .interp           PROGBITS         0000000000400238  00000238
     000000000000001c  0000000000000000   A       0     0     1
......
[liji@null dynamic]$ readelf -x .interp a
Hex dump of section '.interp':
  0x00400238 2f6c6962 36342f6c 642d6c69 6e75782d /lib64/ld-linux-
  0x00400248 7838362d 36342e73 6f2e3200          x86-64.so.2.
```
动态链接器用于程序运行时的重定位，与静态链接器(ld)不是一个东西。
动态链接器的位置并不是环境变量决定的，而是由ELF文件决定的，在加载动态链接器时首先会寻找.interp指向的路径。
理解了.interp section之后，INTERP segment 其实就是 .interp section。

##### 3.dynamic section
通过readelf -d [可执行文件] 即可查看.dynamic段，如下图：
![](Images\dynamic.png)
dynamic段描述了动态链接时相关的内容，比方动态链接器就是通过该段寻找动态符号表、依赖的动态库
通过ldd命令还可以获取到更全的依赖库信息
```shell
[liji@null dynamic]$ ldd a
	linux-vdso.so.1 =>  (0x00007ffc3751e000)
	libmy.so => not found
	libc.so.6 => /lib64/libc.so.6 (0x00007fb64b20d000)
	/lib64/ld-linux-x86-64.so.2 (0x00007fb64b5db000)
```

##### 4.dynsym 动态符号表 & .dynstr & .rela.xxx
这三种section 在静态链接时我们已经讲过类似的了，他们的功能其实差不多。
注意：动态符号表只保存了动态链接相关的符号，而静态符号表则保存了所有的符号
与静态符号表一样，动态符号表也需要dynstr(动态符号字符串表辅助)，即符号表只保存符号的索引，实际的内容保存在字符串表。
.rela.plt和.rela.dyn重定位表分别用于函数引用的修正和数据引用的修正，如下图：
![](Images\rela.png)
重定位的实现与静态重定位类似，都是从全局符号表中去查找，找到之后，直接修改.got表中的地址(关于got后面再讲)

##### 5.动态库相关的段
下面我们梳理下跟动态库相关的section：

|名称|属性|含义|
|---|---|---|
|.dynamic| | 动态库的整体信息 |
|.dynstr| alloc |用于辅助动态符号表，保存符号的字符串|
|.dynsym| alloc |动态符号表|
|.got|alloc+write|全局偏移表(后面讲)|
|.got.plt| alloc+write| 全局偏移表，用于plt|
|.interp | | 记录动态编译器的位置|
|.plt | alloc |procedure linkage table(后面讲)|
|.rela.plt| |对函数引用的修正，修正.got.plt|
|.rela.dyn| |对数据引用的修正，主要修正.got| 

### 动态链接的过程
文章开头我们提到了静态链接的问题，那么动态链接是如何解决多进程内存共享的呢？ -- PIC
##### 1.PIC技术-地址无关代码
在生成so的时候，需要加上-fPIC，就是为了生成地址无关代码。
它的核心思想是把指令分成需要被修改的，和不需要被修改的，进程间共享的是不需要修改的指令，而需要修改的指令(其实是需要重定位的地址)则放到数据段，并且每个进程都有一个数据段的副本。
接下来的问题就是如何进行地址访问，原理看下图：
![](Images\PIC_principle.png)
当需要访问外部模块的变量、函数时，通过数据段里的全局偏移表(got)进行间接访问。
got存储了需要重定位的地址列表，这个地址列表需要等程序运行由动态链接器进行修正
got section以及动态重定位表的内容如下：
![](Images\got_section.png)
再次强调下，静态链接中.rela直接修改的是指令和数据段引用的地址，而在动态链接中.rela修改的是got表中的地址列表。
这里还有.got.plt 也属于全局偏移表，具体有什么区别等会讲。
有没有觉得像c++的虚函数表，只不过c++的虚表在静态链接期间就确定了。

##### 2.PLT技术-延迟绑定
动态库也存在缺点，主要是对性能的影响:
1. PIC技术，每次对函数、数据的访问都需要先经过got表，再寻址
2. 当程序启动时需要动态链接器进行链接，这过程需要装载用到的动态库，然后进行符号查找，并进行重定位。

而PLT技术用于优化第二项，加快启动的速度 ：它的核心思想是当函数第一次被调用的时候才绑定
它有专门的section用于实现它的技术，没错就是.plt，我们看它的汇编实现：
```x86asm
Disassembly of section .plt:
0000000000000550 <.plt>:
 550:	ff 35 b2 0a 20 00    	push   QWORD PTR [rip+0x200ab2] # .got.plt 的第二项，即moduleid，参数2
 556:	ff 25 b4 0a 20 00    	jmp    QWORD PTR [rip+0x200ab4] # .got.plt 的第三项，即_dl_runtime_resolve()
 55c:	0f 1f 40 00          	nop    DWORD PTR [rax+0x0]
# printf的入口,第一次调用时printf地址还没确定，需要_dl_runtime_resolve()进行绑定
0000000000000560 <printf@plt>:
 560:	ff 25 b2 0a 20 00    	jmp    QWORD PTR [rip+0x200ab2] # <printf@GLIBC_2.2.5>
 566:	68 00 00 00 00       	push   0x0                      # 需要决议的函数下标，即.rela.plt的下标，参数1
 56b:	e9 e0 ff ff ff       	jmp    550 <.plt>

Disassembly of section .plt.got:
0000000000000570 <.plt.got>:
 570:	ff 25 6a 0a 20 00    	jmpq   *0x200a6a(%rip)        # 200fe0 <__gmon_start__>
 576:	66 90                	xchg   %ax,%ax
 578:	ff 25 7a 0a 20 00    	jmpq   *0x200a7a(%rip)        # 200ff8 <__cxa_finalize@GLIBC_2.2.5>
 57e:	66 90                	xchg   %ax,%ax

```
plt叫过程连接表，你也可以把它理解成一个函数指针数组，每个需要调用的动态库函数都会有自己的plt条目，如上面的printf就叫printf@plt
他的内存布局图如下：
![](Images\plt_view.png)
再来看下它的计算过程：
560(printf@plt) + 6(指令大小) + rip(当前地址) + 0x200ab2 = 0x201018 + rip(当前地址)
550 + 6(指令大小) + rip(当前地址) + 0x200ab2 = 0x201008 + rip(当前地址)
556 + 6(指令大小) + rip(当前地址) + 0x200ab4 = 0x201010 + rip(当前地址)
.got.plt的地址为：
```shell
[22] .got              PROGBITS         0000000000200fd8  00000fd8
     0000000000000028  0000000000000008  WA       0     0     8
[23] .got.plt          PROGBITS         0000000000201000  00001000
     0000000000000020  0000000000000008  WA       0     0     8
```
正好是.got.plt的2，3，4项，而第一项保存了.dynamic段的地址。
rela.plt用于修正.got.plt的地址：
![](Images\rel_plt.png)
可以看到它的地址指向.got.plt的第四项

最后我们讲讲.got 和 .got.plt 的区别：
.got 用来保存全局变量的地址以及不需要用plt技术的函数，如__gmon_start__
.got.plt 用来保存函数引用的地址

##### 流程图
从网上找到的一张流弊动态图：
![Alt Text](Images\flow.gif)


### 内核装载elf的过程
##### 1.静态链接程序的装载
![](Images\static_run.png)

##### 2.动态链接程序的装载
![](Images\dynamic_run.png)
动态链接器的运行过程：
![](Images\dynamic_ld_flow.png)

### 命令&资料
1. readelf -l [可执行文件]  查看program header table
2. readelf -d [可执行文件]  查看.dynamic section
##### 参考资料
书籍：《程序员的自我修养》
博客: https://luomuxiaoxiao.com/?p=578 