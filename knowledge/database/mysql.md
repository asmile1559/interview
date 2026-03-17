# MySQL

## Q: MySQL的事务隔离级别

A：MySQL的事务隔离级别分为四种：

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | InnoDB默认实现                                                        | 应用场景              |
| -------- | ---- | ---------- | ---- | --------------------------------------------------------------------- | --------------------- |
| 读未提交 | ✅    | ✅          | ✅    | 基本不加锁，性能高，一致性差                                          | 几乎不用              |
| 读已提交 | ❎    | ✅          | ✅    | 每次读都取最新提交版本（快照读+当前读）                               | Oracle，SQLServer默认 |
| 可重复读 | ❎    | ❎          | ✅    | MySQL InnoDB默认实现，使用MVCC+Next-Key Lock（间隙锁+记录锁）防止幻读 | 大多数业务推荐        |
| 序列化   | ❎    | ❎          | ❎    | 所有读写都加锁，并发低                                                | 数据一致性要求高      |

- 脏读：一个事务读到了另一个事务未提交的修改数据。
- 不可重复读：同一个事务内，连续两次读取相同数据，得到的结果不同。
- 幻读：事务基于前一次读到的结果进行后续写操作，却发现了违反约束问题。或出现了不同的数据数量。

设置隔离级别

```sql
SELECT @@transcaction_isolation; -- 查看隔离级别

SELECT @@global.transcation_isolation; -- 查看全局隔离级别

SET SESSION TRANSLATION ISOLATION LEVEL READ COMMITTED;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET GLOBAL TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

## Q: 如何解决幻读

快照读：不加锁的非阻塞读，如普通的select操作。

当前读：MySQL的MVCC决定了同一行数据可能存在多个版本的问题，当前读表示读取的纪录是最新版本的，且在读取时，如果有其他并发事务要修改统一数据行，当前事务会通过加锁让其他事务阻塞等待。比如：SELECT LOCK IN SHARE MODE（共享锁）,SELECT ... FOR UPDATE, UPDATE, INSERT, DELETE（排它锁）.

1. 使用序列化隔离级别
2. 使用可重复读隔离级别：
   1. 基于MVCC：事务第一次SELECT时生成一个Read View,后续所有普通Select都基于这个快照读取一致的版本。
   2. 使用Next-Key Lock：使用行锁和间隙锁。间隙锁锁定记录之间的空隙，防止其他的事务在间隙中插入新纪录。

## Q: 主备复制是什么？

A：