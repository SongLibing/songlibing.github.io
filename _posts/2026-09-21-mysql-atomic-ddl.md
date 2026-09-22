---
title: Atomic DDL 揭秘
description: 深入解析 MySQL 8.0 的 Atomic DDL 实现：统一的 Data Dictionary、InnoDB DDL Log 如何分别当作 Redo/Undo 保证 DDL 的原子性，以及 Binlog 的 DDL Crash Safe 和 CREATE TABLE ... SELECT 的原子化改进。
date: 2026-09-22 10:00:00 +0800
categories: [数据库, MySQL]
tags: [MySQL, DDL, Atomic DDL, Crash Safe]
toc: true
---

在 MySQL-8.0 之前，DDL 是不能做到 Crash Safe 的。主要的问题有 3 个：

1. Server 层的 Metadata 和 InnoDB 的 Metadata/数据不一致。

   Server 层的 Metadata 记录在文件中，例如表结构信息保存在 `frm` 文件中；InnoDB 中也有一份 Metadata 存储在表中。Crash 可能导致 Server 层的 Metadata 和 InnoDB 中的 Metadata、甚至表中的数据不一致。比如 Server 层存在表的 `frm` 文件，认为表存在，但是 InnoDB 的 `ibd` 数据文件已经不存在了，InnoDB 认为表不存在。
2. InnoDB 的 Metadata 和数据不一致。
3. Binlog 和数据不一致。

   比如 Crash 重启后，表已经存在了，但是 `CREATE TABLE` 却没有记录到 Binlog 中。

MySQL-8.0 为了实现 Atomic DDL、彻底解决以上问题，做了三个方面的改动：

1. 去掉 Server 层的 Metadata 文件，所有 Metadata 统一存放到 InnoDB 表中。

   这些表被称为 Data Dictionary。用户、Server 层、引擎都可以通过 DD 的访问接口查询或者更新 Metadata。
2. DDL Log。

   InnoDB 中实现了 DDL Log 表，在 DDL 操作过程中会记录一些 DDL 的操作日志。InnoDB 通过 DDL Log 来保证 DDL 中的文件操作和 Metadata 操作的原子性。
3. Binlog DDL Crash Safe。

   在 DDL 的 Binlog Event 中会记录 DDL 操作的 Xid，通过 Xid 来保证 Binlog 的 Crash Safe。

## DDL 事务

实现 DDL 原子性的基本思路是把 DDL 的过程变成一个事务，通过事务的原子性来保证 DDL 的原子性。MySQL-8.0 将所有的 Metadata 保存到了 InnoDB 的表中，对 Metadata 的操作实际上就是对 InnoDB 表做 `INSERT`、`UPDATE` 或者 `DELETE` 操作。Metadata 的操作完全可以通过一个事务来完成，对于只修改 Metadata 的 DDL 操作，通过一个事务就可以保证其原子性，比如 View、Function、Trigger 的创建、修改、删除操作。

DDL 执行的过程中会开启一个事务，这里把它称作 DDL Trx。DDL Trx Commit 就等同于 DDL 执行成功。Commit 前 DDL 的所有操作都是可以回滚的，Commit 之后进行的操作则不能回滚。

## InnoDB DDL Log

对于涉及数据文件操作的 DDL，InnoDB 将文件的操作指令写入 DDL Log 表中，并将 DDL Log 表的操作和 Metadata 表的操作放入同一个事务，来实现 DDL 操作的原子性。

涉及数据文件操作的 DDL 包括：

* `CREATE TABLE`
* `ALTER TABLE`
* `DROP TABLE`
* `RENAME TABLE`
* `CREATE INDEX`
* `DROP INDEX`
* `DROP DATABASE`

## DDL Log Table

DDL Log Table 定义如下：

```sql
CREATE TABLE mysql.innodb_ddl_log (
  id            BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  thread_id     BIGINT UNSIGNED NOT NULL,
  type          INT UNSIGNED NOT NULL,
  space_id      INT UNSIGNED,
  page_no       INT UNSIGNED,
  index_id      BIGINT UNSIGNED,
  table_id      BIGINT UNSIGNED,
  old_file_path VARCHAR(512) COLLATE UTF8_BIN,
  new_file_path VARCHAR(512) COLLATE UTF8_BIN,
  KEY(thread_id)
);
```

DDL Log Table 中记录了以下几种操作指令：

* DELETE SPACE

  删除指定的 table space 文件。
* DROP

  从 `mysql.innodb_dynamic_metadata` 表中删除指定表的信息。
* RENAME SPACE

  重命名指定表的 table space 文件。
* RENAME TABLE

  重命名指定表，包括修改 Metadata，以及统计信息中的表名等操作。
