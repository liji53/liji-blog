---
title: mysql-并发
date: 2021-09-04 13:20:46
tags:
categories: 数据库
---
# mysql-并发
对于MySql服务器来说，可能同时要处理多个事务。如果在处理多事务的时候，让事务排队进行这样对性能的影响非常大。如何让Mysql既有较好的隔离性，又有较好的性能，这就是MVCC和锁要做的事情。

### 隔离与MVCC

##### 1. 事务并发问题
1. 脏写：一个事务修改了另一个未提交事务修改过的数据
2. 脏读：一个事务读到了另一个未提交事务修改过的数据
3. 不可重复读：一个事务只能读到另一个已经提交的事务修改过的数据，并且其他事务每对该数据进行一次修改并提交后，该事务都能查询得到最新值
4. 幻读：一个事务先查询出一些记录，之后另一个事务又向表中插入一些记录，原先的事务再次按照该条件查询时，能把另一个事务插入的记录也读出来

如果按照问题的严重性来看：
脏写 > 脏读 > 不可重复读 > 幻读

##### 2. 隔离级别
对于上面的4种问题，SQL标准设立了4种隔离级别

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
| -------  | ---- | ---------- | ---- |
|READ UNCOMMITTED| Possible| Possible| Possible|
|READ COMMITTED| Not Possible| Possible| Possible|
|REPEATABLE READ| Not Possible| Not Possible| Possible|
|SERIALIZABLE| Not Possible| Not Possible| Not Possible|

这里没有脏写，是因为对于脏写来说，不管哪种隔离级别，都不允许发生。
当然这只是SQL标准要求实现的，在实际各种数据库的实现中，4种隔离级别的支持程度是不一样的。
如：oracle 只支持READ COMMITTED 和 SERIALIZABLE。
而Mysql虽然支持4种隔离级别，但实现上却并不完全按照这个标准来的（REPEATABLE-READ能禁止幻读的产生）
mysql中默认的隔离级别是REPEATABLE-READ。

##### 3. 事务id和版本链
在搞清楚隔离级别的实现之前，我们先了解下事务ID和版本链的概念。
如果某个事务执行过程中对某个表执行了增、删、改操作，那么InnoDB 存储引擎就会给它分配一个独一无二的事务id（只有第一次增、删、改才分配）,事务ID是向上增加的。
再来回看下数据/索引页的内容：
![](Images/undo_indexPage.png)
roll_pointer指向的是改记录对应的undo log，同时把历史undo log链接起来：
![](Images/MVCC_versionlist.png)
这个就是版本链的结构，其中头节点表示当前记录的最新值。

##### 4. 隔离级别的实现
对于READ UNCOMMITTED来说，由于可以读未提交事务的记录，因此对于该隔离级别只需要读最新版本的记录即可。
对于SERIALIZABLE来说，采用的是加锁的方式来实现的（等会就讲）
而REPEATABLE READ和READ COMMITTED 都需要保证读到的是已经提交事务的记录，因此它们通过版本链来实现(判断版本链中哪个版本是当前事务可见的)。
他们的区别就是：
1. READ COMMITTED在每次读数据之前都会生成一个ReadView
2. REPEATABLE READ只有在第一次读取数据的时候才生成一个ReadView，之后的查询复用这个ReadView

##### 5. ReadView
ReadView简单来说就是记录了当前有哪些活跃的事务，这样在遍历版本链的时候，就能判断该版本是否已经提交。
ReadView的数据格式如下：
```c
struct ReadView{
    int m_ids[n_max_ids];   // 当前活跃读写事务的事务id列表
    int min_trx_id;         // 当前系统中活跃读写事务中最小的事务id
    int max_trx_id;         // 当前系统中应该分配给下一个事务的id 值
    int creator_trx_id;     // 表示生成该ReadView 的事务的事务id,如果是只读事务则为0
};
```
查找已经提交事务的记录过程如下(伪代码)：
```python
query(ReadView, record):
    if record.trx_id == ReadView.creator_trx_id: # 正在访问自己修改的记录，返回这条记录
        return record                           
    if record.trx_id < ReadView.min_trx_id:      # 这条记录在生成ReadView时已经提交了
        return record                           
    if record.trx_id >= ReadView.max_trx_id:     # 有新的事务开启，需要递归查找下一个版本
        query(ReadView, record.roll_pointer)    
    else:                                       
        if record.trx_id in ReadView.m_ids:      # 这条记录的事务仍然活跃，查找下一个版本
            query(ReadView, record.roll_pointer)
        else:                                    # 说明这条记录已经提交
            return record
```

