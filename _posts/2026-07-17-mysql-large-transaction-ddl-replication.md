---
title: MySQL 大事务和 DDL 的复制延迟优化
date: 2026-07-17 17:00:00 +0800
categories: [数据库, MySQL]
tags: [MySQL, 复制, DDL, 大事务, 主从延迟]
toc: true
---

MySQL官方从MySQL-5.6开始优化复制延迟问题，最先实现了 Schema 级别的 Binlog 并发应用。然而这个并发能力并不能解决日常的复制延迟。然后官方在 MySQL-5.7上实现了`基于提交顺序的并发回放策略(Commit Order)`, 这种策略依赖主上的事务并发执行数量。主上并发执行的事务非常多的情况下，从库上回放的速度才会快。如果主上并发少，从库上回放速度仍然会变慢，产生复制延迟。因此官方又在 MySQL-5.7 上又实现了`行级的并发策略(Writeset)`, 这种并发策略无论主上并发事务的多少，都能在从库上快速的并发回放。

我们很早以前就在线上全面使用了基于 writeset 的复制策略，writeset 的使用解决了大概60%的复制延迟问题。另外有30%多的复制延迟问题源自于大事务和 DDL。在 MySQL 的复制体系里，大事务和 DDL 的复制延迟可谓是最难以解决的问题。去年，我们在 AliSQL 上实现了`Binlog 实时复制(Binlog Realtime Replication 简称BRR)`的机制，彻底解决了这个难题。

## Binlog 实时复制的原理

![](https://mmbiz.qpic.cn/sz_mmbiz_png/Jg6JM4XTDcdoiapepLfBPajWlVXCMsanwB0V7j7mx5JFzOU5bWDuOPibD44PlpncgyESHfTXvk6UwdPib37fXZPL9L4EDepiaE8zOicCOibOmjzBM/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=0)

大事务和 DDL 的复制延迟原因如上图所示。Binlog 复制是以事务为单位，只有事务执行完了才会写入 Binlog 文件，然后传输到从库执行( DDL 可以看做一个事务），只有从库执行完毕，业务才能看到。如果事务执行了很长时间，在从库也需要执行同样的时间，就会产生复制延迟。延迟的时间就是从库执行事务的时间。实际上延迟可能会更大：其一对于大事务来说，因为产生的 Binlog Events 非常大，还有一块是传输导致的延迟。其二在大事务尤其是 DDL 执行期间，可能会阻塞其他的事务回放，导致更多的 Relay Log 堆积。大事务和 DDL 回放完成后，这些堆积的事务也需要一定的时间回放，才能够追平。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/Jg6JM4XTDce0NvK8AINIy2arj4SzKM9CdRyxwAYW9YNeZQwnwXaETDVsIonsUxicvK2mXmrbd2weseVzydj3nnK7NFqLIcCEQYlzjdXq5mOA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=1)

优化的思路也非常直观，那就是让从库和主库同时开始执行大事务、DDL，主库提交后，通知从库提交事务。通过这套机制，大事务和 DDL 的复制延迟可以控制在`1秒`以内。以下是优化前、优化后大事务产生的延迟对比，采用了实时复制的机制后大事务不再导致复制延迟，DDL 也一样。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/Jg6JM4XTDcdS9ibNt1n8W69W8ZZ48la0T07ZRRGZHTqj8nulMTazr2IspJhZZpsXRrPjBIYhiaJnpzLxVOUY1k3MgNPvw8cF1FovXH2qjDyYE/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=2)

这个功能已于2025年在线上默认开启，截止目前累积3000+实例使用了该功能。大事务实时复制执行了约30万次，DDL实时复制执行了约6万次。

## 实时复制的实现

实时复制的核心思想只有一句话：`主库执行开始就把 Binlog Events 或 DDL 发给从库，从库同步执行；主库最终提交或回滚，从库跟着提交或回滚`。

