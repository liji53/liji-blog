---
title: mysql-存储布局
date: 2021-08-14 22:20:07
tags:
categories: 数据库
---
# mysql-存储布局
现在的软件开发基本上都离不开数据库的支持，而最常用的关系型数据库有oracle、SQL Server、mysql等，但如果考虑到开源，mysql应该是最受欢迎的。
下面我们将开始解密mysql数据库的底层原理，但在学习mysql之前，我们要先熟知它的整体框架:
![](Images/arch.png)
这里我们跳过server层，直接来到存储引擎一层(只看InnoDB)。
可能我们平常在软件设计时，大部分打交道的都是内存中的数据结构。这次我们看看存储介质中的数据结构设计。

### 表空间布局   
先学习表空间的数据布局，是因为了解存储结构之后（底层机制）更加有助于我们学习上层策略(索引、事务等)。
表空间其实就是一个文件，下面我们就要讲这个文件内部数据是如何管理的。

##### 1.表空间结构
mysql对表空间的管理，按照"组(group)-区(extent)-页(page) "进行管理，其中页的大小为16k，也是mysql操作的最小单元；页的上一级是区，管理了64个页，一个区的大小为1M；最上层的是组，管理了256个区。
如图所示：
![](Images/table_space.png)
其中我们看到像FSP_HDR、IBUF_BITMAP 等这些16k大小的条目就是所谓的页。
这里我们需要清楚mysql这么设计的原因：
1. 让数据连续存储，在进行范围查询(范围查询等讲索引时揭秘)时能减少随机IO。
2. 虽然表空间是一个文件，但物理上可能是不连续的(参考[文件系统](https://liji53.github.io/2021/07/10/operatingSystem/os-fileSystem/))，在数据量到达一定程度时，mysql按区来申请空间，用于保证物理空间的连续。
3. 将表空间划分成组、区，这样能方便管理页

图中的表空间虽然比较简单，但在细入到各个页的结构之前，我们还需要清楚其他的一些基本概念。
 
##### 2.段与碎片区
mysql里面也有段的概念，跟编译时将程序分成代码段、数据段的概念类似，mysql将每个区、甚至页按照功能属性进行逻辑分类，比方说一张表至少可以分为数据和索引2个段。
段是按照区为单位进行申请的，但如果一张表的数据量很小，这时候也按照区为单位进行空间申请就可能照成浪费。
因此有专门的碎片区用于存储这些数据量比较小的表，等这些少量数据增长到一定大小的时候，就开始按照区为单位进行申请。

##### 3.系统表空间
除了上面的独立表空间(用户创建的表)，还存在系统表空间，用于存储整个系统的属性：
![](Images/sys_table_space.png)
我们看到除了第一个组的第一个区不一样，其他结构跟独立表空间一摸一样。而第一个区新增了第4-第8页，分别表示:
1. SYS Insert Buffer Header 存储Insert Buffer的头部信息
2. INDEX Insert Buffer Root 存储Insert Buffer的根页面
3. TRX_SYS Transction System 事务系统的相关信息
4. SYS First Rollback Segment 第一个回滚段的页面
5. SYS Data Dictionary Header 数据字典头部信息

这5种页等用到的时候在单独介绍他们的结构，在这里不再展开。

##### 4.页的通用部分
每个页都是16k，它们(共有11种页)有相同部分的结构：
![](Images/page_common.png)
其中File Header描述了页的通用信息，比方说页号、页类型、校验和、上一个页号、下一个页号、日志序列号等。
File Trailer 用于校验页是否完整，它的校验过程:
1. 修改数据时，File Header记录下新数据的校验和。
2. 先写入File Header和数据内容，等File Header写入成功，再写入File Trailer
3. 如果中间宕机了，只要比对File Header和File Trailer的校验和是否一致就性了

### 表空间头部信息页(FSP_HDR)和扩展描述页(XDES)
FSP_HDR和XDES的结构类似，可以对照着一起看。
##### 数据格式
FSP_HDR页的格式：
![](Images/FSP_HDR.png)
XDES页的格式：
![](Images/XDES.png)

##### 区块描述符(XDES Entry)
FSP_HDR和XDES都有一张XDES Entry表，这张表记录了所属组256个区的信息。
这些信息包括：
1. 属于哪个段(如果是碎片区则无效)；
2. 上一个XDES Entry和下一个XDES Entry的指针；
3. 区的种类（包括空闲区、有剩余空间的碎片区、没有剩余空间的碎片区、附属于某个段的区）；
4. 页的空闲bitmap表

段和碎片区的概念大家已经清楚了，而第4项用于快速找到当前组剩余的空闲页。
下面我们重点看下它的2个指针的作用，这种结构一般用于双向链表。
假设你要插入一条数据，这时候如果数据比较少，则mysql从碎片区找空闲页
而如果当前的数据量比较大，mysql则从对应的段中找有空闲页的区
因此如果不使用链表，而是全遍历，就又影响性能了。
mysql根据区的种类以及段ID将不同类型的区链接了起来。
直属于表空间：
&nbsp;&nbsp;&nbsp;&nbsp;1. 碎片区：
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1.1 空闲碎片区链表
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1.2 有剩余空间的碎片区链表
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1.3 没有剩余空间的碎片区链表
&nbsp;&nbsp;&nbsp;&nbsp;2. 段1：
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2.1 属于段1，空闲区链表
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2.2 属于段1，有剩余空间的链表
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2.3 属于段1，没有剩余空间的链表
&nbsp;&nbsp;&nbsp;&nbsp;3. 段2：
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;...

将这些区链接之后，就不需要对所有的XDES entry进行遍历了。(可以适当的想想内存池管理)

##### File Space Header结构
前面我们讲了XDES Entry的双向链表，但链表的入口在哪里的？
其中碎片区的链表入口在File Space Header，这个结构体属于FSP_HDR特有的，因此我们可以把碎片区理解成是表空间直属的。
由于在FSP_HDR页格式的图中已经描述了每个字段的内容，因此不再细说。

### 段信息节点页(INODE)
XDES Entry用于描述区的信息，而INODE Entry用于描述段的信息。

##### 1. 数据格式
INODE页的格式：
![](Images/INODE_page.png)
如果表空间中的段特别多(索引很多)，一个INODE页可能放不下所有的INODE Entry，因此需要申请新的INODE页，于是mysql也把Inode页链接了起来。
List Node for INODE Page List就是用于链接的(前后指针)，而这个链表的入口也在FSP_HDR的File Space Header中。

##### 2. 段描述符(INODE Entry)
格式如下：
![](Images/INODE_entry.png)
前面在讲XDES Entry的双向链表时，我们已经把碎片区的链表入口找到了，而属于某个段的链表入口则是在这个结构中。
除了3个链表的入口以外，还有32个碎片页的页号记录。这32个碎片页是在表数据小的时候用到。。。（回看"段与碎片区"的描述）

### 数据/索引页(INDEX)
前面讲的都是mysql用于管理所用到的数据格式，下面我们看看用户的实际数据是怎么存储的。
在mysql中存储数据的页和存储索引的页在结构上是没有区别的。

##### 1.数据格式
![](Images/data_index.png)
其中Infimum和Supremum是最小、最大记录，这2条记录是系统自动给我们生成的。

##### 2.行记录格式
每一个用户记录的格式：
![](Images/record_head.png)
图中第一行的内容如下：
1. 变长字段长度列表：存储varchar字符串类型的时候，用于记录字符串的长度
2. NULL值列表：记录列的值是否为NULL的bitmap表，只要创建创建表的时候没有not NULL声明，就会有相应的bit位。
3. 记录头信息：重点关注next_record用于链表(理解成next指针)，这是为了让mysql的数据按照主建的大小顺序排列。

##### 3.页目录（Page Directory）
在一个页中，mysql除了会把记录从小到大排序以外，还会对页内数据进行分组，而且每个分组的记录数在1-8条，如果超过8条，则会将记录进行拆分。
这么做的原因自然还是为了加快查找速度，通过分组只要遍历组和相应组的1-8条记录即可。
页目录就是用来存放分组信息(当前组最大节点的偏移地址)：
![](Images/page_directory.png)
图中内部数据指针省略了。

##### 4.页面头部（Page Header）
这个结构自然用于存放本页的全局信息。比如有多少条记录，第一条记录的入口地址， Page Directory槽的个数，还未使用的空间大小等，感兴趣的自己看书。

### 总结图
上面的这些数据结构除了专业搞mysql的，可能根本记不住，因此如果某一天要用到了，可以参考下面这张全图:
![](Images/overall.png)
这些数据结构背后的设计，更加值得我们学习，比方：
1. 文件内部分组、分区、分页的分层管理
2. 段与碎片区的概念
3. 在文件中如何快速找到空闲的页
4. 数据量大时和数据量小时应该如何处理
5. 文件中如何实现链表、双向链表、页目录(类似散列表)
6. 如何避免随机IO，如何提高磁盘读写IO
7. 如何保证数据的一致性(日志文件系统也有类似的机制)
等等。。。

### 资料
书籍：
《MySQL是怎样运行的：从根儿上理解MySQL》图片均来自本书
