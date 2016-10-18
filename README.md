# rocksdb-replication
A library for replication between master and slave 
# 引言
在我们开发的评论分布式存储系统中, 使用Rocksdb作为单节点的存储引擎,采用Master/Slave异步同步方案来保证单节点数据的高可用性. 该文档主要介绍用于Master/Slave异步同步库的设计和实现

# 功能
 - 支持主从同步
 - 支持一主N从
 - 支持数据拆分
 - 支持角色迁移

# 日志
类似于Mysql的Statement-based binary logging方案, 数据更新流水日志首先持久化到本地磁盘,然后顺序将日志同步给Slave,在Slave上重新replay来达到Slave与Master数据同步目的. 对本地流水日志的实现有一些特定要求

 - Sequence Number
日志模块必须提供一个单调递增的Sequence Number,标记提交给Rocksdb的每一个更新操作,就像TCP协议中的Sequence Number,每个byte都对应一个独立的Sequence Number. 当TCP数据丢包后,能够通过Sequence Number来判断那些数据需要重传.类似的,Sequence Number为Slave提供同步的进度信息.

如果没有这个Sequence Number, 当Slave与Master之间的连接由于网络闪断,重新连接Master后, Master不知道那些日志已经同步给Slave了,那些没有,缺少同步context信息, 只能像Redis早期的数据同步方案一样,从头开始同步,效率差

 - Atmoic
应用的数据操作具有一定的事务性,比如应用的一次操作可能包含多条记录的更新, 这些操作之间不是相互独立的,而是相互有关联.这些操作要么全部成功,要么全部失败.除了DB本身需要支持事务外, 日志记录也需要保留这种操作的事务性或则原子性.不可将包含多条数据更新操作的事务分解为 多条独立的日志记录进行存储, 否则这多条日志记录会单独在Slave上replay,最后造成Slave上的读操作出现不一致,读取到不该读取的数据.

虽然从纯粹数据同步角度来说,分解为多个单独记录没有问题,Slave最终都会与Master数据一致,但却影响了Slave读操作的正确性

# 日志实现
 - 第三方日志流水
在提交请求到Rocksdb之前,将所有操作打包为一条日志记录写入第三方日志流水中(有自己单独的日志文件), 优点是能够服用现有的日志模块,相对简单

 - WAL(Write ahead log)
Rocksdb新写入的数据保存在内存当中, 而内存的数据并没有持久,所以Rocksdb所有写入的数据,首先是写入内部的日志文件WAL. 同时WAL为每个操作(Put/Delete)分配唯一的单调递增的Sequence Number;WAL提供了一定程度的原子性,包含多个操作的WriteBatch,要么全部成功,要不全部失败,不会出现部分提交的情况.Rocksdb内部也是利用WriteBatch来保证数据的一致性

既然Rocksdb内部已经有日志文件,而且满足作为日志流水文件的需求,我们选择基于WAL来实现数据同步,不需要将数据日志重复写入第三方日志文件中,提高效率

# 数据同步
主从同步具体步骤也比较常见,分为两个部分

 - 初始镜像同步 Snapshot Synchronization
 - 增量同步 Delta Synchronization

## 增量同步
Slave启动时,根据本地DB的Sequence Number与Master请求同步, 如果Master本地日志的最小Sequence Number比之小,则直接进行增量同步

Master为每一个Slave开一个同步线程, 从初始Sequence Number开始读取日志记录,然后同步给Slave.为了实现较高的吞吐量,利用TCP保证数据的有序性,Master只需将日志写入Slave通讯的Chanel中,无需等待每个日志的结果.Slave单线程replay日志

## 初始镜像
Slave启动时,根据本地DB的Sequence Number与Master请求同步,如果Sequence Number比Master日志保存的最小Sequence Number还小的话,那么首先需要做一个Snapshot Synchronization

Rocksdb内部的所有数据文件都是Immutable,得到一个初始镜像相对简单,首先Disable内部文件删除操作,获取当前Rocksdb的meta文件(manifest文件)的一个pinpoint snapshot, (Rocksdb的meta文件只会不断append),其实就是一个文件offset; 还有当前有效的数据文件

