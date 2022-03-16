---
title: allocator
date: 2021-10-11 19:04:04
tags:
categories: C++基础
---
# 分配器杂谈
分配器在stl的容器中用于空间的分配与释放，以及对象的初始化与析构，下面我们简单了解下stl 6大组件中的allocator

### 分配器的思想与接口
分配器将内存的分配与对象的构造行为分离出来，一是为了提高效率对于已分配好的内存可以反复利用，避免内存碎片，但更重要的是为了泛型编程的可复用性，降低内存分配与构造行为的耦合性。
##### 1.基本接口
```c++
template <class T>
T* allocator<T>::allocate(size_type n);
template <class T>
void allocator<T>::deallocate(T* ptr);
template <class T>
void allocator<T>::construct(T* ptr);
template <class T>
void allocator<T>::destroy(T* ptr)
```
allocate和deallocate 在gcc中的实现就是::operator new和delete
而construct 用到了placement new，为的是调用对象的构造函数。
而destroy 自然是显式调用析构函数

##### 2.rebind
在allocator的实现中都会有一个rebind的结构体，它的作用是获得其他类型的内存分配器allocator\<other\>
```c++
template <typename _Tp1>
struct rebind
{
    typedef allocator<_Tp1> other;
};
```
因为在stl容器中，除了要给对象分配内存，往往还需要给存数据的节点分配内存，比如list，list节点存放数据和下一个节点，而rebind用来给这个节点分配内存的。

### uninitialized的细节
uninitialized系列的函数虽然不属于allocator，但提供了大规模元素初始值的设置，能大大提高拷贝的效率
##### 1. uninitialized_copy
uninitialized_copy 会把 \[first, last\)上的内容复制到以result为起始处的空间，返回复制结束的位置
由于实际代码嵌套层数太多而且还有各种type_traits的处理，这里写伪代码供参考：
```c++
ForwardIter uninitialized_copy(InputIter first, InputIter last, ForwardIter result){
    if (is_trivially_copy_assignable(result)){  /// 根据result来判断对象的构造函数是否无关紧要(可以理解为是否为POD)
        if (is_pointer_iterator()){   /// 指针版本，实际通过偏特化来实现
            std::memmove(result, first, n * sizeof(Up));  ///  memmove能够保证源串在被覆盖之前将重叠区域的字节拷贝到目标区域中，比memcpy更安全
        }
        else{  /// 迭代器版本
            if (is_random_access_iterator_tag()){   /// 可以看到，为了性能，stl偏特化了随机访问迭代器版本
                for (auto n = last - first; n > 0; --n, ++first, ++result)
                {
                    *result = *first;
                }
                return result;
            }
            else{
                for (; first != last; ++first, ++result)   /// 性能比random_access_iterator差，因为需要调用运算符重载函数
                {
                    *result = *first;
                }
                return result;
            }
        }
    }
    else{
        auto cur = result;
        try{
            for (; first != last; ++first, ++cur){
                std::construct(&*cur, *first);   /// 使用了replacement new
            }
        }
        catch (...){                             /// commit or rollback 构造出所有元素，要么不构造出任何一个
            for (; result != cur; --cur)
                std::destroy(&*cur);             /// 调用析构函数
        }
        return cur;
    }
}
```

##### 2. uninitialized_fill
uninitialized_fill 会在 [first, last) 区间内填充元素值value
伪代码实现如下：
```c++
void  uninitialized_fill(ForwardIter first, ForwardIter last, const T& value){
    if (is_trivially_copy_assignable(first)){  /// 根据first的类型判断构造函数是否无关紧要
        if (is_random_access_iterator_tag()){
            n = last - first;   /// 类型萃取difference_type
            if is_pointer_iterator(){   
                std::memset(first, (unsigned char)value, (size_t)(n));  /// 如果迭代器是指针，直接调用memset
            }
            else{
                for (; n > 0; --n, ++first){
                    *first = value;
                }
            }
        }
        else{
            for (; first != last; ++first){
                *first = value;
            }
        }
    }
    else{
        ......  /// 省略，与uninitialized_copy的实现类似
    }
}
```

