---
title: "Binlog Transmission Optimization for Large MySQL Transactions"
date: 2026-07-16 10:00:00 +0800
categories: [Database, MySQL]
tags: [MySQL, Replication, Binlog, Large Transaction, Semisync]
toc: true
lang: en
hidden: true
---

> This article is also available in Chinese: [中文版](/posts/mysql-large-transaction-binlog-transmission/). Browse [all English articles](/english/).
{: .prompt-tip }

Large transactions are a pain point in MySQL: they cause not only replication lag but also a host of stability problems. A previous article, *[MySQL Large Transaction Commit Optimization](https://mp.weixin.qq.com/s?__biz=MzIyMTQ1NDE0MQ==&mid=2247484449&idx=1&sn=d308f27938b563cb197964fbc2c9b6bd&scene=21#wechat_redirect)*, covered the problems a large transaction causes at commit time and the optimizations we made in AliSQL. This article looks at the problems a large transaction causes during semi-synchronous replication, and how AliSQL solves them.

In *[MySQL Large Transaction Commit Optimization](https://mp.weixin.qq.com/s?__biz=MzIyMTQ1NDE0MQ==&mid=2247484449&idx=1&sn=d308f27938b563cb197964fbc2c9b6bd&scene=21#wechat_redirect)* we noted that writing the binlog when a large transaction commits can produce strange slow queries like these:

![](/assets/img/bigtxn-binlog-1.webp)

- An `INSERT` that normally runs in an instant took `1.3s`, yet the slow-query log shows no long lock wait.
- Every statement in a multi-statement transaction had already finished, yet the `COMMIT` alone took `1.3s`.

Besides writing the binlog at commit, transmitting a large transaction's binlog during semi-synchronous replication produces the same symptom. Below is a simulated test: we used sysbench `oltp_write_only` to simulate a normal write workload, then committed, in the background, a transaction that generated 2 GB of binlog events (with the large-transaction commit optimization already applied). When the large transaction commits, writes drop to zero and don't recover until semisync times out.

![](/assets/img/bigtxn-binlog-2.webp)

## Root Cause

![](/assets/img/bigtxn-binlog-3.webp)

The figure above shows the commit flow of a transaction under semi-synchronous replication:

- On commit, the transaction runs two-phase commit, starting with Prepare.
- It then writes its binlog events to the binlog file.
- After writing the binlog, it waits for its binlog events to be sent to the replica (`after_sync` mode).
- The binlog Dump thread then sends the transaction's binlog events to the replica.
- The replica's IO thread receives these events and writes them into the relay log file.
- Once it has the complete transaction, the IO thread sends the primary an acknowledgment saying it has all of the transaction's binlog events. The ack is expressed as a binlog file name and offset. In the figure, Trx_n's binlog end offset is `530`, so the replica's IO thread sends `master-bin.000001:530` to the primary, meaning every transaction before `master-bin.000001:530` has been received.
- On the primary, the Semisync Ack Receiver thread receives the ack and, based on the offset, wakes the corresponding transaction.
- Once woken, the transaction finishes committing and returns OK to the user.

There is only one Dump thread between the primary and the replica, and it transmits binlog events in the order they were written to the binlog. The replica's IO thread likewise writes received events into the relay log in that same order before acknowledging the primary. So a later transaction can't be sent until the earlier one has finished. If the current transaction has a huge number of binlog events, sending them takes a very long time, and a later transaction — however small — has to wait. That wait includes not just its own transmission time but the large transaction's ahead of it. Hence the slow-log symptom: a small transaction suddenly becomes very slow.

To cope with this, MySQL provides the `rpl_semi_sync_master_timeout` parameter, which sets how long a transaction waits for an ack; once the wait exceeds `rpl_semi_sync_master_timeout`, replication automatically falls back to asynchronous. We can set this to a small value to avoid the severe case where a large transaction makes the whole instance unwritable.

## An RPO = 0 Design Based on Semi-Synchronous Replication

Because a transaction under semi-synchronous replication can't commit until its binlog has been replicated to a replica, it's natural to think of using semisync to build an RPO = 0 (zero data loss) consistency solution.

![](/assets/img/bigtxn-binlog-4.webp)

This architecture needs two replicas, and semisync guarantees that a transaction commits only after it receives an ack from at least one of them.

- If the primary crashes, the data has been replicated to at least one replica.
- If one replica becomes unavailable, cluster availability is unaffected.

To guarantee RPO = 0, semisync must never fall back to async. MySQL semisync has two points where it can degrade to async:

- After a crash and restart, transactions already written to the binlog are committed automatically, even though they may not yet have been replicated to a replica.
- Once the wait reaches `rpl_semi_sync_master_timeout`, it degrades to async.

The former can't be controlled from outside — it requires changing MySQL's code. The latter requires setting `rpl_semi_sync_master_timeout` to a very large value so semisync never degrades. Large transactions are clearly the thorniest issue in an RPO = 0 design: the moment one appears, it makes the whole cluster unwritable, so the design must take countermeasures. A DBA with strong influence over the application can arrange for it to avoid large transactions; but at a large company, with sprawling and complex applications, eliminating them entirely is hard, and an RDS provider has no control over its users at all. In practice, availability usually matters far more than consistency, so many designs adopt a temporary-degradation strategy, falling back to async whenever a large transaction appears.

## Realtime Transmission of Large Transactions

In AliSQL we designed a realtime-transmission mechanism to solve the problems large-transaction transmission causes; with it, there is no need to degrade semisync to async.

![](/assets/img/bigtxn-binlog-5.webp)

The realtime large-transaction transmission mechanism reads a transaction's binlog events out of the Binlog Cache temporary file and sends them to the replica while the transaction is still doing DML. The key steps:

- During DML execution, once the binlog events a transaction has produced exceed a certain amount, the transaction is registered in the large-transaction list and handled as a large transaction.
- Based on that list, the binlog Dump thread reads the large transaction's binlog temporary file and sends its contents to the replica. The large transaction's binlog events and the events from the binlog file are sent interleaved, with flow control on the large transaction: events from the binlog file take priority, so the transaction currently committing is unaffected.
- The large transaction's binlog events carry a special marker and extra information. When the replica's IO thread receives them, it stores them in a temporary file called the `Relay Log Cache`.
- At commit, once the Dump thread has sent all the binlog events, it sends a `Gtid_event` to the replica.
- On receiving the `Gtid_event`, the replica knows it has all of the transaction's binlog events, and it turns the `Relay Log Cache` into a `Relay Log` file.
- When several large transactions run at once, the mechanism can transmit them all in real time simultaneously.

From these steps we can see: *a large transaction's binlog events are sent to the replica bit by bit as they are produced, so at commit only the `Gtid_event` needs to be sent.* The amount of data sent at commit is therefore tiny, and it no longer blocks other transactions' binlog-event transmission. It also removes the sudden burst of network traffic, reducing congestion.

### Relay Log Cache

The realtime-transmission mechanism follows directly from the large-transaction commit optimization and reuses parts of its implementation. A transaction's binlog events are produced and accumulate during DML execution; once they exceed `binlog_cache_size`, they are written to a temporary file, and at commit they are written to the binlog file all at once. In *MySQL Large Transaction Commit Optimization*, a large transaction's temporary file is automatically turned into a new binlog file, which eliminates the problems that large-transaction commit causes.

Realtime large-transaction transmission reuses this logic, reserving some space at the head of the Relay Log Cache. When the Relay Log Cache is turned into a Relay Log file, that head space is filled with the special binlog events a relay log needs, such as the `Format_description_event`.

![](/assets/img/bigtxn-binlog-6.webp)

### Handling Failures

A large transaction runs for a long time, so any failure along the way has to be handled.

- If the large transaction rolls back on the primary, the binlog Dump thread sends a `rollback` to the replica; on receiving it, the IO thread destroys the corresponding Relay Log Cache.
- If the IO thread's connection to the primary drops, or a `STOP SLAVE` is issued, the IO thread destroys all Relay Log Caches. After reconnecting, it restarts realtime replication of the large transaction.

## Results

We used sysbench `oltp_write_only` to simulate a normal write workload, then committed, in the background, a transaction that generated 2 GB of binlog events. The results:

![](/assets/img/bigtxn-binlog-7.webp)

With realtime replication, the application's writes run smoothly, with no more drops to zero.

## Conclusion

In MySQL's semi-synchronous replication architecture, large transactions are a classic problem. To keep them from destabilizing the instance, people have had to work hard to eliminate large transactions from their applications, or simply let replication degrade to async. *Realtime large-transaction transmission* moves the transmission of a large transaction's binlog events from the commit phase up to the execution phase, sending each event to the replica as soon as it is produced. This avoids blocking other transactions' binlog-event transmission for a long time at commit, and avoids network congestion. When a large transaction comes along, semisync no longer needs to degrade to async — clearing away a thorny obstacle on the path to a semisync-based RPO = 0 design.
