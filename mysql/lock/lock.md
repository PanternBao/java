幻读是什么

怎么解决幻读



表除了主键 id 外，还有一个索引 c，初始化语句在表中插入了 6 行数据

```sql
CREATE TABLE `t` ( 
  `id` int(11) NOT NULL, 
  `c` int(11) DEFAULT NULL, 
  `d` int(11) DEFAULT NULL, 
  PRIMARY KEY (`id`), 
  KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values(0,0,0),(5,5,5),(10,10,10),(15,15,15),(20,20,20),(25,25,25);
```

下面的语句序列，是怎么加锁的，加的锁又是什么时候释放的呢？

```sql
begin;
select * from t where d=5 for update;
commit;
```

这个语句会命中 d=5 的这一行，对应的主键 id=5，因此在 select 语句执行完成后，id=5 这一行会加一个写锁，而且由于两阶段锁协议，这个写锁会在执行 commit 语句的时候释放。

由于字段 d 上没有索引，因此这条查询语句会做全表扫描。那么，其他被扫描到的，但是不满足条件的 5 行记录上，会不会被加锁呢？我们知道，InnoDB 的默认事务隔离级别是可重复读，所以本文接下来没有特殊说明的部分，都是设定在可重复读隔离级别下。

## 幻读是什么

现在，我们就来分析一下，如果只在 id=5 这一行加锁，而其他行的不加锁的话，会怎么样。

|      | session A                                                    | session B | session C |
| ---- | ------------------------------------------------------------ | --------- | --------- |
| T1   | begin;<br />select * from t where d=5 for update; /*Q1*/<br /><br />commit; |           |           |
| T2   |                                                              |           |           |
| T3   |                                                              |           |           |
| T4   |                                                              |           |           |
| T5   |                                                              |           |           |
| T6   |                                                              |           |           |

