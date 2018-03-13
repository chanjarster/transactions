# 本地事务

什么是本地事务（Local Transaction）？本地事务也称为*数据库事务*或*传统事务*（相对于分布式事务而言）。它的执行模式就是常见的：

1. transaction begin
1. insert/delete/update
1. insert/delete/update
1. ...
1. transaction commit/rollback

本地事务有这么几个特征:

1. 一次事务只连接一个支持事务的数据库（一般来说都是关系型数据库）
1. 事务的执行结果保证[ACID][wiki-acid]
1. 会用到数据库锁

## ACID

在讨论事务时，我们绕不过一组概念：ACID，我们来看看Wiki是怎么解释ACID的：

### Atomicity 原子性

> Atomicity requires that each transaction be **"all or nothing"**: if one part of the transaction fails, then the entire transaction fails, and the database state is left unchanged. An atomic system must guarantee atomicity in each and every situation, including power failures, errors and crashes. To the outside world, a committed transaction appears (by its effects on the database) to be **indivisible** ("atomic"), and an aborted transaction does not happen.

关键词在于：

1. **all or nothing**，它的意思是数据库要么被修改了，要么保持原来的状态。所谓保持原来的状态不是我先insert再delete，而是压根就没有发生过任何操作。因为insert然后再delete实际上还是修改了数据库状态的，至少在数据库日志层面是这样。
2. **indivisible**，不可分割，一个事务就是一个最小的无法分割的独立单元，不允许部分成功部分失败。

### Consistency 一致性

> The consistency property ensures that any transaction will bring the database from one valid state to another. **Any data written to the database must be valid according to all defined rules, including constraints, cascades, triggers, and any combination thereof**. This does not guarantee correctness of the transaction in all ways the application programmer might have wanted (that is the responsibility of application-level code), but merely that any programming errors cannot result in the violation of any defined rules.

一致性要求任何写到数据库的数据都必须满足于预先定义的规则（比如余额不能小于0、外键约束等），简单来说就是在任何时间点都不能出现违反一致性要求的状态。

### Isolation 隔离性

> The isolation property ensures that the concurrent execution of transactions results in a system state that would be obtained if transactions were executed sequentially, i.e., **one after the other**. Providing isolation is the main goal of concurrency control. Depending on the concurrency control method (i.e., if it uses strict - as opposed to relaxed - serializability), **the effects of an incomplete transaction might not even be visible to another transaction**.

隔离性要求如果两个事务修改同一个数据，则必须按顺序执行，并且前一个事务如果未完成，那么未完成的中间状态对另一个事务不可见。

### Durability 持久性

> The durability property ensures that **once a transaction has been committed, it will remain so, even in the event of power loss, crashes, or errors**. In a relational database, for instance, once a group of SQL statements execute, the results need to be stored permanently (even if the database crashes immediately thereafter). To defend against power loss, transactions (or their effects) must be recorded in a non-volatile memory.

持久性的关键在于一旦“完成提交”（committed），那么数据就不会丢失。

## 数据库锁

在提到隔离性的时候我们提到，在修改同一份数据的情况下，两个事务必须挨个执行以免出现冲突情况。而数据库有四种隔离级别（注意：不是所有数据库支持所有隔离级别）

| Isolation Level  | Dirty Reads    | Non-Repeatable Reads | Phantom Reads |
|------------------|----------------|----------------------|---------------|
| Read uncommitted | 允许   | 允许   | 允许   |
| Read committed   | 不允许 | 允许   | 允许   |
| Repeatable reads | 不允许 | 不允许 | 允许   |
| Serializable     | 不允许 | 不允许 | 不允许 |

PS. 大多数数据库的默认隔离级别是Read committed。

来复习一下Dirty reads、Non-repeatable reads、Phantom reads的概念：

* Dirty reads：A事务可以读到B事务还未提交的数据
* Non-repeatable read：A事务读取一行数据，B事务后续修改了这行数据，A事务再次读取这行数据，结果得到的数据不同。
* Phantom reads：A事务通过`SELECT ... WHERE`得到一些行，B事务插入新行或者更新已有的行使得这些行满足A事务的`WHERE`条件，A事务再次`SELECT ... WHERE`结果比上一次多了一些行。

大多数数据库在实现以上事务隔离级别（Read uncommitted除外）时采用的机制是锁。这也就是为什么经常说当应用程序里大量使用事务或者高并发情况下会出现性能低下、死锁的问题。

## 参考资料

* [wiki - ACID][wiki-acid]
* [wiki - Database Transaction][wiki-database-transaction]
* [wiki - Isolation (database systems)][wiki-isolation-db]
* [JDK中对于事务的介绍][doc-java-transactions]

[wiki-acid]: https://en.wikipedia.org/wiki/ACID
[wiki-database-transaction]: https://en.wikipedia.org/wiki/Database_transaction
[doc-java-transactions]: https://docs.oracle.com/javase/tutorial/jdbc/basics/transactions.html
[wiki-isolation-db]: https://en.wikipedia.org/wiki/Isolation_(database_systems)