* FREE

  删除指定的索引。
* REMOVE CACHE

  从 table cache 中移除指定的表。

## DDL Log 的使用

DDL 语句的执行被分成了两部分：

1. DDL Trx 相关的操作。

   DDL Trx 执行的过程中会往 DDL Log Table 写入一些操作指令。
2. Post DDL 操作。

   读出写入 DDL Log Table 中的指令，按反序执行。不论 DDL Trx 是成功还是失败，DDL Log 表中记录的指令都会被执行（如果有的话）。

DDL Log 可以看作是 Redo Log 和 Undo Log 的一个合集。有些 DDL 把它用作 Redo，有些 DDL 把它用作 Undo，还有些 DDL 会把它同时当作 Redo 和 Undo。

## DDL Log 当作 Redo 的用法

![](/assets/img/atomic-ddl-1.svg)

DROP TABLE 把 DDL Log 当作 Redo 来用。

* 在 DDL Trx 阶段，DROP TABLE 会删除 Metadata，并且往 DDL Log 中插入一条 DELETE SPACE 的指令。DDL Trx Commit 后，就不能回滚了。
* Post DDL 阶段，根据 DDL Log 中的 DELETE SPACE 指令，进行实际的文件删除操作。
* 文件删除操作完成后，删除 DDL Log Table 中的记录。

  DDL Log Table 中的指令都是幂等的，多次执行不影响正确性。所以执行 DDL Log Table 中指令的部分和删除 DDL Log Table 中记录的部分不是原子的。

![](/assets/img/atomic-ddl-2.svg)

当 DDL Trx 阶段有错误发生时，DDL Trx 会回滚，DDL Log Table 中的 DELETE SPACE 也会回滚掉。所以 Post DDL 阶段就什么也不做。

## DDL Log 当作 Undo 的用法

![](/assets/img/atomic-ddl-3.svg)

CREATE TABLE 则将 DDL Log 当作 Undo 来用。

* DDL Trx 执行阶段，CREATE TABLE 首先在 DDL Log 中记录一条 DELETE SPACE 的指令，这个记录要用另外一个事务执行（这里称作 DDL Log Trx），并提交。
* 然后在 DDL Trx 中删除 DDL Log Table 中对应的 DELETE SPACE 记录。
* 最后创建表文件，执行成功后提交 DDL Trx。
* DDL Trx 如果提交了，DDL Log Table 的 DELETE SPACE 记录已经被删除，所以 Post DDL 阶段什么也不做。

![](/assets/img/atomic-ddl-4.svg)

当错误发生后，DDL Trx 会回滚，DDL Log Table 的 DELETE SPACE 记录会保留下来，Post DDL 阶段会根据 DDL Log Table 中的记录将表文件删除掉，并且删除 DDL Log Table 中的记录。

## innodb_print_ddl_logs

为了方便调试，InnoDB 提供了一个功能，可以将 DDL Log 表的所有操作都记录到 error log 里。这个功能可以通过 `innodb_print_ddl_logs` 变量动态地开关。

下面我们就看一下几个 DDL 的详细过程。

## CREATE TABLE

CREATE TABLE 将 DDL Log 当作 Undo 来使用，当 DDL 执行失败时通过 DDL Log 来回滚创建文件的操作。

```sql
CREATE TABLE t1(c1 INT PRIMARY KEY, c2 VARCHAR(20), INDEX(c2));
```

该 DDL 对 DDL Log Table 的操作如下：

![](/assets/img/atomic-ddl-5.png)

1. 通过 DDL Log Trx(1802) 记录 DELETE SPACE 回滚指令，它的记录 ID 是 7。
2. 通过 DDL Trx(1801) 删除 DDL Log Table 中 ID 为 7 的记录。
3. 将 t1 加入 Table Cache。Table Cache 的回滚依赖于 DDL Log Table，所以先通过 DDL Log Trx(1803) 往 DDL Log Table 中添加 REMOVE CACHE 记录，然后在 DDL Trx(1801) 中删除这条记录。
4. 之后的两个 FREE 分别是 Clustered B-Tree 和 Index B-Tree 的回滚日志。对于 CREATE TABLE 来说删除 B-Tree 的操作是不必要的，直接删除文件就可以了，DROP TABLE 就没有删除 B-Tree 的过程。
5. 当 DDL Trx(1801) 提交后，之前添加到 DDL Log 中的所有记录都被删除了，所以 Post DDL 阶段什么也没有做。

## DROP TABLE / DROP DATABASE

DROP TABLE 将 DDL Log 当作 Redo 来使用，当 DDL 操作成功后，在 Post DDL 阶段根据 DDL Log Table 中的记录执行文件的删除操作。

