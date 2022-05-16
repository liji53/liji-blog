---
title: mysql-事务
date: 2021-08-29 18:42:00
tags:
categories: 数据库
---
# mysql-事务
先看下事务的概念：需要保证原子性、隔离性、一致性和持久性（AICD）的一个或多个数据库操作。

### redo log
InnoDB的事务持久性，通过redo log来保证，简单来说就是把对数据的修改记录下来，如果出现宕机，在重启的时候对数据进行恢复(与double write的区别[mysql-存储布局]())。
它的机制可以与日志文件系统如ext3、ext4 对比学习，参考[操作系统-文件系统](https://liji53.github.io/2021/07/10/operatingSystem/os-fileSystem/)

##### 1.redo log格式
InnoDB通过redo log来记录事务对数据库做了哪些修改(也包括undo log的修改)。它的格式如下： 
![](Images/redo_log.png)
1. type 表示日志的类型
2. space ID + page number 表示表空间的ID和page number （指向修改了哪个页）
3. data 表示修改的内容。举个例子（修改字符串），本字段只要记录：页内的偏移 + string的长度 + 新string的值。

但是实际插入一条数据，涉及修改的内容可能会很分散，比如：
![](Images/redo_insert.png)
如果每个修改点都记录一条日志，这个日志就会变得很大。因此针对这种复杂的修改，mysql专门设计多种针对某种复杂操作的日志。
比方针对插入一条紧凑行格式的记录(格式比较复杂，不展示)，在恢复的时候判断type类型，然后调用专门用于恢复这类日志的函数，函数中自己计算哪些数据涉及修改

##### 2. Mini-Transaction（redo log分组）
还是以插入一条数据为例，当插入数据时，如果当前页的数据已经满了，这就会导致页分裂，我们至少需要2条(实际更多)redo log来记录这次的修改。
在修复的时候，这2条redo log 不能只恢复其中1条，因此需要将这两条日志当成一个组进行恢复，这个组的所有日志要么都恢复，要么都不恢复。
怎么解决呢？参考流的处理，一般要么标记这个流的长度，要么在流的尾巴加一个特殊标志如‘\0’。
因此在一组redo log的末尾加入一个特殊的redo log，如下图所示：
![](Images/redo_tail.png)
在系统恢复的时候，只有解析到类型为MLOG_MULTI_REC_END 的redo 日志，才认为是一组完整的redo 日志并恢复。
我们把一次原子操作的过程称为Mini-Transaction，而一条语句可能存在多个Mini-Transaction，如下图：
![](Images/redo_mini_transaction.png)

##### 3.log buffer 管理
还记得[mysql-整体架构]()和[mysql-存储布局]()中提到的redo log的内存和物理结构吗？
下面我们看下redo log buffer的管理。
InnoDB并不是一下子把整个log buffer的数据写入到磁盘中的，因此实际写入的redo log和内存中的redo log是有速度差异的，需要2个指针分别表示：
![](Images/redo_lsn_add.png)

##### 4. checkpoint
如果脏页已经刷到磁盘中了，那么对应的redo log也就没有用处了，后面新产生的redo log就可以覆盖。
结合我们上面已经提到过的2种状态（从哪里开始写内存buffer，从哪里开始写入磁盘），现在新增redo log是否可以被覆盖(脏页已经写入磁盘) 这一种状态。
新的状态如下：
![](Images/redo_checkpoint.png)
同样需要一个指针(checkpoint)记录哪些redo log已经无效了。
现在我们在来整理下这个checkpoint的过程：
1. 脏页数据写入到磁盘，将flush链表中对应的控制块移除
2. 更新内存中无效redo的指针：找到flush链表中最老节点的redo log位置值赋值给checkpoint
3. 更新硬盘中无效redo的指针：将新的checkpoint的位置 写到日志文件

##### 5. 恢复过程
1. 确认恢复的起点：从日志文件的头部信息中获取checkpoint的位置
2. 依次扫描起点后面的redo log，并按照日志的内容进行恢复

但修复的时候一个页面可能有好几个redo log，为了加快修复的过程，mysql设计了哈希表，对于同一个页面的数据一次性修复：
![](Images/redo_repair.png)

### undo log
redo log用来恢复提交事务修改过的物理页，而undo log用来记录事务修改过的记录的版本。
undo log 用于保证事务的原子性和隔离性，也可以说用来帮助事务的回滚和MVCC功能。
了解下回滚的概念：当事务失败了，撤销事务对当前数据库造成的影响，从而保证事务的原子性

##### 1. 需要undo log的操作
需要回滚的操作，就三种：增、删、改。而查询是不需要回滚的。
它的基本思路如下：
1. 对‘增’的操作，只要把这条记录的主键值记下来，在回滚的时候删除这个主键值对应的记录即可
2. 对‘删’的操作，要把删除记录的数据全部记录下来，在回滚的时候再把这些内容插入到表中
3. 对‘改’的操作，要把修改记录的旧值记录下来，回滚的时候把这条记录更新回旧值

##### 2. undo log
InnoDB将插入、删除、更新数据时需要记录的内容写到undo log中，
插入undo log格式如下：
![](Images/undo_insert.png)
可以看到undo日志会将主键的值和长度记录了下来
删除和更新undo log格式比较复杂，思路上面已经写过，不再展示
undo log日志类型虽然有多种，但只分为2类:insert类和update类（包括删除和更新）
这么分类的原因是insert类型的日志在事务提交之后就可以直接删除，而update类的undo log不能删除（MVCC中讲原因）

##### 3. undo log缓存
undo log是作为一种页类型出现在buffer pool中的（注意与redo log buffer 的区别），关于表空间-页的介绍，参考[mysql-存储布局](https://liji53.github.io/2021/08/14/database/mysql-layout/)
一个事务可能包含多个语句，一个语句可能产生多条undo日志。因此一个事务产生的undo log可能一个页面放不下，需要放到多个页面中。
另外，普通表和临时表的的undo日志要分别记录，并且undo log又分为insert和update类型。
如图所示：
![](Images/undo_link.png)
这就是一个事务产生的undo log的内存布局，当然实际中并不会直接分配4个链表，而是按需分配

##### 4. undo log页重用
一个事务可能产生很多的很多条undo log，也可能只有1、2条undo log。因此若为每个事务都分配一个单独undo页会非常浪费空间。
mysql在事务提交之后，在某些情况下对这些undo log页进行了重用。
它的策略如下：
1. 链表中只包含一个Undo页面
2. 该Undo页面已经使用的空间小于整个页面空间的3/4

而insert类型的undo log 和update类型的undo log在重用的时候又不一样：
1. insert类型可以直接覆盖
2. update类型不能直接删除，因此在页的剩余空间中继续写（需要等purge线程回收undo log页）

关于undo log的分配过程有点小复杂，可以从《MySQL是怎样运行的：从根儿上理解MySQL》找答案。
而purge的过程还要再等等。

### 参考资料
书籍：
《MySQL是怎样运行的：从根儿上理解MySQL》
《Mysql技术内幕 InnoDB存储引擎》（第二版）
《高性能MySql》（第三版）
官方资料：
https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html
