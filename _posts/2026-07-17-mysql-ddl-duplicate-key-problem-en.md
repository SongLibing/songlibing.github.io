---
title: 'MySQL Online DDL "Duplicate Key" Optimization'
date: 2026-07-17 16:30:00 +0800
categories: [Database, MySQL]
tags: [MySQL, DDL, Online DDL, InnoDB]
toc: true
lang: en
hidden: true
published: false
---

> This article is also available in Chinese: [中文版](/posts/mysql-ddl-duplicate-key-problem/). Browse [all English articles](/english/).
{: .prompt-tip }

When you rebuild a table with MySQL's Online DDL, the operation can fail partway through with a `Duplicate Entry` error:

```sql
mysql> alter table tt add c3 int, algorithm=inplace;
ERROR 1062 (23000): Duplicate entry '1' for key 'tt.uk_c2'
```

This is a long-standing, well-known problem — it has been around ever since Online DDL landed in MySQL 5.6, and it surfaces across several bug reports:

- [BUG#76895](https://bugs.mysql.com/bug.php?id=76895) — Adding new column OR Drop column causes duplicate PK error
- [BUG#77572](https://bugs.mysql.com/bug.php?id=77572) — The bogus duplicate key error in online ddl with incorrect key name
- [BUG#98600](https://bugs.mysql.com/bug.php?id=98600) — Optimize table fails with duplicate entry on UNIQUE KEY
- [BUG#104626](https://bugs.mysql.com/bug.php?id=104626) — Remove failure of Online ALTER because concurrent Duplicate entry

[BUG#76895](https://bugs.mysql.com/bug.php?id=76895) was the first to report it. It called the problem a `Duplicate PRIMARY KEY` error, but that label was itself a bug: a separate 5.6 issue, [BUG#77572](https://bugs.mysql.com/bug.php?id=77572), printed `PRIMARY KEY` in the message when it should have said `UNIQUE KEY`. Once BUG#77572 was fixed (in 5.6.28 and 5.7.10), the message correctly read `Duplicate UNIQUE KEY` — and someone promptly filed [BUG#98600](https://bugs.mysql.com/bug.php?id=98600) for what was really the same underlying problem.

MySQL closed BUG#76895 as `Not a Bug`, treating the error as an unavoidable side effect of how Online DDL works. But "not a bug" is cold comfort to a DBA: a failed DDL derails a carefully planned change and often pushes it to the next maintenance window. That frustration is what eventually produced [BUG#104626](https://bugs.mysql.com/bug.php?id=104626) — this time filed as a feature request asking for a real fix.

So what actually causes this error, why does MySQL consider it working as intended, and can you do anything about it? Answering all three means looking at how Online DDL works under the hood.

## How Online DDL Works

To keep a table writable while it's being altered, Online DDL splits the job in two: a *full copy* of the rows that already exist, and an *incremental replay* of everything that changes while that copy runs. It copies the existing rows into the new table (when you're adding an index there's no new table — the rows go straight into the new index). The copy takes time, and DML keeps flowing throughout, so every change made along the way is captured to be applied afterward. Two things make this work:

- A precise cut-off in time, so the full copy takes only the rows that existed before it and nothing after.
- An incremental log that captures every change made after the cut-off, so replaying it brings the new table fully up to date.

### Telling Full Data from Incremental Data

The *full data* lives in the *clustered B+Tree*, and Online DDL reads it by scanning that tree. Since DML runs during the scan, the tree also holds changes made after the cut-off — so the two have to be told apart. Online DDL does this with InnoDB's `MVCC` (multi-version concurrency control).

MVCC gives a transaction a consistent snapshot of the data, immune to whatever other transactions are doing. The mechanism behind it is the `read view`: when one is created, it records the IDs of all currently active transactions, then uses that list to decide which version of each row the transaction may see. If a row is modified and committed after the read view was created, the view can still reach the earlier version through the undo log, even once the new version is sitting in the clustered index.

That's exactly what Online DDL leans on to fix the boundary of the full copy: *it opens a read view at one precise moment, and the copy reads only what that view can see. Whatever concurrent DML changes afterward — no matter when it commits — stays invisible to the copy.*

## The Three Phases of Online DDL

InnoDB runs an Online DDL in three phases, each holding a different level of `Metadata Lock (MDL)` to control precisely what concurrent DML is allowed to do.

### Prepare

Prepare runs under `MDL_EXCLUSIVE`, which blocks every read and write on the table. That brief "writes-stopped" window is the cut-off between full and incremental data. During it, InnoDB:

1. Creates a consistent read view via `trx_assign_read_view()`.
2. Sets up the row log (the incremental log) on the target index and marks the index `ONLINE_INDEX_CREATION`. From then on, each insert, update, or delete on that index's B+tree checks the flag to decide whether it also needs to go into the row log.

Both steps happen under `MDL_EXCLUSIVE`, and that exclusivity buys two guarantees:

1. The read view and the row log line up perfectly, so together they cover the existing rows and every later change with no gap.
2. It *shuts out any in-flight transaction*. Without the exclusive lock, a transaction could have modified the table but not yet committed at the moment of the cut-off. The row log isn't installed yet, so its change goes unlogged; and because it's still uncommitted, the read view marks it active and the full copy can't see it either. The change would fall straight through the crack — in neither the copy nor the log — and be lost. `MDL_EXCLUSIVE` guarantees no such transaction is running, so the read view and row log meet at a clean seam: *everything committed before the seam belongs to the full copy; everything after it is caught by the row log.*

Once Prepare finishes, the lock is downgraded from `MDL_EXCLUSIVE` to the shared `MDL_SHARED_UPGRADABLE`.

### Execute

During Execute, Online DDL holds `MDL_SHARED_UPGRADABLE`, so *concurrent DML runs normally*. This is where the full copy happens, along with the first pass of incremental replay — and it's by far the longest phase, the one throughout which the table stays writable. Every DML on the table is recorded in the row log.

### Commit

For Commit, the DDL upgrades the lock from `MDL_SHARED_UPGRADABLE` back to `MDL_EXCLUSIVE`, *blocking all reads and writes again*. Under that lock it replays whatever row log is left and commits the DDL transaction. Because Execute already drained most of the log, there's little left to do here, so the exclusive lock is held only briefly.

The exclusive lock in this phase also guarantees two things:

- Every DML from the Execute phase that touched the table has committed.
- No new DML can start, so no new row log is produced.

The takeaway: Online DDL isn't *fully* online — it blocks DML during Prepare and Commit. And because a metadata lock is held for the life of a transaction, a long-running transaction that's in progress when the DDL reaches for `MDL_EXCLUSIVE` will stall it — and while the DDL waits, it blocks other DML too, leaving the table effectively unavailable.

## The Full Copy

The full copy runs during Execute, under `MDL_SHARED_UPGRADABLE`, with concurrent DML allowed.

It uses `row_merge_read_clustered_index()` to walk the old table's clustered index in ascending PRIMARY KEY order, checking each row's visibility with MVCC as it goes. Only rows visible to the read view are copied into the new table; `delete-marked` rows are skipped.

Thanks to MVCC, even when concurrent DML rewrites a row mid-scan, the copy still sees the version as of the read view — reconstructed from the undo log if need be. What it copies is always one consistent snapshot, untouched by anything happening alongside it.

## Recording the Incremental Log

The row log records page-level changes to a B+tree, and it behaves differently in the two kinds of DDL:

- *Rebuilding the table*: it logs only changes to the clustered B+tree — `INSERT`, `UPDATE`, and `DELETE` — and on replay each entry is applied to *every* index.
- *Adding an index*: it logs only changes to the new index's B+tree — `INSERT` and `DELETE`, never `UPDATE`. Since the index is still being built, these operations are *only* written to the row log and never actually applied to the B+tree — the opposite of a table rebuild, where every existing index on the old table is maintained in real time.

One subtlety is worth pinning down: *when* an entry gets logged. Because the row log tracks B+tree changes, an entry is written only after the B+tree operation succeeds. As the figure shows, *for a table rebuild the entry is logged after the row is inserted into the clustered B+tree; for an added index it's logged at the point the row would go into the secondary index's B+tree* — except that, for an added index, nothing is actually inserted, and only the log entry is written.

![](/assets/img/ddl-dupkey-1.webp)

### What Happens on a DML Rollback

A DML touches the clustered B+tree first, then the secondary indexes. So what happens if the secondary-index step fails?

- The row-log entry stays put — it isn't cleaned up. It records one B+tree operation, and that operation really did happen.
- The statement then rolls back. Rolling back replays the change in reverse against the B+tree (driven by the undo log), and that reverse operation is logged too. So a failed INSERT leaves *two* row-log entries — a `ROW_T_INSERT` followed by a `ROW_T_DELETE` (below). Applied in order, the pair cancels out, exactly as a rollback should.

![](/assets/img/ddl-dupkey-2.webp)

A failed statement is just one way to get here; every rollback works the same way, including an explicit `ROLLBACK` from the user.

## Replaying the Incremental Log

The row log is replayed twice:

- *The first pass* runs in Execute, right after the full copy. The lock is still `MDL_SHARED_UPGRADABLE`, so DML keeps running and keeps adding to the log. The point of this pass is to *burn down as much of the log as possible now, so the exclusive lock in Commit is held for as short a time as possible*.
- *The second pass* runs in Commit, under `MDL_EXCLUSIVE`, with no new log being produced. It mops up the small remainder, and once it's done the new table matches the old one exactly.

Both passes read the log in write order, one entry at a time. For a table rebuild, each entry is applied to every index on the new table; for an added index, only to the index being built.

## What Actually Causes the Duplicate Entry

Recall the rollback case: even when an INSERT hits a `Duplicate Entry` on a unique index, it still leaves two row-log entries behind (below).

![](/assets/img/ddl-dupkey-3.webp)

Replaying the first entry, `ROW_T_INSERT`, does exactly what the original INSERT did: put a row in the clustered B+tree, then add an entry to each index. And when it reaches the unique index, it runs straight into the very same `Duplicate Entry` — this time during replay, which is what takes the whole DDL down.

### Reproducing It

```sql
CREATE TABLE t1 (
  c1 INT AUTO_INCREMENT PRIMARY KEY,
  c2 INT,
  UNIQUE KEY (c2)
) ENGINE=InnoDB;
```

Run these statements in three separate sessions:

```sql
# Session 1
BEGIN;
INSERT INTO t1 VALUES(NULL, 1);

# Session 2
ALTER TABLE t1 ENGINE = InnoDB;

# Session 3
INSERT INTO t1 VALUES(NULL, 1);
```

Now commit Session 1, and both Session 2's ALTER and Session 3's INSERT fail with `Duplicate Entry`:

```sql
# Session 2
mysql> ALTER TABLE t1 ENGINE = InnoDB;
ERROR 1062 (23000): Duplicate entry '1' for key 't1.c2'

# Session 3
mysql> INSERT INTO t1 VALUES(2, 1);
ERROR 1062 (23000): Duplicate entry '1' for key 't1.c2'
```

The setup relies on the MDL behavior described above:

- Session 1's INSERT takes an `MDL_SHARED_WRITE` lock on `t1` and holds it until it commits.
- Session 2's ALTER needs `MDL_EXCLUSIVE` for its prepare phase, so it blocks behind Session 1.
- Session 3's INSERT wants `MDL_SHARED_WRITE`, but now blocks behind Session 2.

You can watch this play out in `performance_schema.metadata_locks` — the first row below is Session 1's lock.

![](/assets/img/ddl-dupkey-4.webp)

When Session 1 commits, Session 2 grabs `MDL_EXCLUSIVE`, finishes Prepare, and downgrades to `MDL_SHARED_UPGRADABLE`. That lock doesn't conflict with `MDL_SHARED_WRITE`, so Session 3 finally runs — and since `c2 = 1` already exists, it fails with a `Duplicate key` error. That failure leaves a row-log entry, and replaying it is what makes Session 2's ALTER fail in turn.

![](/assets/img/ddl-dupkey-5.webp)

### How to Avoid It

Once the mechanism is clear, I'd call this a bug — which is why we fixed it in AliSQL. On community MySQL, the one lever you have is to keep DML from hitting `Duplicate Entry` while a DDL is running.

The classic trigger is a table with an auto-increment primary key that has also gained a unique index. The app inserts a row, lets MySQL assign the auto-increment key, and — if the first INSERT is slow — retries the same statement from another session. That's perfectly sensible on its own: the unique index guarantees only one of the two can win. But that single losing INSERT is all it takes to fail the DDL. So while a DDL is in flight, widening the retry timeout is a practical way to sidestep the problem.

## The Fix in AliSQL

AliSQL takes the straightforward route: *when replay hits a `Duplicate Entry`, ignore it.*

### Real Duplicates vs. False Ones

There's a catch — not every `Duplicate Entry` is safe to ignore.

Online DDL has a genuine *real duplicate* case: *the DDL introduces a new uniqueness constraint (a new or changed primary key, or a new UNIQUE index) while the existing data already contains duplicates.* That new constraint isn't enforced during the DDL, so concurrent DML can keep adding duplicates — and here the DDL *must* fail. Ignoring it would silently break the guarantee the user asked for.

But if the `Duplicate Entry` lands on a unique index that already existed on the table, it's always a *false duplicate* and safe to skip. That's the common case, and the one AliSQL optimizes.

### Cleaning Up the Rollback Entry

As we saw, a failed DML leaves two row-log entries. For an INSERT:

```
<ROW_T_INSERT, pk1, ...>
<ROW_T_DELETE, pk1>
```

On replay, the insert into the unique index fails and we ignore it — but the insert into the primary-key B+tree did succeed. When the second entry (`ROW_T_DELETE`) is replayed, deleting from the primary key is fine; deleting from the unique index is not, because nothing was ever inserted there, so the record can't be found and the replay fails.

So whenever we skip a `Duplicate Entry` on a pre-existing unique index, we have to remember it and skip the matching operation when the rollback's row log comes through later, like this:

![](/assets/img/ddl-dupkey-6.webp)

UPDATE is trickier. When an UPDATE changes a secondary-index column, the secondary index sees two operations:

1. Delete the old entry.
2. Insert the new one.

The replay fails at step 2, so on the rollback we can't just skip the whole index — we skip step 1 but still perform step 2:

![](/assets/img/ddl-dupkey-7.webp)

With this handling, AliSQL clears away the `Duplicate Entry` failures that Online DDL should never have raised in the first place.

## Wrapping Up

Online DDL can fail with a `Duplicate Entry`, and because it turns on timing, it's hard to rule out entirely — and each failure can knock a planned change off course. The chain is short: a concurrent DML hits a `Duplicate Entry`, that error rides along on Online DDL's row log, and replaying the log fails the DDL. AliSQL breaks the chain by ignoring `Duplicate Entry` errors that occur on an already-existing unique index, so Online DDL no longer dies on an error that was never really its own.