```sql
DROP TABLE t1, t2;
```

该 DDL 对 DDL Log Table 的操作如下：

![](/assets/img/atomic-ddl-6.png)

1. 记录 DROP 指令，用来删除 t1 在 `mysql.innodb_dynamic_metadata` 中的信息。`innodb_dynamic_metadata` 是一个比较特殊的表，该表的操作不记录 Undo Log，不能在 DDL Trx 里操作。因此会记录一条指令到 DDL Log Table，然后放到 Post DDL 阶段执行。
2. 然后记录 DELETE SPACE 指令，用来删除 `db1/t1.ibd`。
3. 接着对表 t2 重复以上的操作。
4. 当所有表的删除操作完成后，提交 DDL Trx(1916)，这时 DDL Log Table 中插入了 4 条记录。
5. 在 Post DDL 阶段，会按反序执行这 4 条记录，执行实际的删除操作。Redo 实际上是可以不按顺序执行的，但是 Undo 必须要反序执行，因为会有前后依赖关系。估计是为了保持简单和统一，Post DDL 时统一按反序执行。

DROP DATABASE 对 DDL Log Table 的操作等同于 DROP 指定数据库中的所有表。

## CREATE INDEX

CREATE INDEX 将 DDL Log 当作 Undo 来使用，当创建 Index 失败时通过 DDL Log Table 中的记录回滚正在创建的 Index。

```sql
ALTER TABLE t2 ADD INDEX ind1(c3);
```

该 DDL 对 DDL Log Table 的操作如下：

![](/assets/img/atomic-ddl-7.png)

1. 创建 B-Tree 前，通过 DDL Log Trx(1872) 记录 FREE 回滚指令，它的记录 ID 是 30。
2. 通过 DDL Trx(1871) 删除 DDL Log Table 中的 FREE 指令。
3. 通过 DDL Trx 更新 Metadata，并创建 B-Tree。
4. 最后提交 DDL Trx(1871)。事务提交后 DDL Log Table 中的记录已经被删除，Post DDL 阶段什么也不做。

## DROP INDEX

DROP INDEX 将 DDL Log 当作 Redo 来使用。

```sql
ALTER TABLE t2 DROP INDEX ind1;
```

该 DDL 对 DDL Log Table 的操作如下：

![](/assets/img/atomic-ddl-8.png)

1. 通过 DDL Trx，在 DDL Log Table 中插入一条 FREE 指令。
2. 更新 Metadata 后提交 DDL Trx。
3. Post DDL 阶段，根据 DDL Log Table 中的 FREE 指令删除 Index 的 B-Tree。

## RENAME TABLE

RENAME TABLE 将 DDL Log 当作 Undo 来使用。

```sql
RENAME TABLE t2 TO t20;
```

该 DDL 对 DDL Log Table 的操作如下：

![](/assets/img/atomic-ddl-9.png)

1. 重命名文件前，通过 DDL Log Trx(1909)，在 DDL Log Table 中插入一条 RENAME SPACE 指令。这是 Undo 记录，所以这条指令是将 `t20.ibd` 重命名为 `t2.ibd`。
2. 通过 DDL Trx 将 DDL Log Trx 中的 RENAME SPACE 记录删除。
3. 通过 DDL Log Trx(1909)，在 DDL Log Table 中插入一条 RENAME TABLE 指令。这是 Undo 记录，所以这条指令是将 t20 重命名为 t2。

   RENAME 时需要修改一些内存结构，另外修改 `innodb_table_stats` 中的表名是通过单独的事务更新的，无法回滚。所以这里记录一个 RENAME TABLE 指令，通过反向命名实现回滚。
4. 通过 DDL Trx 将 DDL Log Trx 中的 RENAME TABLE 记录删除。
5. DDL Trx(1908) 事务提交后，DDL Log Table 中的记录已经被删除，Post DDL 阶段什么也不做。

## ALTER TABLE

ALTER TABLE 有很多种，最为复杂的是需要重建表的情形。在这种情况下，DDL Log 既是 Redo，也是 Undo。

```sql
ALTER TABLE t2 ADD COLUMN c3 int;
```

该 DDL 对 DDL Log Table 的操作如下：

![](/assets/img/atomic-ddl-10.png)

ALTER TABLE 集合了 CREATE TABLE、RENAME TABLE 和 DROP TABLE 的操作：

1. 创建临时表 `db1/#sql-ib1066-677171833`，记录删除该临时表的 Undo 指令。
2. 重命名 `db1/t2` 为 `db1/#sql-ib1071-677171834`，记录反向命名的 Undo 指令。
3. 重命名 `db1/#sql-ib1066-677171833` 为 `db1/t2`，记录反向命名的 Undo 指令。
4. 删除表 `db1/#sql-ib1071-677171834`，记录删除文件的 Redo 指令。