##### 6. MVCC(Multi-Version Concurrency Control)
所谓的MVCC指的就是REPEATABLE READ和READ COMMITTED这两种隔离级别访问版本链的过程。
现在再回头看下REPEATABLE READ和READ COMMITTED的过程区别：
```shell
Read Committed隔离级别：
        事务1                                                  事务2
                                                            update row = X
      select row(生成ReadView)      
                                                            committed
      select row(并不会返回X，因为ReadView中事务2是活跃的)
      
Repeatable Read隔离级别：
        事务1                                                  事务2
                                                            update row = X
      select row(生成ReadView)      
                                                            committed
      select row(重新生成ReadView，事务2不活跃，返回X)
```

##### 7.purge的过程
我们已经知道ReadView和MVCC是干什么的了，现在我们来看下purge的过程。
purge用于完成最终的delete和update操作，由于MVCC的存在，事务在提交之后，并不能立即把undo log删除，同时像delete操作也只是做了一个标记，并不是真正删除。而这些脏活累活就是由purge来完成的。
在事务提交之后，如果存在update和delete的操作，则会根据事务的提交顺序，生成history链表。
如图所示：
![](Images/purge_history.png)
purge线程会先从history list中找undo log，再从undo log所在的页中找undo log是否在被使用，这样可以避免大量的随机读写。
如果一个undo log页中存在仍然在使用的undo log，则表示整个页都不能回收。怎么判断是否仍然在使用呢？
通过找到最老的ReadView即可，最老的ReadView保存了那个时刻的活跃事务的快照，这些事务即使提交了，也不能回收。

### 锁
在开发中难免会遇到并发的处理，常见的并发处理参考[os-并发](https://liji53.github.io/2021/06/29/operatingSystem/os-concurrency/)，但在InnoDB中mutex、rwlock这种锁称之为latch，并不是我们这次要讨论的对象。
它们的区别如下：
![](Images/lock_latch.png)

##### 1. 多粒度锁
首先介绍两个基本的锁：
共享锁(S lock)：允许事务读一行数据
独占锁(X lock)：允许事务删除/更新 一行数据（插入通过隐式锁实现）
这一对锁的语义跟读写锁一致。因此不再介绍它们的互斥情况，但独占锁和共享锁是不存在饥饿现象的，同时存在要加独占锁和共享锁的情况下，总是独占锁优先。

刚才说的锁针对的是一行记录，称为行锁，但事务也可以对表进行加锁，这种就是表锁。表锁也分为共享锁和独占锁。
但表锁和行锁该如何配合呢，比方说一条记录已经加了X锁了，这时候要对这张表加S锁，如何知道这种表的其中某些记录有加X锁呢？
通过意向锁(Intention Lock)来实现，简单来说就是在加行S锁之前，先对这张表加IS锁，这样当需要对表加X锁时，就能知道这张表是否存在行S锁
他们的兼容情况如下：

|兼容性|X| IX| S| IS|
|---|---|---|---|---|
|X |不兼容|不兼容|不兼容|不兼容|
|IX|不兼容|兼容|不兼容|兼容|
|S|不兼容|不兼容|兼容|兼容|
|IS|不兼容|兼容|兼容|兼容|

虽然InnoDB有表锁的存在，单实际上表锁却是server层实现的东西，如对表进行Alter 、Drop等DDL操作。

##### 2. Gap锁和Next-key锁
在InnoDB中有3种行锁：
1. Record lock：对一行记录加锁
2. Gap lock：间隙锁，能锁定一个范围，单不包含记录本身
3. Next-key lock：前面2种锁功能相加，Gap lock + Record lock

前面我们说过REPEATABLE READ这种隔离级别是可以解决幻读问题的，解决幻读的办法就是使用gap锁。
![](Images/gap_lock.png)
但由于gap锁定的数据并不存在，因此对于gap锁来说，独占锁和共享锁是没有区别的，只要加锁了，如果其他事务要在这个范围内插入记录，就会阻塞。

##### 3. 锁的结构
如果每条记录都要生成对应的一个锁结构，对于要锁定大量数据的InnoDB来说，就又容易照成空间浪费。
因此它的如图所示：
![](Images/lock_struct.png)
其中”一堆比特位“表示的就是页内哪条数据被锁定了（一个比特对应一条记录）。
而对于行锁来说，图中n_bit表示的是”一堆比特位“的长度。

##### 4.死锁处理
既然有并发有锁，那就不可避免的会发生死锁。对于线程的死锁，应用程序一般不会加死锁检测机制，有死锁一般就要重启了。但对于数据库事务来说，死锁是很容易操作的事情，就不能简单重启来解决了。
当前数据库一般采用超时机制和wait for graph的方式来解决，其中wait for graph就是通过破坏锁的环形依赖这个条件来解决的。
如图所示：
![](Images/wait_for_graph.png)
图中t1和t2有环形依赖。
通常来说InnoDB选择回滚undo量最小的事务。

### 参考资料
书籍：
《MySQL是怎样运行的：从根儿上理解MySQL》
《Mysql技术内幕 InnoDB存储引擎》（第二版）
《高性能MySql》（第三版）
官方资料：
https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html
