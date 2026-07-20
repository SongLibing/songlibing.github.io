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

Since MySQL 5.6, the MySQL replication team has been working to reduce replication lag. The first step was schema-level parallel application of the binlog, but schema-level parallelism only helps when writes are spread across many databases; in the common case, where most write traffic hits a single database, it provides almost no parallelism. MySQL 5.7 then introduced the `Commit-Order` parallel-replay strategy, which depends on how many transactions run concurrently on the primary: the replica can replay quickly only when the primary is highly concurrent. When concurrency on the primary is low, the replica still replays slowly and lag builds up. To fix that, MySQL 5.7 also introduced the `Writeset` (row-level) strategy, which lets the replica replay in parallel quickly no matter how concurrent the primary is.

We rolled out the writeset-based strategy across our fleet long ago, and it eliminated roughly 60% of our replication-lag problems. Another more than 30% comes from large transactions and DDL — the hardest replication-lag problem to solve in MySQL. Last year we built a mechanism in AliSQL called `Binlog Realtime Replication (BRR)` that solves it completely.

## How Binlog Realtime Replication Works

![](/assets/img/bigtxn-repl-1-en.png)

The figure above shows why large transactions and DDL cause replication lag. Binlog replication works at the granularity of a transaction: a transaction's events are written to the binlog file only after it commits, then shipped to the replica and executed there (a DDL can be treated as a single transaction). The change becomes visible to applications only once the replica finishes executing. If a transaction takes a long time on the primary, it takes just as long on the replica, and the lag equals the replica's execution time. In practice the lag is often worse. First, a large transaction produces very large binlog events, which adds transmission delay. Second, while a large transaction — especially a DDL — is running, it can block the replay of other transactions, so relay log piles up; once the large transaction or DDL finishes replaying, that backlog also needs time to drain before the replica catches up.

![](/assets/img/bigtxn-repl-2-en.png)

The idea behind the optimization is simple: have the replica start executing the large transaction or DDL at the same time as the primary, and once the primary commits, tell the replica to commit too. With this mechanism, replication lag for large transactions and DDL stays under one second. The chart below compares the lag from a large transaction before and after the optimization: with realtime replication, large transactions no longer cause lag, and neither do DDLs.

![](/assets/img/bigtxn-repl-3-en.jpg)

The feature has been enabled by default in our RDS service since 2025. To date more than 3,000 instances have used it, running realtime replication about 300,000 times for large transactions and about 60,000 times for DDL.

## Implementing Realtime Replication

The core idea of realtime replication fits in one sentence: *as soon as the primary starts executing, it ships the binlog events (or DDL) to the replica, which executes them in lockstep; when the primary finally commits or rolls back, the replica does the same.*

Realtime replication has two parts: realtime transmission and realtime application. Realtime transmission streams the binlog events a large transaction produces on the primary to the replica as they are generated; that part is covered in *[Binlog Transmission Optimization for Large MySQL Transactions](/posts/mysql-large-transaction-binlog-transmission-en/)*. Realtime application replays those events on the replica as they arrive, using a dedicated group of replay threads, as shown below:

![](/assets/img/bigtxn-repl-4-en.png)

While a transaction runs on the primary, the binlog events it produces are first buffered in the Binlog Cache. If the transaction is large (the Binlog Cache exceeds a threshold), the primary's Dump thread reads the Binlog Cache temporary file and sends the events straight to the replica. The replica writes them into a dedicated `Brr Cache` (not the relay log file), where a new group of `Brr Worker` threads applies them in real time.

For DDL, binlog events are produced later than for a large transaction — a DDL writes its `Query_log_event` into the Binlog Cache only during the commit phase. BRR therefore handles DDL specially: once the primary starts executing the DDL, it builds the `Query_log_event` directly and puts it in an in-memory buffer, `ddl_query_buffer`; the Dump thread reads events from this buffer and sends them to the replica, where a Brr Worker again executes the DDL in real time.

As a result, replica execution of DDL and large transactions shifts from *run only after the primary finishes* to *run on the primary and replica in parallel*, leaving only network transmission and commit as the residual lag — typically on the order of tens of milliseconds.

Below we look at how BRR is implemented, from both the primary and the replica side.

### Overall BRR Architecture

![](/assets/img/bigtxn-repl-5-en.jpg)

#### Primary Side

When a large transaction or DDL needs realtime replication, a `Brr_trx` is created and registered with the `Brr_trx_manager`.

