---
title: 静态链接
date: 2021-05-20 14:44:02
tags: 
categories: 编译、汇编、链接
---
# 静态链接
当程序编译成可执行文件，需要经历4个步骤，预处理，编译，汇编，链接。本文我们来看看静态链接的原理。

### elf文件格式
elf文件可以分4种，包括可执行文件、可重定位文件(.o .a)、共享目标文件(.so)、核心转储文件(core dump)
##### 1.文件结构
一个elf文件由4部分组成，文件头(elf header)、程序头部表(program header table)、节(section)或段(segment)、节区头部表(section header table),如下图：
![](Images\ELF_view.png)
下图描绘了可重定位文件的格式，也是本次讲的重点:
![](Images\relocatable_view.png)

##### 2.elf header
通过readelf -h [目标文件] 就可以查看elf文件头，它描绘了整个文件的基本属性。
![](Images\elf_header.png)
entry point address对于可重定位文件为0；对于可执行文件指向C库中的_start
由于可重定位文件没有program header，因此相关的参数都为0
##### 3.section header table
通过readelf -S [目标文件] 可以查看完整的信息（通过objdump -h 只能看到重要信息）。它描绘了elf各个section的信息。
![](Images\section_header_table.png)
每个section的name字段，需要通过.shstrtable来获取。
同时也可以看到可重定位文件的虚拟地址还没有分配，等链接之后才能确定。

##### 4.section
section的数目比较多，也可以自定义section，这里列举了系统保留的重要section，还有其他的section在动态链接的时候再讲

|name | 属性| 含义|
|---- |-----|-----|
|.text | alloc+exec| 代码段|
|.init | alloc+exec | 调用mian函数之前执行的代码段，如c++中全局对象的构造函数|
|.data | alloc+write| 数据段-初始化的全局数据|
|.bss | alloc+write | bss段-未初始化的全局数据，在装载时分配空间，不占文件空间|
|.rodata | alloc | 只读数据段|
|.symtab | | 符号表|
|.strtab | | 字符串表，保存了符号表用到的符号名，如函数名、变量名|
|.shstrtab | | section header table的字符串表，保存了section名|
|.rel.xxx | | 重定位表，对代码段或数据段需要重定位才有|
|.line | | 行号信息，描述了源程序与机器指令之间的对应关系|
|.note | | 注释信息|

##### 5.符号表
符号表作为一项section，在开发中会经常用到，符号的收集一般发生在编译的词法、语法分析阶段。
通过readelf -s [目标文件] 即可以查看符号表，如下图：
![](Images\symbol_table.png)
WEAK表示弱符号，指的是那些通过__attribute__((weak))定义的弱符号，未初始化的全局变量也属于弱符号，但显示的是GLOBAL和COM。
弱符号可以被定义多次，因此可以用来hook，比方可以hook malloc、new来定位内存问题。
COM用于重定位，当存在多个相同名字的弱符号，重定位时以占用空间最大的那个为准。
这也可以说明为什么未初始化的全局变量不放在BSS段，因为大小还没有确定，在重定位完成之后，该变量最终还是放到BSS段

### 静态链接的过程
从原理上来讲，链接器的工作就是：
1.确定虚拟地址，
2.把一些指令对其他符号地址的引用加以修正。

测试的源文件如下:
```c
extern int shared;
extern void swap(int* a, int* b);
int c = 1;
int main(){
        int a = 100;
        swap(&a, &shared);
}
```
##### 编译后链接前
读取section header table可以看到虚拟地址都是0
![](Images\flow_section_header_table.png)
读取汇编代码之后，可以看到对外部引用的函数与变量地址都为0
![](Images\objdump_d.png)
再次重申编译的工作就是修改上面2张图的地址
而编译器也为地址分配做出了贡献，比方确定了定义在源文件中的函数、数据、指令的相对地址(即本文件中的地址)

