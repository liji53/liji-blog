---
title: mysql-整体架构
date: 2021-08-11 19:20:30
tags:
categories: 数据库
---
# mysql-整体架构
现在的软件开发基本上都离不开数据库的支持，而最常用的关系型数据库有oracle、SQL Server、mysql等，但如果考虑到开源，mysql应该是最受欢迎的。
在《Mysql技术内幕 InnoDB存储引擎》中作者认为一个新的OLTP项目不使用mysql InnoDB存储引擎是非常愚蠢的（非常的自信）。


### 数据库的分类
关系型数据库(sql)：适合存储结构化数据(抽象成二维表格模型)，支持事务保证ACID，支持SQL适合复杂、多表的查询，缺点是海量数据的读写效率以及可扩展性。
非关系型数据库(NoSql)：常给web应用提供可扩展的高性能数据库，有redis、HBase、BigTable、MongoDB

行数据库和列数据库：行数据库存储数据的方式是一行行存储的，适合OLTP应用。列数据库按列存储，因此每一列数据都为索引(数据即索引)，适合OLAP应用

OLTP(on line transaction processing)：对数据做增删改的操作频繁的应用，要求稳定性、实时性。如银行交易系统、淘宝购物等。
OLAP(on line analytical processing)：对数据的查询频繁、查询量大的应用，适合做数据分析，为决策提供支持。

按照上面分类，我们的主角InnoDB属于关系型数据库-行数据库-目标群体主要面向OLTP的应用。

### 整体架构
在学习mysql之前，我们要先熟知它的整体框架:
![](Images/arch_mysql.png)
server层(后续介绍优化器)在这里不做介绍。而存储引擎我们只看InnoDB的实现。
InnoDB与其他mysql存储引擎的比较，参考官网：https://dev.mysql.com/doc/refman/8.0/en/storage-engines.html
InnoDB的架构：
![](Images/arch_InnoDB.png)
聚焦到内存池以及存储磁盘的架构：
![](Images/innodb-architecture.png)
这张图有点复杂，我们先看内存部分。

