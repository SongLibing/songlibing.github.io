---
title: 'DDL "Duplicate Key" 错误详解'
date: 2026-07-17 16:30:00 +0800
categories: [数据库, MySQL]
tags: [MySQL, DDL, Online DDL, InnoDB]
toc: true
---

MySQL 在执行 Online DDL 重建表的时候，可能会碰到 `Duplicate Entry` 错误，从而导致 DDL 中途失败。失败信息如下所示：

``` sql
mysql> alter table tt add c3 int, algorithm=inplace;  
ERROR 1062 (23000): Duplicate entry '1' for key 'tt.uk_c2'
```

这是一个非常知名的问题，自从 MySQL-5.6 引入了 Online DDL 之后就一直存在。相关的Bug包括:

- BUG#76895[1] Adding new column OR Drop column causes duplicate PK error
- BUG#77572[2] The bogus duplicate key error in online ddl with incorrect key name
- BUG#98600[3] Optimize table fails with duplicate entry on UNIQUE KEY
- BUG#104626[4] Remove failure of Online ALTER because concurrent Duplicate entry.

BUG#76895[5]最早报告了这个问题。BUG#76895 上提到的是 `Duplicate PRIMARY KEY` 问题，实际上是 `Duplicate UNIQUE KEY`。因为 MySQL-5.6 上有另一个 BUG#77572[6]在返回的失败消息中，错误的写了 `PRIMARY KEY`。BUG#77572 在 MySQL-5.6.28 和 MySQL-5.7.10 做了修复。后来的版本中人们看到的就是`Duplicate UNIQUE KEY` 错误，所以又有人报告了 BUG#98600.

BUG#76895中，MySQL官方回复`Not a Bug`，认为这是 Online DDL 带来的副作用，所以一直没有修复这个问题。然而这个问题会导致 DDL 操作失败，对 DBA 运维来说是非常痛苦的一个问题。一旦 DDL 失败，原本的运维、变更计划就会被打乱甚至推迟。所以社区又有人提报了 BUG#104626，将其作为功能需求提了出来。

这个 Bug 的根因是什么？为什么官方认为这不是一个Bug？ 有没有什么办法规避？要回答以上的三个问题，我们要先了解 Online DDL 的原理。

## Online DDL 的基本原理

为了实现在执行 DDL 期间可以同时执行 DML 操作，Online DDL 设计了一套`全量拷贝`加`增量回放`的机制。Online DDL 先将表中已经存在的全量数据应用到新表中(如果是添加索引则不需要创建新表，而是直接应用到新的索引上)。全量应用的过程比较长，期间允许执行DML，并记录增量。全量回放完成后，将增量应用到新表中。这个机制有两个关键点：

- 要有一个方法可以确切的区分一个特定时间点前后产生的数据，全量拷贝时只拷贝这个时间点前产生的数据。
- 这个时间点后对数据的更新都要记录到一个增量日志中，全量拷贝完成后通过回放增量日志来补齐所有数据。

### 如何区分全量和增量数据

Online DDL 中`全量数据`是`Clustered B+Tree`中的数据。Online DDL 通过扫描`Clustered B+Tree`来获取全量数据。在扫描的过程中允许 DML 操作，因此 B+Tree 上实际上包含了增量的修改。Online DDL 利用了 InnoDB 的 `MVCC （多版本并发控制）`机制来区分全量和增量数据。

InnoDB 的 MVCC 机制允许一个事务看到一个一致的数据快照，而不受其他并发事务的影响。其核心是 `read view`：read view 创建时会记录当前所有活跃事务的 ID 列表，之后通过这个列表判断每条记录的哪个版本对自己可见。对于 read view 创建之后提交的数据修改，即使已经写入了聚簇索引，read view 也能通过 undo log 找到旧版本来读取。

Online DDL 正是利用这个能力来划定全量拷贝的边界：`在一个确定的时间点创建 read view，全量拷贝只读取这个 read view 可见的数据。之后并发 DML 产生的变更，无论何时提交，都不会影响全量拷贝读到的内容。`

