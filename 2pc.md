# 2PC

在[上一篇文章][local.md]中我们介绍了本地事务，随着软件复杂度的上升，我们会需要一种可以在多个数据库之间完成事务（[分布式事务][wiki-dtp]）的方法，而这个方法也必须能够保证ACID。于是就出现了[2PC - Two phase commit protocol][wiki-2pc]。事实上2PC不仅仅适用于多数据库事务场景下使用，也适用于所有支持2PC的参与方（Participants)。

## 算法介绍

2PC的参与方有：

1. 一个作为Coordinator的节点
2. 多个作为Cohort的网络节点

2PC假设：

1. 所有节点都有一个稳定存储用以保存WAL（[write-ahead log][wiki-write-ahead log]）
1. 没有一个节点会永远崩溃（即会最终恢复）
1. write-ahead log中存储的数据永远不会丢失，不会因崩溃而损坏
1. 任意两个节点都能够互相通信

第四个假设太过严格，实际上有不是所有的2PC实现都满足。第一、二个假设则大多数2PC实现都能满足。

PS. 如果某个节点完全损坏（比如服务器物理损毁），那么数据就直接丢失了。

2PC的执行步骤：

1. Commit request phase /Voting phase。这个阶段做：
  1. Coordinator发送一个查询是否同意commit的请求到所有Cohort，并且等待所有Cohort给出应答。
  2. Cohort收到请求，开始执行事务，执行到就差commit为止（不commit）。
  3. 每个Cohort根据操作结果返回Yes或No
1. Commit phase / Completion phase。这个阶段分两种情况：
  1. 成功。所有Cohort应答Yes
    1. Coordinator发送commit指令到所有Cohort
    2. 每个Cohort执行commit，并发送ack到Coordinator
    3. 当Coordinator收到每个Cohort的ack之后则事务完成
  2. 失败。任意Cohort应答No，或者在commit request阶段超时
    1. Coordinator发送rollback指令到所有Cohort
    2. 每个Cohort执行rollback，并发送ack到Coordinator
    3. 当Coordinator收到每个Cohort的ack之后则事务撤销

消息流（摘自wiki）：

```
Coordinator                                         Cohort
                              QUERY TO COMMIT
                -------------------------------->
                              VOTE YES/NO           prepare*/abort*
                <-------------------------------
commit*/abort*                COMMIT/ROLLBACK
                -------------------------------->
                              ACKNOWLEDGMENT        commit*/abort*
                <--------------------------------  
end
```

2PC的通信次数是：

1. 如果实现没有要求任意两个Cohort可以通信，那么是2n（n=Cohort数量）
2. 如果实现要求任意两个Cohort可以通信，那么是n^2

## 异常处理

我们把上面的流程简化以便说明异常处理：

1. Coordinator发送query to commit
2. Cohort执行prepare
3. Cohort返回ack
4. Coordinator发送commit/rollback
5. Cohort执行commit/rollback
6. Cohort返回ack

从Coordinate角度来看出现异常要怎么处理：

1. step 1发生异常，Coordinator需要执行rollback
2. step 2、3发生异常，意味着Coordinator没有收到Cohort的响应，这个时候因认定为失败，执行rollback
3. step 4发生异常，Coordinator重试commit/rollback
4. step 5、6发生异常，意味着Coordinator没有收到Cohort的响应，这个时候因认定为失败，重试commit/rollback

从Cohort角度来看看看出现异常怎么处理：

1. step 1，意味着Cohort没有收到请求，什么都不需要做
2. step 2，意味着Cohort没有执行成功，什么都不需要做
3. step 3，意味着Coordinator没有收到结果，什么都不需要做，等待Coordinator重试即可。Cohort要保证prepare是幂等的。
4. step 4，等待Coordinator重试即可，这里有点tricky，如果Coordinator迟迟不retry，那么Cohort要自行rollback，否则就会造成资源死锁。
5. step 5，等待Coordinator重试即可
6. step 6，意味着Coordinator没有收到结果，什么都不需要做，等待Coordinator重试即可，Cohort要保证commit/rollback是幂等的。

观察一下你就会发现依然存在漏洞——会出现违反一致性的情况：

1. 若Coordinator/Cohort因崩溃遗失了信息，有的Cohort已commit，有的Cohort则恢复到commit之前的状态。
2. 若Coordinator在step 4发送commit，而Cohort在rollback（因timeout导致的rollback）。

出现上面的情况就需要人工介入了。

更多2PC的异常处理推理详见这篇[slides][slides-dcp]。

## 缺点

根据上面的算法介绍可以看出2PC是一个阻塞协议：

* 如果两个事务针对同一个数据，那么后面的要等待前面完成，这是由于Cohort采用的是本地事务所决定的
* Cohort在commit request phase之后会阻塞，直到进入Coordinator告之Cohort进入commit phase

## 对于ACID的保证

2PC所保证的ACID和本地事务所提到的ACID不太一样——事实上对于所有分布式事务来说都不太一样：

* A，正常情况下保证
* C，在某个时间点，会出现A库和B库的数据违反一致性要求的情况，但是最终是一致的
* I，在某个时间点，A事务能够读到B事务部分提交的结果
* D，和本地事务一样，只要commit则数据被持久

## XA

[XA][wiki-xa]是一个针对[分布式事务][wiki-dtp]的spec，它实现了2PC协议。在XA中定义了两种参与方：Transaction Manager（TM）和Resource Manager（RM），其中TM=2PC中的Coordinator，RM=2PC中的Cohort。

Java规范中的JTA（Java Transaction API）定义了XA的Java接口，JTA的实现有Bitronix、Atomikos等等。

## 参考资料

* [wiki - 2PC][wiki-2pc]
* [wiki - XA][wiki-xa]
* [wiki - Distributed Transaction][wiki-dtp]
* [wiki - Write-ahead log][wiki-write-ahead log]
* [For 2 phase commit, why is the worst-case communication complexity of a failed transaction O(n^2)?][quora-comm-complexity]
* [slides - Distributed Commit Protocols][slides-dcp]


[local.md]: local.md
[wiki-2pc]: https://en.wikipedia.org/wiki/Two-phase_commit_protocol
[wiki-xa]: https://en.wikipedia.org/wiki/X/Open_XA
[wiki-dtp]: https://en.wikipedia.org/wiki/Distributed_transaction
[wiki-write-ahead log]: https://en.wikipedia.org/wiki/Write_ahead_logging
[quora-comm-complexity]: https://www.quora.com/For-2-phase-commit-why-is-the-worst-case-communication-complexity-of-a-failed-transaction-O-n-2
[slides-dcp]: http://www.inf.fu-berlin.de/lehre/SS10/DBS-TA/folien/07-10-TA-2PC-2-1.pdf