实时复制分为实时传输和实时应用两部分。实时传输将主库上大事务实时产生的 Binlog Events 流式的发送到从库上，这一部分在《[MySQL大事务的Binlog传输优化](https://mp.weixin.qq.com/s?__biz=MzIyMTQ1NDE0MQ==&mid=2247484516&idx=1&sn=096bb73138047bf48187e1d33d892e91&scene=21#wechat_redirect)》做了介绍。实时应用则是实时的将这些传输过来的 Binlog Events 回放到从库，为此引入了一组额外的回放线程。如下图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/Jg6JM4XTDce6rbM87pLtEcibicxvCzMte7ic7cQEC9mVpmBjPyvPL67hBENthvEpib7fJXOQmWuxGDTZvrGzscQ29pUj8L6LtLeYbOQV9LTv8bQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=3)

事务在主库执行的过程中，产生的 Binlog Events 会先暂存在 Binlog Cache 里；如果这是一个大事务（Binlog Cache 大小超过阈值），主库上的 Dump 线程会读取 Binlog Cache 临时文件，把Binlog Cache 里的 Binlog Events 直接发送到从库。从库收到后，写入一个专门的 `Brr Cache`（不是 Relay Log 文件），由一组新的 `Brr Worker` 线程实时应用。

对 DDL 来说，Binlog Events 的产生时机比大事务更晚——DDL 是在提交阶段才把 `Query_log_event` 写入 Binlog Cache。所以 BRR 对 DDL 做了一个特别处理：在主库开始DDL执行后，直接构造 Query\_log\_event 放到一个内存 Buffer `ddl_query_buffer` 里，Dump 线程从这个 Buffer 里读事件发给从库，从库上同样是通过 Brr Worker 实时执行 DDL。

这样一来，DDL 和大事务的从库执行就从`主库执行完再执行`变成了`主从并行执行`，最终延迟只剩下网络传输和提交的耗时，通常在十毫秒量级。

下面我们从主库端和从库端两个视角，看看 BRR 是怎么实现的。

### BRR 的整体架构

![](https://mmbiz.qpic.cn/mmbiz_png/Jg6JM4XTDcfVeoNNH6TNmD2lf88hsicic8n67m5hKiaVh3U3qDBdJzJfr4O0HnoX7rZCxyjmP4bWkOedfVCgU0u4AqDPy7xrVlAy5HtV6oh6Rs/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=4)

#### 主库端

当一个大事务或者 DDL 需要进行实时复制时，会产生一个`Brr_trx` 注册到`Brr_trx_manager`里。

`Brr_binlog_sender` 是 Dump 线程的扩展，负责从 Brr\_trx 读取 Events 并推送到从库。Dump 线程原本只做一件事：从 Binlog 文件读事件发给从库。BRR 给它增加了第二个职责：轮询每个活跃的 Brr\_trx 从他们的 Binlog Cache 的临时文件 或者 ddl\_query\_buffer 里读 Binlog Events 发给从库。

实时传输借用原有的 Dump 通道传输信息，为了区分 BRR 流量和普通流量，这里借鉴了 Semisync 的机制，给每个Event 添加了一个额外的 `BRR Header`。 通过 BRR Header 里的信息可以区分是 BRR 流量还是普通复制流量。 为了不让 BRR 事件把普通的 Binlog Events 通道堵死，BRR 做了流量控制。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/Jg6JM4XTDcdIF6icybicbBrBfObXXwBRNHtLhVR398rO5Qic2cEozSVII1v4aJ8sSl7rRuYd5KYiciakdD4ibdPW4pmISM9lVicILVpjXoymvlf5PA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=5)

#### 从库端

从库 IO 线程根据 `BRR Header` 把事件分成 BRR 事件和 Normal 事件两类：BRR 事件走 `Brr_cache`，Normal 事件按原路走 Relay Log。

`Brr_cache` 是 BRR 事务在从库对应的存储，一个 BRR 事务对应一个 Brr\_cache。从库 IO 线程收到一个 BRR 事件后，根据 BRR Header 里的 `brr_index` 找到对应的 Brr\_cache（如果是第一个事件，就创建一个新的 Brr\_cache，并唤醒 Brr Worker），然后把 Events 写入 Brr\_cache 的临时文件，更新可读位置。

`Brr_rpl_info` 负责管理这些 BRR 事务。

`BRR Worker` 线程专门用来应用 BRR 事务。Brr Worker 在空闲时会找到一个未开始应用的 Brr\_cache，把自己设为它的 owner。之后 Brr Worker 就绑定在这个 Brr\_cache 上，循环读 Binlog Events 回放，直到 `Gtid_log_event`（说明主库已经提交），或者收到 `BRR_ROLLBACK_EVENT` (主库回滚)才结束。

### gtid\_executed 快照

主库上这些`未提交`的 BRR 事务和`已经提交`的事务在从库上并行的执行，如果 BRR 事务和已经提交的事务有依赖关系，那么 BRR 事务的 Binlog Events 必须要等依赖的事务在从库回放完成后才能开始。否则就可能导致死锁、复制中断，严重的可能数据不一致。比如下面这个例子：

```sql
INSERT INTO t1(pk, c2) VAUES(pk1, 1);   
UPDATE t1 SET c2 = 2;  // 大事务
```

UPDATE 是大事务，它必须要在 INSERT 回放完成后，才能开始执行。如果 UPDATE 先执行，则在更新 `pk1` 这条记录时因为记录不存在而报错。

BRR 机制里使用 `gtid_executed 快照` 作为前后依赖关系的判定。当主库某个 DDL 或大事务开始执行时，主库当前的 `gtid_executed` 值就代表了这个事务在主库看到的所有前序事务。只要从库的 `gtid_executed` 也追上了这个值（是它的超集），就说明这个事务在从库依赖的所有前序事务都已经回放完毕，可以开始应用了。

为此，BRR 引入了一个新的 event 类型 `Brr_gtid_executed_log_event`，body 存放一个 `gtid_executed` 集合。主库在特定的时刻会打 gtid\_executed 快照，写入 BRR 通道；从库 Brr Worker 读到这个快照的 Binlog Events，会等待这个快照里的所有 Gtid 对应的事务都执行完毕，然后再继续应用后续操作。

### 大事务实时复制

![](https://mmbiz.qpic.cn/sz_mmbiz_png/Jg6JM4XTDccSWbOmWwQreBV9icTA9T40tg3mdpMHI5JgoTcpPicdgbfWQLblPUmJSzZiaSnXVHQ55ZXanWib1JNzgDNyf5mic4dlJrFjIOmbYLHU/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=6)

#### Brr\_trx 的创建与更新

一个事务在主库上执行时，Binlog Events 会先写入 Binlog Cache（一个内存 Buffer + 临时文件的结构）。MySQL 的实现里，当 Binlog Cache 用满内存 Buffer 后，会写入临时文件里。

BRR 在这里插了一个钩子：每次向 Binlog Cache 写完一批 event 后，检查临时文件的大小。如果超过了一定的大小，就创建一个 `Brr_trx`，记录临时文件名和当前的可读位置，然后把它注册到 `Brr_trx_manager` 里。之后每次向 Binlog Cache 追加 Binlog Events，都会更新 `Brr_trx` 的 `end_position`，并唤醒 Dump 线程发送这些 Binlog Events 到从库。

#### Binlog Events 传输

Dump 线程每次发送一批 Binlog Events 之前都要产生一个 `Brr_gtid_executed_log_event`, 作为当前这批 Binlog Events 的依赖事务的快照发送到从库，然后再发送这批 Binlog Events.

#### 事务提交

对大事务来说，Brr Worker 读到 `Gtid_log_event` 之前，Binlog Events 都写在 `Brr_cache` 的临时文件里，还不算 Relay Log。当主库最终提交时，`Gtid_log_event` 通过 BRR 通道发到从库，IO 线程做两件事：

1. 把 `Brr_cache` 的临时文件 rename 成一个 Relay Log 文件，主上的 Dump 线程会根据 Gtid 跳过这个这个事务的发送。这样这些 Binlog Events 就不会当作普通事务再传一遍。
2. 通知 Brr Worker 读取 `Gtid_log_event`、`Xid_log_event`，完成事务的提交。

#### 事务的回滚

回滚路径很简单，主库回滚时通过 BRR 通道发一个 `BRR_ROLLBACK_EVENT`；从库 IO 线程收到后，直接给对应的 Brr Worker 发 `KILL_QUERY` 信号。Brr Worker 检测到 KILL\_QUERY 后，回滚当前事务，做清理工作，然后继续处理下一个 Brr\_cache。

注意被 Kill 了之后，Brr Worker 不会退出，也不会传播错误到 SQL 线程。这一点和普通 Worker 完全不一样，普通 Worker 遇到错误必须让整个复制停下来。因为主库那边事务已经提交，从库如果放弃执行，就会导致主从数据不一致；而 Brr Worker 执行的事务，是和主库同时执行的，主库回滚是正常路径，从库也必须回滚。

### DDL 的实时应用

![](https://mmbiz.qpic.cn/sz_mmbiz_png/Jg6JM4XTDcdsF2NtrKSSL52Nt46PuT7xNpiaN3iasSZ9KicNb1bOklia5lRtkOp7z5R0cWeKE1MwibpaKIpE89yhIHduMKvyPJOkDiaML5klLE968/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=7)

#### Brr\_trx 的创建

在大事务里，我们通过 Binlog Cache 里的 Binlog Events 的总大小来判定是不是一个大事务。DDL 则复杂一些，有些 DDL 只操作元数据，这些 DDL 执行很快。操作了数据的 DDL 的执行时间和操作的数据量有关，要评估的精确会比较复杂。 因此对于 DDL 是否要进行实时复制，我们不是通过提前预估 DDL 的执行时间来决定， 而是`通过 DDL 执行时间超时的方法来决定是否要执行实时复制`。

所有的 DDL 都会创建一个 Brr\_trx， 但是这个 Brr\_trx 不会立刻发送给从库。DDL 的 Brr\_trx 还有一个门限，默认 1000 毫秒：只有当 DDL 的执行时长超过这个门限，Dump 线程才会开始发送这个 DDL 的 Brr\_trx。如果一个 DDL 执行很快，在1秒内就完成了，`Brr_trx` 会被静默地清理掉，DDL 走普通的 Binlog 通道传到从库回放，和不开启 BRR 完全一样。

DDL Brr\_trx 是在 DDL 的 Prepare 阶段创建的，也就是在 DDL `获取到 MDL X 锁之后`。因为只有获取了 X 锁才意味着 DDL 对这个表有了操作权限。其他有冲突的操作要么已经提交，要么要等待 DDL 释放了 X锁 或者到 DDL 结束才能开始。

### 两次 gtid\_executed 快照

对于 Online DDL 来说，会将执行分为 `Prepare`, `Execute`, `Commit` 三个阶段。在 Prepare 之后，会将 MDL 的 `X 锁`降级为 `S 锁`，因此在 Execute 阶段 DML 和 DDL 可以并行执行。在 Commit 阶段，又将 `S 锁` 升级 为 `X 锁`。升级为 X 锁 则意味着那些并行执行的 DML 都已经提交了。所以 DDL 在从库执行时也要遵循这个原则，要让这些已经提交的 DML 先回放完成，才能进入 Commit 阶段。

因此对于 Online DDL 的实时复制，从库执行DDL时有两个必须要同步的时间点，一个是在进入 Prepare 阶段之前，一个是在进入 Commit 阶段之前。对应地，主库上需要抓取两次 `gtid_executed` 快照，一次是在 DDL 进入 Prepare 阶段之后，一次是在 DDL 进入 Execute 阶段之后。

### 传两份 Binlog Events

在大事务章节，我们提到大事务是通过 BRR 传输到从库上的，Binlog 文件中的大事务不会再次传送到从库。DDL 则会传输两次，`BRR 会传输一次，Binlog 文件中的 Binlog Events 还会传一次。`

DDL 的 `Query_log_event` 尺寸很小，重复传的成本可以忽略。但如果只传一份，就要走大事务那套 rename 逻辑（把 `Brr_cache` 的临时文件 rename 成 Relay Log），涉及一堆边界处理。对 DDL 来说，直接传两份`Brr_cache` 用完删掉是最简单的方案。

顺序上，Dump 线程保证先传 BRR 事件，再传普通事件。这样 Brr Worker 一定能先拿到 DDL 开始执行；等普通事件到达 Relay Log，Brr Worker 已经在应用 DDL 了。

普通 Worker 读到 Relay Log 里的 DDL 时，会检查这个 gtid 是不是在 `owned_gtids` 里。如果是（说明 Brr Worker 正在执行），就等待；等 Brr Worker 提交后把 gtid 加到 `gtid_executed`、释放 `owned_gtids`，普通 Worker 醒来发现 gtid 已经在 `gtid_executed` 里，就会跳过整个 DDL。

如果 Brr Worker 回滚了该DDL，那 Gtid 会从 `owned_gtids` 移除，也不会加到 `gtid_executed`。普通 Worker 醒来后，发现该事务没有被执行，就会正常执行这个DDL —— `这是兜底路径，等价于关闭 BRR 的场景。`

## 结论

AliSQL 的 Binlog 实时复制通过主、从并行执行的机制解决了 MySQL Binlog 复制中最为棘手的复制延迟问题：大事务和DDL的复制延迟问题。此外，我们还针对 Writeset 机制、海量并发场景、批处理产生的中等事务场景也做了许多的优化。通过这些优化，消除了线上95%的复制延迟问题。
