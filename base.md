# BASE模式

## ACID的局限

在[本地事务][local.md]这篇文章里我们讲到了数据库事务必须保证ACID，在[2PC][2pc.md]这篇文章里，我们探讨了跨数据库事务是如何保证ACID的。

当数据量越来越大的时候，我们会对将大数据库拆分成若干小库，随着数据库数量越来越多，2PC（及XA）就显得有些捉襟见肘了：

1. 性能低下，2PC协议是阻塞式的。当协调的数据库越来越多时，性能无法接受。
2. 无法水平扩展以提升性能，只能靠垂直扩展（提升硬件）——更快的CPU、更快更大的硬盘、更大更快的内存——但是这样很贵，并且很容易遇到极限。

## CAP理论

[CAP理论][wiki-cap]是Eric Brewer教授针对分布式数据库所提出的一套理论，他认为在实现分布式数据库，需要考虑3个需求：

* Consistency：当写发生时，每个节点的数据必须都被更新到。当查询发生时，如果还未全部更新到则返回error或timeout；如果全部更新到了，则直接返回结果。
* Availability：当查询发生时，不用考虑上一次写是否更新到了每个节点，直接提供当下的数据作为结果。
* Partition-tolerance：除非所有节点都挂，它都应该能继续提供服务。

CAP理论指出，当每个节点都工作正常的时候，C、A、P是可以都满足的，当出现节点故障时，我们只能3选2。

如果我们不想因为数据库的某个节点出现故障就让数据库停止服务，那么我们必定选择P，那么我们就只能在C和A之间做选择。如果选择C：后续的读都将失败。如果选择A：会读到不一致的结果。

## BASE模式

**B**asically **A**vailable, **S**oft state, **E**ventually consistent，简称BASE。

BASE和ACID相反，ACID是悲观的，它要求所有操作都必须保证一致性，而BASE是乐观的，它接受数据库的一致性在不断变化当中。同时，BASE对于CAP中的C做出了一定的妥协——接受临时的不一致，采用最终一致性。最终一致性，听上去怪怪的，一些开发人员觉得这是个坏东西。不过我们真的要**时时刻刻**保证一致性吗？BASE认为我们可以做一些妥协，因此如果我们按照BASE设计系统的话就能够保证：

1. ACID - A，不保证，一旦开始“写”则不可能回滚。
1. ACID - C，保证最终一致性。
1. ACID - I，不保证，是因为一个大事务是由多个小事务组成，每个小事务都会独立提交。
1. ACID - D，保证，因为数据库保证D。
1. CAP - C，保证最终一致性。
1. CAP - A，保证基本可用。
1. CAP - P，保证。

举个例子，现在我们有3条insert要执行（至于是否是3个不同的表、数据库不重要），那么只要保证下面几点就能够满足BASE：

1. 最终都能够执行成功
2. 任何一条语句执行失败都会重试
3. 任意一条语句重复执行结果都一样——幂等

正确地使用BASE模式也不是那么容易，比如刷卡消费业务，我们要保证“检查余额”和“扣款、记录消费日志”这两组动作不会产生交叉，否则就会因为高并发场景而发生透支，在这个例子里我们可以对“扣款、记录消费日志”做最终一致性，但是如何保证下达“扣款、记录消费日志”这两个指令肯定不会产生透支的情况则不是BASE解决的问题了。

所以总结一下BASE的特点就是：

1. 解决的是提交的问题
2. 2PC将提交动作放在数据库，而BASE将提交动作放在应用程序

关于BASE可以详见这篇文章[BASE: An Acid Alternative][article-base]。

## 参考资料

* [Wiki - CAP Theorem][wiki-cap]
* [Wiki - Eventual Consistency][wiki-eventual-consistency]
* [Article - BASE: An Acid Alternative][article-base]
* [Article - Better Explaining CAP Theorem][article-better-explaining-cap]

[local.md]: local.md
[2pc.md]: 2pc.md
[article-base]: https://dl.acm.org/citation.cfm?doid=1394127.1394128
[article-better-explaining-cap]: https://dzone.com/articles/better-explaining-cap-theorem
[wiki-eventual-consistency]: https://en.wikipedia.org/wiki/Eventual_consistency
[wiki-cap]: https://en.wikipedia.org/wiki/CAP_theorem

