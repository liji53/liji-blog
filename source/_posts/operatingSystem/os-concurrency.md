---
title: os-并发
date: 2021-06-29 15:54:28
tags:
categories: 操作系统
---
# 并发
并发、互斥、同步在实际开发过程中经常会用到，本文将聚焦于互斥与同步，除了了解底层的互斥原理，还将重点学习glic提供的几种互斥机制。

### 底层原子操作
原子操作：将多个步骤合成一个操作，这个操作要么成功，要么失败。
底层原子操作由硬件提供，在阅读同步相关的源码时，会经常看到类似atomic_compare_exchange，atomic_fetch_add等类似的函数，其实这些函数就是封装了这些底层的原子操作指令。
因此熟悉这部分原子操作对阅读理解pthread相关的代码十分重要，下面我们看下4种基本的原子操作

##### test-and-set
返回旧值，设置新值
```c
int TestAndSet(int *ptr, int new) {
    int old = *ptr; // fetch old value at ptr
    *ptr = new; // store ’new’ into ptr
    return old; // return the old value
}
```

##### compare-and-swap
返回旧值，如果旧值与期望值一致，则设置新值
```c
int CompareAndSwap(int *ptr, int expected, int new) {
    int actual = *ptr;
    if (actual == expected)
        *ptr = new;
    return actual;
}
```

##### fetch-and-add
返回旧值，更新值
```c
int FetchAndAdd(int *ptr, int add_value) {
    int old = *ptr;
    *ptr = old + add_value;
    return old;
}
```
利用fetch-and-add我们可以实现基于ticket的自旋锁,ticket lock能保证执行顺序，实现先来先拿锁：
```c
typedef struct __lock_t {
    int ticket;
    int turn;
} lock_t;

void lock_init(lock_t *lock) {
    lock->ticket = 0;
    lock->turn = 0;
}

void lock(lock_t *lock) {
    int myturn = FetchAndAdd(&lock->ticket, 1);   
    while (lock->turn != myturn)  
        ; // spin
}

void unlock(lock_t *lock) {
    FetchAndAdd(&lock->turn, 1);
}
```

##### load 和 store
load指每次读最新的值
store如果没有人写ptr则更新ptr，并返回成功，否则返回失败
```c
int LoadLinked(int *ptr) {
    return *ptr;
}
int StoreConditional(int *ptr, int value) {
    if (如果没有人更新ptr) {
        *ptr = value;
        return 1; // success!
    } else {
        return 0; // failed to update
    }
}
```

### 互斥锁
了解了硬件提供的几种原子操作之后，我们可以基于这些原子操作实现一些互斥功能。
首先来看下mutex，但在看它的实现之前，我么需要清楚如何评估一个锁的好坏，主要从下面2方面考虑：
1. 公平性，主要看是否可能存在“饥饿”现象
2. 性能，从实际环境来评估：cpu是否多核，线程数，临界区的长度

