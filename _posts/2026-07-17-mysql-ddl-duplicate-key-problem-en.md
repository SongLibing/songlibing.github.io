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

This is a long-standing, well-known problem — it has existed ever since Online DDL was introduced in MySQL 5.6, and it appears across several bug reports:

- [BUG#76895](https://bugs.mysql.com/bug.php?id=76895) — Adding new column OR Drop column causes duplicate PK error
- [BUG#77572](https://bugs.mysql.com/bug.php?id=77572) — The bogus duplicate key error in online ddl with incorrect key name
- [BUG#98600](https://bugs.mysql.com/bug.php?id=98600) — Optimize table fails with duplicate entry on UNIQUE KEY
- [BUG#104626](https://bugs.mysql.com/bug.php?id=104626) — Remove failure of Online ALTER because concurrent Duplicate entry

[BUG#76895](https://bugs.mysql.com/bug.php?id=76895) was the first to report it. It called the problem a `Duplicate PRIMARY KEY` error, but that label was itself a bug: a separate 5.6 issue, [BUG#77572](https://bugs.mysql.com/bug.php?id=77572), printed `PRIMARY KEY` in the message when it should have said `UNIQUE KEY`. Once BUG#77572 was fixed (in 5.6.28 and 5.7.10), the message correctly read `Duplicate UNIQUE KEY` — and [BUG#98600](https://bugs.mysql.com/bug.php?id=98600) was filed later for what was really the same underlying problem.

MySQL closed BUG#76895 as `Not a Bug`, treating the error as an unavoidable side effect of how Online DDL works. But "not a bug" is of little consolation to a DBA: a failed DDL derails a carefully planned change and often pushes it to the next maintenance window. That inconvenience is what eventually produced [BUG#104626](https://bugs.mysql.com/bug.php?id=104626) — this time filed as a feature request asking for a real fix.

So what actually causes this error, why does MySQL consider it working as intended, and can you do anything about it? Answering all three means looking at how Online DDL works internally.

## How Online DDL Works

To keep a table writable while it's being altered, Online DDL splits the job in two: a *full copy* of the rows that already exist, and an *incremental replay* of everything that changes while that copy runs. It copies the existing rows into the new table (when you're adding an index there's no new table — the rows go straight into the new index). The copy takes time, and DML keeps running throughout, so every change made along the way is captured to be applied afterward. Two things make this work:

- A precise cut-off in time, so the full copy takes only the rows that existed before it and nothing after.
- An incremental log that captures every change made after the cut-off, so replaying it brings the new table fully up to date.

### Distinguishing Full Data from Incremental Changes

The *full data* lives in the *clustered B+Tree*, and Online DDL reads it by scanning that tree. Since DML runs during the scan, the tree also holds changes made after the cut-off, so Online DDL needs a way to separate the original rows from those later changes. It does this with InnoDB's `MVCC` (multi-version concurrency control).

MVCC lets a transaction read from a consistent snapshot of the data — one that isn't affected by changes other transactions make while it runs. The mechanism behind it is the `read view`: when one is created, it records the IDs of all currently active transactions, then uses that list to decide which version of each row the transaction may see. If a row is modified and committed after the read view was created, the view can still reach the earlier version through the undo log, even once the new version is sitting in the clustered index.

That is exactly what Online DDL relies on to set the boundary of the full copy: *it opens a read view at one precise moment, and the copy reads only what that view can see. Whatever concurrent DML changes afterward — no matter when it commits — stays invisible to the copy.*

## The Three Phases of Online DDL

InnoDB runs an Online DDL in three phases, each holding a different level of `Metadata Lock (MDL)` to control precisely what concurrent DML is allowed to do.

### Prepare

Prepare runs under `MDL_EXCLUSIVE`, which blocks every read and write on the table. That brief "writes-stopped" window is the cut-off between full and incremental data. During it, InnoDB:

1. Creates a consistent read view via `trx_assign_read_view()`.
2. Sets up the row log (the incremental log) on the target index and marks the index `ONLINE_INDEX_CREATION`. From then on, each insert, update, or delete on that index's B+tree checks the flag to decide whether it also needs to go into the row log.

Both steps happen under `MDL_EXCLUSIVE`, which guarantees two things:

1. The read view and the row log line up perfectly, so together they cover the existing rows and every later change with no gap.
2. It *shuts out any in-flight transaction*. Without the exclusive lock, a transaction could have modified the table but not yet committed at the moment of the cut-off. The row log isn't installed yet, so its change goes unlogged; and because it's still uncommitted, the read view marks it active and the full copy can't see it either. The change would then be in neither the copy nor the log, and would be lost. `MDL_EXCLUSIVE` guarantees no such transaction is running, so the read view and row log meet at a clean boundary: *everything committed before it belongs to the full copy; everything after it is captured by the row log.*

Once Prepare finishes, the lock is downgraded from `MDL_EXCLUSIVE` to the shared `MDL_SHARED_UPGRADABLE`.

### Execute

During Execute, Online DDL holds `MDL_SHARED_UPGRADABLE`, so *concurrent DML runs normally*. This is where the full copy happens, along with the first pass of incremental replay — and it is by far the longest phase, the one throughout which the table stays writable. Every DML on the table is recorded in the row log.

### Commit

For Commit, the DDL upgrades the lock from `MDL_SHARED_UPGRADABLE` back to `MDL_EXCLUSIVE`, *blocking all reads and writes again*. Under that lock it replays whatever row log is left and commits the DDL transaction. Because Execute already replayed most of the log, there is little left to do here, so the exclusive lock is held only briefly.

The exclusive lock in this phase also guarantees two things:

- Every DML from the Execute phase that touched the table has committed.
- No new DML can start, so no new row log is produced.

In short, Online DDL is not *fully* online — it blocks DML during Prepare and Commit. And because a metadata lock is held for the life of a transaction, a long-running transaction that is in progress when the DDL tries to acquire `MDL_EXCLUSIVE` will block it — and while the DDL waits, it blocks other DML too, leaving the table effectively unavailable.

## The Full Copy

The full copy runs during Execute, under `MDL_SHARED_UPGRADABLE`, with concurrent DML allowed.

It uses `row_merge_read_clustered_index()` to scan the old table's clustered index in ascending PRIMARY KEY order, checking each row's visibility with MVCC as it goes. Only rows visible to the read view are copied into the new table; `delete-marked` rows are skipped.

Thanks to MVCC, even when concurrent DML rewrites a row mid-scan, the copy still sees the version as of the read view — reconstructed from the undo log if necessary. What it copies is always one consistent snapshot, unaffected by concurrent activity.

## Recording the Incremental Log

The row log records page-level changes to a B+tree, and it behaves differently in the two kinds of DDL:

- *Rebuilding the table*: it logs only changes to the clustered B+tree — `INSERT`, `UPDATE`, and `DELETE` — and on replay each entry is applied to *every* index.
- *Adding an index*: it logs only changes to the new index's B+tree — `INSERT` and `DELETE`, never `UPDATE`. Since the index is still being built, these operations are *only* written to the row log and never actually applied to the B+tree — the opposite of a table rebuild, where every existing index on the old table is maintained in real time.

One detail here is worth paying attention to: *when* an entry is logged. Because the row log tracks B+tree changes, an entry is written only after the B+tree operation succeeds. As the figure shows, *for a table rebuild the entry is logged after the row is inserted into the clustered B+tree; for an added index it is logged at the point the row would go into the secondary index's B+tree* — except that, for an added index, nothing is actually inserted, and only the log entry is written.

![](/assets/img/ddl-dupkey-1-en.svg)

### What Happens on a DML Rollback

A DML operates on the clustered B+tree first, then the secondary indexes. So what happens if the secondary-index step fails?

- The row-log entry remains — it isn't cleaned up. It records one B+tree operation, and that operation really did happen.
- The statement then rolls back. Rolling back replays the change in reverse against the B+tree (driven by the undo log), and that reverse operation is logged too. So a failed INSERT leaves *two* row-log entries — a `ROW_T_INSERT` followed by a `ROW_T_DELETE` (below). Applied in order, the two cancel out, exactly as a rollback should.

![](/assets/img/ddl-dupkey-2-en.svg)

A failed statement is just one way to reach this state; every rollback works the same way, including an explicit `ROLLBACK` from the user.

## Replaying the Incremental Log

The row log is replayed twice:

- *The first pass* runs in Execute, right after the full copy. The lock is still `MDL_SHARED_UPGRADABLE`, so DML keeps running and keeps adding to the log. The purpose of this pass is to *replay as much of the log as possible now, so the exclusive lock in Commit is held for as short a time as possible*.
- *The second pass* runs in Commit, under `MDL_EXCLUSIVE`, with no new log being produced. It handles the small remainder, and once it is done the new table matches the old one exactly.

Both passes read the log in write order, one entry at a time. For a table rebuild, each entry is applied to every index on the new table; for an added index, only to the index being built.

## What Actually Causes the Duplicate Entry

Recall the rollback case: even when an INSERT encounters a `Duplicate Entry` on a unique index, it still leaves two row-log entries behind (below).

![](/assets/img/ddl-dupkey-3-en.svg)

Replaying the first entry, `ROW_T_INSERT`, does exactly what the original INSERT did: insert a row into the clustered B+tree, then add an entry to each index. When it reaches the unique index, it encounters the same `Duplicate Entry` — this time during replay, which is what fails the whole DDL.

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

The example relies on the MDL behavior described above:

- Session 1's INSERT takes an `MDL_SHARED_WRITE` lock on `t1` and holds it until it commits.
- Session 2's ALTER needs `MDL_EXCLUSIVE` for its prepare phase, so it is blocked by Session 1.
- Session 3's INSERT wants `MDL_SHARED_WRITE`, but is now blocked by Session 2.

You can observe this in `performance_schema.metadata_locks` — the first row below is Session 1's lock.

![](/assets/img/ddl-dupkey-4-en.png)

When Session 1 commits, Session 2 acquires `MDL_EXCLUSIVE`, finishes Prepare, and downgrades to `MDL_SHARED_UPGRADABLE`. That lock doesn't conflict with `MDL_SHARED_WRITE`, so Session 3 runs — and since `c2 = 1` already exists, it fails with a `Duplicate key` error. That failure leaves a row-log entry, and replaying it is what makes Session 2's ALTER fail in turn.

![](/assets/img/ddl-dupkey-5-en.svg)

### How to Avoid It

Once the mechanism is clear, I would call this a bug — which is why we fixed it in AliSQL. On community MySQL, the only option is to prevent DML from encountering `Duplicate Entry` while a DDL is running.

A common trigger is a table with an auto-increment primary key that has also gained a unique index. The application inserts a row, lets MySQL assign the auto-increment key, and — if the first INSERT is slow — retries the same statement from another session. That is perfectly sensible on its own: the unique index guarantees only one of the two can succeed. But a single failing INSERT is enough to make the DDL fail. So while a DDL is in progress, increasing the retry timeout is a practical way to avoid the problem.

## The Fix in AliSQL

AliSQL takes a straightforward approach: *when replay encounters a `Duplicate Entry`, ignore it.*

### Real Duplicates vs. False Ones

There is one caveat: not every `Duplicate Entry` is safe to ignore.

Online DDL has a genuine *real duplicate* case: *the DDL introduces a new uniqueness constraint (a new or changed primary key, or a new UNIQUE index) while the existing data already contains duplicates.* That new constraint isn't enforced during the DDL, so concurrent DML can keep adding duplicates — and here the DDL *must* fail. Ignoring it would silently break the guarantee the user asked for.

But if the `Duplicate Entry` occurs on a unique index that already existed on the table, it is always a *false duplicate* and safe to skip. That is the common case, and the one AliSQL optimizes.

### Cleaning Up the Rollback Entry

As we saw, a failed DML leaves two row-log entries. For an INSERT:

```
<ROW_T_INSERT, pk1, ...>
<ROW_T_DELETE, pk1>
```

On replay, the insert into the unique index fails and we ignore it — but the insert into the primary-key B+tree did succeed. When the second entry (`ROW_T_DELETE`) is replayed, deleting from the primary key is fine; deleting from the unique index is not, because nothing was ever inserted there, so the record can't be found and the replay fails.

So whenever we skip a `Duplicate Entry` on a pre-existing unique index, we have to record it and skip the matching operation when the rollback's row log arrives later, as shown below:

![](/assets/img/ddl-dupkey-6-en.svg)

UPDATE is more complex. When an UPDATE changes a secondary-index column, the secondary index performs two operations:

1. Delete the old entry.
2. Insert the new one.

The replay fails at step 2, so on the rollback we cannot simply skip the whole index — we skip step 1 but still perform step 2:

![](/assets/img/ddl-dupkey-7-en.svg)

With this handling, AliSQL eliminates the `Duplicate Entry` failures that Online DDL should never have raised in the first place.

## Conclusion

Online DDL can fail with a `Duplicate Entry`, and because it is nondeterministic — it depends on the timing of concurrent DML — it is hard to rule out entirely, and each failure can disrupt a planned change. The failure happens in just a few steps: a concurrent DML encounters a `Duplicate Entry`, that error is carried through Online DDL's row log, and replaying the log fails the DDL. AliSQL breaks this chain by ignoring `Duplicate Entry` errors that occur on an already-existing unique index, so Online DDL is no longer aborted by an error that did not originate from the DDL itself.
