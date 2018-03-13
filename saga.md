# 分布式事务Pattern - Saga

Saga是一种分布式事务模式，该模式和传统XA模式不同。区别点在于：

1. XA模式是直接连接多个数据库的模式，而Saga则是连接多个服务
2. XA模式要求数据库也支持事务，而Saga则不要求
3. XA模式是[ACID][wiki-acid]的，而Saga模式本身不保证ACID

所以，虽然说Saga是一种分布式事务模式，但它和关系型数据库的“事务”的概念并不相同。准确地说Saga是一种协调机制，协调各个服务完成一件业务，并且保证其结果最终是一致的：要么全成功，要么全失败。

## 机制

## ACID

下面仔细说说Saga是如何不能保证ACID的，以便理解Saga所说的“要么全成功，要么全失败”到底是什么意思：

### Atomicity 原子性

> Atomicity requires that each transaction be **"all or nothing"**: if one part of the transaction fails, then the entire transaction fails, and the database state is left unchanged. An atomic system must guarantee atomicity in each and every situation, including power failures, errors and crashes. To the outside world, a committed transaction appears (by its effects on the database) to be **indivisible** ("atomic"), and an aborted transaction does not happen.

以上是Wiki中关于Atomicity的解释。看关键词**"all or nothing"**，它的意思是数据库要么被修改了，要么保持原来的状态。所谓保持原来的状态不是我先insert再delete，而是压根就没有发生过任何操作。因为insert然后再delete实际上还是修改了数据库状态的，至少在数据库日志层面是这样。

再看关键词**"indivisible"**，它的意思是不可分割，一个事务就是一个最小的无法分割的独立单元。

我们回过头来看Saga，Saga在执行过程中是由多个独立的Sub-transaction组成，所以首先就违反了**"indivisible"**。其次Saga中的补偿机制是一种业务上的“回滚”，所以数据库本身肯定是发生了操作的，即不满足**"all or nothing"**。所以Saga不满足Atomicity。

### Consistency 一致性

> The consistency property ensures that any transaction will bring the database from one valid state to another. **Any data written to the database must be valid according to all defined rules, including constraints, cascades, triggers, and any combination thereof**. This does not guarantee correctness of the transaction in all ways the application programmer might have wanted (that is the responsibility of application-level code), but merely that any programming errors cannot result in the violation of any defined rules.

一致性要求当数据库从一个状态转移到另一个状态后，其依然能够满足各种一致性要求（业务约束、外键等等），简单来说就是不能出现中间状态。

而Saga在执行各个Sub-transaction的过程中，会出现中间状态，且中间状态会违反一致性要求，所以它不满足Consistency。

### Isolation 隔离性

> The isolation property ensures that the concurrent execution of transactions results in a system state that would be obtained if transactions were executed sequentially, i.e., **one after the other**. Providing isolation is the main goal of concurrency control. Depending on the concurrency control method (i.e., if it uses strict - as opposed to relaxed - serializability), **the effects of an incomplete transaction might not even be visible to another transaction**.

隔离性要求如果两个事务修改同一个数据，则必须按顺序执行，并且前一个事务的中间状态对另一个事务不可见。

Saga在执行过程中，会出现中间状态，而且这个中间状态对于其他服务、进程是可见的。所以Saga不满足Isolation。

### Durability 持久性

> The durability property ensures that once a transaction has been committed, it will remain so, even in the event of power loss, crashes, or errors. In a relational database, for instance, once a group of SQL statements execute, the results need to be stored permanently (even if the database crashes immediately thereafter). To defend against power loss, transactions (or their effects) must be recorded in a non-volatile memory.

持久性的关键在于一旦“完成提交”（committed），那么数据就不会丢失。

Saga是由sub-transaction组成，每个sub-transaction由对应的服务负责，那么这个服务到底是否能够做到“完成提交数据就不会丢失”Saga无法控制，所以它也就不保证Durability。

### Eventual Consistency

那么Saga能够保证什么？即使各个服务自身保证ACID，也不代表Saga也能保证ACID，或者说Saga压根就不能保证，你也不应该要求Saga保证ACID。Saga能够保证的就只有“最终一致性”，即经过若干时间后结果能够满足一致性。

PS. 最终一致性也有另一个名字：BASE - **B**asically **A**vailable, **S**oft state, **E**ventual consistency。

## 优点


## 缺点

## 注意

## 参考资料

* [Eventual Data Consistency Solution in ServiceComb - part 1][service-comb-saga-blog-1]
* [Eventual Data Consistency Solution in ServiceComb - part 2][service-comb-saga-blog-2]
* [Eventual Data Consistency Solution in ServiceComb - part 3][service-comb-saga-blog-3]
* [Paper - Sagas][paper-sagas]
* [Wiki - Eventual Consistency][wiki-eventual-consistency]
* [Wiki - CAP theorem][wiki-cap]
* [Pattern: Saga][site-pattern-saga]

[service-comb-saga-blog-1]: https://servicecomb.incubator.apache.org/docs/distributed_saga_1/
[service-comb-saga-blog-2]: https://servicecomb.incubator.apache.org/docs/distributed_saga_2/
[service-comb-saga-blog-3]: https://servicecomb.incubator.apache.org/docs/distributed_saga_3/
[paper-sagas]: https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf
[wiki-acid]: https://en.wikipedia.org/wiki/ACID
[wiki-eventual-consistency]: https://en.wikipedia.org/wiki/Eventual_consistency
[wiki-cap]: https://en.wikipedia.org/wiki/CAP_theorem
[site-pattern-saga]: http://microservices.io/patterns/data/saga.html