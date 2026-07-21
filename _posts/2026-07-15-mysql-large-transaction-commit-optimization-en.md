---
title: "Commit Optimization for Large MySQL Transactions"
date: 2026-07-15 10:00:00 +0800
categories: [Database, MySQL]
tags: [MySQL, Binlog, Large Transaction, Commit Optimization]
toc: true
lang: en
hidden: true
---

> This article is also available in Chinese: [中文版](/posts/mysql-large-transaction-commit-optimization/). Browse [all English articles](/english/).
{: .prompt-tip }

If you use and operate MySQL, you've surely run into a strange slow query like this:

![](/assets/img/bigtxn-commit-1.webp)

- An `INSERT` that's normally instant took `1.3s`, and the slow-query log shows no long lock wait.
- Every statement in a multi-statement transaction had already finished, yet the `COMMIT` alone took `1.3s`.

When this happens, the most likely cause is a large transaction committing. Below is a simulated test: we used sysbench to simulate a normal workload, then ran a large UPDATE in the background every 5 seconds. You can see the large UPDATE severely hurts performance.

![](/assets/img/bigtxn-commit-2.webp)

## Root Cause

![](/assets/img/bigtxn-commit-3.webp)

The figure above shows the execution of two transactions:

- A transaction runs in two phases: an execution phase and a commit phase.
- During execution, when a statement updates data it generates binlog events. These are stored in the Binlog Cache, which has two parts: an in-memory buffer and a temporary file. When the buffer fills up, the events are written to the temporary file.
- At commit, all the binlog events in the Binlog Cache are copied into the binlog file.
- *Writing binlog events to the binlog file must be serialized — one transaction can't do it until the previous one has finished.* So while `Trx_n` is writing to the binlog file, `Trx_m` has to wait.
- In the figure, `Trx_n` is a large transaction that produced a lot of binlog events. *The time to copy binlog events into the binlog file is linear in the size of the events the transaction produced — the more events, the longer the copy takes.*
- `Trx_m` is a small transaction. Even though its execution phase finished quickly, at commit it runs into the large transaction `Trx_n` committing, so it must wait for `Trx_n` to finish copying its binlog events before it can proceed. `Trx_m` spends most of its commit phase waiting for `Trx_n` to write the binlog file — and that's why the small transaction becomes slow.

## How Serious the Problem Is

As our simulated test shows, committing a large transaction has a major impact on workload stability. In real-world scenarios it can be far worse, and it's common.

- A GB-scale transaction can make the instance unwritable for a long time. Since storage IO bandwidth is fixed, the time to write a large transaction's binlog depends on the transaction's size. The largest transaction we've seen in production produced `104 GB` of binlog events.
- A GB-scale transaction can push IO throughput up and slow it down, or even saturate IO, which also slows queries.
- A few-hundred-MB transaction won't cause a long outage, but it can still add hundreds of milliseconds to application DML. For latency-sensitive workloads, even that may be unacceptable.
- On top of this, all of the above can raise the number of active connections. If those active connections aren't cleared in time, CPU spikes, and it can turn into a vicious cycle — eventually an avalanche and a much bigger problem.

## Optimizing How Large Transactions Write the Binlog

In AliSQL we optimized how a large transaction writes the binlog, completely eliminating the stability impact of large-transaction commit. RDS 5.7 and RDS 8.0 both enable this optimization by default. Last year we contributed it to MariaDB, and the feature shipped in MariaDB 11.7[^1].

### The Approach

Here is the implementation in MariaDB 11.7. MySQL and MariaDB have diverged quite a bit in code, but the underlying logic — and therefore the approach — is the same.

![](/assets/img/bigtxn-commit-4.webp)

The idea is simple and clean: since the Binlog Cache has already written the binlog events to a file, we just *rename that file directly into a binlog file.* This avoids copying the binlog events, so there is no extra IO. And a rename takes constant time regardless of the Binlog Cache's size, which fully solves the large-transaction problem. Let's look at the implementation.

### The #binlog_cache_files Directory

The Binlog Cache's file is a system temporary file, which can't be renamed into a regular file directly. So we create a directory, `#binlog_cache_files`, in the binlog directory; the file the Binlog Cache creates then becomes a regular file in this directory instead of a system temp file.

```shell
$ls var/mysqld.1/data/#binlog_cache_files
ML_140413554102520
```

### Reserving Head Space

The Binlog Cache file contains only the transaction's binlog events. To turn it into a binlog file, we need to reserve some space for the binlog header events, such as the `Format_description_event`.

![](/assets/img/bigtxn-commit-5.webp)

The reserved space is 4 KB-aligned, so at least 4 KB is reserved, which is enough in most cases. But in some situations the `Gtid_list_log_event` (similar to MySQL's `Previous_gtids_event`, recording the GTID set generated before this binlog) can be very large. To keep the feature usable in that case, when generating a new binlog file we adjust the reserved space based on how much the header events actually occupy; the Binlog Cache file's reserved space is then adjusted when the next transaction begins. The binlog header events usually take less than 4 KB, so after writing them some space may be left over. How do we handle the leftover? Thanks to MariaDB's mechanism of padding a `Gtid_log_event` with trailing zeros, the leftover space is absorbed into the corresponding `Gtid_log_event`. After the Binlog Cache file is turned into a binlog file, its structure looks like this:

![](/assets/img/bigtxn-commit-6.webp)

### The Rename Process

![](/assets/img/bigtxn-commit-7.webp)

The rename works roughly as follows:

- Persist the Binlog Cache file. At this point the rename hasn't started, so it doesn't block other transactions from committing.
- Perform a rotate: close the current binlog file and create a new one.
- Copy the new file's header into the head of the Binlog Cache.
- Generate the `Gtid_log_event`.
- Delete the newly generated binlog file, and rename the Binlog Cache file into the new binlog file.

## Results

Again we used sysbench to simulate a workload, then ran a large UPDATE in the background every 5 seconds, each producing 512 MB of binlog events. The results:

![](/assets/img/bigtxn-commit-8.webp)

With the large-transaction commit optimization, sysbench's TPS is fairly steady, with no violent swings. There's still a small dip every 5 seconds, but that comes from the large UPDATE itself using some CPU, not from transaction commit.

We also simulated the DML latency caused by transactions of different sizes. The results:

![](/assets/img/bigtxn-commit-9.webp)

- Without the optimization, once a large transaction exceeds 64 MB, sysbench's max latency starts to climb noticeably, and rises rapidly as the transaction grows.
- With the optimization on, sysbench's max latency stays stable no matter how large the transaction, holding at normal workload levels. At 1024 GB, one extra binlog rotate adds a slight bump in latency.

## Conclusion

In MySQL's binlog replication architecture, large transactions are a classic trigger for trouble, causing stability and replication-lag problems. By renaming the Binlog Cache's temporary file directly into a binlog file, we avoid copying binlog events and eliminate the extra IO, keeping large-transaction commit fast and stable — and fully resolving the various stability problems that large-transaction commit causes.

[^1]: MariaDB 11.7 — [Binlog Commit Optimization for Large Transactions](https://mariadb.com/resources/blog/binlog-commit-optimization-for-large-transaction/)