也就是初始镜像由两部分组成:

 - 所有immutable的sst文件
 - 描述这些immutable的meta文件
Master将这些镜像文件通过Socket发送给Slave,Slave接受完Snapshot文件后,打开DB获得当前最大的Sequence Number.然后根据该初始Sequence Number去与Master做增量同步.

初始镜像同步完成后,Master在增量同步之前,Master还需要检查Sequence Number是否小于本地日志中的最小Sequence Number. 如果小于,则说明Snapshot同步时间过长,WAL日志文件保留太小,Slave需要的日志已经被删除. 出现这种情况两种方法,要么加速Snapshot过程,要么增大WAL日志

初始镜像同步完成后一定不要忘了Enable内部文件删除操作!

# 数据拆分
数据拆分需求的场景:

单节点存储容量达到上限,无法继续写入新的数据
单节点数据访问出现热点,拆分数据来提供更高的并发度
一个单节点存储容量达到上限后,需要对节点数据进行平滑拆分.支持对单节点数据一拆N. 采用的数据拆分方案, 首先拆分节点作为一个新的slave,连接到master同步一个全量镜像数据, 在slave中对数据进行拆分,删除不属于自己维护的数据,然后增量过滤从master同步过来的增量数据, master等待所有的slave追赶上来后,停写,更新路由. 停写之前,master支持读写操作. 实现数据平滑拆分

1. 数据拆分接口

提供拆分过滤接口,应用自定义数据拆分逻辑

2. 数据删除实现

有几种方案

 - Delete
Scan Rocksdb数据,将不属于拆分分片的数据采用Delete接口删除,实现简单,但这种方式的缺点, Rocksdb的delete不是一个in place的删除,而是新写入一个新的tomber marker,这样会造成数据膨胀,本来可能进行数据拆分操作就是因为单节点存储容量已经达到了上限, 现在反而写入了更多的数据, 如果磁盘冗余容量不足,删除操作可能无法完成

同时对Rocksdb本身来说,大量的delete marker会影响查询的效率,而且会触发内部剧烈的compaction操作

 - 重构Rocksdb
为了达到最高的效率,我们采用了这样一种方案,对Rocksdb内部的数据文件sst,逐个进行重写,删除不必要的数据,一对一的重新生成新的sst文件; 重新构建Rocksdb的meta文件. 通过这种方式来达到快速删除数据的目的, 效率更高, 对后续读操作没有影响

## 重写优化
测试过程中发现重写单线程,CPU是瓶颈. 最后重写线程改为多线程,snapshot synchronization的时间缩短几倍
老sst文件增量删除, 当对一个sst文件重写完成后,尝试马上删除老sst来减少对存储空间的占用
拆分增量更新
拆分增量同步过程中, 从master推送过来的增量日志记录WriteBatch,需要经过拆分过滤接口进行选择, 只将属于本分片的数据进行replay,否则丢弃

大部分应用场景下,一个WriteBatch包含了对某一个主题的相关修改,比如修改某一个用户的名字和生日等信息. 由于Rocksdb是一个Key/Value的数据模型,所以大部分应用通过设计共同的前缀key的方式来将一个用户的信息聚合在一起,方便数据的操作. 而且也会根据前缀key来进行sharding. 所以从应用的角度来讲,一个WriteBatch只包含了一个相关主题的修改,被一个分片进行存储和提供服务. 但在存储上却稍有区别

在多核机器上,Rocksdb内部为了提高并发写入速度,实现所谓的Group Commit功能. 将多个线程提交的WriteBatch合并为一个大的WriteBatch,批量提交,提高吞吐量.

这样一个WriteBatch可能包含多个相关主题的修改,不是完全保留或者完全丢弃, 需要对WriteBatch进行拆分,构建新的WriteBatch

# 开源

The library will be open sourced on Github soon,Stay tuned