`Brr_binlog_sender` is an extension of the Dump thread; it reads events from a `Brr_trx` and pushes them to the replica. Originally the Dump thread did just one thing: read events from the binlog file and send them to the replica. BRR gives it one more job — poll each active `Brr_trx`, read binlog events from its Binlog Cache temporary file or from `ddl_query_buffer`, and send them to the replica.

Realtime transmission reuses the existing Dump channel. To tell BRR traffic apart from ordinary traffic, BRR borrows an idea from Semisync and attaches an extra `BRR Header` to each event; the header identifies whether an event is BRR or ordinary replication traffic. And to keep BRR events from choking the ordinary binlog-event channel, BRR applies flow control.

![](/assets/img/bigtxn-repl-6-en.png)

#### Replica Side

Using the `BRR Header`, the replica's IO thread splits events into two kinds: BRR events go into the `Brr_cache`, while normal events take the original path into the relay log.

`Brr_cache` is the replica-side storage for a BRR transaction; each BRR transaction has one `Brr_cache`. When the IO thread receives a BRR event, it uses the `brr_index` in the header to locate the matching `Brr_cache` (if it's the first event, it creates a new `Brr_cache` and wakes a Brr Worker), writes the event into the `Brr_cache` temporary file, and updates the readable position.

`Brr_rpl_info` manages these BRR transactions.

The `BRR Worker` threads are dedicated to applying BRR transactions. When idle, a Brr Worker picks a `Brr_cache` that hasn't started being applied and makes itself its owner. From then on it is bound to that `Brr_cache`, looping to read and replay binlog events until it sees a `Gtid_log_event` (the primary has committed) or receives a `BRR_ROLLBACK_EVENT` (the primary rolled back).

### The gtid_executed Snapshot

The `uncommitted` BRR transactions from the primary run in parallel on the replica alongside `already-committed` transactions. If a BRR transaction depends on an already-committed one, its binlog events must not start until that dependency has finished replaying on the replica; Otherwise you get escalating failures: a deadlock, then a broken replication channel, and in the worst case data inconsistency between primary and replica. Take this example:

```sql
INSERT INTO t1(pk, c2) VALUES(pk1, 1);
UPDATE t1 SET c2 = 2;  -- large transaction
```

The `UPDATE` is the large transaction, and it must not begin until the `INSERT` has finished replaying. If the `UPDATE` runs first, it fails when updating the `pk1` row because that row doesn't exist yet.

BRR uses a `gtid_executed snapshot` to enforce these ordering dependencies. When a DDL or large transaction starts on the primary, the primary's current `gtid_executed` captures every preceding transaction it saw. Once the replica's `gtid_executed` has caught up to that value (that is, is a superset of it), all the transactions this one depends on have been replayed on the replica, and it is safe to start applying it.

To do this, BRR adds a new event type, `Brr_gtid_executed_log_event`, whose body holds a `gtid_executed` set. At specific moments the primary takes a gtid_executed snapshot and writes it to the BRR channel; when a replica Brr Worker reads the snapshot, it waits for all the GTIDs in it to finish before continuing.

### Realtime Replication of Large Transactions

![](/assets/img/bigtxn-repl-7-en.png)

#### Creating and Updating a Brr_trx

When a transaction runs on the primary, its binlog events go first into the Binlog Cache (an in-memory buffer backed by a temporary file). In MySQL, once the Binlog Cache fills its in-memory buffer, it spills to the temporary file.

BRR hooks in here: after each batch of events is written to the Binlog Cache, it checks the temporary file's size. Once the file exceeds a certain size, BRR creates a `Brr_trx`, records the temporary file name and the current readable position, and registers it with `Brr_trx_manager`. From then on, every append to the Binlog Cache updates the `Brr_trx`'s `end_position` and wakes the Dump thread to send those events to the replica.

#### Transmitting Binlog Events

Before sending each batch of binlog events, the Dump thread emits a `Brr_gtid_executed_log_event` as that batch's dependency snapshot, then sends the batch itself.

#### Committing the Transaction

For a large transaction, the binlog events sit in the `Brr_cache` temporary file — not yet relay log — until the Brr Worker reads the `Gtid_log_event`. When the primary finally commits, it sends the `Gtid_log_event` over the BRR channel, and the IO thread does two things:

1. Renames the `Brr_cache` temporary file into a relay log file. Based on the GTID, the primary's Dump thread then skips sending this transaction, so its events aren't shipped again as an ordinary transaction.
2. Notifies the Brr Worker to read the `Gtid_log_event` and `Xid_log_event` and complete the commit.

#### Rolling Back the Transaction

The rollback path is straightforward: when the primary rolls back, it sends a `BRR_ROLLBACK_EVENT` over the BRR channel; on receiving it, the replica's IO thread sends a `KILL_QUERY` signal to the corresponding Brr Worker. The Brr Worker detects `KILL_QUERY`, rolls back the current transaction, cleans up, and moves on to the next `Brr_cache`.

Note that after being killed, a Brr Worker neither exits nor propagates the error to the SQL thread — unlike an ordinary Worker, which must halt all replication on an error. The reason: for an ordinary Worker the transaction has already committed on the primary, so if the replica gives up, the two diverge. A Brr Worker's transaction, by contrast, runs concurrently with the primary, so a primary rollback is the normal path and the replica must roll back as well.

### Realtime Application of DDL

![](/assets/img/bigtxn-repl-8-en.png)

#### Creating a Brr_trx

For large transactions, we decide whether a transaction is "large" by the total size of its binlog events in the Binlog Cache. DDL is trickier: some DDLs only touch metadata and finish almost instantly, while for DDLs that touch data the run time depends on how much data is involved and is hard to estimate accurately. So instead of predicting a DDL's run time up front, we decide whether to realtime-replicate it *by whether its execution exceeds a timeout.*

Every DDL creates a `Brr_trx`, but that `Brr_trx` isn't sent to the replica right away. A DDL's `Brr_trx` has a threshold — 1000 ms by default — and only when the DDL's run time exceeds it does the Dump thread start sending the `Brr_trx`. If a DDL finishes quickly, within one second, its `Brr_trx` is silently discarded and the DDL ships to the replica over the ordinary binlog channel, exactly as if BRR were off.

A DDL's `Brr_trx` is created during the DDL's Prepare phase — that is, *after the DDL has acquired the MDL X lock* — because only with the X lock does the DDL have permission to operate on the table. Any conflicting operations have either already committed or must wait until the DDL releases the X lock or finishes.

### Two gtid_executed Snapshots

An Online DDL runs in three phases: `Prepare`, `Execute`, and `Commit`. After Prepare, the MDL `X lock` is downgraded to an `S lock`, so during Execut, DML and DDL can run in parallel. During Commit, the `S lock` is upgraded back to an `X lock`; regaining the X lock means all those parallel DMLs have already committed. The replica must honor the same rule: those committed DMLs have to finish replaying before the replica can enter the Commit phase.

So realtime replication of an Online DDL has two points on the replica that must be synchronized: one before entering Prepare, and one before entering Commit. Correspondingly, the primary takes two `gtid_executed` snapshots — one after the DDL enters Prepare, and one after it enters Commit.

### Shipping Binlog Events Twice

In the large-transaction section we saw that a large transaction is transmitted to the replica via BRR, and the copy in the binlog file is not shipped again. DDL is different: it ships twice — *once over BRR, and again as the binlog events in the binlog file.*

A DDL's `Query_log_event` is tiny, so shipping it twice costs almost nothing. Shipping it only once would force us into the large-transaction rename logic (renaming the `Brr_cache` temporary file into relay log), with all its edge cases. For DDL, simply shipping it twice and discarding the `Brr_cache` afterward is the simplest approach.

As for ordering, the Dump thread guarantees BRR events ship before ordinary events. That way the Brr Worker is sure to get the DDL first and start executing it; by the time the ordinary events reach the relay log, the Brr Worker is already applying the DDL.

When an ordinary Worker reads the DDL from the relay log, it checks whether the GTID is in `owned_gtids`. If it is (a Brr Worker is executing it), the ordinary Worker waits; once the Brr Worker commits, the GTID is added into `gtid_executed`. The ordinary Worker wakes and finds the GTID already in `gtid_executed`, so it skips the whole DDL.

If the Brr Worker rolled the DDL back, the GTID is removed from `owned_gtids` and never added to `gtid_executed`. The ordinary Worker then wakes and sees the transaction wasn't executed. It runs the DDL normally — *the fallback path, equivalent to running with BRR off.*

## Conclusion

AliSQL's Binlog Realtime Replication tackles the thorniest lag in MySQL binlog replication — lag from large transactions and DDL — by executing on the primary and replica in parallel. On top of that, we've made optimizations for the writeset mechanism, for massively concurrent workloads, and for the medium-sized transactions that batch jobs produce. Together, these have eliminated 95% of the replication lag in our production environment.