## Online DDL 的三个阶段

InnoDB 将 Online DDL 的过程分为三个阶段。三个阶段各自持有不同级别的`元数据锁(Metadata Lock, MDL)`，精确控制并发 DML 的行为。

### Prepare 阶段

Prepare 阶段在 `MDL_EXCLUSIVE` 保护下执行，此时这个表的所有读写都被阻塞。在这个"停写"窗口就是区分全量和增量数据的关键时间点。在这个阶段

1. 通过 `trx_assign_read_view()` 创建一致性读视图
2. 在相应的索引上初始化 row log（增量日志）， 并为索引设置 `ONLINE_INDEX_CREATION` 标记。后续索引 B+tree 上的增删改操作都会根据 `ONLINE_INDEX_CREATION` 来决定是否需要记录到 row log 中。

Row log 的安装和 read view 的创建都在 `MDL_EXCLUSIVE` 下完成。`MDL_EXCLUSIVE` 保证了两件事：

1. 确保 row log 和 read view 作为一个整体，能够无缝覆盖存量数据和数据变更。
2. 2. `排除正在执行中的事务`。如果没有 MDL\_EXCLUSIVE，可能存在一个事务已经修改了此表的数据但尚未提交。此时 row log 尚未安装，变更未被记录到 row log。尚未提交的事务在 read view 中标记为活跃事务，全量拷贝看不到。这条 DML 的修改既不在全量拷贝中也不在 row log 中，造成数据丢失。MDL\_EXCLUSIVE 确保此刻没有任何修改了此表的事务在执行，从而让 row log 和 read view 建立一个精确的分界点：`分界点之前已提交的数据由全量拷贝负责，分界点之后的所有 DML 都被 row log 捕获。`

Prepare 阶段完成后，MDL 从 `MDL_EXCLUSIVE` 降级为 `MDL_SHARED_UPGRADABLE`，这是一个共享锁。

### Execute 阶段

Execute 阶段 Online DDL 持有 `MDL_SHARED_UPGRADABLE`，`允许并发 DML 正常执行`。全量拷贝和第一次增量回放都在这个阶段完成。这是 Online DDL 中最耗时的阶段，也是 DML 不受阻塞的阶段。该表的 DML 操作都会记录到 row log 中。

### Commit 阶段

Commit 阶段 DDL 将 MDL 从 `MDL_SHARED_UPGRADABLE` 升级回 `MDL_EXCLUSIVE`，`阻塞所有读写操作`。在此保护下完成 row log 的回放，并提交 DDL 事务。由于 Execute 阶段已经回放了大部分 row log，Commit 阶段需要处理的数据很少，持有排他锁的时间很短。

这个阶段的 `MDL_EXCLUSIVE` 也保证了两件事：

- 保证 Execute 阶段所有修改了该表的 DML 事务都提交了。
- 阻塞新的 DML 执行，确保不会产生新的 Row Log。

从上面的内容可以看到 Online DDL并`不是全程 Online`，在 Prepare 和 Commit 阶段会阻塞 DML 操作。此外由于 Metadata Lock 是事务级别的锁，如果在 DDL 获取 MDL\_EXCLUSIVE 锁时有一个长事务在执行，DDL 就会被阻塞很长的时间。这个 DDL 在等待获取 MDL\_EXCLUSIVE 锁时也会阻塞其他 DML 因此导致此表不可读写。

## 全量拷贝

全量拷贝在 Execute 阶段执行，此时 MDL 为 `MDL_SHARED_UPGRADABLE`，并发 DML 可以执行。

全量拷贝调用 `row_merge_read_clustered_index()` 按 PRIMARY KEY 顺序从小到大扫描旧表的聚簇索引。扫描过程中对每一条记录通过 MVCC 判断可见性。只有 Read View 可见的数据才会被拷贝到新表中，`delete-marked` 的记录会被跳过。

由于 MVCC 的特性，即使并发 DML 在扫描过程中修改了旧表的数据，全量拷贝通过 undo log 仍然能读到 read view 时刻的旧版本。全量拷贝看到的始终是一个一致性快照，不受并发 DML 的影响。

