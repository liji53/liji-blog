---
title: atomic
date: 2022-03-04 03:24:16
tags:
---
## 锁的机制与开销
现在锁的机制在linux2.5.7之后开始使用futex(fast Userspace mutexes),即内核态和用户态的混合机制。这个机制的主要优势就是在加锁的时候根据mmap共享内存里的futex变量，判断该变量是否有竞争，如果没有竞争则通过原子操作把共享的futex变量改成1，这样就不需要进入内核态，就可以完成加锁了。而如果有竞争则执行系统调用完成处理(wait)。
为了帮助理解，下面是我根据pthread_mutex_lock的源代码所写的伪代码：
```c++
int __pthread_mutex_lock (pthread_mutex_t *mutex)
{
    // 普通锁
    if (type == PTHREAD_MUTEX_TIMED_NP)
    {
        LLL_MUTEX_LOCK(mutex);
    }
    // 嵌套锁/递归锁，允许同一个线程多次加锁
    elif (type == PTHREAD_MUTEX_RECURSIVE_NP)
    {
        // 获取线程id
        pid_t id = THREAD_GETMEM(THREAD_SELF, tid)
        // 已经持有锁直接返回
        if (mutex->__data.__owner == id){return;}
        // 获取锁
        LLL_MUTEX_LOCK(mutex);
    }
    // 适应锁，跟自旋锁类似，尝试获取，但过一段时间仍然获取不到，就放弃，并让出CPU
    elif (type == PTHREAD_MUTEX_ADAPTIVE_NP)
    {
        if (LLL_MUTEX_TRYLOCK (mutex) != 0)
        {
            // 一直尝试获取
            do{
                // 过一段时间仍然获取不到，则放弃，让出CPU
                if (++cnt > max_cnt){
                    LLL_MUTEX_LOCK(mutex);
                    break;
                }
            }while(LLL_MUTEX_TRYLOCK(mutex) != 0)
        }
    }
    // PTHREAD_MUTEX_ERRORCHECK_NP 检错锁，不展开
    else{......}
}
void LLL_MUTEX_LOCK()
{
    // CAS原子操作，将0变为1,
    if（atomic_compare_and_exchange(futex, 0, 1)）
    {
        return
    }
    /// 有竞争，则阻塞
    else{
        INTERNAL_SYSCALL()
    }
}
```
从锁的机制来看，如果锁的冲突比较多，则会让线程从用户态切换到内核态，同时由于要让出CPU还存在上下文切换的开销。
## 其他并发访问解决方案
使用mutex性能开销大，那有什么方案来优化锁或者替代锁呢? 
1. 使用事务内存方案，主要是借鉴数据库里的事务来实现
2. 细粒度锁算法，基于“轻量级”的原子操作(自旋锁)，这种方案适合任何锁持有时间少于将一个线程阻塞和唤醒所需要的时间的场合
3. 无锁数据结构，将共享数据放入无锁的数据结构中，采用原子操作来访问共享数据
## C++11 atomic原子类型
C++11的原子变量以及原子操作为上面3方案的实现提供了极大的便利，我们结合源码先来熟悉下atomic的操作。源码来自vs的atomic.h头文件
#### 自定义对象必须是POD
我们知道atomic\<T\>的类型可以是自定义对象，但自定义对象是有限制的
```c++
/// 模板类中，有一个静态检查，表示T的类型必须是POD类型
static_assert(is_trivially_copyable_v<_Ty> && is_copy_constructible_v<_Ty> && is_move_constructible_v<_Ty>
        && is_copy_assignable_v<_Ty> && is_move_assignable_v<_Ty>,"...")
```
#### 原子变量不能拷贝，赋值
因为两个原子类型之前操作不能保证原子化
```c++
    atomic(const atomic&) = delete;
    atomic& operator=(const atomic&) = delete;
```
#### 非atomic_flag不一定无锁
除了atomic_flag,其他任何原子类型不一定是无锁的,具体跟平台相关。在本例中，只要是1,2,4,8大小的数据，就是无锁
```c++
_NODISCARD bool is_lock_free() const volatile noexcept {
        constexpr bool _Result = sizeof(_Ty) <= 8 && (sizeof(_Ty) & sizeof(_Ty) - 1) == 0;
        return _Result;
    }
```
#### 如果不是无锁的原子类型，则用mutex
从atomic的底层实现可以看到，使用了自旋锁来实现原子类型
```c++
struct _Atomic_storage {
        void store(const _TVal _Value, const memory_order _Order = memory_order_seq_cst) noexcept {
        _Check_store_memory_order(_Order);
        _Guard _Lock{_Spinlock};
        _Storage = _Value;
    }
}；
```
#### 原子操作
_Atomic_storage<_Ty, 8> 指的是数据结构大小为8 的原子类型，除了8以外，还有1，2，4
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
#### 赋值运算符默认std::memory_order_seq_cst
话说自加等运算符的重载去哪了，我没找到
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
#### compare_exchange_weak和compare_exchange_strong的区别
weak的CAS允许偶然出乎意料的返回(比如在字段值和期待值一样的时候却返回了false)，通常它比起strong有更高的性能。可惜在vs的这个版本中并没有实现，与strong版本是一样的。
```c++
    bool compare_exchange_weak(_Ty& _Expected, const _Ty _Desired) volatile noexcept {
        // we have no weak CAS intrinsics, even on ARM32/ARM64, so fall back to strong
        static_assert(_Deprecate_non_lock_free_volatile<_Ty>, "Never fails");
        return this->compare_exchange_strong(_Expected, _Desired);
    }
```
#### atomic_flag的实现
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
## 内存顺序
程序实际的执行过程可能与我们写的顺序是不一致的，这个不一致来自2个方面：
1. 编译器可以依情况在不影响程序运行结果的前提下，为提高运行效率，调整执行顺序
2. 弱顺序内存模型的多核处理器不一定按照顺序执行
要保证CPU顺序执行，就需要加入内存栅栏，就像大坝一样，需要上面的代码执行完，才能执行后面的代码
先看结论吧
#### 结论
英文说明是来自：https://en.cppreference.com/w/cpp/atomic/memory_order

