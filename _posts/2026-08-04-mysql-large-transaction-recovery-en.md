---
title: "Recovery Optimization for Large MySQL Transactions"
date: 2026-08-04 10:00:00 +0800
categories: [Database, MySQL]
tags: [MySQL, Large Transaction, Recovery, Binlog, InnoDB]
toc: true
lang: en
hidden: true
published: false # TODO: remove this line to publish
---

> This article is also available in Chinese: [中文版](/posts/mysql-large-transaction-recovery/). Browse [all English articles](/english/).
{: .prompt-tip }

Have you ever seen a `mysqld` process that stays down for a very long time after being started? When that happens, `perf top` will tell you what the process is actually busy with. If the output looks like the screenshot below — the MySQL `main thread (the one starting from mysqld_main)` spending almost all of its time rolling back a transaction — you are most likely hitting a large-transaction rollback.

![](/assets/img/bigtxn-recovery-1.webp)

The most common way to get into this state is a large transaction that fills up the disk while writing its binlog, crashing the instance. The largest binlog file I have run into was over `114GB`. Since the Binlog Cache's temporary file is only cleaned up after the binlog has been written, that single transaction occupied `228GB` of space in total. The MySQL parameter `binlog_error_action` controls what happens when writing the binlog file fails. The default is `ABORT_SERVER`, which shuts the process down. You can also set it to `IGNORE_ERROR`, which closes the binlog file on a write failure so that subsequent transactions produce no binlog at all. That obviously leaves the primary and the replica inconsistent, so don't use it unless you have no other choice.

## Root Cause

Why does the main thread have to roll transactions back during startup? It comes from the binlog crash-safe mechanism; here is a brief summary. DML inside a transaction produces binlog events, and on commit those events are written to the binlog file and persisted. To keep the data and the binlog consistent across a crash and restart, MySQL uses a crash-safe mechanism that applies two-phase commit (2PC) to ordinary transactions — also known as internal XA.

![](/assets/img/bigtxn-recovery-2.webp)

As the figure shows, under internal XA a commit happens in three steps:

1. The storage engine `prepares` the transaction. The transaction state changes from `ACTIVE` to `PREPARED`, and both the state and the `XID` are persisted into the redo log.
2. The transaction generates an `Xid_event`, which is written to the binlog file together with the DML binlog events and persisted.
3. The transaction commits.

When the server crashes, a transaction can be in one of the following states:

- `Active`: under two-phase commit, such a transaction was never written to the binlog.
- `Prepared but not written to the binlog (or only partially written)`: the transaction is in the Prepared state, but its XID does not appear in the binlog files.
- `Prepared and written to the binlog`: the transaction is in the Prepared state and its XID appears in the binlog files.
- `Committed`: the transaction was written to the binlog and committed.

For a `Committed` transaction, the design already guarantees that its binlog events made it into the binlog file, so the binlog and the data are consistent and nothing needs to be done at startup. For an `Active` transaction, the binlog events definitely never reached the binlog file, and `InnoDB has a background rollback thread that rolls it back automatically`. A `Prepared` transaction has to be handled based on the XID information in the last binlog file: if its `XID` is present in the binlog, the transaction must be committed to keep the binlog and the data consistent; otherwise it must be rolled back.

![](/assets/img/bigtxn-recovery-3.webp)

Processing `Prepared` transactions is called `Binlog Recovery`, and it `must finish before MySQL starts serving users`. Committing is usually fast, but rolling back typically costs about as much time as executing the transaction did. If a transaction took an hour to execute, its rollback may well take another hour — and MySQL is unavailable for all of it.

Why must every transaction be resolved before the server starts serving? It has to do with how the `XID` is built. An XID is the `MySQL` prefix plus the `query_id`, and `query_id` is a global counter that restarts from 1 after a reboot. If Prepared transactions from before the restart were left neither committed nor rolled back, two Prepared transactions could end up with the same XID, and recovery would have no way to tell which one to commit and which one to roll back.

## Rolling Back Prepared Transactions Asynchronously

In AliSQL we designed an asynchronous rollback mechanism to solve this problem.

![](/assets/img/bigtxn-recovery-4.webp)

As the figure shows, the rollback of a Prepared transaction is split into two parts:

1. The main thread sets the transaction state to `Active` and persists that state.
2. InnoDB's background rollback thread asynchronously rolls back all of the transaction's changes.

Binlog Recovery can start serving traffic as soon as the first part is done. Since that step is very fast, Binlog Recovery completes in a very short time.

After a crash and restart, `Active` transactions are rolled back directly by InnoDB's background thread, with no need for the `XID` to drive the decision. So during recovery, simply changing the state of the transactions to be rolled back from `Prepared` to `Active` avoids the problem of two Prepared transactions sharing an `XID`. The key point is persisting the `Active` state, so that the transaction is still `Active` if the server crashes again — InnoDB will then roll it back automatically.

Community InnoDB already rolls back a Prepared transaction by first setting it to `Active` and then undoing its changes from the undo records. That `Active` state is written to the redo log; it is simply not flushed at that moment. However, InnoDB persists the redo log once per second by default, so the `Active` state gets persisted very soon after the change. This means that when a large-transaction rollback keeps an instance from starting, `even on community MySQL you can just force a restart of the mysqld process, and the large transaction turns into a background rollback that no longer blocks startup`.

The source code for this feature was contributed to MariaDB and has been merged into MariaDB 11.7; see [MDEV-33853](https://jira.mariadb.org/browse/MDEV-33853) for details.

## Conclusion

With asynchronous rollback, the `Binlog Recovery` phase only has to set Prepared transactions to `Active`, while the genuinely expensive rollback work is carried out asynchronously by InnoDB's background rollback thread. This optimization shrinks a startup that used to take tens of minutes — or even hours — down to seconds.
