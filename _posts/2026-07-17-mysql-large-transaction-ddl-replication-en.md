---
title: "Optimizing Replication Lag for Large Transactions and DDL in MySQL"
date: 2026-07-17 17:30:00 +0800
categories: [Database, MySQL]
tags: [MySQL, Replication, DDL, Large Transaction, Replication Lag]
toc: true
lang: en
hidden: true
---

> This article is also available in Chinese: [中文版](/posts/mysql-large-transaction-ddl-replication/). Browse [all English articles](/english/).
{: .prompt-tip }

Starting from MySQL 5.6, the official MySQL team has been working on replication lag, first implementing schema-level parallel application of the binlog. However, that level of parallelism does not solve everyday replication lag. MySQL 5.7 then introduced the `Commit-Order` parallel replay strategy, which depends on how many transactions run concurrently on the primary: only when the primary has a high degree of concurrency can the replica replay quickly. If concurrency on the primary is low, replay on the replica is still slow and lag builds up. To address this, MySQL 5.7 also introduced the `Writeset` (row-level) parallel strategy, which lets the replica replay in parallel quickly regardless of how concurrent the primary is.

We adopted the writeset-based replication strategy across our fleet a long time ago, and it solved roughly 60% of our replication-lag problems. Another 30%+ of replication lag comes from large transactions and DDL. Within MySQL's replication architecture, replication lag caused by large transactions and DDL is arguably the hardest problem to solve. Last year we implemented a mechanism in AliSQL called `Binlog Realtime Replication (BRR)` that solves this problem completely.

## How Binlog Realtime Replication Works

![](/assets/img/bigtxn-repl-1.webp)

The cause of replication lag for large transactions and DDL is shown above. Binlog replication works at the granularity of a transaction: only after a transaction finishes are its events written to the binlog file, shipped to the replica, and executed there (a DDL can be viewed as a single transaction). Only once the replica finishes executing can the change become visible to applications. If a transaction takes a long time to run, it takes the same amount of time on the replica, producing lag equal to the replica's execution time. In practice the lag can be even larger: first, a large transaction produces very large binlog events, so there is additional lag from transmission; second, while a large transaction — especially a DDL — is executing, it can block the replay of other transactions, causing more relay log to pile up. After the large transaction or DDL finishes replaying, those piled-up transactions also need time to replay before the replica can catch up.

![](/assets/img/bigtxn-repl-2.webp)

The idea behind the optimization is intuitive: let the replica start executing the large transaction or DDL at the same time as the primary, and once the primary commits, notify the replica to commit as well. With this mechanism, replication lag for large transactions and DDL can be kept under `1 second`. Below is a comparison of the lag produced by a large transaction before and after the optimization; with realtime replication, large transactions no longer cause replication lag, and neither do DDLs.

![](/assets/img/bigtxn-repl-3.webp)

This feature has been enabled by default in production since 2025. To date more than 3,000 instances have used it, with realtime replication running about 300,000 times for large transactions and about 60,000 times for DDL.

## Implementing Realtime Replication

The core idea of realtime replication is a single sentence: `as soon as the primary starts executing, it ships the binlog events (or DDL) to the replica, which executes them in step; when the primary finally commits or rolls back, the replica follows suit`.