## 增量日志记录

Row log 是对一个 B+tree 上数据页变化的记录，分两种情形：

- 重建表：只记录 clustered B+tree 上的变化，包括`INSERT`、`UPDATE`、`DELETE`。在回放 row log 时，这条记录会被`回放到所有的索引上`。
- 创建索引：只记录正在创建的索引的 B+tree 的变化，包括 `INSERT` 和 `DELETE`，没有`UPDATE`。因为这是正在创建的索引，所以 B+Tree 上的操作只会记录到 row log 中，而`不会真正的修改 B+tree`。这和重建表是不一样的，重建表的时候老表的所有索引都要实时更新。

这里要特别注意 row log 记录的时机。Row log 记录的是 B+tree 上的变化，所以是在一个 B+tree 的操作成功后记录的。如下图所示:`重建表场景下，当 clustered B+tree 插入成功后会记录 row log。在添加索引场景下，则是在 secondary index B+tree 插入时记录的。`添加索引的场景下不会真正的往 B+tree 上插入记录，只是记录一下 row log。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/Jg6JM4XTDccian4Lj1heMsw0Ajael82yHFpJiaiao53cq7w17Ak4wf7KYpibQicgXe6x2nQt6OWHf73TXzViadrMvNANG2xyT3QXSNzBXibrOuiaLQo/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=0)

### DML 回滚场景

当 DML 执行时，要先操作 clustered B+tree 然后操作 secondary index B+tree. 这里就有一个疑问，如果二级索引的操作失败了，会发生什么？

- 首先，这条 row log 仍然存在，不会被清理掉。row log 记录的是 B+tree 上的一次操作，这一次操作是确实发生了的。

• 其次， DML 语句会回滚。DML 回滚时，会根据 undo log 的内容，再一次操作 B+tree, 这一次的操作同样也会记录到 row log 中。因此一个失败的 INSERT 会记录两条 row log，一条是 ROW\_T\_INSERT, 一条是 ROW\_T\_DELETE，如下图所示。 两条 row log 按顺序执行后就相当于没有产生这条记录，符合回滚预期。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/Jg6JM4XTDcczrNSbfxJ6LkPo1a3a2AnonA6h4iaPfzSdib6GWPMZibewOdEYVtSeKQlsUzahF3KISo46HF4tVEVzeJg5hrV0eicibV98zvM0eFCo/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=1)

失败导致回滚只是其中一个特例，实际上所有的回滚操作都是这样的逻辑，包括用户手动执行 `ROLLBACK`。

## 增量日志回放

Row log 的回放一共执行两次：

- `第一次回放`在 Execute 阶段，全量拷贝完成后立即执行。此时 MDL 为 `MDL_SHARED_UPGRADABLE`，并发 DML 仍在继续，还会产生新的 row log。这次回放的目的是`尽量消化已有的 row log，减少 Commit 阶段持有排他锁的时间`。
- `第二次回放`在 Commit 阶段，MDL 已升级为 `MDL_EXCLUSIVE` ，不会有新的 row log 产生。这次回放处理剩余的少量增量数据，完成后新表的数据与旧表完全一致。

两次回放都是按写入 row log 的顺序逐条读取回放。对于重建表的场景，这条 row log 会被应用到新表的所有 Index 的 B+tree 上；对于创建索引的场景，则只会将 row log 应用到新创建的索引上。

## Duplicate Entry 的根因

在前面的`DML 回滚场景`中介绍了 INSERT 时即使 Unique Index 上发生了`Duplicate Entry`错误，也会记录 row log，而且记录的是两条。如下图所示：

