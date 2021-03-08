MVCC(Multi Version Concurrency Control的简称)，代表多版本并发控制。与MVCC相对的，是基于锁的并发控制，Lock-Based Concurrency Control)。


mvcc实现关键

- 并发控制协议 ( concurrency control protocol )
- 历史版本数据存储 ( version storage )
- 垃圾回收 ( garbage collection )
- 索引管理 ( index management )

是指为dbms里面的逻辑对象维护多个物理版本，这样同一个对象允许不同的并行操作。

没有标准实现，不同的实现有不同的取舍和性能。

MVCC最大的优势：读不加锁，读写不冲突。在读多写少的OLTP应用中，读写不冲突是非常重要的，极大的增加了系统的并发性能。

实现

原数据

事务：事务在创建时，系统会分配一个单调递增的时间戳作为标识符。通过这个可以标识数据库里面逻辑对象的版本或者作为事务的线性顺序

元组：每个物理版本包含四个元数据字段

txn-id 充当当前版本的写锁，没有被持有则是0