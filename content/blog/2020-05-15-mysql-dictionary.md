#### Double Write
Double Write(DW)的用处：
如果写盘的时候挂掉，硬盘的表就挂了。
不知道此时undo log是否可以undo，估计要回退的多一些
DW是说我先写到另外一个位置：
如果DW时候挂了，那么表没问题，可以直接redo
如果DW没问题，写表的时候挂了，那么可以直接从DW文件恢复
DW是顺序写，所以对性能影响不大
ref: https://www.percona.com/blog/2006/08/04/innodb-double-write/

#### Fsync Performance
对于7200rpm的HDD，一次fsync大约需要2转多，算下来只能支持50TPS。
SSD的性能差别很大，从100到10000TPS。
ref: https://www.percona.com/blog/2018/02/08/fsync-performance-storage-devices/
对于表文件的读写，一批page会用一次fsync。

#### Adaptive Hash Index
innodb会自动为经常查询的索引key做hash索引。这显然是动态hash索引
(DSC 24.5.2.3)
自适应hash索引的性能分析如下：
1. 如果hash索引miss，回到b+树索引
2. 维护hash索引的开销：表大小，缓冲区大小
   
表增大，hash碰撞的几率增加，查询和维护的开销都增加
如果表不能完全放在缓冲区中，维护的开销会极度增加
3. hash索引竞争的开销
ref: https://www.percona.com/blog/2016/04/12/is-adaptive-hash-index-in-innodb-right-for-my-workload/

#### Gap Lock
`SELECT * FROM id > 1000 FOR UPDATE`
innodb对满足WHERE条件的行加X锁，对间隙加S锁
ref: https://www.percona.com/blog/2012/03/27/innodbs-gap-locks/

#### Metadata Lock
为了防止事务运行的时候表结构改变，事务开始时会获取元数据锁，提交之后释放。
ref: https://dev.mysql.com/doc/refman/8.0/en/metadata-locking.html