##### 修正相对地址(地址分配)
首先会对属性相同的段进行合并，比方.init和.text进行合并，这里以数据段为例，如下图:
![](Images\link.png)
在合并的过程中会计算大小、位置，再根据程序的入口地址，以及偏移地址、相对地址就可以确定虚拟地址
同时根据已经确定下来的虚拟地址更新符号表(上面符号表一节，我们可以看到符号的地址大部分为0)

##### 引用符号的重定位
经过上一步的修正，函数、变量的虚拟地址已经更新到全局符号表了，剩下的就是将引用这些全局变量、函数的地址修正。
但链接器怎么知道哪些函数、变量需要修正呢？-- 靠重定位表
通过objdump -r [目标文件] 我们看下.rel.text重定位表(如果数据段需要修正,则是.rel.data)的内容
![](Images\rel_text.png)
其中R_X86_64_32 表示通过绝对寻址修正
R_X86_64_PC32 通过相对寻址修正

修改之后，结果如下：
![](Images\objdump_d2.png)

##### 流程图
最后把自己绘制的流程图献上
![](Images\ld_flow.png)

### 实验：c++虚表的内存布局
利用上面学的知识，我们现学现用，看下c++虚表
比方，有下面源文件：
```c++
struct t_base{
        int a;
        virtual void f_call(){}
        virtual void f_show(){}
};
struct t_derive:public t_base{
        void f_call(){a=1;}
        void f_show(){a=2;}
};
int main(){
        t_base* a = new t_derive();
        a->f_call();
        a->f_show();
        t_base* b = new t_base();
        b->f_call();
        b->f_show();
}
```
##### 符号表找函数地址
通过符号表，我们看下相关的地址
```shell
51: 000000000040074a    37 FUNC    WEAK   DEFAULT   14 _ZN8t_deriveC2Ev
58: 0000000000400870    24 OBJECT  WEAK   DEFAULT   16 _ZTI8t_derive
64: 000000000040071e    21 FUNC    WEAK   DEFAULT   14 _ZN8t_derive6f_showEv
66: 0000000000400820    32 OBJECT  WEAK   DEFAULT   16 _ZTV8t_derive
67: 0000000000400860    10 OBJECT  WEAK   DEFAULT   16 _ZTS8t_derive
68: 0000000000400708    21 FUNC    WEAK   DEFAULT   14 _ZN8t_derive6f_callEv
71: 000000000040074a    37 FUNC    WEAK   DEFAULT   14 _ZN8t_deriveC1Ev
48: 0000000000400890    16 OBJECT  WEAK   DEFAULT   16 _ZTI6t_base
53: 00000000004006fe    10 FUNC    WEAK   DEFAULT   14 _ZN6t_base6f_showEv
60: 0000000000400840    32 OBJECT  WEAK   DEFAULT   16 _ZTV6t_base
61: 0000000000400734    21 FUNC    WEAK   DEFAULT   14 _ZN6t_baseC2Ev
72: 0000000000400888     8 OBJECT  WEAK   DEFAULT   16 _ZTS6t_base
74: 0000000000400734    21 FUNC    WEAK   DEFAULT   14 _ZN6t_baseC1Ev
77: 00000000004006f4    10 FUNC    WEAK   DEFAULT   14 _ZN6t_base6f_callEv
```
上面只截取了跟2个类相关的符号表，如果对c++的符号有疑问，可以通过c\+\+filt直接查看源名
```shell
[liji@null test]$ c++filt _ZTI8t_derive
typeinfo for t_derive
[liji@null test]$ c++filt _ZTV8t_derive
vtable for t_derive
[liji@null test]$ c++filt _ZTS8t_derive
typeinfo name for t_derive
```
##### 查看数据段内容
我们知道虚表是放在数据段的，而虚表在编译期间就已经确定而且只有一份，实际虚表放在.rodata数据段
```shell
[16] .rodata           PROGBITS         0000000000400800  00000800
     00000000000000a0  0000000000000000   A       0     0     32
[26] .bss              NOBITS           0000000000601040  0000102c
     00000000000000c0  0000000000000000  WA       0     0     32
```
通过查看section header table，可以知道.rodata 的起始虚拟地址是0x400800，同时在可执行文件中的偏移位置是0x800，下面我们就从文件的.rodata段中找下有没有虚表
通过命令 readelf -x .rodata [目标文件]，也可以通过hexdump 来查看
![](Images\hexdump_rodata.png)
比对符号表中的地址与rodata中数据，可以得出虚表的内存布局。
也可以看到在linux上typeinfo位于虚表的第二项（看前面符号表_ZTI8t_derive的地址正是第二项）
而且还能看出typeinfo的内存布局(除了第一项指向了bss段不知道以外，后面分别指向类名，父类的typeinfo地址)

