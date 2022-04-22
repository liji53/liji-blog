---
title: c/c++反汇编
date: 2021-06-10 19:51:03
tags:
categories: 编译、汇编、链接
---
# c/c++反汇编
有人说编程的最高境界是人机合一，为了达到这个境界，我们开始学习反汇编，做到写一段代码，脑中就自动反射出相应的汇编代码
本文我们分析c/c++常用的语句，看它的汇编代码是怎样的，下面例子都是linux上编译的（对新手来说，直接用vs看反汇编，缺少了思考的过程）

### 基本语句反汇编
编译器不同、优化选项不同会导致生成的汇编代码不一致，接下来的汇编代码是gcc在没有优化的情况下编译出来的。

##### 1.全局变量与局部变量
```c++
// 没有指令，编译器会把这个全局变量放到.data段
/*Hex dump of section '.data':
  0x00601020 00000000 11000000                   ........
*/
int g = 0x11;
int main(){
    // 计算过程：0x4004d1 + 0xa(指令长度) + 0x200b49 = 0x601024（g的地址）
    /* 指令：  
    4004d1:	c7 05 49 0b 20 00 88 	mov    DWORD PTR [rip+0x200b49],0x88  # 601024 <g>
    4004d8:	00 00 00 
    */
    g = 0x88;
    // mov    DWORD PTR [rbp-0x4],0x1
    int a = 1;
}
```
全局变量放在数据段中，在链接时修改引用它的指令的地址，可参考[《静态链接》](https://liji53.github.io/2021/05/20/compileAssemblyLink/staticLd/)
局部变量放在栈中。

##### 2.引用与指针
```c++
// mov    DWORD PTR [rbp-0x14],0x1
int a = 1;
// lea    rax,[rbp-0x14]
// mov    QWORD PTR [rbp-0x8],rax
int* p = &a;
// lea    rax,[rbp-0x14]
// mov    QWORD PTR [rbp-0x10],rax
int& r = a;
```
从底层实现来看，引用和指针完全没有区别，引用也占用栈空间。

##### 3.const变量
```c++
// mov    DWORD PTR [rbp-0x10],0x88
const int c = 0x88;
// lea    rax,[rbp-0x10]
// mov    QWORD PTR [rbp-0x8],rax
int* a= (int*)&c;
// mov    rax,QWORD PTR [rbp-0x8]
// mov    DWORD PTR [rax],0x22
*a = 0x22;
// mov    DWORD PTR [rbp-0xc],0x88
int b = c;
```
被const修饰之后的变量其实与普通变量没有任何区别，只不过编译器在遇到const修饰的变量时会直接优化成常量值。

##### 4.if和switch
```c++
// cmp    DWORD PTR [rbp-0x4],0x0
// jne    4004e7 <main+0x1a>     ; else的地址
if(a == 0){...;}
// cmp    DWORD PTR [rbp-0x4],0x1
// jne    4004f6 <main+0x29>     ; else的地址
else if(a == 1){...;}
else{...;}
```
有多少个if分支，就有多少个cmp指令
```c++
/*cmp    DWORD PTR [rbp-0x4],0x5
  ja     400518 <main+0x4b>
  mov    eax,DWORD PTR [rbp-0x4]
  mov    rax,QWORD PTR [rax*8+0x4005b0]
*/
switch(i){
    case 1: ...;break;
    case 2: ...;break;
    case 3: ...;break;
    case 4: ...;break;
    case 5: ...;break;
    default: break;
}
```
switch 不管case分支多少(根据我的测试要5个分支及以上)，采用数组索引的方式，直接跳转，相比if语句不仅占用空间小，而且耗时上是O(1)的算法。
如果case 不连续怎么办？按照《老码识途》所说，自己多动手思考下吧。

##### 5.数组与结构体
```c++
// mov    QWORD PTR [rbp-0x20],0x0
// mov    QWORD PTR [rbp-0x18],0x0
// mov    DWORD PTR [rbp-0x10],0x0
int a[5] = {0};
// mov    DWORD PTR [rbp-0x30],0x1
// mov    DWORD PTR [rbp-0x2c],0x2
typedef struct cStruct{int a; int b;}cStruct;
cStruct c = {1,2}; 
```
数组和结构体的内存布局一样，无非就是编译器给我们提供了一个自动求取偏移量的方法。

### 函数的反汇编
每个函数的开头和结尾大部分都有这么一段code（它的含义就不介绍了),应该做到看到类似代码本能的就想到这是一个函数：
```x86asm
  push   rbp
  mov    rbp,rsp
  ...
  pop    rbp
  ret    
# 或者
  push   rbp
  mov    rbp,rsp
  sub    rsp,0x10
  ...
  leave
  ret    
```