| 内存顺序 | 说明 |
| --- | --- |
| std::memory_order_relaxed | 同一个线程中, 不同原子变量可以是乱序的 |
| std::memory_order_consume	| no reads or writes in the current thread dependent on the value currently loaded can be reordered before this load. Writes to data-dependent variables in other threads that release the same atomic variable are visible in the current thread. On most platforms, this affects **compiler optimizations only** |
| std::memory_order_acquire	| no reads or writes in the current thread can be reordered before this load. All writes in other threads that release the same atomic variable are visible in the current thread |
| std::memory_order_release	| no reads or writes in the current thread can be reordered after this store. All writes in the current thread are visible in other threads that acquire the same atomic variable |
| std::memory_order_acq_rel	| 略 |
| std::memory_order_seq_cst	| 顺序一致性，所有的线程观察到的整个程序中内存修改顺序是一致的 |

这部分源码我看了，但没看出哪里实现了这些功能，可能是编译器又做了什么骚操作，又或者根本就没有实现。

## 原子操作的使用场景
回到最关键的问题上，什么时候用原子操作？
让我们先看看原子类型和原子操作能做什么吧
#### 自旋锁
最简单的就是利用atomic_flag来实现一个自旋锁，但自旋锁只有在持有锁的时间比较短的情况下(也可以认为锁的粒度比较细的情况下)才比互斥锁有优势，而且上面提到锁的实现时有适应锁的选项，可以当作自旋锁来用。
#### 无锁编程
一个简单的例子
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
#### 无锁编程的ABA问题
另一个线程可能会把变量的值从A改成B，又从B改回成A。这就是ABA问题。
很多情况下，ABA问题不会影响你的业务逻辑因此可以忽略。但有时不能忽略，这时要解决这个问题，一般的做法是给变量关联一个只能递增、不能递减的版本号。在compare时不但compare变量值，还要再compare一下版本号。