##### 反汇编验证
最后看下通过汇编c++是如何实现多态的
```x86asm
......
  400646: mov    edi,0x10                     ; 给new传参
  40064b: call   400530 <_Znwm@plt>           ; 调用 operator new
  400650:	mov    rbx,rax                      ; new的返回值赋值给rbx
  400653:	mov    QWORD PTR [rbx],0x0          ; 给虚函数指针赋值0，构造时才初始化
  40065a:	mov    DWORD PTR [rbx+0x8],0x0      ; 给成员变量a赋值0
  400661:	mov    rdi,rbx                      ; 给t_derive构造函数传参
  400664:	call   40074a <_ZN8t_deriveC1Ev>    ; 调用t_derive构造函数
  400669:	mov    QWORD PTR [rbp-0x18],rbx     ; 对象地址(也是虚表的地址)放入栈中
  40066d:	mov    rax,QWORD PTR [rbp-0x18]     ; rax指向对象的首地址
  400671:	mov    rax,QWORD PTR [rax]          ; rax指向虚函数表
  400674:	mov    rax,QWORD PTR [rax]          ; rax指向虚函数
  400677:	mov    rdx,QWORD PTR [rbp-0x18]     ; 
  40067b:	mov    rdi,rdx                      ; 传参this
  40067e:	call   rax                          ; 调用f_call()
  400680:	mov    rax,QWORD PTR [rbp-0x18]     ; rax指向对象的首地址
  400684:	mov    rax,QWORD PTR [rax]          ; rax指向虚函数表
  400687:	add    rax,0x8                      ; 虚函数表的第二项
  40068b:	mov    rax,QWORD PTR [rax]          ; rax指向虚函数
  40068e:	mov    rdx,QWORD PTR [rbp-0x18]
  400692:	mov    rdi,rdx                      ; 传参this
  400695:	call   rax                          ; 调用f_show()
......
000000000040074a <_ZN8t_deriveC1Ev>:
  ...
  400752:	mov    QWORD PTR [rbp-0x8],rdi      ; 参数，即this指针
  400756:	mov    rax,QWORD PTR [rbp-0x8]      ; 对象地址赋值给rax
  40075a:	mov    rdi,rax                      ; 父类构造传参数
  40075d:	call   400734 <_ZN6t_baseC1Ev>      ; 调用父类的构造函数
  400762:	mov    rax,QWORD PTR [rbp-0x8]      ; 
  400766:	mov    QWORD PTR [rax],0x400830     ; 关键：给虚函数指针赋值，
                                              ; 这里的0x400830正是我们之前看到的地址
  ...
```
写的注释已经够详细了，直接下结论：
1.虚函数指针指向的是对应类的虚函数表第一项
2.虚函数指针由构造函数初始化

### 命令&参考资料
##### 用到的命令
1. hexdump [目标文件]  查看二进制内容
2. readelf -h [目标文件] 查看 elf header
2. readelf -S [目标文件] 查看 section header table
2. objdump -d [目标文件] -M intel 查看 .text 的汇编代码
5. readelf -s [目标文件] 查看符号表
6. objdump -r [目标文件] 查看重定位表
7. readelf -x [section名,如.data] [目标文件] 查看指定段的二进制内容
##### 参考资料
书籍：《程序员的自我修养》
博客：https://blog.csdn.net/mergerly/article/details/94585901