## DDL Log 用法总结

* DDL 的执行过程分为 DDL Trx 阶段和 Post DDL 阶段。
* DDL Trx 阶段向 DDL Log Table 中插入 Redo 或/和 Undo 指令。
* Post DDL 阶段执行 DDL Log Table 中的指令。
* 当作 Redo 使用时，通过 DDL Trx 向 DDL Log Table 插入 Redo 指令，Post DDL 阶段执行这些指令。
* 当作 Undo 使用时，则通过 DDL Log Trx 向 DDL Log Table 插入 Undo 指令，然后通过 DDL Trx 删除这些 Undo 指令。如果 DDL Trx 提交了，Post DDL 阶段什么也不做；如果 DDL Trx 回滚了，Post DDL 阶段按照 DDL Log Table 中的指令进行回滚操作。
* DDL Log Table 中的指令都是幂等的，可以多次执行而不影响正确性。

## DDL Crash Recovery

Crash 可能发生在 Post DDL 完成之前，所以重启后做 Recovery 时，Server 需要检查 DDL Log Table，并按 DDL Log Table 中的记录完成所有 Post DDL 操作。

## Binlog Crash Safe

![](/assets/img/atomic-ddl-11.png)

现在对于支持 Atomic DDL 的存储引擎，每个 DDL 都是一个事务了，事务过程中失败了可以回滚，事务提交则意味着 DDL 执行成功。当 Binlog 开启时，DDL 的事务也会采用两阶段提交。MySQL-8.0 对 Binlog 的 `Query_log_event` 做了扩展，DDL 事务的 Xid 会存储到 `Query_log_event` 中。

![](/assets/img/atomic-ddl-12.png)

这样在做 Crash Recovery 的时候，就可以根据 DDL 的 `Query_log_event` 中的 Xid 来决定提交或回滚 Prepare 状态的 DDL 事务了。

## CREATE TABLE ... SELECT

CREATE TABLE ... SELECT 在 Binlog 中会被分成 CREATE TABLE 和 INSERT 两部分（Row Format），如下所示：

```text
CREATE TABLE t1   // Query_log_event
BEGIN             // Query_log_event
                  // Table_map_log_event
INSERT            // Write_rows_log_event
COMMIT            // Xid_log_event
```

MySQL-8.0.21 之前，CREATE 部分和 INSERT 部分是两个独立的事务，所以没办法实现原子性。因此当开启 GTID 后，CREATE TABLE ... SELECT 是被禁止的。

在实现了 Atomic DDL 后，MySQL-8.0.21 对 CREATE TABLE ... SELECT 做了改进。现在 CREATE TABLE 部分和 INSERT 部分可以在同一个事务中执行，并且保证原子性。因此开启 GTID 时，也可以使用 CREATE TABLE ... SELECT 语句了。

```text
BEGIN                                      // Query_log_event
CREATE TABLE t1 (......) START TRANSACTION // Query_log_event
                                           // Table_map_log_event
INSERT                                     // Write_rows_log_event
COMMIT                                     // Xid_log_event
```

下图是一个实际的例子：

![](/assets/img/atomic-ddl-13.png)

可以看到，MySQL 对 CREATE TABLE 语句做了扩展，从而实现 CREATE TABLE 和 DML 在同一个事务中执行。

```sql
CREATE TABLE ... START TRANSACTION;
```

DDL 在执行完毕时，会自动 COMMIT 当前的事务。有了 START TRANSACTION 这个扩展后，
CREATE TABLE 语句就不会自动结束当前事务了，所以后续的 DML 就和 CREATE TABLE 在同
一个事务中执行了。目前 CREATE TABLE ... START TRANSACTION 是为复制线程重放 CREATE TABLE ... SELECT 引入的内部事务化形式。创建表后只允许继续应用 Binlog 行事件，不能作为通用的“DDL 后继
续执行普通 DML”语法。但是从这个改进可以看到，在实现了 Atomic DDL 之后，让 DDL 支持事务就不难了，今后也许更多 DDL 可以和 DML 放到同一个事务中执行。

## 参考

* [Atomic Data Definition Statement Support](https://dev.mysql.com/doc/refman/8.0/en/atomic-ddl.html)
* [WL#13355: Make CREATE TABLE...SELECT atomic and crash safe](https://dev.mysql.com/worklog/task/?id=13355)
* [WL#9536: InnoDB: Support crash-safe DDL](https://dev.mysql.com/worklog/task/?id=9536)
