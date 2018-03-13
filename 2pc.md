# 两阶段提交分布式事务

在[上一篇文章][local.md]中我们介绍了本地事务，随着软件复杂度的上升，我们会需要一种可以在多个数据库之间完成事务的方法，而这个方法也必须能够保证ACID。于是就出现了2PC - Two phase commit protocol。

## 参考资料

* [wiki - 2PC][wiki-2pc]
* [XA transactions using Spring][article-xa-transactions-using-spring]
* [Distributed transactions in Spring, with and without XA][article-distributed-transactions-in-spring--with-and-without-xa]

[local.md]: local.md
[wiki-2pc]: https://en.wikipedia.org/wiki/Two-phase_commit_protocol
[article-xa-transactions-using-spring]: https://www.javaworld.com/article/2077714/java-web-development/xa-transactions-using-spring.html
[article-distributed-transactions-in-spring--with-and-without-xa]: https://www.javaworld.com/article/2077963/open-source-tools/distributed-transactions-in-spring--with-and-without-xa.html