##### 1.自旋锁
这个实现比较简单，利用前面的4种原子操作都能实现，如利用compare-and-swap：
```c
void lock(lock_t *lock) {
    while (CompareAndSwap(&lock->flag, 0, 1) == 1)
        ; // spin
}
```
下面重点看下它的优劣势：
从公平性角度来看，可能存在饥饿现象。（除了上面基于ticket 的spin lock）
从性能角度来看，对于单核cpu来说，采用自旋的锁，必然是浪费cpu的，参考[虚拟cpu](https://liji53.github.io/2021/06/15/operatingSystem/os-virtualization/)的进程调度
但是对于多核的cpu来说（线程数接近cpu核数），如果临界区很短(立马就释放锁)，采用自旋锁，性能将比普通的锁更好

##### 2.nptl互斥锁的实现
自旋锁会一直占用cpu，浪费资源，如何让出cpu，让程序更高效。下面我们通过glibc的源码，来看下pthread_mutex的锁实现，
pthread_mutex_lock的近似代码：
```c++
int __pthread_mutex_lock (pthread_mutex_t *mutex){
    // 普通锁
    if (type == PTHREAD_MUTEX_TIMED_NP){
        LLL_MUTEX_LOCK(mutex);
    }
    // 嵌套锁/递归锁，允许同一个线程多次加锁
    elif (type == PTHREAD_MUTEX_RECURSIVE_NP){
        pid_t id = THREAD_GETMEM(THREAD_SELF, tid);  // 获取线程id
        if (mutex->__data.__owner == id){return;}  // 已经持有锁直接返回
        LLL_MUTEX_LOCK(mutex);
    }
    // 适应锁，跟自旋锁类似，尝试获取，但过一段时间仍然获取不到，就放弃，并让出CPU
    elif (type == PTHREAD_MUTEX_ADAPTIVE_NP){
        if (LLL_MUTEX_TRYLOCK (mutex) != 0){            
            do{  // 一直尝试获取
                if (++cnt > max_cnt){  // 过一段时间仍然获取不到，则放弃，让出CPU
                    LLL_MUTEX_LOCK(mutex);
                    break;
                }
            }while(LLL_MUTEX_TRYLOCK(mutex) != 0);
        }
    }
    // PTHREAD_MUTEX_ERRORCHECK_NP 检错锁，如果一个线程2次获取同一个锁则，返回失败
    else{
        pid_t id = THREAD_GETMEM (THREAD_SELF, tid);
        if (__glibc_unlikely (mutex->__data.__owner == id)){return EDEADLK;}
        LLL_MUTEX_LOCK(mutex);
    }
}
```
代码里清楚的展示了4种锁的上层实现策略，而最关键的LLL_MUTEX_LOCK代码，我们下面讲

##### 3.futex机制
先简单介绍下futex，futex机制在linux2.5.7之后开始使用(fast Userspace mutexes)。
它的主要优势是在加锁的时候根据共享内存里的futex变量，判断该变量是否有竞争，如果没有竞争则通过原子操作把共享的futex变量置1，这样不需要进入内核态，就可以完成加锁了。
而如果有竞争则执行系统调用futex_wait，将需要等待的进程(线程)加入到futex的等待队列中，直到通过futex_wake进行唤醒。
futex的结构维护在内核中.它的数据结构大概如下：
![](Images/futex_struct.png)
加入等待队列以及唤醒的基本接口：
```c
//uaddr代表futex word的地址，val代表这个地址期待的值，当*uaddr==val时，才会进行wait
int futex_wait(int *uaddr, int val);
//唤醒n个在uaddr指向的锁变量上挂起等待的进程
int futex_wake(int *uaddr, int n);
```
因此前面pthread_mutex_lock的LLL_MUTEX_LOCK函数如下（参考lowlevellock.h）：
```
void LLL_MUTEX_LOCK(pthread_mutex_t *mutex){
    // CAS原子操作，如果等于0则置为1，表示没有锁竞争
    // mutex->__data.__lock 也就是所谓的futex word
    if（atomic_compare_and_exchange_bool_acq(mutex->__data.__lock, 0, 1)）{
        return;
    }
    /// 有竞争，则阻塞，加入到futex等待队列
    else{
        futex_wait(mutex->__data.__lock， 1);
    }
}
```
了解完futex的机制之后，我们再来评估下pthread_mutex的好坏：
1. 公平性：通过futex来管理等待线程，并不能保证它绝对的唤醒先后顺序，取决于OS的调度策略。
2. 性能：无竞争时，不需要进程(线程)切换，性能极好；但存在锁冲突时，需要线程切换，存在性能浪费，但好在提供了自适应锁这种兼容spin lock优势的参数

##### 4.nptl读写锁的实现
在读操作频繁，写操作少的情况下，使用读写锁能提高性能，下面我们看下pthread中读写锁的实现(来自glibc-2.35)：
```c
___pthread_rwlock_rdlock (pthread_rwlock_t *rwlock){
    ...
    // 原子操作，让reader + 1
    r = (atomic_fetch_add_acquire (&rwlock->__data.__readers,
			    (1 << PTHREAD_RWLOCK_READER_SHIFT)) + (1 << PTHREAD_RWLOCK_READER_SHIFT));
    // 如果当前为读阶段(即没有写)，则直接返回成功，不阻塞
    if (__glibc_likely ((r & PTHREAD_RWLOCK_WRPHASE) == 0)){return 0;}
    // 如果处于写阶段，但并没有占有锁
    while ((r & PTHREAD_RWLOCK_WRPHASE) != 0 && (r & PTHREAD_RWLOCK_WRLOCKED) == 0){
        // 尝试改成读阶段
        if (atomic_compare_exchange_weak_acquire (&rwlock->__data.__readers, &r, r ^ PTHREAD_RWLOCK_WRPHASE)){
            // 更新写阶段的futex变量，设置为0意味着解锁
            if ((atomic_exchange_relaxed (&rwlock->__data.__wrphase_futex, 0) & PTHREAD_RWLOCK_FUTEX_USED) != 0){
                // 唤醒其他阻塞的读线程
                int private = __pthread_rwlock_get_private (rwlock);
                futex_wake (&rwlock->__data.__wrphase_futex, INT_MAX, private);
            }
            return 0;
        }else{// 不停的重新尝试}
    }
    for (;;)
    {
        // 处于写阶段，并且锁已经被其他线程占有
        while (((wpf = atomic_load_relaxed (&rwlock->__data.__wrphase_futex))
	            | PTHREAD_RWLOCK_FUTEX_USED) == (1 | PTHREAD_RWLOCK_FUTEX_USED))
        {
            int private = __pthread_rwlock_get_private (rwlock);
            // 如果中途 锁已经释放了
            if (((wpf & PTHREAD_RWLOCK_FUTEX_USED) == 0) && (!atomic_compare_exchange_weak_relaxed
	                (&rwlock->__data.__wrphase_futex, &wpf, wpf | PTHREAD_RWLOCK_FUTEX_USED)))
            {
                continue; // 继续尝试，看锁是否又被写进程占有了
            }
            // futex_wait,加入到futex的等待队列
            int err = __futex_abstimed_wait64 (&rwlock->__data.__wrphase_futex,
					1 | PTHREAD_RWLOCK_FUTEX_USED, clockid, abstime, private);
        }
        ...
}
```
上面的代码如果理解有困难，可以参考下面读写锁各个阶段的情况：
```c
// WP 指的是读写阶段他，WL 指的是写锁， R 指的是读的线程数(也代表读锁)， RW 指的是是否有读线程在等待
   State WP  WL  R   RW  Notes
   ---------------------------
   #1    0   0   0   0   Lock is idle (and in a read phase).
   #2    0   0   >0  0   Readers have acquired the lock.    
   #3    0   1   0   0   Lock is not acquired; a writer will try to start a
			 write phase.                                          
   #4    0   1   >0  0   Readers have acquired the lock; a writer is waiting
			 and explicit hand-over to the writer is required. 
   #4a   0   1   >0  1   Same as #4 except that there are further readers
			 waiting because the writer is to be preferred.
   #5    1   0   0   0   Lock is idle (and in a write phase).
   #6    1   0   >0  0   Write phase; readers will try to start a read phase
			 (requires explicit hand-over to all readers that
			 do not start the read phase).
   #7    1   1   0   0   Lock is acquired by a writer.
   #8    1   1   >0  0   Lock acquired by a writer and readers are waiting;
			 explicit hand-over to the readers is required.
```
注意：读不占用锁，只有写才占用锁，#3，#4，#4a 这三个阶段 为wrlock加锁的情况，需要等R的个数为0，才能进行写。
关于读写锁饿死的情况, 在rdlock源码中并没有看到处理，因此如果写频繁，请不要使用读写锁。
引用一句话“Big and dumb is better”，不要过分追求像读写锁这种听起来炫酷，但使用复杂的玩意。

##### 5.提高并发性能
这里仅指优化锁的开销，从而提高并发性能
从前面的futex机制，我们知道如果出现锁冲突，就会有进程(线程)切换的开销(上下文切换、进程调度)，因此如何优化锁，核心在于减少锁的冲突：
1. 减少锁的持有时间，在锁的过程中不要使用会阻塞的接口，从时间颗粒度的角度考虑。
2. 减少锁的空间颗粒度，将临界数据打散，每个分散后的临界数据各使用一把锁，代替全局的锁。因为数据分散之后，访问某一个数据的概率就减少了，锁冲突也就减少了。
3. 避免使用锁,使用线程本地存储(比方TLS)。思路就是把临界数据变成线程局部数据，一段时间之后再更新回临界数据中。
4. 在读操作频繁，写操作少的数据结构中，使用读写锁;在临界区短，冲突频繁的场景中使用自旋锁。
5. 放弃用锁，使用wait-free的思路，比方使用无锁的数据结构。

下面针对第二点，我们看下常见的数据结构如何从空间颗粒度角度，减少锁的开销：
1. 双向队列，使用2把锁代替1把锁
```c
typedef struct __queue_t {
    node_t *head;
    node_t *tail;
    pthread_mutex_t headLock;
    pthread_mutex_t tailLock;
} queue_t;
```
2. 哈希表，每个哈希桶一把锁，哈希值不冲突就不会存在锁冲突
```c
typedef struct __hash_t {
    list_t lists[BUCKETS];
    pthread_mutex_t bucketLock[BUCKETS];
} hash_t;
```
针对第三点采用TLS数据，看个多线程计数器的列子：
```c
typedef struct __counter_t {
    int global; // global count
    pthread_mutex_t glock; // global lock
    int local[NUMCPUS]; // 每个线程单独访问，不存在冲突
    int threshold; // update frequency
} counter_t;
void update(counter_t *c, int threadID, int amt) {
    c->local[threadID] += amt; // assumes amt > 0
    if (c->local[threadID] >= c->threshold) { // transfer to global
        pthread_mutex_lock(&c->glock);
        c->global += c->local[threadID];
        pthread_mutex_unlock(&c->glock);
        c->local[threadID] = 0;
    }
}
```

### 条件变量
条件变量用于线程的同步，保证线程的执行先后顺序
##### 1.用法
条件变量的使用虽然容易犯错，但它的套路其实比较固定的：
```c
// 场景：A线程先执行，再执行B线程
// A线程：
Pthread_mutex_lock(&m);
设置条件
Pthread_cond_signal(&c);
Pthread_mutex_unlock(&m);
// B线程：
Pthread_mutex_lock(&mutex); // p1
while (条件不满足) // p2
    Pthread_cond_wait(&cond, &mutex); // p3
Pthread_mutex_unlock(&mutex); // p6
```
常见的错误就是：
1. 没有和锁一起配合使用，或者锁顺序有问题，牢记条件变量要传锁的原因：即Pthread_cond_wait内部实现： 1.先释放锁，2.加入等待队列，3.唤醒之后再加锁
2. Pthread_cond_wait在等待条件满足时，不用while 而用if， Pthread_cond_wait可能存在虚假唤醒(被信号中断唤醒);也有可能同时唤醒2个线程，但另一个线程执行很快，又把值改回去了(ABA问题)
3. 该使用2个条件变量，却只使用了一个条件变量，参考下面的消费者-生产者的错误模型：
```c
pthread__cond_t cond;
pthread__mutex_t mutex;
void *producer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        Pthread_mutex_lock(&mutex); // p1
        while (count == 1) // p2
            Pthread_cond_wait(&cond, &mutex); // p3
        put(i); // p4
        Pthread_cond_signal(&cond); // p5
        Pthread_mutex_unlock(&mutex); // p6
    }
}
void *consumer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        Pthread_mutex_lock(&mutex); // c1
        while (count == 0) // c2
            Pthread_cond_wait(&cond, &mutex); // c3
        int tmp = get(); // c4
        Pthread_cond_signal(&cond); // c5
        Pthread_mutex_unlock(&mutex); // c6
    }
}
```
这一段代码，如果只有1个消费者和1个生产者似乎没什么问题，但如果consumer(消费者)的线程数是2个就有问题了。
错误原因：如果consumer1 发出signal ，但唤醒的不是producer，而是consumer2，consumer2由于条件不满足，又睡眠了（并没有发signal），这时候三个线程就将永远睡眠。

##### 2.生产者-消费者模型
正确的代码如下：
```c
cond_t empty, fill;
mutex_t mutex;
void *producer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        Pthread_mutex_lock(&mutex); // p1
        while (count == MAX) // p2
            Pthread_cond_wait(&empty, &mutex); // p3
        put(i); // p4
        Pthread_cond_signal(&fill); // p5
        Pthread_mutex_unlock(&mutex); // p6
    }
}
void *consumer(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        Pthread_mutex_lock(&mutex); // c1
        while (count == 0) // c2
            Pthread_cond_wait(&fill, &mutex); // c3
        int tmp = get(); // c4
        Pthread_cond_signal(&empty); // c5
        Pthread_mutex_unlock(&mutex); // c6
    }
}
```

##### 3.nptl的实现
pthread_cond_wait的源码（来自glibc-2.35）
```c
int __pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex){
    ...
    // wseq低位是group的索引，因此需要加2才表示seq加1
    uint64_t wseq = __condvar_fetch_add_wseq_acquire (cond, 2);  
    unsigned int g = wseq & 1;  // g是group[1]的索引
    uint64_t seq = wseq >> 1;   // seq 为序列号
    // __wrefs 低3位有其他用处，+8 表示等待者计数 +1
    unsigned int flags = atomic_fetch_add_relaxed (&cond->__data.__wrefs, 8);
    // 【用户锁，解锁】
    err = __pthread_mutex_unlock_usercnt (mutex, 0);
    // 需要唤醒的等待者个数，__g_signals + g表示当前group
    unsigned int signals = atomic_load_acquire (cond->__data.__g_signals + g);
    do{
        while (1){
            // 如果有信号
            if (signals != 0) {break;}
            // __g_refs 当前组的引用计数+1
            atomic_fetch_add_acquire (cond->__data.__g_refs + g, 2);
            // 【futex_wait 加入等待队列，阻塞】
            err = __futex_abstimed_wait_cancelable64 (cond->__data.__g_signals + g, 0, clockid, abstime, private);
            // 重新加载信号个数，被唤醒之后，signal有变化
            signals = atomic_load_acquire (cond->__data.__g_signals + g);
        }
    }
    // 当前有需要唤醒的等待者，尝试获取signal
    while (!atomic_compare_exchange_weak_acquire (cond->__data.__g_signals + g, &signals, signals - 2));
    // 当前group的起始序列号
    uint64_t g1_start = __condvar_load_g1_start_relaxed (cond);
    // 优先消费之前被wait的等待者，而不是本次的等待者
    if (seq < (g1_start >> 1)){
        if (((g1_start & 1) ^ 1) == g){
            // 再次更新信号量
            unsigned int s = atomic_load_relaxed (cond->__data.__g_signals + g);
            while (__condvar_load_g1_start_relaxed (cond) == g1_start){
                if (((s & 1) != 0) || atomic_compare_exchange_weak_relaxed(cond->__data.__g_signals + g, &s, s + 2)){
                    // 优先唤醒其他等待者
                    futex_wake (cond->__data.__g_signals + g, 1, private);
                    break;
                }
            }
        }
    }
done:
    // 【用户锁，加锁】
    err = __pthread_mutex_cond_lock (mutex);
    return (err != 0) ? err : result;
}
```
上面的代码把异常处理、group的处理等都给省去了，即使如此，阅读这个代码可能依然有困难。
建议去glibc阅读源码（里面的注释很详细），搞清楚pthread_cond_t的每个字段的含义之后，阅读就轻松多了。
其实我们简化成以下伪代码就好理解了：
```c
pthread_cond_wait(cond_t* t, mutex_t* m){
    unlock(m);
    if(atomic_load(t->futex_signal) == 0){ //如果signal为0，则进入等待队列
        futex_wait(t->futex_signal, 0);
    }
    lock(m);
}
pthread_cond_signal(cond_t* t){
    atmoic_ftech_and_add(t->futex_signal，1);  // signal 加1
    futex_wake(t->futex_var, 1);  // 唤醒一个等待者
}
```

### 信号量
##### nptl中的实现
```c
int __new_sem_post (sem_t *sem){
    unsigned int v = atomic_load_relaxed (&isem->value);
    do{
        // 溢出
        if ((v >> SEM_VALUE_SHIFT) == SEM_VALUE_MAX){
	        __set_errno (EOVERFLOW);
	        return -1;
        }
    }/// 原子+1，如果有其他post导致value变换，则循环尝试
    while (!atomic_compare_exchange_weak_release(&isem->value, &v, v + (1 << SEM_VALUE_SHIFT)));
    // 如果有等待的线程，则唤醒一个线程
    if ((v & SEM_NWAITERS_MASK) != 0)
        futex_wake (&isem->value, 1, private);
}
```
信号量的源码比读写锁、条件变量好理解多了。
将这些代码当作lock-free编程去学习，你再去网上的其他lock-free的数据结构也会轻松不少。

### 并发问题
##### Non-Deadlock
在实际软件项目中，非死锁导致的问题可能比死锁导致的问题更多，下面我们看看常见的两种非死锁并发问题：
1. 违背原子性：简单来说就是对于临界数据没有加锁
2. 违背执行顺序：在多线程中由于执行顺序不正确导致的问题，通过条件变量、信号量 来保证同步就可以解决

这类问题看起来很简单，但如果项目工程很大，在处理这些问题时，你需要对代码有较深入的理解才能解决。

##### DeadLock
死锁产生的条件与解决方案：
1. 互斥：资源是独占的且排他使用
   解决（实现复杂）：不使用互斥锁，而采用wait-free的方案。
2. 请求和保持：进程每次申请它所需要的一部分资源，在申请新的资源的同时，继续占用已分配到的资源。
   解决（会导致性能差）：在线程获取多个锁之前，先加一把大范围的锁。
3. 不可剥夺：进程所获得的资源在未使用完毕之前，不被其他进程强行剥夺，而只能由获得该资源的进程资源释放。
   解决（可读性差，异常处理复杂）：使用trylock，尝试获取锁，如果获取锁失败，则释放已经占用的锁，从而让其他进程能获取锁，但释放锁的时候需要考虑资源释放与重新申请的问题。
4. 循环等待：比方A进程占用a锁，同时请求b锁，B进程占用b锁，请求a锁，导致AB进程互相等待
   解决（最常用的方法）：保持获取锁的顺序一致。
   
看完死锁的原因以及解决方案之后，并不能杜绝死锁的产生，究其原因，还是软件的复杂度导致的。
一般在软件设计中，组件之间的互相依赖、接口的封装 才是导致死锁的根本原因。
因此适当的接口使用说明、代码评审、测试覆盖以及代码漏洞检测，在提升软件质量上尤为关键

### 线程模型
最后，我们简单看下进程与线程的区别：
进程是资源分配的基本单位，线程是CPU调度的基本单位。线程有独立的线程栈、寄存器。

在《Modern Operating Systems》中提到的线程模型：
在linux中pthread库中采用的是1对1的线程模型，即一个用户对应一个内核线程，内核负责每个线程的调度。
但这种一对一的模型存在用户态、内核态切换频繁的问题。
因此为提升效率，由用户实现支持多对一的定时器模型，能提高一定的效率，但这种模型调度的任务不能阻塞，否则会导致其他任务也不能执行。（之前在某公司做嵌入式开发时，就遇到过这个问题）
![](Images/thread_module.png)

### 资料
书籍：《Operating Systems: Three Easy Pieces》
[线上书籍](https://pages.cs.wisc.edu/~remzi/OSTEP/)
《Modern Operating Systems》（第四版：有介绍futex，但真的只是介绍）
glibc源码地址：http://ftp.gnu.org/gnu/glibc/