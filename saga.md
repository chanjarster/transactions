# Saga

1987年普林斯顿大学的Hector Garcia-Molina和Kenneth Salem发表了一篇Paper [Sagas][paper-sagas]，讲述的是如何处理long lived transaction（长活事务）。听起来是不是觉得和分布式事务很像？没错，下面来看看这个来自1987年的解决方案是如何启发当今的分布式事务问题的。

## 协议介绍

Saga的组成：

* 每个Saga由一系列sub-transaction T<sub>i</sub> 组成
* 每个T<sub>i</sub> 都有对应的补偿动作C<sub>i</sub>，补偿动作用于撤销T<sub>i</sub>造成的结果

可以看到，和[TCC][tcc.md]相比，Saga没有“预留”动作，它的T<sub>i</sub>就是直接提交到库。

Saga的执行顺序有两种：

* T<sub>1</sub>, T<sub>2</sub>, T<sub>3</sub>, ..., T<sub>n</sub>
* T<sub>1</sub>, T<sub>2</sub>, ..., T<sub>j</sub>, C<sub>j</sub>,..., C<sub>2</sub>, C<sub>1</sub>，其中0 < j < n

Saga定义了两种恢复策略：

* backward recovery，向后恢复，即上面提到的第二种执行顺序，其中j是发生错误的sub-transaction，这种做法的效果是撤销掉之前所有成功的sub-transation，使得整个Saga的执行结果撤销。
* forward recovery，向前恢复，适用于必须要成功的场景，执行顺序是类似于这样的：T<sub>1</sub>, T<sub>2</sub>, ..., T<sub>j</sub>(失败), T<sub>j</sub>(重试),..., T<sub>n</sub>，其中j是发生错误的sub-transaction。该情况下不需要C<sub>i</sub>。

## 对于ACID的保证

Saga对于ACID的保证和TCC一样：

* A，正常情况下保证。
* C，在某个时间点，会出现A库和B库的数据违反一致性要求的情况，但是最终是一致的。
* I，在某个时间点，A事务能够读到B事务部分提交的结果。
* D，和本地事务一样，只要commit则数据被持久。

## 和TCC对比

Saga相比TCC的缺点是缺少预留动作，导致补偿动作的实现比较麻烦：T<sub>i</sub>就是commit，比如一个业务是发送邮件，在TCC模式下，先保存草稿（Try）再发送（Confirm），撤销的话直接删除草稿（Cancel）就行了。而Saga则就直接发送邮件了（T<sub>i</sub>），如果要撤销则得再发送一份邮件说明撤销（C<sub>i</sub>），实现起来有一些麻烦。

如果把上面的发邮件的例子换成：A服务在完成T<sub>i</sub>后立即发送Event到ESB（企业服务总线，可以认为是一个消息中间件），下游服务监听到这个Event做自己的一些工作然后再发送Event到ESB，如果A服务执行补偿动作C<sub>i</sub>，那么整个补偿动作的层级就很深。

不过没有预留动作也可以认为是优点：

* 有些业务很简单，套用TCC需要修改原来的业务逻辑，而Saga只需要添加一个补偿动作就行了。
* TCC最少通信次数为2n，而Saga为n（n=sub-transaction的数量）。
* 有些第三方服务没有Try接口，TCC模式实现起来就比较tricky了，而Saga则很简单。
* 没有预留动作就意味着不必担心资源释放的问题，异常处理起来也更简单（请对比Saga的恢复策略和TCC的异常处理）。

## 实现Saga的注意事项

对于服务来说，实现Saga有以下这些要求：

1. T<sub>i</sub>和C<sub>i</sub>是幂等的。
1. C<sub>i</sub>必须是能够成功的，如果无法成功则需要人工介入。
1. T<sub>i</sub> - C<sub>i</sub>和C<sub>i</sub> - T<sub>i</sub>的执行结果必须是一样的：sub-transaction被撤销了。

第一点要求T<sub>i</sub>和C<sub>i</sub>是幂等的，举个例子，假设在执行T<sub>i</sub>的时候超时了，此时我们是不知道执行结果的，如果采用forward recovery策略就会再次发送T<sub>i</sub>，那么就有可能出现T<sub>i</sub>被执行了两次，所以要求T<sub>i</sub>幂等。如果采用backward recovery策略就会发送C<sub>i</sub>，而如果C<sub>i</sub>也超时了，就会尝试再次发送C<sub>i</sub>，那么就有可能出现C<sub>i</sub>被执行两次，所以要求C<sub>i</sub>幂等。

第二点要求C<sub>i</sub>必须能够成功，这个很好理解，因为，如果C<sub>i</sub>不能执行成功就意味着整个Saga无法完全撤销，这个是不允许的。但总会出现一些特殊情况比如C<sub>i</sub>的代码有bug、服务长时间崩溃等，这个时候就需要人工介入了。

第三点乍看起来比较奇怪，举例说明，还是考虑T<sub>i</sub>执行超时的场景，我们采用了backward recovery，发送一个C<sub>i</sub>，那么就会有三种情况：

1. T<sub>i</sub>的请求丢失了，服务之前没有、之后也不会执行T<sub>i</sub>
2. T<sub>i</sub>在C<sub>i</sub>之前执行
3. C<sub>i</sub>在T<sub>i</sub>之前执行

对于第1种情况，容易处理。对于第2、3种情况，则要求T<sub>i</sub>和C<sub>i</sub>是可交换的（commutative)，并且其最终结果都是sub-transaction被撤销。

## 参考资料

* [Paper - Sagas][paper-sagas]
* [Eventual Data Consistency Solution in ServiceComb - part 1][service-comb-saga-blog-1]
* [Eventual Data Consistency Solution in ServiceComb - part 2][service-comb-saga-blog-2]
* [Eventual Data Consistency Solution in ServiceComb - part 3][service-comb-saga-blog-3]
* [Distributed Sagas: A Protocol for Coordinating Microservices][presentation-saga]

[paper-sagas]: ftp://ftp.cs.princeton.edu/reports/1987/070.pdf
[tcc.md]: tcc.md
[service-comb-saga-blog-1]: https://servicecomb.incubator.apache.org/docs/distributed_saga_1/
[service-comb-saga-blog-2]: https://servicecomb.incubator.apache.org/docs/distributed_saga_2/
[service-comb-saga-blog-3]: https://servicecomb.incubator.apache.org/docs/distributed_saga_3/
[presentation-saga]: https://www.youtube.com/watch?v=1H6tounpnG8