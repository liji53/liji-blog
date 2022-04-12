---
title: OS-虚拟memory
date: 2021-06-19 16:42:07
tags:
categories: 操作系统
---
# 虚拟memory
我们知道每个进程都有自己的连续内存空间，比方在64位系统中进程的内存布局：
![](Images\memory_layout.png)
让进程误以为自己拥有完整的内存空间，这就是虚拟内存。而实现虚拟内存技术主要从以下几个角度考虑：
1. 透明，不应该让用户进程感知到物理内存（基本功能）
2. 高效，每一条指令都会跟内存打交道，效率是最重要的指标之一
3. 保护，不光是进程间的保护，还包括内部数据的保护比方代码段只读属性

类似[虚拟cpu](https://liji53.github.io/2021/06/15/operatingSystem/os-virtualization/)的实现，虚拟内存也需要一些底层机制与上层策略

### 分页
分页是实现虚拟内存的基本机制，简单来说就是把虚拟地址及物理地址分成诺干个固定大小的页(一般4K)，每个页会被OS(或MMU)映射到实际的物理内存，被映射的物理内存页称为页框(page frame)。
##### 1. 页表和页表项
承载这个映射关系的数据结构叫做页表(page Table)(数组)，页表的每一项数据即页表项(page table entry)它的结构如下：
![](Images\page_table_entry.png)
其中PFN代表page frame number(页框号) 即实际物理地址的页号
其他位的含义这里简单介绍下：
present bit(P)：表示这个页是否在物理内存中，如果为0表示在硬盘中（缺页用到）
accessed bit(A)：表示这个页是否最近被访问（缺页用到）
dirty bit(D)：表示这个页是否写入过数据，如果为1则需要定时写入硬盘
read/write bit(R/W)：表示这个页是否可写
user/supervisor bit(U/S)：表示这个页属于用户模式还是内核模式能访问
PWT,PCD,PAT,G：跟硬件缓存相关

##### 2. 如何映射
虚拟地址由2部分(后面讲多级页表时再更新)组成：
1. VPN：virtual page number 即虚拟页号
2. offset：页内偏移

将虚拟地址转为物理地址，如下图(这里假设6位的地址空间，16byte的页大小)：
![](Images\address_translation.png)
图中Address Translation就是从页表中获取数据得到PFN再进行转换，但这里有2个问题需要搞清楚：
1. Address Translation由谁来完成
2. 页表到底在哪里

现代的操作系统Address Translation一般由MMU这个硬件来完成，我们先来看地址转化的伪代码（这段伪代码后面还会继续完善）：
```c
// 从虚拟地址中提取VPN
VPN = (VirtualAddress & VPN_MASK) >> SHIFT
// PTBR指的是页表的基地址，PTE指页表项
// 从页表中找到当前虚拟地址的页表项的地址
PTEAddr = PTBR + (VPN * sizeof(PTE))
// 获取当前页表项（注意：这里需要访问内存）
PTE = AccessMemory(PTEAddr)
// 检查访问的虚拟地址是否合法
if (PTE.Valid == False)
    RaiseException(SEGMENTATION_FAULT)
else if (CanAccess(PTE.ProtectBits) == False)
    RaiseException(PROTECTION_FAULT)
else
    // 转化成物理地址
    offset = VirtualAddress & OFFSET_MASK
    PhysAddr = (PTE.PFN << PFN_SHIFT) | offset
    // 访问实际的物理内存
    Register = AccessMemory(PhysAddr)
```
如果Address Translation这个动作由操作系统来完成，意味着每次访问内存都需要2次，性能直接下降了一半以上。而通过MMU则可以缓存常用的页表项，因此大部分时候只需访问一次内存。
页表位于内存中，只需要告诉MMU页表的基地址，MMU就可以自动根据VPN计算出页表的偏移从而取到PTE。
页表不需要通过虚拟地址映射，由OS告诉硬件页表的物理地址。(在进程切换时，通过设置CR3寄存器来告诉页表地址)

##### 3.遗留问题
根据上面的算法，我们的内存管理存在两个严重的问题：
1. 正如前面所说，地址转化的时候需要访问2次内次，效率低下
2. 页表如果采用1维数组的方式实现，在32位系统中1个进程的虚表就占据了4M，而如果是64位系统，则。。。

### TLB
第一个问题是如何如何让mmu的转化更快，这就需要缓存(TLB)进行帮助.
##### 1.什么时候用缓存
我们再来看下上面的伪代码：PTE = AccessMemory(PTEAddr) 如果这次的内存访问，能提前通过硬件缓存起来，这样就能减少一次内存访问了，这就是TLB的作用。
缓存作为一种常用技术，它的有效性常常依赖局部性原理：
1. 时间局部性：最近使用的数据大概率再次使用
2. 空间局部性：数据附近的数据很可能被使用

TLB正是基于这2个局部性原理，从而加快页表的查询

##### 2.更新算法
现在基于TLB，来更新地址转化的算法：
```c
VPN = (VirtualAddress & VPN_MASK) >> SHIFT
// 从TLB中查找页表项
(Success, TlbEntry) = TLB_Lookup(VPN)
if (Success == True) // 缓存命中
    if (CanAccess(TlbEntry.ProtectBits) == True)
        Offset = VirtualAddress & OFFSET_MASK
        PhysAddr = (TlbEntry.PFN << SHIFT) | Offset
        AccessMemory(PhysAddr)
    else
        RaiseException(PROTECTION_FAULT)
else // TLB 没找到
    PTEAddr = PTBR + (VPN * sizeof(PTE))
    PTE = AccessMemory(PTEAddr)
    if (PTE.Valid == False)
        RaiseException(SEGMENTATION_FAULT)
    else if (CanAccess(PTE.ProtectBits) == False)
        RaiseException(PROTECTION_FAULT)
    else
        // 更新TLB
        TLB_Insert(VPN, PTE.PFN, PTE.ProtectBits)
    // 重新执行指令
    RetryInstruction()
```
这个算法中，如果TLB miss，则通过页表查找页表项，找到之后再更新TLB，并重新执行指令。
TLB miss的过程其实即可以由MMU完成也可以由OS来完成，比方RISC指令集的MIPS就是通过OS完成的，而像X86这些CISC指令集的架构则通过硬件完成TLB miss
因此，如果由OS完成TLB miss，则是以下算法：
```c
VPN = (VirtualAddress & VPN_MASK) >> SHIFT
// 从TLB中查找页表项
(Success, TlbEntry) = TLB_Lookup(VPN)
if (Success == True) // 缓存命中
    if (CanAccess(TlbEntry.ProtectBits) == True)
        Offset = VirtualAddress & OFFSET_MASK
        PhysAddr = (TlbEntry.PFN << SHIFT) | Offset
        AccessMemory(PhysAddr)
    else
        RaiseException(PROTECTION_FAULT)
else // TLB 没找到
    // 通过产生中断，让OS来查找页表项，并更新TLB
    RaiseException(TLB_MISS)
```

##### 3.TLB项
TLB其实也是一个数组，一般有32、64或者128项，它的每一项内容如下：
![](Images\TLB_entry.png)
VPN：虚拟页号,这里只有19位，因为操作系统的虚拟地址占了一半(MIPS架构)
PFN：物理页号，有24位，因此最高可以支持64G的物理内存
G：全局共享位，如果为1表示这个虚拟地址被所有进程共享（可以用于共享库）
ASID：用于区分哪个进程，OS在进程切换时通过上下文切换，设置ASID的寄存器，这样MMU就能区分进程了。
C：内存页是如何被缓存的(跟硬件相关)
D：这个内存页是否被写入过
V：表示这个页的转化是否合法(PTE的V表示的这个页OS有没有分配)

看到这里，其实我自己有几个未解之谜：
1. 硬件是如何根据VPN来查找TLB entry的？从它的数据结构来看，它需要一项一项的比较才行
2. ASID只有8位，如果进程超过256个，怎么表示？

不管如何，TLB作为底层机制，它的核心作用就是缓存部分PTE，从而加快MMU的地址转化

### 多级页表
多级页表用于解决一维线性页表占用内存大的问题。
##### 1.新的数据结构
![](Images\multi_Level_page.png)
这部分内容个人觉得存粹就是数据结构的知识了，通过增加一层页目录，从而让大部分无用的虚拟地址不在占用页表空间。
这里仅仅展示了2级的页表，在实际linux内核中，为了支持64位地址空间，会有4级页表。

##### 2.新的转化
由于页表的数据结构改变，因此映射的方式也需要做细微调整。
内核在设计时一个页表、页目录的大小应该要刚好等于页大小。这里我们以2级页表、4K页大小、32位地址空间做示例，如果页表项的大小为4byte，则页目录刚好占10位。

它的虚拟地址结构如下：
![](Images\vritual_address.png)
它的MMU地址转化伪代码：
```c
VPN = (VirtualAddress & VPN_MASK) >> SHIFT
(Success, TlbEntry) = TLB_Lookup(VPN)
if (Success == True) // TLB Hit
    ...  //参考上一节的代码
else // TLB Miss
    // 页目录的索引
    PDIndex = (VPN & PD_MASK) >> PD_SHIFT
    // 找到对应页目录项
    PDEAddr = PDBR + (PDIndex * sizeof(PDE))
    PDE = AccessMemory(PDEAddr)
    if (PDE.Valid == False)
        RaiseException(SEGMENTATION_FAULT)
    else
        // 页表的索引
        PTIndex = (VPN & PT_MASK) >> PT_SHIFT
        // 从页目录项中找到页表的基地址，在加上页内偏移地址，得到页表项的地址
        PTEAddr = (PDE.PFN << SHIFT) + (PTIndex * sizeof(PTE))
        PTE = AccessMemory(PTEAddr)
        if (PTE.Valid == False)
            RaiseException(SEGMENTATION_FAULT)
        else if (CanAccess(PTE.ProtectBits) == False)
            RaiseException(PROTECTION_FAULT)
        else
            TLB_Insert(VPN, PTE.PFN, PTE.ProtectBits)
            RetryInstruction()
```

### 缺页
从前面的分页机制中，我们已经实现了虚拟内存的功能，让用户感知不到物理内存，但物理内存空间毕竟是有限的，如果进程，把物理空间用完了，这怎么办呢？
这就需要再制造一个假象，让进程误以为有无穷大的物理内存。内核通过将物理内存的页置换到硬盘，实现这个假象。
##### 缺页处理
如果发生TLB miss，并且虚拟地址映射的物理页不存在时(或物理页满)，则需要缺页处理，新的伪代码如下：
```c
VPN = (VirtualAddress & VPN_MASK) >> SHIFT
(Success, TlbEntry) = TLB_Lookup(VPN)
if (Success == True) // TLB Hit
    ...  //参考前面的代码
else // TLB Miss
    PTEAddr = PTBR + (VPN * sizeof(PTE))
    PTE = AccessMemory(PTEAddr)
    if (PTE.Valid == False)
        RaiseException(SEGMENTATION_FAULT)
    else
        if (CanAccess(PTE.ProtectBits) == False)
            RaiseException(PROTECTION_FAULT)
        else if (PTE.Present == True)
            TLB_Insert(VPN, PTE.PFN, PTE.ProtectBits)
            RetryInstruction()
        else if (PTE.Present == False)
            // 触发缺页中断，由OS处理中断
            RaiseException(PAGE_FAULT)
```
一旦没有找到物理页，硬件就会触发page fault，于是OS就去处理缺页中断，它的伪代码如下：
```c
// 找一个空闲的物理页。
PFN = FindFreePhysicalPage()
if (PFN == -1) // no free page found
    PFN = EvictPage() // 通过页置换算法，淘汰页
    DiskRead(PTE.DiskAddr, pfn) // 从硬盘中读取代码、数据
    PTE.present = True // 表示在物理内存中了
    PTE.PFN = PFN // 
    RetryInstruction() // 从新执行指令
```
前面PTE中提到的present bit在这里终于有用武之地了

### 页置换策略
前面讲了缺页的处理，但哪个内存页置换出去呢？这关乎到进程的内存访问效率，因此页置换算法至关重要。
##### LRU算法
如果OS能知道进程未来需要访问哪个内存页，就可以根据哪个内存页最后使用就把这个页置换出去，这种算法无疑是最好的，但这就跟进程调度算法一样，OS并不知道未来是怎么样的。
虽然未来不可知，但我们可以通过历史来推断未来，经典的算法LRU和LFU就是基于历史的使用情况来推断未来的，其实它的思想跟前面讲TLB缓存的局部性思想是一致的。

由于要实现精准的LRU算法，效率往往比较低下，因此一种近似LRU的算法在OS中更加常见：
![](Images\LRU_clock.png)
这种算法需要硬件和OS配合，当访问内存页时，由MMU硬件设置页表项的access bit(reference bit)为1，表示这个页最近被访问了。
当OS在处理缺页中断时，会遍历环形链表，如果reference bit位为0，则说明最近没有使用，因此可以置换，如上图所示。
进一步完善这个算法：如果页面写入过数据，则最好不要置换，因为OS后面会再次把页写入到磁盘，如果加上dirty bit，有以下优先级：
1. reference bit为0，且dirty bit为0。优先级最高
2. reference btt为0，且dirty bit为1
3. reference btt为1，且dirty bit为1。优先级最低，需要经过一轮循环之后

### 参考资料
书籍：《Operating Systems: Three Easy Pieces》
       [线上书籍](https://pages.cs.wisc.edu/~remzi/OSTEP/)
