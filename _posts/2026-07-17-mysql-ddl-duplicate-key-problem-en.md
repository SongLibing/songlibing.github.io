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

When MySQL rebuilds a table with Online DDL, it can hit a `Duplicate Entry` error that makes the DDL fail partway through. The failure looks like this:

```sql
mysql> alter table tt add c3 int, algorithm=inplace;
ERROR 1062 (23000): Duplicate entry '1' for key 'tt.uk_c2'
```

This is a well-known problem that has existed ever since MySQL 5.6 introduced Online DDL. Related bugs include:

- [BUG#76895](https://bugs.mysql.com/bug.php?id=76895) — Adding new column OR Drop column causes duplicate PK error
- [BUG#77572](https://bugs.mysql.com/bug.php?id=77572) — The bogus duplicate key error in online ddl with incorrect key name
- [BUG#98600](https://bugs.mysql.com/bug.php?id=98600) — Optimize table fails with duplicate entry on UNIQUE KEY
- [BUG#104626](https://bugs.mysql.com/bug.php?id=104626) — Remove failure of Online ALTER because concurrent Duplicate entry

[BUG#76895](https://bugs.mysql.com/bug.php?id=76895) was the first report of this problem. It described it as a `Duplicate PRIMARY KEY` problem, but it was really a `Duplicate UNIQUE KEY`: because of another bug on MySQL 5.6, [BUG#77572](https://bugs.mysql.com/bug.php?id=77572), the failure message wrongly said `PRIMARY KEY`. BUG#77572 was fixed in MySQL 5.6.28 and 5.7.10. In later versions people saw the `Duplicate UNIQUE KEY` error, which led to yet another report, [BUG#98600](https://bugs.mysql.com/bug.php?id=98600).

For BUG#76895, MySQL officially responded `Not a Bug`, considering it a side effect of Online DDL, so it was never fixed. But this problem makes DDL operations fail, which is a real headache for DBAs in production — once a DDL fails, the planned maintenance or change is disrupted or even postponed. So the community filed [BUG#104626](https://bugs.mysql.com/bug.php?id=104626), raising it as a feature request.

What is the root cause of this bug? Why does the official team consider it not a bug? And is there any way to avoid it? To answer these three questions, we first need to understand how Online DDL works.

## How Online DDL Works

To let DML run at the same time as a DDL, Online DDL uses a *full copy* plus *incremental replay* mechanism. Online DDL first applies all the existing data in the table to the new table (for adding an index, no new table is created — the data is applied directly to the new index). The full apply takes a while, during which DML is allowed and its changes are recorded as increments. After the full replay finishes, the increments are applied to the new table. Two things are key to this mechanism:

- There must be a way to precisely distinguish data produced before and after a specific point in time; the full copy copies only the data produced before that point.
- Every update made after that point must be recorded in an incremental log, and once the full copy finishes, replaying that log fills in the rest.

### Distinguishing Full Data from Incremental Data

In Online DDL, the *full data* is the data in the *clustered B+Tree*, which Online DDL obtains by scanning that tree. DML is allowed during the scan, so the B+Tree actually contains incremental changes too. Online DDL uses InnoDB's `MVCC (multi-version concurrency control)` to tell full data from incremental data.

InnoDB's MVCC lets a transaction see a consistent snapshot of the data, unaffected by other concurrent transactions. At its core is the `read view`: when a read view is created, it records the list of all currently active transaction IDs, and later uses that list to decide which version of each row is visible to it. For modifications committed after the read view was created — even if already written into the clustered index — the read view can still find the older version through the undo log.

Online DDL uses exactly this ability to draw the boundary for the full copy: *it creates a read view at a definite point in time, and the full copy reads only the data visible to that read view. Changes made by concurrent DML afterward, whenever they commit, never affect what the full copy reads.*

## The Three Phases of Online DDL

InnoDB divides an Online DDL into three phases. Each phase holds a different level of `Metadata Lock (MDL)`, precisely controlling how concurrent DML behaves.

### Prepare Phase

The Prepare phase runs under `MDL_EXCLUSIVE`, so all reads and writes to the table are blocked. This "write-stopped" window is the key point in time for separating full data from incremental data. In this phase:

1. A consistent read view is created via `trx_assign_read_view()`.
2. The row log (incremental log) is initialized on the relevant index, and the index is marked `ONLINE_INDEX_CREATION`. Subsequent insert/update/delete operations on the index B+tree then use `ONLINE_INDEX_CREATION` to decide whether they need to be recorded in the row log.

Installing the row log and creating the read view both happen under `MDL_EXCLUSIVE`, which guarantees two things:

1. It ensures the row log and the read view act as a whole, seamlessly covering both the existing data and the data changes.
2. It *excludes in-flight transactions*. Without `MDL_EXCLUSIVE`, a transaction might have modified the table's data but not yet committed. At that moment the row log isn't installed, so the change isn't recorded in it; and the uncommitted transaction is marked active in the read view, so the full copy can't see it either. That DML's change would be in neither the full copy nor the row log, causing data loss. `MDL_EXCLUSIVE` ensures no transaction that has modified the table is running at that instant, letting the row log and read view establish a precise dividing line: *committed data before the line is handled by the full copy, and all DML after the line is captured by the row log.*

After the Prepare phase, the MDL is downgraded from `MDL_EXCLUSIVE` to `MDL_SHARED_UPGRADABLE`, a shared lock.

### Execute Phase

In the Execute phase, Online DDL holds `MDL_SHARED_UPGRADABLE`, which *lets concurrent DML run normally*. The full copy and the first incremental replay both happen here. This is the most time-consuming phase of Online DDL, and the phase where DML is not blocked. All DML on the table is recorded in the row log.

### Commit Phase

In the Commit phase, the DDL upgrades the MDL from `MDL_SHARED_UPGRADABLE` back to `MDL_EXCLUSIVE`, *blocking all reads and writes*. Under this protection it finishes replaying the row log and commits the DDL transaction. Since the Execute phase already replayed most of the row log, there is little data left for the Commit phase to handle, so the exclusive lock is held only briefly.

The `MDL_EXCLUSIVE` in this phase also guarantees two things:

- It ensures all DML transactions from the Execute phase that modified the table have committed.
- It blocks new DML, ensuring no new row log is produced.

From the above we can see that Online DDL is *not online the whole way through*: DML is blocked during the Prepare and Commit phases. Also, since the metadata lock is transaction-level, if a long transaction is running when the DDL tries to acquire `MDL_EXCLUSIVE`, the DDL is blocked for a long time — and while it waits for `MDL_EXCLUSIVE`, it also blocks other DML, making the table unreadable and unwritable.

## The Full Copy

The full copy runs in the Execute phase, when the MDL is `MDL_SHARED_UPGRADABLE` and concurrent DML is allowed.

It calls `row_merge_read_clustered_index()` to scan the old table's clustered index in ascending PRIMARY KEY order. For each row during the scan, MVCC decides its visibility. Only data visible to the read view is copied to the new table; `delete-marked` records are skipped.

Because of MVCC, even if concurrent DML modifies the old table's data during the scan, the full copy can still read the version as of the read view moment through the undo log. What the full copy sees is always a consistent snapshot, unaffected by concurrent DML.

## Recording the Incremental Log

The row log records changes to the data pages of a B+tree, in two cases:

- Rebuilding the table: it records only changes to the clustered B+tree, including `INSERT`, `UPDATE`, and `DELETE`. When the row log is replayed, each record is *replayed onto all indexes*.
- Creating an index: it records only changes to the B+tree of the index being created, including `INSERT` and `DELETE`, with no `UPDATE`. Because this is the index being created, operations on its B+tree are only recorded in the row log and *don't actually modify the B+tree*. This differs from rebuilding a table, where all of the old table's indexes must be updated in real time.

Pay special attention to *when* the row log is recorded. The row log records changes to a B+tree, so it's recorded after a B+tree operation succeeds. As shown below: *when rebuilding a table, the row log is recorded after the insert into the clustered B+tree succeeds; when adding an index, it's recorded when inserting into the secondary index B+tree.* In the add-index case, no record is actually inserted into the B+tree — only the row log is recorded.

![](/assets/img/ddl-dupkey-1.webp)

### The DML Rollback Case

When a DML runs, it operates on the clustered B+tree first and then the secondary index B+tree. This raises a question: what happens if the secondary-index operation fails?

- First, the row log still exists and isn't cleaned up. The row log records one operation on a B+tree, and that operation did happen.
- Second, the DML statement rolls back. On rollback, based on the undo log, it operates on the B+tree again, and this operation is likewise recorded in the row log. So a failed INSERT records two row-log entries — one `ROW_T_INSERT` and one `ROW_T_DELETE`, as shown below. Executing the two entries in order is equivalent to never having produced the record, which matches the rollback's intent.

![](/assets/img/ddl-dupkey-2.webp)

A failure-triggered rollback is just one special case; in fact all rollbacks follow this same logic, including a user's manual `ROLLBACK`.

## Replaying the Incremental Log

The row log is replayed twice in total:

- *The first replay* is in the Execute phase, right after the full copy finishes. The MDL is `MDL_SHARED_UPGRADABLE`, concurrent DML is still running and still producing new row log. The goal of this replay is to *consume as much existing row log as possible, reducing the time the exclusive lock is held in the Commit phase*.
- *The second replay* is in the Commit phase, where the MDL has been upgraded to `MDL_EXCLUSIVE` and no new row log is produced. This replay handles the small amount of remaining incremental data; when it finishes, the new table's data is fully consistent with the old table's.

Both replays read and replay the row log entry by entry, in the order it was written. When rebuilding a table, each entry is applied to the B+trees of all of the new table's indexes; when creating an index, each entry is applied only to the newly created index.

## The Root Cause of Duplicate Entry

In the earlier *DML Rollback Case*, we saw that even when a `Duplicate Entry` error occurs on a unique index during an INSERT, the row log is still recorded — two entries, in fact. As shown below:

![](/assets/img/ddl-dupkey-3.webp)

When replaying the first entry, `ROW_T_INSERT`, the logic is the same as executing an INSERT statement: first insert a row into the clustered B+tree, then insert a record into each index. So when inserting the record into the unique index, it again reports a `Duplicate Entry` error. It's exactly this duplicate entry that makes the DDL statement fail.

### A Test Case

```sql
CREATE TABLE t1 (
  c1 INT AUTO_INCREMENT PRIMARY KEY,
  c2 INT,
  UNIQUE KEY (c2)
) ENGINE=InnoDB;
```

Open three sessions and run the following statements:

```sql
# Session 1
BEGIN;
INSERT INTO t1 VALUES(NULL, 1);

# Session 2
ALTER TABLE t1 ENGINE = InnoDB;

# Session 3
INSERT INTO t1 VALUES(NULL, 1);
```

After running the above, commit Session 1's transaction. You will then find that Session 2's ALTER and Session 3's INSERT both report `Duplicate Entry`:

```sql
# Session 2
mysql> ALTER TABLE t1 ENGINE = InnoDB;
ERROR 1062 (23000): Duplicate entry '1' for key 't1.c2'

# Session 3
mysql> INSERT INTO t1 VALUES(2, 1);
ERROR 1062 (23000): Duplicate entry '1' for key 't1.c2'
```

This example is constructed using the `Metadata Lock` mechanism of Online DDL described earlier.

- Session 1's INSERT first holds the `MDL_SHARED_WRITE` lock on t1, which is released only when the transaction commits.
- Session 2's ALTER needs the `MDL_EXCLUSIVE` lock in its prepare phase, and is blocked by Session 1.
- Session 3's INSERT also needs the `MDL_SHARED_WRITE` lock, but is blocked by Session 2.

The `metadata_locks` table in `performance_schema` shows these sessions' metadata locks, as below; the first row is the lock held by Session 1.

![](/assets/img/ddl-dupkey-4.webp)

After Session 1's transaction commits, Session 2 acquires the `MDL_EXCLUSIVE` lock, and after finishing its prepare phase, downgrades to the `MDL_SHARED_UPGRADABLE` lock. This lock doesn't conflict with `MDL_SHARED_WRITE`, so Session 3 acquires `MDL_SHARED_WRITE` and starts running. During execution, because `c2 = 1` already exists, it reports a `Duplicate key` error. This process records a row log, which in turn causes the error in Session 2's ALTER statement.

![](/assets/img/ddl-dupkey-5.webp)

### How to Avoid This Problem

Once you understand why this happens, I consider it a bug, so we fixed it in AliSQL. If you're on community MySQL, the way to avoid it is to keep `Duplicate Entry` situations from happening during DML as much as possible while a DDL is running.

A common scenario is a table with an auto-increment primary key to which a unique index has been added. When the application inserts a row, MySQL generates the auto-increment key. The application has retry logic: once a previous INSERT is slow, it may retry the same SQL in another session. Logically this is reasonable — because of the unique index, only one INSERT can succeed. But it's precisely this logic that causes the DDL to fail. So during a DDL, you can try increasing the retry timeout to avoid the problem.

## The Optimization in AliSQL

AliSQL takes a fairly intuitive and simple approach: *when a Duplicate Entry error is hit, ignore it.*

### Real vs. False Duplicates

This ignore strategy has a precondition — not every `Duplicate Entry` can be ignored.

During Online DDL there is also a *real duplicate* case: *the DDL introduces a new uniqueness constraint (adding or changing the primary key, or adding a UNIQUE index), and the original data already contains duplicate records.* The new uniqueness constraint isn't in effect during Online DDL, so DML during this period can still introduce duplicate records. *When this happens, the DDL must fail.*

If the `Duplicate Entry` happens on a unique index that already existed in the original table, it's definitely a *false duplicate* and can be skipped. This is the most common case, and it's the one AliSQL optimizes.

### Handling the Subsequent Undo Row Log

As noted earlier, when a DML fails it actually records two row-log entries. Take an INSERT:

```
<ROW_T_INSERT, pk1, ...>
<ROW_T_DELETE, pk1>
```

During row-log replay, inserting into the unique-index B+tree fails and is ignored. But the insert into the primary-key B+tree did succeed. When the second entry is replayed, the record in the primary-key B+tree is deleted, which is fine. But since the unique index never had a successful insert, the record can't be found there, which makes the replay fail.

So when we hit a `Duplicate Entry` error on an already-existing unique index, we need to record that error. When replaying the row log produced by the rollback, we likewise skip the operation on that index. As shown below:

![](/assets/img/ddl-dupkey-6.webp)

UPDATE is more complex. If an UPDATE modifies a secondary-index column, two operations are performed on the secondary index:

1. Delete the old record.
2. Insert the new record.

The failure during row-log replay happens at step 2, so when doing the subsequent rollback we can't simply skip all operations on that index — we skip step 1, but step 2 still has to run, as shown below:

![](/assets/img/ddl-dupkey-7.webp)

With this design, AliSQL avoids the unnecessary `Duplicate Entry` errors during Online DDL.

## Conclusion

Online DDL sometimes reports a `Duplicate Entry` error that makes the DDL fail. This error is nondeterministic and hard to avoid entirely, and once a DDL fails, the planned maintenance or change is disrupted or even postponed. The cause is that a concurrent DML during Online DDL hits a `Duplicate Entry` error, and that DML error is carried to the DDL through Online DDL's row log, making the DDL fail. AliSQL optimized Online DDL to ignore `Duplicate Entry` errors that occur on an already-existing unique index, so Online DDL is no longer interrupted by this error.
