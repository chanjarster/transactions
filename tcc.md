# TCC

在前一篇文章中讲到[2PC][2pc.md]是针对跨数据库的事务的一种解决方案，不过随着SOA、微服务的兴起与发展，2PC已经不能满足需求了：

1. 大量兴起兴起的NoSQL不支持XA
2. 各个应用之间的数据库是分离的，不允许A应用插手到B应用的数据库

于是就有人提出了[TCC模式][presentation-transactions-http-rest]（Try、Confirm、Cancel），在国内TCC因阿里巴巴的推广而广为人知。

## 算法介绍

## 不一样的ACID

TCC模式在于应用程序而非数据库，因此其所保证的ACID实际上和[本地事务][local.md]所保证的并不一样。

### Atomicity

实际上TCC并不能保证和本地事务一样的Atomicity，它只能保证最终Atomicity，或者从业务层面看起来像Atomicity，因为：

1. TCC只要一开始，就修改了数据库，违反了all or nothing
2. TCC是分步骤提交的，不是一次提交，而是各个应用程序有自己的本地事务提交
3. TCC执行过程中崩溃了，那么就会出现部分成功/失败的结果，违反了indivisible

### Consistency

TCC保证的是“最终一致性”，但这个保证也不是那么强，而是依赖于应用程序代码没有BUG。

而且TCC是分步执行的，所以在执行过程中会出现违反一致性的结果。

### Isolation

TCC的Try阶段做的一般是预留操作，比如新建一个PENDING状态的订单，不过这条订单记录也是进入数据库并提交的（本地事务）。

应用程序可以在代码层面来对PENDING状态的订单做特殊处理（比如在报表逻辑里排除PENDING），但还是那句话，这依赖于程序代码没有BUG。

### Durability

TCC的Durability也是依赖于各个应用程序，要求各个应用程序正确的实现了业务逻辑。

## 要点

* TCC模式在于应用程序而非数据库
* TCC模式依赖于各应用程序代码的正确实现，每个应用程序自我保证ACID（可以采用本地事务）


对于Try、Confirm、Cancel的要求：

* 要求都是幂等的
* Confirm/Cancel一定要能够成功，否则会需要人工介入

优点：

* 因为有Try的动作，能够做到准隔离性。比如处于PENDING状态的订单可以在程序代码做控制，以对其他事务不可见

缺点：

* 需改造应用程序逻辑
* 需要2N次通信（N=服务数量，每个服务都会被Try、Confirm/Cancel）


## 参考资料

* [Paper - Rest TCC][pdf-tcc]
* [Presentation - Transactions for the REST of Us][presentation-transactions-http-rest]
* [Article - Transactions for the REST of Us][article-transactions-http-rest]
* [Article - Atomic Distributed Transactions: a RESTful Design][article-tcc-wsrest]
* [大规模SOA系统中的分布事务处事_程立][slides-tcc-alibaba] 
* [深入解读微服务架构下分布式事务解决方案][article-microservice-transactions-in-depth]
* [分布式事务之说说TCC事务][article-talk-about-tcc]

[2pc.md]: 2pc.md
[local.md]: local.md
[presentation-transactions-http-rest]: https://www.infoq.com/presentations/Transactions-HTTP-REST
[article-transactions-http-rest]: https://dzone.com/articles/transactions-for-the-rest-of-us
[pdf-tcc]: http://design.inf.usi.ch/sites/default/files/biblio/rest-tcc.pdf
[slides-tcc-alibaba]: https://wenku.baidu.com/view/be946bec0975f46527d3e104.html
[article-microservice-transactions-in-depth]: https://www.jianshu.com/p/f04cc1a696b4
[article-talk-about-tcc]: https://www.toutiao.com/a6340518979443032322/
[article-tcc-wsrest]: http://www.pautasso.info/biblio-pdf/tcc-wsrest2014.pdf
