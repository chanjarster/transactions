# 分布式事务Pattern - Saga

Saga是一种分布式事务模式，该模式和传统XA模式不同。区别点在于：

1. XA模式是直接连接多个数据库的模式，而Saga则是连接多个服务
2. XA模式要求数据库也支持事务，而Saga则不要求
3. XA模式是[ACID][wiki-acid]的，而Saga模式本身不保证ACID

所以，虽然说Saga是一种分布式事务模式，但它和关系型数据库的“事务”的概念并不相同。准确地说Saga是一种协调机制，协调各个服务完成一件业务，并且保证其结果最终是一致的：要么全成功，要么全失败。

## 机制

## 不一样的ACID

Saga模式和[TCC][tcc.md]一样，它只能保证ACD，而且对于ACD的保证与[本地事务][local.md]所保证的并不一样。

### Atomicity

实际上Saga并不能保证和本地事务一样的Atomicity，它只能保证最终Atomicity，或者从业务层面看起来像Atomicity，因为：

1. Saga只要一开始，就修改了数据库，违反了all or nothing
2. Saga是分步骤提交的，不是一次提交，而是各个应用程序有自己的本地事务提交
3. Saga执行过程中崩溃了，那么就会出现部分成功/失败的结果，违反了indivisible

### Consistency

Saga保证的是[最终一致性][wiki-eventual-consistency]，但这个保证也不是那么强，而是依赖于应用程序代码没有BUG。

而且Saga是分步执行的，所以在执行过程中会出现违反一致性的结果。

### Isolation

Saga和[TCC][tcc.md]不一样，他没有Try阶段，因此它无法提供任何形式/强度的Isolation。

### Durability

Saga的Durability也是依赖于各个应用程序，要求各个应用程序正确的实现了业务逻辑。

## 要点

* Saga模式在于应用程序而非数据库
* Saga模式依赖于各应用程序代码的正确实现，每个应用程序自我保证ACD（可以采用本地事务）

对于do、compensate的要求：

* 要求幂等
* 要求两者可交换，即可颠倒次序执行，如compensate在前do在后
* compensate动作要一定可以成功，否则会需要人工介入

优点：

* 需改造应用程序逻辑
* 相比TCC，最好情况下，只需要2N次通信（N=服务数量）

缺点：

* 不提供任何形式/强度的隔离性


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

[local.md]: local.md
[tcc.md]: tcc.md
[service-comb-saga-blog-1]: https://servicecomb.incubator.apache.org/docs/distributed_saga_1/
[service-comb-saga-blog-2]: https://servicecomb.incubator.apache.org/docs/distributed_saga_2/
[service-comb-saga-blog-3]: https://servicecomb.incubator.apache.org/docs/distributed_saga_3/
[paper-sagas]: https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf
[wiki-acid]: https://en.wikipedia.org/wiki/ACID
[wiki-eventual-consistency]: https://en.wikipedia.org/wiki/Eventual_consistency
[wiki-cap]: https://en.wikipedia.org/wiki/CAP_theorem
[site-pattern-saga]: http://microservices.io/patterns/data/saga.html