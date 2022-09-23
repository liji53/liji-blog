---
title: 原子类型
date: 2022-02-04 03:24:16
tags:
categories: C++基础
---
- [原子类型](#%E5%8E%9F%E5%AD%90%E7%B1%BB%E5%9E%8B)
    - [原子类型的实现](#%E5%8E%9F%E5%AD%90%E7%B1%BB%E5%9E%8B%E7%9A%84%E5%AE%9E%E7%8E%B0)
        - [1.基本类型](#1-%E5%9F%BA%E6%9C%AC%E7%B1%BB%E5%9E%8B)
        - [2.内部类的关系与功能](#2-%E5%86%85%E9%83%A8%E7%B1%BB%E7%9A%84%E5%85%B3%E7%B3%BB%E4%B8%8E%E5%8A%9F%E8%83%BD)
        - [3.基本操作](#3-%E5%9F%BA%E6%9C%AC%E6%93%8D%E4%BD%9C)
        - [4.原子变量不能拷贝，赋值](#4-%E5%8E%9F%E5%AD%90%E5%8F%98%E9%87%8F%E4%B8%8D%E8%83%BD%E6%8B%B7%E8%B4%9D-%E8%B5%8B%E5%80%BC)
        - [5.atomic\<T\>自定义对象必须是POD](#5-atomic-lt-T-gt-%E8%87%AA%E5%AE%9A%E4%B9%89%E5%AF%B9%E8%B1%A1%E5%BF%85%E9%A1%BB%E6%98%AFPOD)
        - [6.非atomic_flag不一定无锁](#6-%E9%9D%9Eatomic-flag%E4%B8%8D%E4%B8%80%E5%AE%9A%E6%97%A0%E9%94%81)
        - [7.如果不是无锁的原子类型，其实底层实现就是mutex](#7-%E5%A6%82%E6%9E%9C%E4%B8%8D%E6%98%AF%E6%97%A0%E9%94%81%E7%9A%84%E5%8E%9F%E5%AD%90%E7%B1%BB%E5%9E%8B-%E5%85%B6%E5%AE%9E%E5%BA%95%E5%B1%82%E5%AE%9E%E7%8E%B0%E5%B0%B1%E6%98%AFmutex)
        - [8.原子读写的内部实现](#8-%E5%8E%9F%E5%AD%90%E8%AF%BB%E5%86%99%E7%9A%84%E5%86%85%E9%83%A8%E5%AE%9E%E7%8E%B0)
        - [9.默认内存顺序std::memory_order_seq_cst](#9-%E9%BB%98%E8%AE%A4%E5%86%85%E5%AD%98%E9%A1%BA%E5%BA%8Fstd-memory-order-seq-cst)
        - [10.compare_exchange_weak和compare_exchange_strong的区别](#10-compare-exchange-weak%E5%92%8Ccompare-exchange-strong%E7%9A%84%E5%8C%BA%E5%88%AB)
        - [11.atomic_flag的实现](#11-atomic-flag%E7%9A%84%E5%AE%9E%E7%8E%B0)
    - [内存顺序](#%E5%86%85%E5%AD%98%E9%A1%BA%E5%BA%8F)
        - [1.编译器指令重排序](#1-%E7%BC%96%E8%AF%91%E5%99%A8%E6%8C%87%E4%BB%A4%E9%87%8D%E6%8E%92%E5%BA%8F)
        - [2.内存模型](#2-%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B)
        - [3.结论](#3-%E7%BB%93%E8%AE%BA)
    - [原子类型的使用场景](#%E5%8E%9F%E5%AD%90%E7%B1%BB%E5%9E%8B%E7%9A%84%E4%BD%BF%E7%94%A8%E5%9C%BA%E6%99%AF)
        - [无锁编程](#%E6%97%A0%E9%94%81%E7%BC%96%E7%A8%8B)
    - [参考资料](#%E5%8F%82%E8%80%83%E8%B5%84%E6%96%99)

# 原子类型
在c\+\+11中，最大的一个变化就是对多线程的支持，而其中最重要的部分就是引入了原子类型。
学习原子类型之前，需要具备一定的多线程基础，需要理解互斥、同步、原子操作等的相关概念，可以参考学习[os-并发](https://liji53.github.io/2021/06/29/operatingSystem/os-concurrency/)。
从原子类型对并发编程的意义来看，它是对临界资源的原子操作的抽象，比方将test-and-set、compare-and-swap等这些原子操作抽象成了原子类型的方法。
c\+\+11以前，我们从逻辑上可能只需要对数据进行互斥即可，但实际却必须对代码段进行互斥，而**有了原子类型之后，就可以真正地实现数据之间的互斥而不是代码段的互斥。**


### 原子类型的实现
下面我们结合源码(来自vs的atomic.h)来看看c++11的原子类型是如何实现的

##### 1.基本类型
在使用原子类型的时候，你可以用atomic_bool、atomic_char等类型，但更多的是用std::atomic\<T\>类模板。其中T除了支持基本数据类型以外，还支持自定义的数据类型，自定义数据类型也能保证互斥，但不一定是lock-free的。
在源码中，我们可以找到相关的定义：
```c++
using atomic_bool = atomic<bool>;
using atomic_char = atomic<char>;
/// 等等
```

##### 2.内部类的关系与功能
除了atomic\<T\>模板类之外，在它的内部实现中，我们还可以看到如_Atomic_integral_facade、_Atomic_integral、_Atomic_pointer、_Atomic_storage等。
下面介绍下这些类的功能与关系：
```c++
// 1. 最低层的类，实际存储数据的类，提供最基本的读写（load和store）原子操作
// 除了下面2种以外，还特例化了数据大小为2，4，8的类型
template <class _Ty, size_t /* = ... */>
struct _Atomic_storage{}； // 自定义数据结构，大小超过8的，平台不支持，内部使用的是自旋锁实现
template <class _Ty>      
struct _Atomic_storage<_Ty, 1>{}；  // 类型大小为1的，通过底层原子操作实现

// 2. 继承自_Atomic_storage的模板类，表示整型的原子类型，提供了整数的一系列操作如fetch_add、operator++等
// _Atomic_storage类可以是自定义对象，因此只实现了读写的操作操作，而_Atomic_integral是对整型应该支持的原子操作的补充
struct _Atomic_integral<_Ty, 1> : _Atomic_storage<_Ty>

// 3. 继承自_Atomic_storage的模板类,与_Atomic_integral类似，是对指针的原子操作的补充，比如指针++操作
struct _Atomic_pointer : _Atomic_storage<_Ty> {}
struct _Atomic_pointer<_Ty&> : _Atomic_storage<_Ty&> {}

// 4. _Atomic_integral_facade只是用来生成特定的_Atomic_integral类的，属于工厂类
struct _Atomic_integral_facade : _Atomic_integral<_Ty>
struct _Atomic_integral_facade<_Ty&> : _Atomic_integral<_Ty&>

// 5. 供用户使用的模板类，_Choose_atomic_base_t用于选择继承上述的基类，是实现的核心
struct atomic : _Choose_atomic_base_t<_Ty> {};
// _Choose_atomic_base_t的实现
template <class _TVal, class _Ty = _TVal>
using _Choose_atomic_base2_t =
    typename _Select<is_integral_v<_TVal> && !is_same_v<bool, _TVal>>::template _Apply<_Atomic_integral_facade<_Ty>,
        typename _Select<is_pointer_v<_TVal> && is_object_v<remove_pointer_t<_TVal>>>::template _Apply<
            _Atomic_pointer<_Ty>, _Atomic_storage<_Ty>>>;

// 对上面如何选择继承的泛型编程看不懂的，可以看下面的伪代码
T _Choose_atomic_base_t(T){
    if (is_integral(T) && !is_bool(T)){
        return _Atomic_integral_facade<T>;
    }
    if (is_pointer(T)){
        return _Atomic_pointer<T>;
    }
    else{
        return _Atomic_storage<T>;
    }
}
```
这里的注释我写的挺详细的了,最后给出一张类关系图：
![](Images/class.png)

##### 3.基本操作
经过上一小节对atomic内部模板类的关系梳理，接下来对于不同原子类型所支持的操作应该就很好理解了。

|操作|atomic_flag|atomic\<bool\>|atomic\<整型\>|atomic\<指针\>|atomic\<自定义\>|
|---|:---:|:---:|:---:|:---:|:---:|
|test_and_set|√||||
|clear|√|
|is_lock_free||√|√|√|√|
|load,store||√|√|√|√|
|exchange||√|√|√|√|
|fetch_*|||√|√||

##### 4.原子变量不能拷贝,赋值
这是因为两个原子类型之间的操作不能保证原子化
```c++
atomic(const atomic&) = delete;
atomic& operator=(const atomic&) = delete;
```

##### 5.atomic\<T\>自定义对象必须是POD
我们知道atomic\<T\>的类型可以是自定义对象，但自定义对象必须是POD
```c++
/// 模板类中，有一个静态检查，表示T的类型必须是POD类型
static_assert(is_trivially_copyable_v<_Ty> && is_copy_constructible_v<_Ty> && is_move_constructible_v<_Ty>
        && is_copy_assignable_v<_Ty> && is_move_assignable_v<_Ty>,"...")
```

##### 6.非atomic_flag不一定无锁
除了atomic_flag,其他任何原子类型都不一定是无锁的,具体跟平台相关。但在这个版本的VS中，只要是1,2,4,8大小的数据类型，就是无锁
```c++
_NODISCARD bool is_lock_free() const volatile noexcept {
    constexpr bool _Result = sizeof(_Ty) <= 8 && (sizeof(_Ty) & sizeof(_Ty) - 1) == 0;
    return _Result;
}
```

##### 7.如果不是无锁的原子类型,其实底层实现就是mutex
这个机制保证了一定的兼容性
```c++
struct _Atomic_storage {
    void store(const _TVal _Value, const memory_order _Order = memory_order_seq_cst) noexcept {
        _Check_store_memory_order(_Order);
        _Guard _Lock{_Spinlock};
        _Storage = _Value;
    }
}；
```

##### 8.原子读写的内部实现
这里用的是windows的原子操作接口：_InterlockedExchange64
```c++
struct _Atomic_storage<_Ty, 8>{
     void store(const _TVal _Value) noexcept { // store with sequential consistency
        const auto _Mem           = _Atomic_address_as<long long>(_Storage);
        const long long _As_bytes = _Atomic_reinterpret_as<long long>(_Value);
#if defined(_M_IX86)
        _Compiler_barrier();
        __iso_volatile_store64(_Mem, _As_bytes);
        _STD atomic_thread_fence(memory_order_seq_cst);
#elif defined(_M_ARM64)
        _Memory_barrier();
        __iso_volatile_store64(_Mem, _As_bytes);
        _Memory_barrier();
#else // ^^^ _M_ARM64 / ARM32, x64 vvv
        (void) _InterlockedExchange64(_Mem, _As_bytes);
#endif // _M_ARM64
    }
}；
```

##### 9.默认内存顺序std::memory_order_seq_cst
std::memory_order_seq_cst指顺序一致性，关于内存顺序，我们稍后唠嗑，
```c++
_Ty operator=(const _Ty _Value) noexcept {
    this->store(_Value); 
    return _Value;
}
```

##### 10.compare_exchange_weak和compare_exchange_strong的区别
weak的意思是允许偶然出乎意料的返回(比如实际值和期待值一样的时候却返回了false)，通常它比起strong有更高的性能。
可惜在vs的这个版本中并没有实现，与strong版本是一样的。
```c++
bool compare_exchange_weak(_Ty& _Expected, const _Ty _Desired) volatile noexcept {
    // we have no weak CAS intrinsics, even on ARM32/ARM64, so fall back to strong
    static_assert(_Deprecate_non_lock_free_volatile<_Ty>, "Never fails");
    return this->compare_exchange_strong(_Expected, _Desired);
}
```

##### 11.atomic_flag的实现
atomic_flag 不一定是bool，比方这里就是long类型，atomic_flag一般是符合标准的最小的硬件实现类型，所以一定是无锁的
```c++
struct atomic_flag {
    bool test_and_set(const memory_order _Order = memory_order_seq_cst) noexcept {
        return _Storage.exchange(true, _Order) != 0;
    }
    void clear(const memory_order _Order = memory_order_seq_cst) noexcept {
        _Storage.store(false, _Order);
    }
#if 1 // TRANSITION, ABI
    atomic<long> _Storage;
#else // ^^^ don't break ABI / break ABI vvv
    atomic<bool> _Storage;
#endif // TRANSITION, ABI
};
```

### 内存顺序
这部分的内容，其实我个人认为除了追求极致的并发性能以外，并不用太关注这个内存顺序，使用默认的内存顺序即可。
现在我们再来看看为什么会有所谓的内存顺序？这里我们需要一些计算机组成原理、编译器相关的知识。

##### 1.编译器指令重排序
编译器编译的时候会经历Front-end(词法、语法等)、Optimizer、CodeGen 这三个阶段。在优化阶段（Optimizer）如果编译器认为代码的执行顺序不影响最终的结果，则可以依据情况将指令重排序从而提高性能。
如:
```c++
// b的赋值不影响最终结果，可以重排序
void func(){
    int t = 1;
    a = t;  // 全局原子类型
    b = 2;  // 全局原子类型
}
```
这段代码如果是在多线程情况下，可能就会因为重排序而发生问题。比如另一个线程需要根据b的值，对a进行某些操作，这时我们需要“顺序一致”。
但也有可能，我们仅仅只是对a、b进行赋值，顺序的先后并不重要，这时我们仅需要互斥即可。
因此c++原子类型需要内存顺序的一个原因就是因为编译器指令重排序的问题。

##### 2.内存模型
关于编译器的重排序问题，可能有人想到了用volatile来防止编译器的优化，但如果是cpu对指令顺序进行优化，可能就没办法了。

在计算机组成原理中，有一个叫指令流水线的概念，简单来说，就是将一条指令分解成诺干阶段，如取指、译码、执行等阶段，由于每个阶段用到的硬件不一样，可以让这些指令分阶段的并发执行，从而加快指令的执行速度。
![](Images/instruction.png)
而超标量流水线技术，是指令流水线的升级，以前取指、译码、执行等各个阶段各自只有一套硬件环境，现在硬件可以存在多套，于是就可以做到让每个时钟周期内并发多条独立指令。
![](Images/superInstruction.png)
这种超标量流水线技术会导致在不影响结果的情况下，cpu会优化指令的执行顺序。
在实现环境中，X86平台其实并不会改变指令执行顺序(强顺序)，但如果是ArmV7这些架构的，则会改变指令执行顺序（弱顺序）。
因此对于这些弱顺序的平台架构，为了保证指令的执行先后顺序，通常需要加入memory barrier。
细心的同学可能已经发现前面讲原子类型的实现时，代码中已经出现过内存栅栏了。从[原子读写的内部实现](#8-%E5%8E%9F%E5%AD%90%E8%AF%BB%E5%86%99%E7%9A%84%E5%86%85%E9%83%A8%E5%AE%9E%E7%8E%B0)中可以看到在对store的实现中就对_M_IX86和_M_ARM64平台加入了内存栅栏，而x64架构就不需要内存栅栏。

##### 3.结论
在清楚了底层的指令执行顺序的之后，我们可能希望编译器或者cpu对指令顺序进行优化，从而提高性能，也可能要求指令必须顺序执行，因此就有了内存顺序这一概念。
在atomic\<T\>的接口中，最基本的几种接口(像store、load)是支持内存顺序，但正如一开始所说的，除了追求极限的执行效率，个人认为使用默认的强顺序一致即可。
不用内存顺序的另一个原因则是编译器对内存顺序的支持情况可能是一样的，比如我在看VS对内存顺序的实现时，就发现除了memory_order_relaxed，其他的内存顺序都是强顺序一致的。
最后，给出内存顺序的结论：

| 内存顺序 | 说明 |
| --- | --- |
| std::memory_order_relaxed | 同一个线程中, 不同原子变量可以是乱序的 |
| std::memory_order_consume	| no reads or writes in the current thread dependent on the value currently loaded can be reordered before this load. Writes to data-dependent variables in other threads that release the same atomic variable are visible in the current thread. On most platforms, this affects **compiler optimizations only** |
| std::memory_order_acquire	| no reads or writes in the current thread can be reordered before this load. All writes in other threads that release the same atomic variable are visible in the current thread |
| std::memory_order_release	| no reads or writes in the current thread can be reordered after this store. All writes in the current thread are visible in other threads that acquire the same atomic variable |
| std::memory_order_acq_rel	| 略 |
| std::memory_order_seq_cst	| 顺序一致性，所有的线程观察到的整个程序中内存修改顺序是一致的 |

### 原子类型的使用场景

##### 无锁编程
无锁编程的优势主要是2个：
1. 避免了死锁、饥饿的产生
2. 临界区非常短、竞争激烈的场景

在内存数据库领域就用到了相关的技术。 业界应用参考：https://www.zhihu.com/question/52629893
最后从网上找了一个简单的无锁数据结构，仅供参考：
```c++
template<class T>
struct node
{
	T data;
	node* next;
	node(const T& data) : data(data), next(nullptr) {}
};
template<class T>
struct stack
{
	std::atomic<node<T>*> head;
	void push(const T& data) {
		node<T>* new_node = new node<T>(data);
		new_node->next = head.load(std::memory_order_relaxed);
		while (!head.compare_exchange_strong(new_node->next, new_node, std::memory_order_release));
	}
};
```

### 参考资料
《深入理解C++11：C++11新特性解析与应用》
内存顺序的解释[英]：https://en.cppreference.com/w/cpp/atomic/memory_order