---
title: atomic
date: 2022-02-04 03:24:16
tags:
categories: C++基础
---
# 锁与原子操作
在c++11中，其中最大的一个变化就是对多线程的支持，而其中最重要的部分就是引入了原子操作和原子类型。
关于锁的机制与原理，可以参考[os-并发](https://liji53.github.io/2021/06/29/operatingSystem/os-concurrency/)

### C++11的atomic原子类型
pthread库的锁(包括信号量、条件变量)通过原子操作来实现的，下面我们结合源码来看看c++11的原子类型是怎么实现的（来自vs的atomic.h）

##### 1. atomic<T>自定义对象必须是POD
我们知道atomic\<T\>的类型可以是自定义对象，但自定义对象是有限制的
```c++
/// 模板类中，有一个静态检查，表示T的类型必须是POD类型
static_assert(is_trivially_copyable_v<_Ty> && is_copy_constructible_v<_Ty> && is_move_constructible_v<_Ty>
        && is_copy_assignable_v<_Ty> && is_move_assignable_v<_Ty>,"...")
```

##### 2. 原子变量不能拷贝，赋值
因为两个原子类型之间的操作不能保证原子化
```c++
    atomic(const atomic&) = delete;
    atomic& operator=(const atomic&) = delete;
```

##### 3. 非atomic_flag不一定无锁
除了atomic_flag,其他任何原子类型不一定是无锁的,具体跟平台相关。在本例中，只要是1,2,4,8大小的数据类型，就是无锁
```c++
_NODISCARD bool is_lock_free() const volatile noexcept {
        constexpr bool _Result = sizeof(_Ty) <= 8 && (sizeof(_Ty) & sizeof(_Ty) - 1) == 0;
        return _Result;
    }
```

##### 4. 如果不是无锁的原子类型，其实底层实现就是mutex
```c++
struct _Atomic_storage {
        void store(const _TVal _Value, const memory_order _Order = memory_order_seq_cst) noexcept {
        _Check_store_memory_order(_Order);
        _Guard _Lock{_Spinlock};
        _Storage = _Value;
    }
}；
```

##### 5. 原子操作
_Atomic_storage<_Ty, 8> 指的是数据结构大小为8 的原子类型，由于平台不一样，这里用的是windows接口：_InterlockedExchange64
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

##### 6. 赋值运算符的默认内存顺序std::memory_order_seq_cst
```c++
    _Ty operator=(const _Ty _Value) noexcept {
        this->store(_Value);
        return _Value;
    }
    void store(const _TVal _Value, const memory_order _Order) noexcept { // store with given memory order
        ......
        switch (_Order) {
        ......
        case memory_order_seq_cst:
            store(_Value);
            return;
        }
    }
```

##### 7. compare_exchange_weak和compare_exchange_strong的区别
weak的意思是允许偶然出乎意料的返回(比如实际值和期待值一样的时候却返回了false)，通常它比起strong有更高的性能。
可惜在vs的这个版本中并没有实现，与strong版本是一样的。
```c++
    bool compare_exchange_weak(_Ty& _Expected, const _Ty _Desired) volatile noexcept {
        // we have no weak CAS intrinsics, even on ARM32/ARM64, so fall back to strong
        static_assert(_Deprecate_non_lock_free_volatile<_Ty>, "Never fails");
        return this->compare_exchange_strong(_Expected, _Desired);
    }
```

##### 8. atomic_flag的实现
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
程序实际的执行过程可能与我们写的顺序是不一致的，这个不一致来自2个方面：
1. 编译器可以依情况在不影响程序运行结果的前提下，为提高运行效率，调整执行顺序
2. 弱顺序内存模型的多核处理器不一定按照顺序执行
要保证CPU顺序执行，就需要加入内存栅栏，就像大坝一样，需要上面的代码执行完，才能执行后面的代码
先看结论吧

##### 结论
英文解释：https://en.cppreference.com/w/cpp/atomic/memory_order

| 内存顺序 | 说明 |
| --- | --- |
| std::memory_order_relaxed | 同一个线程中, 不同原子变量可以是乱序的 |
| std::memory_order_consume	| no reads or writes in the current thread dependent on the value currently loaded can be reordered before this load. Writes to data-dependent variables in other threads that release the same atomic variable are visible in the current thread. On most platforms, this affects **compiler optimizations only** |
| std::memory_order_acquire	| no reads or writes in the current thread can be reordered before this load. All writes in other threads that release the same atomic variable are visible in the current thread |
| std::memory_order_release	| no reads or writes in the current thread can be reordered after this store. All writes in the current thread are visible in other threads that acquire the same atomic variable |
| std::memory_order_acq_rel	| 略 |
| std::memory_order_seq_cst	| 顺序一致性，所有的线程观察到的整个程序中内存修改顺序是一致的 |

这部分源码其实根本就没有实现。除了relaxed，其他的内存顺序就是加了内存栅栏

### 原子类型的使用场景

##### 无锁编程
无锁编程的优势主要是2个：
1. 避免了死锁、饥饿的产生
2. 临界区非常短、竞争激烈的场景，常见的就是无锁的数据结构，在内存数据库领域用的很多。 参考业界应用：https://www.zhihu.com/question/52629893
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