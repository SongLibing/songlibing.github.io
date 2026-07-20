---
title: MySQL 大事务的 Binlog 传输优化
date: 2026-07-16 10:00:00 +0800
categories: [数据库, MySQL]
tags: [MySQL, 复制, Binlog, 大事务, 半同步]
toc: true
---

大事务是MySQL中一个痛点问题， 不仅会带来复制延迟，也会带来大量的稳定性问题。上一篇文章《[MySQL大事务提交优化](https://mp.weixin.qq.com/s?__biz=MzIyMTQ1NDE0MQ==&mid=2247484449&idx=1&sn=d308f27938b563cb197964fbc2c9b6bd&scene=21#wechat_redirect)》介绍了大事务在提交时带来的问题，以及AliSQL中做的技术优化。这篇文章会介绍大事务在`半同步`复制时的问题，以及AliSQL中如何解决这个问题。

在《[MySQL大事务提交优化](https://mp.weixin.qq.com/s?__biz=MzIyMTQ1NDE0MQ==&mid=2247484449&idx=1&sn=d308f27938b563cb197964fbc2c9b6bd&scene=21#wechat_redirect)》中，我们提到大事务提交时写Binlog会导致系统出现如下奇怪的慢SQL。

![](/assets/img/bigtxn-binlog-1.webp)

- 平时执行很快的INSERT语句，竟然执行了`1.3s`，并且慢SQL记录里也没有看到长时间的锁等待。
- 多语句事务的所有语句都已经执行完了，但是COMMIT语句竟然执行了`1.3s`。

除了大事务写Binlog会导致这种表现外，半同步复制时大事务的Binlog传输也会导致这种表现。以下是一个模拟测试的结果，我们用sysbench oltp\_write\_only来模拟正常的写业务，然后在后台提交了一个产生了2GB Binlog Events的事务(`已经做了大事务提交优化`)。可以看到在大事务提交时，写操作跌0，直到半同步复制超时后才恢复。

![](/assets/img/bigtxn-binlog-2.webp)

## 根因分析

![](/assets/img/bigtxn-binlog-3.webp)

上图是半同步复制的事务提交过程图：

- 当事务提交时，需要执行两阶段提交，首先进行Prepare。
- 然后将Binlog Events写入到Binlog文件。
- 写完Binlog后，开始等待自己的Binlog Events被发送到备库(`after_sync`模式)。
- 此后Binlog dump线程将该事务的Binlog Events发送到备节点。
- 备节点的IO线程负责接收这些Binlog Events，并且将收到的Binlog Events写入到Relay Log文件中。
- 当接收到完整的事务后，IO线程会给主库一个应答，告诉主库收到了这个事务的所有Binlog Events。这个应答是用Binlog文件名和位点来表示的。上图中Trx\_n的binlog结束位点是`530`，因此备节点的IO线程会发送`master-bin.000001:530`给主库，这表示`master-bin.000001:530`之前的所有事务都已经收到了。
- 主库的Semisync Ack Receiver线程收到应答后，会根据位点唤醒相应的事务。
- 事务被唤醒后，就继续进行提交操作，提交完成后返回用户OK。

主节点和备节点之间只有一个Dump线程，Dump线程按照Binlog Events写入Binlog的顺序进行传输。备节点的IO线程也是按照这个顺序将收到的Binlog Events写入Relay log中，然后应答主库。因此只有前面的事务发送完成，才能发送后面的事务。如果当前的事务的Binlog Events非常多，那么这个事务的Binlog Events发送的时间就会非常的长。后面的事务尽管很小，也必须要等待。这个等待不仅包含自己事务的传输时间，也包含前面大事务的传输时间。因此就会出现上面慢日志的情况，一个小的事务突然变的很慢。

为了应对这种情况，MySQL提供了`rpl_semi_sync_master_timeout`参数。该参数定义了事务等待ACK的时长，当等待的时长超过`rpl_semi_sync_master_timeout`后，自动退化成异步复制。我们可以将该参数设置一个较小的值，来避免大事务导致整个实例不可写的严重情况。

## 基于半同步复制的RPO = 0方案

因为在半同步复制中，事务要等Binlog复制到备节点后才能提交，人们自然而然就会想到通过半同步复制来构建RPO=0的数据一致性方案。

![](/assets/img/bigtxn-binlog-4.webp)

这个架构中需要两个备节点，半同步复制保证事务在收到任意一个备节点的应答后才能提交。

- 如果主宕机，那么数据至少复制到了一个备上。
- 如果一个备节点不可用，也不会影响集群的可用性。

要保证RPO=0，就要保证半同步不能退化成异步。MySQL的半同步复制有两个可退化异步的点：

- 宕机重启后，已经写入Binlog的事务会自动提交，此时这些事务可能还没有复制到备库上。
- 当等待时间达到`rpl_semi_sync_master_timeout`设置的时间后，退化成异步。

前者是没办法控制的，需要修改MySQL的代码来解决。后者则需要给`rpl_semi_sync_master_timeout`设置一个非常大的值，让半同步不能退化成异步。大事务无疑是RPO=0方案中非常棘手的问题。因为一旦有了大事务，就会导致整个集群不可写。因此在RPO=0的技术方案中，我们必须要采取一些措施。如果DBA对业务有比较高的约束力，则会采取措施在业务中规避大事务的出现。然而对于大公司而言，业务纷繁复杂很难做到全面消除大事务。RDS服务提供商，则对使用者完全没有约束力。在实际的环境中，可用性往往比数据一致性重要的多。因此许多方案中都会采用临时退化的策略，当有大事务时临时退化成异步的形式。

## 大事务实时传输

在AliSQL中，我们设计了一套实时传输的机制来解决大事务传输导致的问题，有了这套传输机制就`不需要`将半同步退化为异步了。

![](/assets/img/bigtxn-binlog-5.webp)

`大事务实时传输机制`在事务做DML期间，就将事务产生的Binlog Events从Binlog Cache的临时文件中读出，传送到备节点。关键的步骤包括：

- 在DML执行期间，当事务产生的Binlog Events超过一定的量时，该事务会被注册到大事务列表中，当作大事务进行处理。
- Binlog Dump线程会根据大事务列表中的信息，读取大事务的Binlog临时文件，将内容发送到备节点。大事务的Binlog Events和Binlog文件中的Binlog Events交替发送，并且对大事务的发送做了限流处理，优先发送Binlog文件中的内容，保证当前正在提交的事务不受影响。
- 大事务的Binlog Events发送时有特殊的标记和额外信息。当备节点的IO线程收到大事务的Events，将其存储到一个临时文件中，这里称作`Relay Log Cache`。
- 事务提交时，Dump线程发送完所有Binlog Events，然后将`Gtid_even`发送给备库。
- 备库收到`Gtid_event`后，知道已经接收到了事务的所有Binlog Events。然后将`Relay Log Cache`转成一个`Relay Log`文件。
- 当多个大事务同时执行时，这套机制支持多个大事务同时进行实时传输。

根据以上的步骤我们可以知道：`大事务的Binlog Events是在产生时就一点一点的发送到了备库，当事务提交时只需要发送Gtid_event`。因此提交阶段发送的数据量非常的小，就不会阻塞其他事务的Binlog Events传输。此外，也消除了突发了大量网络传输，减少了网络的拥塞。

### Relay Log Cache

实时传输的机制和大事务的提交优化一脉相承，并且复用了大事务优化中的一些实现。事务的Binlog Events是在DML执行的过程中产生，逐渐累积。当事务产生的Binlog Events超过`binlog_cache_size`时，这些Binlog Events会被写入一个临时文件中。在事务提交时这些Binlog Events一次性的写入到Binlog文件中。《MySQL大事务提交优化》中，大事务的临时文件会自动转成一个新的Binlog文件，因此大事务的提交导致的问题被消除。

大事务实时传输复用了这个逻辑，在Relay Log Cache的头部保留一定的空间。当Relay Log Cache被转成Relay Log文件时，需要在头部填充Relay Log需要的一些特殊Binlog Events，比如Format\_description\_event等。

![](/assets/img/bigtxn-binlog-6.webp)

### 异常处理

大事务执行时间比较长，如果中间出了异常则需要做相应的处理。

- 如果大事务在主库回滚了，Binlog dump线程会发送`rollback`给备库，IO线程收到`rollback`后，会将相应的Relay Log Cache销毁。
- 如果IO线程到主库的连接断开了，或者执行了`STOP SLAVE`的操作，IO线程则会销毁所有的Relay Log Cache. 当重新连接后，则会重新进行大事务的实时复制。

## 优化效果

我们用sysbench oltp\_write\_only来模拟正常的写业务，然后在后台提交了一个产生了2GB Binlog Events的事务。测试结果如下：

![](/assets/img/bigtxn-binlog-7.webp)

采用了实时复制后，业务写入运行平稳，不再有业务跌0的情况出现。

## 总结

在MySQL半同步复制架构中，大事务是一个比较典型的问题。为了避免大事务造成实例的不稳定，人们不得不努力的去消除业务中的大事务，或者干脆让其退化为异步复制。`大事务实时传输`则将大事务的binlog Events的传输从提交阶段提前到了执行阶段，Binlog Events产生后立刻传输到备库。因此避免了在提交阶段长时间阻塞其他事务的Binlog Events传输，也避免了网络的拥塞。当遇到大事务时半同步复制不再需要退化为异步复制，为实现基于半同步的RPO=0方案解决了一个棘手问题。