##### 3. uninitialized_move
uninitialized_move 会把[first, last)上的内容移动到以result为起始处的空间，返回移动结束的位置
由于实现上与uninitialized_copy基本一致，大部分内容省略
```c++
ForwardIter uninitialized_move(InputIter first, InputIter last, ForwardIter result){
    if (is_trivially_copy_assignable(result)){
        if (is_pointer_iterator()){
            std::memmove(result, first, n * sizeof(Up));
        }
        else{
            ......
            *result = std::move(*first);
            ......
        }
    }
    else{
        ...... 
        std::construct(&*cur, std::move(*first));  /// 实际调用移动构造函数
        ......
    }
}
```

### 内存池版本的allocator弃用
##### 1.std::alloc的原理
参考：《stl源码剖析》
原理图：
![](Images\alloc.png)
当申请的内存较大时，直接通过new进行内存分配，当申请小块内存时，使用内存池管理。
内存池通过16个数组进行管理，每个数组各自管理大小分别为 8， 16， 24，…128 bytes(8 的倍数)的小额区块，这些区块通过链表的方式链接起来

##### 2.什么时候弃用的，为什么弃用
弃用原因参考(官网)：
https://gcc.gnu.org/onlinedocs/libstdc++/manual/memory.html#allocator.design_issues
弃用版本(官网)：
https://gcc.gnu.org/onlinedocs/libstdc++/manual/api.html#table.extension_allocators
从gcc3.4版本开始默认使用__gnu_cxx::new_allocator，同时这个版本对分配器做了较大的改动，新增了如bitmap_allocator、mt_allocator等的分配器
可是使用内存池的版本比直接使用new进行分配要有优势(减少内存碎片,提高效率)，但为什么GNU要废弃内存池版本呢？
通过官网的解释是为了稳定、兼容。（原文：has the advantage of working correctly across a wide variety of hardware and operating systems, including large clusters）
同时指出了内存池的缺点：由于内存的创建销毁顺序的不确定，在内存中加载和释放共享对象时可能有问题。（原文：order-of-destruction and order-of-creation for memory pools may be difficult to pin down with certainty, which may create problems when used with plugins or loading and unloading shared objects in memory）

### Gnu提供的allocator其他版本
##### 1.现有版本
new_allocator：默认版本，使用new和delete管理
malloc_allocator：使用malloc和delete管理
debug_allocator：会申请相比于目标值稍大一些的内存，并用额外的内存存储size信息。在deallocate中用一个assert()来检查被存储的size信息和将要被释放的内存的size是否一致
throw_allocator：具有内存跟踪和标记的功能
__pool_alloc ：即上面的旧版本内存池分配器
__mt_alloc ： 多线程内存池分配器
bitmap_allocator：内存池的基础上用bitmap标记内存块的使用和释放
##### 2.__mt_alloc的原理
资料：https://gcc.gnu.org/onlinedocs/libstdc++/manual/mt_allocator.html
mt_alloc分为多线程版本和单线程版本，遗憾的是多线程版本只给出了接口，并没有实现。
但实现原理其实是在pool_allocator的基础上，从1维数组变成2维数组，第二维指向线程id。
```c++
/// 多线程池版的内存池，数据结构
class __pool<true> : public __pool_base{
    /// 线程id节点，这些节点会组成一个链表，用于从真实线程id映射到内部线程id（1-4096）
    struct _Thread_record
    {
        _Thread_record*	    _M_next;  /// 下一个空闲线程id
        size_t              _M_id;  /// 内部使用的线程id，范围是1到4096
    };
    union _Block_record{ 
        _Block_record*      _M_next;	/// 下一个空闲节点
        size_t              _M_thread_id;  /// 申请该内存的线程id
    };
    struct _Bin_record
    {
        _Block_record**     _M_first;  /// 一维数组，idx为线程id，存free_list的头节点
        _Block_address*     _M_address;  /// 实际内存块的地址
        size_t*             _M_free;   /// 一维数组，idx为线程id，存空闲内存块计数
        size_t*             _M_used;   /// 一维数组，idx为线程id，存使用内存块计数
        __gthread_mutex_t*  _M_mutex;  /// 互斥锁,申请和释放内存小块时用
    };
    _Bin_record*            _M_bin;   /// 一维数组，内存池按2的指数对小内存块进行管理，idx为2的指数
    size_t                  _M_bin_size;  ///bin的个数
    _Thread_record*         _M_thread_freelist;  /// 线程id的列表
};
```
##### 3.bitmap_allocator的原理
资料：https://gcc.gnu.org/onlinedocs/libstdc++/manual/bitmap_allocator.html
网友分析：https://www.jianshu.com/p/425ab2e81b59