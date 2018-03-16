# TCC

在前一篇文章中讲到了[BASE模式][base.md]，这种模式可以应用在单库or跨库事务的场景下。事实上BASE模式不仅仅局限于数据库层面，还可以应用于分布式系统，这类分布式系统最典型的例子就是电商平台，它们有以下几个特征：

1. SOA化/微服务化：单体应用拆分成多个小应用，。
2. 数据库的各种拆分技术的运用：分表、分库、分区。
3. 大量NoSQL数据库的兴起。
4. 应用之间的通信手段并非直接读数据库：RESTful、RPC、消息中间件等。
5. 大量跨应用事务的出现。

在这种场景下，[2PC][2pc.md]（及XA）已经无法满足需求，因为它：

1. 性能低下，2PC协议是阻塞式的。当协调的数据库越来越多时，性能无法接受。
2. 无法水平扩展以提升性能，只能靠垂直扩展（提升硬件）——更快的CPU、更快更大的硬盘、更大更快的内存——但是这样很贵，并且很容易遇到极限。
3. 染指其他数据库。
4. 依赖于数据库是否支持2PC（XA）。

而BASE只解决最后提交的问题，不能解决诸如在[上一篇文章][base.md]中最后提到的如何保证刷卡消费不透支的问题.于是就有人提出了[TCC模式][presentation-transactions-http-rest]（Try、Confirm、Cancel），这一模式在国内因阿里巴巴的推广而广为人知。

## 协议介绍

### 三种动作

TCC是Try、Confirm、Cancel的简称，它们分别的职责是：

* Try：负责做业务检查（比如看看余额是否足够）、预留资源（比如新建一条状态=PENDING的订单）
* Confirm：负责落地Try所预留的资源（比如扣费、把订单状态变成COMPLETED）
* Cancel：负责撤销Try所预留的资源（比如把订单状态变成CANCELED）

可以看到TCC是对于BASE的一种扩充，增加了业务检查和撤销事务的功能。

### 状态变化

TCC的每一个动作都会造成状态的变化，详见这张图：

![TCC的状态](images/tcc-state-machine.png)

### 流程

下面称呼发起事务的应用为“发起方”，接受请求的应用为“参与方”

1. 发起方发起Try到参与方
2. 参与方执行Try，状态变为预留
3. 发起方发起Confirm到参与方

### 异常处理


下面介绍TCC的流程：

1. 

TODO



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

[base.md]: base.md
[2pc.md]: 2pc.md
[local.md]: local.md
[presentation-transactions-http-rest]: https://www.infoq.com/presentations/Transactions-HTTP-REST
[article-transactions-http-rest]: https://dzone.com/articles/transactions-for-the-rest-of-us
[pdf-tcc]: http://design.inf.usi.ch/sites/default/files/biblio/rest-tcc.pdf
[slides-tcc-alibaba]: https://wenku.baidu.com/view/be946bec0975f46527d3e104.html
[article-microservice-transactions-in-depth]: https://www.jianshu.com/p/f04cc1a696b4
[article-talk-about-tcc]: https://www.toutiao.com/a6340518979443032322/
[article-tcc-wsrest]: http://www.pautasso.info/biblio-pdf/tcc-wsrest2014.pdf
