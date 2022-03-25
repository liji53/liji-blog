---
title: smart_ptr
date: 2022-01-08 21:48:07
tags:
categories: C++基础
---

# 智能指针的实现
shared_ptr 的实现我们都知道靠引用计数，但引用计数的生命周期是怎么样的，智能指针是否线程安全，weak_ptr是否需要计数？现在我们通过阅读源码，来一探究竟。

## shared_ptr的实现
代码来自vs的memory.h,用的是c\++14标准
#### 构造函数(c++11动态数组，需要显式提供delete functor)
在vs中神奇的发现c\++17才支持的动态数组，c\++14就支持了
```c++
template <class _Ty>
class shared_ptr : public _Ptr_base<_Ty> {
    explicit shared_ptr(_Ux* _Px) { // construct shared_ptr object that owns _Px
        /// 已经能自动选择数组类型的删除器
        if constexpr (is_array_v<_Ty>) {
            _Setpd(_Px, default_delete<_Ux[]>{});
        } else {
            _Temporary_owner<_Ux> _Owner(_Px);
            /// 由于shared_ptr的生命周期随时可以结束，因此引用计数器必须是在heap上
            _Set_ptr_rep_and_enable_shared(_Owner._Ptr, new _Ref_count<_Ux>(_Owner._Ptr));
            _Owner._Ptr = nullptr;
        }
    }
   _NODISCARD _Elem& operator[](ptrdiff_t _Idx) const noexcept /* strengthened */ {
        return get()[_Idx];
    }

}；
// 测试
std::cout << __cplusplus << std::endl;  // 201402
std::shared_ptr<int[]> p_list(new int[4]{ 1,2,3,4 }); // 本来应该在c++17中才支持的
std::cout << p_list[1] << std::endl;  // 2，同上
```
#### 拷贝构造/移动构造函数/赋值
移动系列的函数，会使原shared_ptr变成空指针
赋值会使原shared_ptr引用计数减一
```c++
shared_ptr(const shared_ptr& _Other) noexcept { // construct shared_ptr object that owns same resource as _Other
    this->_Copy_construct_from(_Other);
}
void _Copy_construct_from(const shared_ptr<_Ty2>& _Other) noexcept {
    // implement shared_ptr's (converting) copy ctor
    _Other._Incref();
    _Ptr = _Other._Ptr;
    _Rep = _Other._Rep;
}
void _Move_construct_from(_Ptr_base<_Ty2>&& _Right) noexcept {
    // implement shared_ptr's (converting) move ctor and weak_ptr's move ctor
    _Ptr = _Right._Ptr;
    _Rep = _Right._Rep;

    _Right._Ptr = nullptr;
    _Right._Rep = nullptr;
}
/// 减一是因为shared_ptr(_Right)是个临时变量，在swap出作用域之后就析构
shared_ptr& operator=(const shared_ptr<_Ty2>& _Right) noexcept {
    shared_ptr(_Right).swap(*this);
    return *this;
}
```

## weak_ptr的实现
weak_ptr 类型指针不会导致堆内存空间的引用计数增加或减少。
另外weak_ptr没有实现operator->和operator*
#### 构造函数
```c++
template <class _Ty>
class weak_ptr : public _Ptr_base<_Ty> {
    weak_ptr(const weak_ptr& _Other) noexcept {
        this->_Weakly_construct_from(_Other); // same type, no conversion
    }
    weak_ptr(const shared_ptr<_Ty2>& _Other) noexcept {
        this->_Weakly_construct_from(_Other); // shared_ptr keeps resource alive during conversion
    }
    /// 不增加shared_ptr的引用计数，增加weak_ptr的引用计数
    void _Weakly_construct_from(const _Ptr_base<_Ty2>& _Other) noexcept { // implement weak_ptr's ctors
        if (_Other._Rep) {
            _Ptr = _Other._Ptr;
            _Rep = _Other._Rep;
            _Rep->_Incwref();
        } else {
            _STL_INTERNAL_CHECK(!_Ptr && !_Rep);
        }
    }
}
```
#### lock函数
```c++
_NODISCARD shared_ptr<_Ty> lock() const noexcept { // convert to shared_ptr
    shared_ptr<_Ty> _Ret;
    (void) _Ret._Construct_from_weak(*this);
    return _Ret;
}
bool _Construct_from_weak(const weak_ptr<_Ty2>& _Other) noexcept {
    // implement shared_ptr's ctor from weak_ptr, and weak_ptr::lock()
    if (_Other._Rep && _Other._Rep->_Incref_nz()) {
        _Ptr = _Other._Ptr;
        _Rep = _Other._Rep;
        return true;
    }

    return false;
}
```

## 引用计数
通过前面的源码大家也可以看出来，shared_ptr和weak_ptr在交换指针时并不是线程安全的，但引用计数却是线程安全的
shared_ptr的引用计数为0，管理的对象可以销毁，但引用计数对象可能仍然存在，需要weak_ptr的引用计数也为0，才销毁
#### 接口与成员变量
```c++
#define _MT_INCR(x) _INTRIN_RELAXED(_InterlockedIncrement)(reinterpret_cast<volatile long*>(&x))
#define _MT_DECR(x) _INTRIN_ACQ_REL(_InterlockedDecrement)(reinterpret_cast<volatile long*>(&x))
class __declspec(novtable) _Ref_count_base { 
    virtual void _Destroy() noexcept     = 0; // destroy managed resource
    virtual void _Delete_this() noexcept = 0; // destroy self
    _Atomic_counter_t _Uses  = 1;
    _Atomic_counter_t _Weaks = 1;
/// 原子操作
    void _Incref() noexcept { // increment use count
        _MT_INCR(_Uses);
    }
    void _Incwref() noexcept { // increment weak reference count
        _MT_INCR(_Weaks);
    }
/// shared_ptr的引用计数为0，则释放资源对象
    void _Decref() noexcept { // decrement use count
        if (_MT_DECR(_Uses) == 0) {
            _Destroy();
            _Decwref();
        }
    }
/// 只有weak_ptr的引用计数为0，才释放引用计数
    void _Decwref() noexcept { // decrement weak reference count
        if (_MT_DECR(_Weaks) == 0) {
            _Delete_this();
        }
    }
};
```