### 内存架构
##### 1. buffer pool - 结构
InnoDB会申请一大块内存，并对这块内存进行分页，一个页的大小是16K。
![](Images/buffer_pool.png)
其中控制块记录了对应缓存页的表空间ID、页号、链表节点信息等。
而在多线程的环境中，为了减少锁的开销，一般会通过减少锁的空间颗粒度从而减少锁的冲突(参考[操作系统-并发](https://liji53.github.io/2021/06/29/operatingSystem/os-concurrency/))
因此将buffer pool设计成多个小的buffer pool，能有效提升多线程的性能。
如图所示：
![](Images/buffer_linklist.png)
其中chunk是mysql申请内存的单位，这样可以细化内存申请的颗粒度，避免大规模的内存拷贝。

##### 2. buffer pool - free链表、Flush链表、LRU链表
mysql针对不同的业务场景，将不同状态的缓存页的控制块按照链表的形式链接了起来。
1. free链表, 当我们要访问数据时，先通过哈希表（key为表空间ID+页号，散列算法为mod，哈希桶的个数为比buffer pool页数*2大一点的质数）来快速定位是否在buffer pool中，如果不在pool中，则从free链表中取一个空闲的缓存页控制块
![](Images/buffer_freelist.png)
2. flush链表，当我们修改了这个页的数据，于是就变成了脏页，需要在未来的某个时间点写入磁盘。而flush链表就是记录了哪些页需要flush到磁盘。
3. LRU链表，如果buffer pool的空间满了，而需要访问的页没有命中时，就需要从磁盘中读一个页，并把当前buffer pool中的一个页淘汰掉，LRU链表管理了缓存池中页的可用性。

Free和flush链表不涉及太复杂的策略，但LRU的策略是我们需要清楚的（LRU策略会经常遇到，比方在[操作系统-虚拟memory](https://liji53.github.io/2021/06/19/operatingSystem/os-menoryVirtual/)就遇到过）
mysql会把LRU链表分成2部分：young和old,分表表示热数据和冷数据
如图所示：
![](Images/buffer_LRUlist.png)
它的策略如下：
1. 每次从磁盘中读一个新的页时，会把新的页放入到old区的头部。这样就能防止把一些真正的热数据淘汰
2. 记录缓存页第一次访问时间，如果在一定时间间隔后，再次访问了这个缓存页，就会把它移动到young区域的头部。防止出现短时间内访问大量使用频率非常低的页面。
3. 第二条策略会导致链表节点移动频繁，因此只有被访问的缓存页位于young 区域的1/4 后边，才会被移动到LRU链表头部。

##### 3. change buffer(insert buffer)
change buffer是insert buffer的升级，官方文档中就叫change buffer。
change buffer只有在对非唯一的二级索引进行DML操作才会使用。
必须是二级索引的原因：二级索引对主键的存放是散列的，对于HDD存储，随机读写的代价是巨大的，参考[文件系统](https://liji53.github.io/2021/07/10/operatingSystem/os-fileSystem/)。因此在DML操作时，如果需要修改的二级索引页不在buffer pool中，则会先放入到change buffer，避免随机读写。
而必须是非唯一索引的原因：非唯一索引不需要读整个索引判断是否唯一，再次避免了随机读
它的架构如图所示：
![](Images/innodb-change-buffer.png)
从图中可以知道：
1. change buffer 属于buffer pool的一部分（默认最大使用25%的buffer pool）
2. InnoDB会以一定的周期将insert buffer中的索引页和实际数据页合并。如果宕机了，inset buffer数据较多的话，需要花大量时间恢复数据。

change buffer页的数据也是按照B+树组织的，其中叶子节点存放了二级索引的记录。
insert buffer bitmap用于记录实际二级索引页的剩余可用空间(下一节看它的物理位置)，帮助change buffer 与二级索引进行合并。

##### 4. 自适应哈希索引
从内存池的架构图中，我们可以知道，自适应哈希索引是对buffer pool中的数据页建立索引，并不是对全表建立索引。
而有意思的是自适应这个词，这表示InnoDB会监控对索引页的查询，只有在一定情况下才对数据页建立索引，它的策略如下：
1. 对数据页的访问模式必须是一样的（如where a = 'xxx', 不支持范围查询）
2. 以条件1的模式访问了100次
3. 以条件1的模式访问了N次，其中N=页中记录*1/16

同时为了减少锁的冲突，InnoDB将自适应哈希索引默认分成8个分区(可以通过参数修改)
注意：在buffer pool中我们提到了用于判断数据是否在buffer pool的哈希表，这两个不是一回事。

##### 5. log buffer
这里的log 指redo log，不包括undo log（后面详细介绍这两个）。
log也需要写入到磁盘，跟普通数据一样，不可能有一条log就立马写入磁盘，因此redo log同样需要buffer（默认16M）。
redo log buffer同样也进行分页管理，一个页是512B（跟HDD的扇区大小一致，这样能保证原子，因此redo log 不需要double write策略）。
如图所示：
![](Images/redo_buffer.png)
关于redo log buffer 的管理，我们后面再讲，先知道有这个东西即可。

### 后台线程
这里我们主要讲InnoDB中的后台线程，而不是server层用于负责连接以及负责执行sql的线程。
##### 1.线程分类
InnoDB中主要有以下4种线程:
1. Master Thread: 主要负责将缓存池中的数据异步刷新到磁盘。
2. IO Thread：主要负责异步IO的回调处理，默认有4个read、4个write、1个insert buffer、1个log
3. Purge Thread：事务提交后，undo log可能不再需要了，该线程用来回收无用的undo log页。（回收的策略由于涉及到read View，在后面讲）
4. Page cleaner Thread: 将脏页的更新都放到这个线程中执行

##### 2.Master Thread
Master Thread的逻辑参考《Mysql技术内幕 InnoDB存储引擎》，只写了部分逻辑
```python
def master_thread():
    while True:
        if InnoDB is idle:  
            # innodb_io_capacity(参数)表示磁盘吞吐量，默认200
            # 官方给出的参考是如果是7200RPM的HDD，建议把这个参数改成100，如果是SSD磁盘，建议把这个参数改大
            if last_ten_second_io < innodb_io_capacity:
                do_buffer_pool_flush_to_disk(100)   # 将100个脏页写如磁盘
            do_merge_insert_buffer()                # insert buffer的索引合并到硬盘
            do_log_buffer_flush_to_disk()           # 将redo log刷新到磁盘
            do_full_purge()                         # 回收undo 页
            if buf_get_modified_ratio_pct > 70%:     
                do_buffer_pool_flush_to_disk()      # 脏页刷新到硬盘，新版本中已经放到Page cleaner Thread中执行。
        else: 
            do_log_buffer_flush_to_disk()           # 将redo log刷新到磁盘
            if last_one_second_io < 5 % innodb_io_capacity:
                do_merge_insert_buffer()            # 磁盘io压力不大,将insert buffer的索引合并到硬盘
            # buf_get_modified_ratio_pct(参数)，默认90
            # 当脏页的数量大于90%时，则进行脏页刷新
            if buf_get_modified_ratio_pct > innodb_max_dirty_pages_pct:
                do_buffer_pool_flush_to_disk()      # 脏页刷新到硬盘，新版本中已经放到Page cleaner Thread中执行。
            if no_user_active:
                do_full_purge()                     # 回收undo 页
            sleep(1)
```
主要是干这些事情：
1. 刷新脏页到磁盘（已经交给新的线程做了）
2. 合并插入缓存的数据（尽量在空闲的时候）
3. 将redo log数据刷新到磁盘（为保证原子性，总时执行）
4. 回收无用的undo log页（已经交给新的线程执行了）

### 参考资料
书籍：
《MySQL是怎样运行的：从根儿上理解MySQL》
《Mysql技术内幕 InnoDB存储引擎》（第二版）
《高性能MySql》（第三版）
官方资料：
https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html
