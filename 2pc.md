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


## 2PC的缺点

根据上面的算法介绍可以看出2PC有几个缺点：

1. 这是一个阻塞协议
2. 如果Coordinator永远崩溃，那么Cohorts永远无法完成它们的事务
3. Cohort在commit request phase之后会阻塞，直到进入Coordinator告之Cohort进入commit phase

## 陷阱

阅读完毕上面的算法介绍里你会发现，所以2PC所保证的ACID实际上和本地事务所提到的ACID不太一样：

1. 对于A，在commit request phase/commit phase，在某个时间点，Coordinator/Cohort崩溃了，那么可能会出现无法恢复事务的情况（继续执行 or 撤销），需要人工介入，详见这篇[slides][slides-dcp]
2. 对于C，在commit phase，在某个时间点，会出现A库和B库的数据违反一致性要求的情况
3. 对于I，在commit phase，在某个时间点，A事务能够读到B事务部分提交的结果

## XA

[XA][wiki-xa]是一个针对[分布式事务][wiki-dtp]的spec，它采用的是2PC协议。在XA中定义了两种参与方：Transaction Manager（TM）和Resource Manager（RM），其中TM=2PC中的Coordinator，RM=2PC中的Cohort。

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