##### 1.函数调用
```c++
// 调用方
// mov    esi,0x4
// mov    edi,0x2
// call   4004cd <add>
// mov    DWORD PTR [rbp-0x4],eax
int s = add(2, 4);

// 被调用函数
/*
  push   rbp
  mov    rbp,rsp
  mov    DWORD PTR [rbp-0x4],edi
  mov    DWORD PTR [rbp-0x8],esi
  mov    eax,DWORD PTR [rbp-0x8]
  mov    edx,DWORD PTR [rbp-0x4]
  add    eax,edx
  pop    rbp
  ret    
*/
int add(int a, int b){return a+b;}
```
没有亲自动手反汇编之前，或者仅从老的书本了解，你可能以为是通过栈传递参数的。
但x86-64通过寄存器传递参数(参数个数小于等于6，超过6个的通过栈传递)。
你可能知道这跟fastcall一样，但其实x86-64中只有一种调用约定：调用方清理堆栈。
一般你可能要记住：
1. e[r]di 是第一个参数，e[r]si 是第二个参数，剩下的参数用到的时候百度
2. 看到rbp-xxx 的时候，你要知道这是参数，遇到rbp+xxx的时候，知道这是函数内部变量

##### 2.系统调用
我们拿mmap做为例子：
```c++
/*
  4005d0:	41 b9 00 00 00 00    	mov    r9d,0x0
  4005d6:	41 b8 ff ff ff ff    	mov    r8d,0xffffffff
  4005dc:	b9 22 00 00 00       	mov    ecx,0x22
  4005e1:	ba 03 00 00 00       	mov    edx,0x3
  4005e6:	be 00 00 02 00       	mov    esi,0x20000
  4005eb:	bf 00 00 00 00       	mov    edi,0x0
  4005f0:	e8 9b fe ff ff       	call   400490 <mmap@plt>
  4005f5:	48 89 45 f8          	mov    QWORD PTR [rbp-0x8],rax
*/
char* p=(char*)mmap(0, 128*1024, PROT_READ|PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
```
其中\<mmap@plt\>是什么，可以看我之前写的[《动态链接》](https://liji53.github.io/2021/05/26/compileAssemblyLink/dynamicLd/)
nmmap的用户层入口在glibc中，通过符号表，找到mmap的地址,并反汇编：
```shell
[liji@null disassembly]$ readelf -s /usr/lib64/libc.so.6 | grep "mmap"
   321: 00000000000f8d10   183 FUNC    WEAK   DEFAULT   13 mmap64@@GLIBC_2.2.5
   661: 00000000000f8d10   183 FUNC    WEAK   DEFAULT   13 mmap@@GLIBC_2.2.5
   874: 00000000000751b0   320 FUNC    LOCAL  DEFAULT   13 _IO_wfile_underflow_mmap
   ...
[liji@null disassembly]$ objdump -d --start-address=0xf8d10 --stop-address=0xf8e00  /usr/lib64/libc.so.6 -M intel
00000000000f8d10 <mmap>:
   ...
   # 系统调用号，在unistd.h中定义
   f8d43:	b8 09 00 00 00       	mov    eax,0x9  
   # 系统调用中断，在i386中是 int 80
   f8d48:	0f 05                	syscall 
   f8d4a:	48 3d 00 f0 ff ff    	cmp    rax,0xfffffffffffff000
   f8d50:	77 52                	ja     f8da4 <mmap+0x94>
   ...
   # 设置errno，在多线程中errno是线程局部变量
   f8da4:	48 8b 15 a5 e0 2c 00 	mov    rdx,QWORD PTR [rip+0x2ce0a5]        # 3c6e50 <.got+0x100>
   f8dab:	f7 d8                	neg    eax
   f8dad:	64 89 02             	mov    DWORD PTR fs:[rdx],eax
   ...
```
系统调用号(mmap的系统调用号是9)放在eax中，其他参数放在其他通用寄存器中(可以百度)，返回值通过rax。
关于系统调用的原理，参考这张流程图：《Operating_Systems_Three_Easy_Pieces》
![](Images\regs_restore.jpg)

##### 3.类成员函数
```c++
struct A{
    int a;
    // mov    QWORD PTR [rbp-0x8],rdi   ; this指针
    // mov    rax,QWORD PTR [rbp-0x8]   ; rax = this
    // mov    DWORD PTR [rax],0x1       ; *this = 1
    void func(){a=1;}
};
int main(){
    // mov    DWORD PTR [rbp-0x10],0x0
    A a = A();
    // lea    rax,[rbp-0x10]            ; rax = &a
    // mov    rdi,rax                   ; 传参
    // call   400520 <_ZN1A4funcEv>
    a.func();
}
```
从这段汇编代码可以看出：
1.编译器不一定给每个类都生成默认构造函数和析构函数；
2.类成员函数会隐藏参数this，通过把对象的首地址传进去，从而实现访问对象的数据
进一步：
类成员函数放在代码段，也就是说每个对象共享这个代码段，对象拥有的是数据的副本

##### 4.虚函数
虚函数的反汇编就不在这里分析了，可看[《静态链接》](https://liji53.github.io/2021/05/20/compileAssemblyLink/staticLd/)对虚表反汇编的分析

### c++11新特性的反汇编
我们选择比较简单的进行分析，而像std:function、智能指针的反汇编限于篇幅不在本次分析中。
反汇编的基础学完之后，大家可以好好利用vs和IDA，因为到目前为止，所有的分析都是静态的，而实际中动态的分析更加实用。
##### 1.右值引用和移动语义
记得编译时加上-fno-elide-constructors，强制生成对应的构造函数，否则编译器很可能优化掉无用的构造函数
```c++
struct A{
    int* a;
    A():a(new int(1)){}
    ~A(){delete a;}
/*
  400910:	mov    QWORD PTR [rbp-0x8],rdi   # this指针
  400914:	mov    QWORD PTR [rbp-0x10],rsi  # right对象的地址  
  400918:	mov    edi,0x4                   # new的参数
  40091d:	call   400690 <_Znwm@plt>        # operateor new
  400922:	mov    rdx,QWORD PTR [rbp-0x10]
  400926:	mov    rdx,QWORD PTR [rdx]       # rdx = *right
  400929:	mov    edx,DWORD PTR [rdx]       # edx = right.a
  40092b:	mov    DWORD PTR [rax],edx       # new出来的空间 = right.a
  40092d:	mov    rdx,QWORD PTR [rbp-0x8]   # 
  400931:	mov    QWORD PTR [rdx],rax       # this.a = new返回的地址
*/
    A(const A& right):a(new int(*right.a)){}
/*
  40093a:	mov    QWORD PTR [rbp-0x8],rdi   # this
  40093e:	mov    QWORD PTR [rbp-0x10],rsi  # right
  400942:	mov    rax,QWORD PTR [rbp-0x10]  
  400946:	mov    rdx,QWORD PTR [rax]        
  400949:	mov    rax,QWORD PTR [rbp-0x8]   
  40094d:	mov    QWORD PTR [rax],rdx       # a(right.a)
  400950:	mov    rax,QWORD PTR [rbp-0x10]
  400954:	mov    QWORD PTR [rax],0x0       # right.a = nullptr
*/
    A(A&& right):a(right.a){right.a = nullptr;}
};
```
从汇编来看，右值引用与普通引用是一回事，进一步来说如果引动构造函数和拷贝构造函数的实现一样，两者的汇编也一样。

##### 2.lambda匿名函数
```c++
/*  
  400540:	mov    DWORD PTR [rbp-0x4],0x1
  400547:	mov    eax,DWORD PTR [rbp-0x4]
  40054a:	mov    DWORD PTR [rbp-0x10],eax
  40054d:	lea    rdx,[rbp-0x10]                    ; 变量s的地址
  400551:	lea    rax,[rbp-0x20]                    ; lambda对象的地址
  400555:	mov    rsi,rdx                              
  400558:	mov    rdi,rax
  40055b:	call   40051e <`_ZZ4mainENUliiE_C1EOS_`> ; 构造函数
  400560:	lea    rax,[rbp-0x20]
  400564:	mov    edx,0x3
  400569:	mov    esi,0x2
  40056e:	mov    rdi,rax
  400571:	call   4004fe <_ZZ4mainENKUliiE_clEii>   ; 仿函数
  400576:	mov    DWORD PTR [rbp-0x8],eax
*/
int s = 1;
auto f = [=](int a, int b)->int {return s + a + b; };
int c = f(2, 3);
```
这里调用了2个函数：\_ZZ4mainENUliiE_C1EOS\_ 和\_ZZ4mainENKUliiE_clEii
我们用c++filt来看下他们到底是什么？
```text
_ZZ4mainENUliiE_C1EOS_
main::{lambda(int, int)#1}::main({lambda(int, int)#1}&&)
_ZZ4mainENKUliiE_clEii
main::{lambda(int, int)#1}::operator()(int, int) const
```
第一个是构造函数？还是移动构造函数？，第二个是仿函数，下面分别看下它们的实现：
```x86asm
00000000004004fe <_ZZ4mainENKUliiE_clEii>:
  ...
  400502:	mov    QWORD PTR [rbp-0x8],rdi  ; this
  400506:	mov    DWORD PTR [rbp-0xc],esi  ; 2
  400509:	mov    DWORD PTR [rbp-0x10],edx ; 3
  40050c:	mov    rax,QWORD PTR [rbp-0x8]  ; 
  400510:	mov    edx,DWORD PTR [rax]      ; 捕获的变量s
  400512:	mov    eax,DWORD PTR [rbp-0xc]  ; 
  400515:	add    edx,eax                  ; edx=s+2
  400517:	mov    eax,DWORD PTR [rbp-0x10] 
  40051a:	add    eax,edx                  ; eax = edx+3
  ...
000000000040051e <_ZZ4mainENUliiE_C1EOS_>:
  ...
  400522:	mov    QWORD PTR [rbp-0x8],rdi   ; this
  400526:	mov    QWORD PTR [rbp-0x10],rsi  ; 变量s的地址
  40052a:	mov    rax,QWORD PTR [rbp-0x8]
  40052e:	mov    rdx,QWORD PTR [rbp-0x10]
  400532:	mov    edx,DWORD PTR [rdx]       ; 值传递捕获变量s
  400534:	mov    DWORD PTR [rax],edx       ; 捕获到的变量值放lamdba对象的第一项
  ...
```
看完反汇编，让我们来反推下它的数据结构与实现代码(这部分仅个人猜测的代码，并无验证)：
```c++
namespace main{
    class lambdaIntInt{
    private:
        int s;
    public:
        lambdaIntInt(int cap):s(cap){}
        int operator()(int a, int b) const{
            return s + a + b;
        }
    };
}
```
看到这里，大家应该对编译器实现的lambda已经了然于心，至于lambda的其他用法的原理，大家自己反汇编即可。

##### 3.初始化列表
看到现在，我们应该对反汇编已经有点感觉了，因此找了初始化列表做例子(够简单)，通过仅阅读汇编代码，看看能不能猜出对应的数据结构与实现算法
```c++
template<class _E>
class initializer_list{
    // 40072e: mov    QWORD PTR [rbp-0x8],rdi
    // 400732: mov    rax,QWORD PTR [rbp-0x8]
    // 400736: mov    rax,QWORD PTR [rax+0x8]
    size_t size();
    // 400740: mov    QWORD PTR [rbp-0x8],rdi
    // 400744: mov    rax,QWORD PTR [rbp-0x8]
    // 400748: mov    rax,QWORD PTR [rax]
    const_iterator begin();
    //400757:	mov    QWORD PTR [rbp-0x18],rdi
    //40075b:	mov    rax,QWORD PTR [rbp-0x18]
    //40075f:	mov    rdi,rax
    //400762:	call   40073c <_ZNKSt16initializer_listIiE5beginEv>
    //400767:	mov    rbx,rax
    //40076a:	mov    rax,QWORD PTR [rbp-0x18]
    //40076e:	mov    rdi,rax
    //400771:	call   40072a <_ZNKSt16initializer_listIiE4sizeEv>
    //400776:	shl    rax,0x2
    //40077a:	add    rax,rbx
    const_iterator end();
};
```
函数原型已经给出，函数开头和结尾的汇编代码去掉了。
看完反汇编代码，可以自己去linux上找源码，注意：vs的实现跟gnu的实现并不一致。

### 参考资料
书籍： 
《老码识途 从机器码到框架的系统观逆向修炼之路》
《C++反汇编与逆向分析技术揭秘》
《汇编语言》
《程序员的自我修养》