Realtime replication has two parts: realtime transmission and realtime application. Realtime transmission streams the binlog events produced by a large transaction on the primary to the replica in real time; this part is described in *[Binlog Transmission Optimization for Large MySQL Transactions](https://mp.weixin.qq.com/s?__biz=MzIyMTQ1NDE0MQ==&mid=2247484516&idx=1&sn=096bb73138047bf48187e1d33d892e91&scene=21#wechat_redirect)*. Realtime application replays those transmitted binlog events on the replica in real time, for which a dedicated group of replay threads is introduced, as shown below:

![](/assets/img/bigtxn-repl-4.webp)

While a transaction executes on the primary, the binlog events it produces are first buffered in the Binlog Cache. If this is a large transaction (the Binlog Cache size exceeds a threshold), the Dump thread on the primary reads the Binlog Cache temporary file and sends the events directly to the replica. On receiving them, the replica writes them into a dedicated `Brr Cache` (not the relay log file), and a new group of `Brr Worker` threads applies them in real time.

For DDL, binlog events are produced later than for a large transaction — a DDL only writes its `Query_log_event` into the Binlog Cache during the commit phase. So BRR handles DDL specially: after the primary begins executing the DDL, it constructs the `Query_log_event` directly and places it in an in-memory buffer, `ddl_query_buffer`; the Dump thread reads events from this buffer and sends them to the replica, where the DDL is likewise executed in real time by a Brr Worker.

As a result, replica execution of DDL and large transactions changes from `execute only after the primary finishes` to `execute on primary and replica in parallel`, leaving only the network transmission and commit as the final lag — typically on the order of tens of milliseconds.

Below we look at how BRR is implemented from both the primary and replica sides.

### Overall BRR Architecture

![](/assets/img/bigtxn-repl-5.webp)

#### Primary Side

When a large transaction or DDL needs realtime replication, a `Brr_trx` is created and registered with the `Brr_trx_manager`.

`Brr_binlog_sender` is an extension of the Dump thread; it reads events from a `Brr_trx` and pushes them to the replica. Originally the Dump thread did only one thing: read events from the binlog file and send them to the replica. BRR gives it a second responsibility: poll each active `Brr_trx` and read binlog events from its Binlog Cache temporary file or from `ddl_query_buffer`, then send them to the replica.

Realtime transmission reuses the existing Dump channel. To distinguish BRR traffic from ordinary traffic, BRR borrows an idea from Semisync and attaches an extra `BRR Header` to each event; the information in that header tells BRR traffic apart from ordinary replication traffic. To keep BRR events from choking the ordinary binlog-event channel, BRR performs flow control.

![](/assets/img/bigtxn-repl-6.webp)

#### Replica Side

Based on the `BRR Header`, the replica's IO thread splits events into two categories, BRR events and normal events: BRR events go into the `Brr_cache`, while normal events follow the original path into the relay log.

`Brr_cache` is the replica-side storage for a BRR transaction; one BRR transaction corresponds to one `Brr_cache`. When the replica's IO thread receives a BRR event, it uses the `brr_index` in the BRR Header to find the corresponding `Brr_cache` (if it is the first event, it creates a new `Brr_cache` and wakes a Brr Worker), then writes the events into the `Brr_cache` temporary file and updates the readable position.

`Brr_rpl_info` manages these BRR transactions.

The `BRR Worker` threads are dedicated to applying BRR transactions. When idle, a Brr Worker finds a `Brr_cache` whose application has not yet started and sets itself as its owner. From then on the Brr Worker is bound to that `Brr_cache`, looping to read and replay binlog events until it sees a `Gtid_log_event` (meaning the primary has committed) or receives a `BRR_ROLLBACK_EVENT` (the primary rolled back).

### The gtid_executed Snapshot

On the primary, these `uncommitted` BRR transactions and `already-committed` transactions execute in parallel on the replica. If a BRR transaction has a dependency on an already-committed transaction, the BRR transaction's binlog events must not start until the dependency has finished replaying on the replica; otherwise it could cause a deadlock, break replication, or in the worst case cause data inconsistency. Consider this example:

```sql
INSERT INTO t1(pk, c2) VALUES(pk1, 1);
UPDATE t1 SET c2 = 2;  -- large transaction
```

The `UPDATE` is a large transaction; it must not begin until the `INSERT` has finished replaying. If the `UPDATE` runs first, it will fail when updating the `pk1` row because that row does not yet exist.

BRR uses a `gtid_executed snapshot` to judge these ordering dependencies. When a DDL or large transaction begins executing on the primary, the primary's current `gtid_executed` value represents all the preceding transactions this transaction saw on the primary. As long as the replica's `gtid_executed` has caught up to that value (i.e., is a superset of it), all the preceding transactions this transaction depends on have already been replayed on the replica and it is safe to start applying it.

To this end BRR introduces a new event type, `Brr_gtid_executed_log_event`, whose body holds a `gtid_executed` set. At specific moments the primary takes a gtid_executed snapshot and writes it to the BRR channel; when a replica Brr Worker reads the binlog events of this snapshot, it waits for all the transactions of the GTIDs in the snapshot to finish before continuing with the subsequent operations.

### Realtime Replication of Large Transactions

![](/assets/img/bigtxn-repl-7.webp)

#### Creating and Updating a Brr_trx

When a transaction executes on the primary, its binlog events are first written to the Binlog Cache (a structure combining an in-memory buffer and a temporary file). In MySQL's implementation, once the Binlog Cache fills its in-memory buffer, it spills to the temporary file.

BRR hooks in here: every time a batch of events is written to the Binlog Cache, it checks the size of the temporary file. If it exceeds a certain size, it creates a `Brr_trx`, records the temporary file name and the current readable position, and registers it with `Brr_trx_manager`. Thereafter, every append of binlog events to the Binlog Cache updates the `Brr_trx`'s `end_position` and wakes the Dump thread to send those events to the replica.

#### Transmitting Binlog Events

Before sending each batch of binlog events, the Dump thread produces a `Brr_gtid_executed_log_event` as the dependency snapshot for that batch and sends it to the replica, then sends the batch itself.

#### Committing the Transaction

For a large transaction, until the Brr Worker reads the `Gtid_log_event` the binlog events sit in the `Brr_cache` temporary file and are not yet relay log. When the primary finally commits, the `Gtid_log_event` is sent to the replica over the BRR channel and the IO thread does two things:

1. Renames the `Brr_cache` temporary file into a relay log file. The Dump thread on the primary will skip sending this transaction based on its GTID, so these binlog events are not shipped again as an ordinary transaction.
2. Notifies the Brr Worker to read the `Gtid_log_event` and `Xid_log_event` and complete the commit.

#### Rolling Back the Transaction

The rollback path is simple: when the primary rolls back, it sends a `BRR_ROLLBACK_EVENT` over the BRR channel; on receiving it, the replica's IO thread sends a `KILL_QUERY` signal to the corresponding Brr Worker. When the Brr Worker detects `KILL_QUERY`, it rolls back the current transaction, cleans up, and moves on to the next `Brr_cache`.

Note that after being killed, the Brr Worker does not exit and does not propagate the error to the SQL thread. This is completely different from an ordinary Worker, which must stop all of replication when it hits an error. The reason: for an ordinary Worker the transaction has already committed on the primary, so if the replica gives up it causes inconsistency; but a Brr Worker's transaction runs concurrently with the primary, and a primary rollback is a normal path, so the replica must roll back too.

### Realtime Application of DDL

![](/assets/img/bigtxn-repl-8.webp)

#### Creating a Brr_trx

For large transactions we decide whether it is "large" by the total size of the binlog events in the Binlog Cache. DDL is more complicated: some DDLs only touch metadata and finish very quickly, while for DDLs that touch data the execution time depends on the amount of data, which is complex to estimate accurately. So rather than deciding whether to realtime-replicate a DDL by predicting its execution time in advance, we `decide by whether the DDL's execution exceeds a timeout`.

Every DDL creates a `Brr_trx`, but this `Brr_trx` is not sent to the replica immediately. A DDL's `Brr_trx` has a threshold, 1000 ms by default: only when the DDL's execution time exceeds this threshold does the Dump thread begin sending the DDL's `Brr_trx`. If a DDL finishes quickly, within 1 second, the `Brr_trx` is cleaned up silently and the DDL is shipped to the replica over the ordinary binlog channel — exactly as if BRR were off.

A DDL's `Brr_trx` is created during the DDL's Prepare phase, that is, `after the DDL has acquired the MDL X lock`, because only after acquiring the X lock does the DDL have permission to operate on the table. Any conflicting operations have either already committed or must wait until the DDL releases the X lock or the DDL finishes.

### Two gtid_executed Snapshots

An Online DDL divides its execution into three phases: `Prepare`, `Execute`, and `Commit`. After Prepare, the MDL `X lock` is downgraded to an `S lock`, so during the Execute phase DML and DDL can run in parallel. During the Commit phase the `S lock` is upgraded back to an `X lock`; upgrading to the X lock means all those parallel DMLs have already committed. So when the DDL executes on the replica it must follow the same principle: those already-committed DMLs must finish replaying before the replica can enter the Commit phase.

Therefore, for realtime replication of an Online DDL, the replica has two points that must be synchronized when executing the DDL: one before entering the Prepare phase and one before entering the Commit phase. Correspondingly, the primary must take two `gtid_executed` snapshots: one after the DDL enters the Prepare phase, and one after the DDL enters the Execute phase.

### Shipping Binlog Events Twice

In the large-transaction section we noted that a large transaction is transmitted to the replica via BRR, and the large transaction in the binlog file is not shipped to the replica again. A DDL, by contrast, is shipped twice: `BRR ships it once, and the binlog events in the binlog file ship it a second time.`

A DDL's `Query_log_event` is small, so the cost of shipping it twice is negligible. But if it were shipped only once, we would have to use the large-transaction rename logic (renaming the `Brr_cache` temporary file into relay log), which involves a lot of edge-case handling. For DDL, simply shipping it twice and discarding the `Brr_cache` when done is the simplest approach.

As for ordering, the Dump thread guarantees that BRR events are shipped before ordinary events. This way the Brr Worker is guaranteed to get the DDL first and start executing it; by the time the ordinary events reach the relay log, the Brr Worker is already applying the DDL.

When an ordinary Worker reads the DDL from the relay log, it checks whether this GTID is in `owned_gtids`. If it is (meaning a Brr Worker is executing it), it waits; after the Brr Worker commits, adds the GTID to `gtid_executed`, and releases `owned_gtids`, the ordinary Worker wakes, finds the GTID already in `gtid_executed`, and skips the entire DDL.

If the Brr Worker rolled the DDL back, the GTID is removed from `owned_gtids` and not added to `gtid_executed`. When the ordinary Worker wakes and finds the transaction was not executed, it executes the DDL normally — `this is the fallback path, equivalent to the BRR-off scenario.`

## Conclusion

AliSQL's Binlog Realtime Replication solves the thorniest replication-lag problems in MySQL binlog replication — lag from large transactions and DDL — through a mechanism of parallel execution on the primary and replica. In addition, we have made many optimizations for the writeset mechanism, massively concurrent scenarios, and the medium-transaction scenarios produced by batch processing. Together these optimizations have eliminated 95% of the replication lag in our production environment.