![](https://mmbiz.qpic.cn/sz_mmbiz_png/Jg6JM4XTDce8ylyEaKDYyC1jAtDwBib0b8PFhUd5MQehjVh3iaPgVib56tSf45DZIYRbufJWN9vNUvk9quDHK4IrbMgDeVqibbvibavRFbiak4Y9E/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=2)

在回放第一条 row log`ROW_T_INSERT`时，和 INSERT 语句的执行逻辑是一样。要先插入一行记录到 clustered B+tree，然后到每个索引上插入一条记录。因此当向唯一索引插入这条记录时，同样也会报`Duplicate Entry`的错误。正是这个 duplicate entry 导致了 DDL 语句的失败。

### 测试用例

``` sql
CREATE TABLE t1 (  
  c1 INT AUTO_INCREMENT PRIMARY KEY,  
  c2 INT,  
  UNIQUE KEY (c2)  
) ENGINE=InnoDB;
```

开启 3 个会话分别执行如下的语句

``` sql
# Session 1  
BEGIN；  
INSERT INTO t1 VALUES(NULL, 1);  
  
# Session 2  
ALTER TABLE t1 ENGINE = InnoDB;  
  
# Session 3  
INSERT INTO t1 VALUES(NULL, 1); 
```

执行完上述的语句后，将 Session 1 的事务提交。然后你会发现 Session 2 的 ALTER 和 Session 3 的 INSERT 都报了 `Duplicate Entry`

``` sql
# Session 2  
mysql> ALTER TABLE t1 ENGINE = InnoDB;  
ERROR 1062 (23000): Duplicate entry '1' for key 't1.c2'  
  
# Session 3  
mysql> INSERT INTO t1 VALUES(2, 1);   
ERROR 1062 (23000): Duplicate entry '1' for key 't1.c2'
```

这个例子利用了前面提到的 Online DDL 的 `Metadata Lock` 的机制来构造。

- Session 1 的 INSERT 首先持有了 t1 的 `MDL_SHARED_WRITE` lock，这个 lock 在事务提交时才会释放。
- Session 2 的 ALTER 在 prepare 阶段时 需要申请 `MDL_EXCLUSIVE` lock, 被 Session 1 阻塞。
- Session 3 的 INSERT 同样要申请 `MDL_SHARED_WRITE` lock，但是被 Session 2 阻塞。

通过 performance\_schema 的 metadata\_locks 表可以看到这些 session 上的metadata locks。如下图所示：其中第一行是 Session 1 持有的锁。

![](https://mmbiz.qpic.cn/mmbiz_png/Jg6JM4XTDcev9QnoEBBPTuUV5ThNES3thgL8gQCFKNbgpk8WaF5zmL4G6sjSibOvqk0IqDOicV15twSY8kqhphSNmWFxjwXRmj7rcVQ40Vwrs/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=3)

当 Session 1 的事务提交后， Session 2 获得`MDL_EXCLUSIVE`lock，执行完 prepare 阶段后，降级到`MDL_SHARED_UPGRADABLE`lock。这个锁和`MDL_SHARED_WRITE`不冲突，所以 Session 3 获得了`MDL_SHARED_WRITE`开始执行。执行的过程中因为`c2 = 1`已经存在，就报了`Duplicate key`的错误。这个过程中记录了 row log，因此也导致了 Session 2 ALTER 语句的错误。

![](https://mmbiz.qpic.cn/mmbiz_png/Jg6JM4XTDceM8qtdCiaY3Yt6ZDicBMByoN2T2DUVxK3VtbYlbGRZGGSrXLwjVCT25tTQibJwasia1aiafzmxmSB9Fl6XLOJlAHbNsvfEt7IQVlVY/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=4)

### 如何避免这个问题

在理解了这个问题产生的原因后，我认为这就是一个 Bug，于是我们在 AliSQL 上修复了这个问题。如果你使用的是社区版 MySQL, 要想避免这个问题，那就是尽量去避免 DML 执行时出现 `Duplicate Entry` 的情况。

一个常见的场景是给自增主键的表添加了唯一索引，业务在 INSERT 一条记录时，自增键由 MySQL 产生。业务上有重试的逻辑，一旦前一个 INSERT 慢了，可能就会在另一个会话里重试这个SQL。从业务上来说逻辑是合理的，因为有唯一索引，只有一个 INSERT 能成功。但是恰恰就是这个逻辑，导致了 DDL 的失败。因此在 DDL 期间可以尝试加大重试的超时时间来避免这个问题。

## AliSQL 上的优化

AliSQL 上选择了一个比较直观，也比较简单的方案: `当碰到 Duplicate Entry 错误时，忽略掉这个错误。`

### 真重复、假重复

这个忽略的策略是有限制条件的，并不是所有的 `Duplicate Entry` 都能忽略掉。

在 Online DDL 期间还存在一种`真重复`的场景：`DDL 引入了新的唯一性约束(新增/更改主键,或新增 UNIQUE 索引)，原本的数据中存在重复记录`。新增的唯一性约束在 Online DDL 期间是不生效的，因此这期间的 DML 操作仍然能够引入重复的记录。`当碰到这样的情况时，DDL 的执行必须要失败`。

如果 `Duplicate Entry` 发生在原表已经存在的唯一索引上，一定是`假重复`，可以被跳过。这是最常见的场景，AliSQL 的优化的就是这个场景。

### 后续 Undo Row Log 的处理

前面说了 DML 执行失败时，实际上记录了两条 row log。以 INSERT 为例：

```
<ROW_T_INSERT, pk1, ...>  
<ROW_T_DELETE, pk1>
```

Row Log 回放时，往唯一索引 B+tree 上插入数据失败，被忽略掉了。但是主键 B+tree 上是插入成功了的。当回放第二条日志时，主键 B+tree 上的记录会被删掉，这个没有问题。但是唯一索引上本来就没有插入成功，就找不到这条记录，这会导致回放失败。

因此当遇到已经存在的唯一索引上的`Duplicate Entry`错误时，我们需要将这个错误记录下来。当回放回滚产生的 row log 时，同样也跳过这个索引上的操作。如下图所示：

![](https://mmbiz.qpic.cn/sz_mmbiz_png/Jg6JM4XTDccs6064s0dt0DP5IibozsZN1v80wm4YyK6wJA7E69rJblUuWVlFsib83ZYlynPuJWnYpgict6Fg2yToLIw2wH00saZ6oJg0GyYb58/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=5)

UPDATE 的情况会更加复杂，如果一个 UPDATE 修改了一个二级索引列，要在二级索引上执行两个操作：

1. 删除老的记录
2. 插入新的记录

回放 row log 时的失败发生在第 2 步，因此执行后续的回滚操作时就不能简单的跳过这个索引上的所有操作，而是要跳过第1步，第2步还要执行, 如下图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/Jg6JM4XTDcchvhEicJaQKlv5hL46qMs55Vsp26MXza5KFaOcEcbviakTQKvVia0QmB3lHYq7ZKCMjDIqRAsGZV9H3b66wFickvPucDiaGfdAK0QE/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=6)

通过以上的设计，AliSQL 避免了 Online DDL 时不必须要的`Duplicate Entry`错误。

## 总结

Online DDL 有时会报 `Duplicate Entry` 的错误，导致 DDL 执行失败。这个报错具有不确定性，很难完全避免。一旦 DDL 失败，原本的运维、变更计划就会被打乱甚至推迟。这个问题产生的原因是 Online DDL 期间有并发的 DML 出现了 'Duplicate Entry' 错误导致的。 DML 的这个错误通过 Online DDL 的 row log 传到到了 DDL 上，导致了 DDL 的错误发生。 AliSQL 上对 Online DDL 做了优化，针对已经存在的唯一索引上发生的 `Duplicate Entry` 错误做了忽略处理，让 Online DDL 不再因这个错误而中断。

#### 引用链接

`[1]` BUG#76895:*https://bugs.mysql.com/bug.php?id=76895*  
`[2]`BUG#77572:*https://bugs.mysql.com/bug.php?id=77572*  
`[3]`BUG#98600:*https://bugs.mysql.com/bug.php?id=98600*  
`[4]`BUG#104626:*https://bugs.mysql.com/bug.php?id=104626*  
`[5]`BUG#76895:*https://bugs.mysql.com/bug.php?id=76895*  
`[6]`BUG#77572:*https://bugs.mysql.com/bug.php?id=77572*
