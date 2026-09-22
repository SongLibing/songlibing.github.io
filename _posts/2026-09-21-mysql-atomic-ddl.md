---
title: Atomic DDL揭秘
description: "解析 MySQL-8.0 Atomic DDL 的实现原理，介绍 Data Dictionary、InnoDB DDL Log、Binlog Crash Safe，以及各类 DDL 的 Redo、Undo 和 Crash Recovery 流程。"
date: 2026-09-21 23:43:00 +0800
categories: [数据库, MySQL]
tags: [MySQL, InnoDB, DDL, Atomic DDL]
toc: true
---

在MySQL-8.0之前DDL是不能做到Crash Safe的。主要的问题有3个:

1. Server层的Metadata和InnoDB的Metadata/数据不一致。  
   Server层的Metadata是记录在文件中的。例如表结构信息保存在frm文件中。InnoDB中也有一份Metadata存储在表中。Crash可能导致server层的Metadata和InnoDB中Metadata、甚至表中的数据不一致。比如Server层存在表的frm文件，认为表存在。但是InnoDB的ibd数据文件不存在了，InnoDB认为表不存在。
2. InnoDB的Metadata和数据不一致。
3. Binlog和数据的不一致  
   比如Crash重启后，表已经存在了但是CREATE TABLE却没有记录到Binlog中。

MySQL-8.0中为了实现Atomic DDL，彻底解决以上的问题做了三个方面的改动：

1. 去掉Server层的Metadata文件，所有Metadata统一存放到InnoDB表中。  
   这些表被称为Data Dictionary。用户、Server层、引擎都可以通过DD的访问接口查询或者更新Metadata.
2. DDL_log  
   InnoDBL中实现了DDL_log表，在DDL操作过程中会记录一些DDL的操作日志。InnoDB通过DDL log来保证DDL中的文件操作和Metadata操作的原子性。
3. Binlog DDL Crash Safe  
   在DDL的Binlog Event中会记录DDL操作的Xid,通过Xid来保证binlog的crash safe。

## DDL事务

实现DDL原子性的基本思路是把DDL的过程变成一个事务。通过事务的原子性特性来保证DDL的原子性。MySQL-8.0将所有的Metadata保存到了InnoDB的表中，对Metadata的操作实际上就是对InnoDB表做INSERT, UPDATE或者DELETE操作。Metadata的操作完全是可以通过一个事务来完成的。对于只修改Metadata的DDL操作通过一个事务就可以保证其原子性。比如View, Function, Trigger的创建、修改、删除操作。

DDL执行的过程中会开启一个事务，这里把它称作DDL Trx。当DDL Trx Commit则等同于DDL执行成功了。Commit前DDL的所有操作是可以回滚。Commit之后进行的操作则不能回滚。

## InnoDB DDL Log

对于涉及到数据文件操作的DDL，InnoDB将文件的操作指令写入DDL Log表中。将DDL Log表的操作和Metadata表的操作放入同一个事务来实现DDL操作的原子性。

涉及数据文件操作的DDL包括：

- CREATE TABLE
- ALTER TABLE
- DROP TABLE
- RENAME TABLE
- CREATE INDEX
- DROP INDEX
- DROP DATABASE

### DDL Log Table

DDL Log Table 定义如下：

```sql
CREATE TABLE mysql.innodb_ddl_log (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  thread_id BIGINT UNSIGNED NOT NULL,
  type INT UNSIGNED NOT NULL,
  space_id INT UNSIGNED,
  page_no INT UNSIGNED,
  index_id BIGINT UNSIGNED,
  table_id BIGINT UNSIGNED,
  old_file_path VARCHAR(512) COLLATE UTF8_BIN,
  new_file_path VARCHAR(512) COLLATE UTF8_BIN,
  KEY(thread_id)
);
```

DDL Log Table中记录了以下几种操作指令:

- DELETE SPACE  
   删除指定的table space文件
- DROP  
   从mysql.innodb_dynamic_metadata表中删除指定表的信息. 
- RENAME SPACE  
   重命名指定表的table space文件
- RENAME TABLE  
   重命名指定表，包括修改metadata,以及统计信息中的表名等操作。
- FREE  
   删除指定的索引
- REMOVE CACHE  
   从table cache中移除指定的表

### DDL Log 的使用

DDL语句的执行被分成了两部分：

1. DDL Trx相关的操作  
   DDL Trx执行的过程中会往DDL Log Table写入一些操作指令。
2. Post DDL操作  
   读出写入DDL Log Table中的指令，按反序执行。不论DDL Trx是成功还是失败，DDL Log表中记录的指令都会被执行（如果有的话)。

DDL Log可以看作是Redo Log和Undo Log的一个合集。有些DDL把它用作Redo，有些DDL把它用做Undo，还有些DDL会把它同时当作Redo和Undo。

### DDL Log当作Redo的用法

![](/assets/img/atomic-ddl-01.svg)

DROP TABLE把DDL log当作Redo来用。

- 在DDL Trx阶段， DROP TABLE会删除metadata，并且往DDL Log中插入一条DELETE SPACE的指令。DDL trx commit后，就不能回滚了。
- Post DDL阶段，根据DDL Log中的DELETE SPACE指令，进行实际的文件删除操作。
- 文件删除操作完成后，删除DDL Log Table中的记录。  
   DDL Log Table表中的指令都是幂等的，多次执行不影响正确性。所以执行DDL Log Table的执行部分和删除DDL Log Table中记录的部分不是原子的。

![](/assets/img/atomic-ddl-02.svg)

当DDL Trx阶段有错误发生时，DDL Trx会回滚。DDL Log Table中的 DELETE SPACE会回滚掉。所以，Post DDL阶段就什么也不做。

### DDL Log当作Undo的用法

![](/assets/img/atomic-ddl-03.svg)

CREATE TABLE则将DDL log当作Undo来用。

- DDL Trx执行阶段，CREATE TABLE首先在DDL Log中记录一条DELETE SPACE的指令，这个记录要用另外一个事务执行(这里称作DDL Log Trx), 并提交。
- 然后在DDL Trx中 删除DDL Log Table中的对应的DELETE SPACE记录。
- 最后创建表文件，执行成功后提交DDL Trx。
- DDL Trx如果提交了，DDL Log Table的DELETE SPACE记录已经被删除，所以Post DDL阶段什么也不做。

![](/assets/img/atomic-ddl-04.svg)

当错误发生后, DDL Trx会回滚。DDL Log Table的DELETE SPACE记录将会保留下来，Post DDL阶段会根据DDL Log Table中的记录将表文件删除掉，并且删除DDL LogTable中的记录。

### innodb_print_ddl_logs

为了方便调试，InnoDB提供了一个功能，可以将ddl log表的所有操作都记录到error log里。这个功能可以通过innodb_print_ddl_logs变量动态的开关。

下面我们就看一下几个DDL的详细过程。

### CREATE TABLE

CREATE TABLE将DDL Log当作Undo来使用，当DDL执行失败时通过DDL Log来回滚创建文件的操作。

```sql
CREATE TABLE t1(c1 INT PRIMARY KEY, c2 VARCHAR(20), INDEX(c2));
```

该DDL对DDL Log Table的操作如下：

![](/assets/img/atomic-ddl-05.png)

1. 通过DDL Log Trx(1802)记录DELETE SPACE回滚指令。它的记录ID是7.
2. 通过DDL Trx(1801),删除 DDL Log Table的ID为7的记录。
3. 将t1加入Table Cache。Table Cache的回滚依赖于DDL Log Table，所以先通过DDL Log Trx(1803)往DDL Log Table中添加REMOVE CACHE的记录，然后在DDL Trx(1801)中删除这条记录。
4. 之后的两个FREE分别是Clustered Btree和Index Btree的回滚日志。对于CREATE TABLE来说删除B-Tree的操作是不必要的，直接删除文件就可以了。DROP TABLE就没有删除B-Tree的过程。
5. 当DDL Trx(1801)提交后，之前添加到DDL Log中的所有记录都被删除了，所以Post DDL阶段什么也没有做。

### DROP TABLE/DROP DATABASE

DROP TABLE将DDL Log当作Redo来使用，当DDL操作成功后。在Post DDL阶段时，根据DDL Log Table中的记录执行文件的删除操作。

```sql
DROP TABLE t1, t2;
```

该DDL对DDL Log Table的操作如下：

![](/assets/img/atomic-ddl-06.png)

1. 记录DROP指令用来删除t1在mysql.innodb_dynamic_metadata中的信息。innodb_dynamic_metadata是一个比较特殊的表，该表的操作不记录Undo Log，不能在DDL Trx里操作。因此会记录一条指令到DDL Log Table，然后放到post ddl阶段执行。
2. 然后记录DELETE SPACE指令用来删除db1/t1.ibd
3. 接着对表t2重复以上的操作。
4. 当所有表的删除操作完成后，提交DDL trx(1916).这时DDL Log Table中插入了4条记录.
5. 在Post DDL阶段时，会按反序执行这4条记录，执行实际的删除操作。Redo实际上是可以不按顺序执行的，但是Undo是必须要反序执行的，因为会有前后依赖关系。估计是为了保持简单和统一，Post DDL时统一按反序执行。

DROP DATABASE对DDL Log Table的操作等同于DROP指定数据库中的所有表。

### CREATE INDEX

CREATE INDEX将DDL Log当作Undo操作来使用, 当创建Index失败时通过DDL Log Table中的记录回滚正在创建的Index.

```sql
ALTER TABLE t2 ADD INDEX ind1(c3);
```

该DDL对DDL Log Table的操作如下：

![](/assets/img/atomic-ddl-07.png)

1. 创建B-Tree前，通过DDL Log Trx(1872)记录FREE 回滚指令。它的记录ID是30.
2. 通过DDL Trx(1871)删除DDL Log Table中的FREE指令。
3. 通过DDL Trx更新Metadata，并创建B-Tree。
4. 最后提交DDL trx(1871)。事务提交后DDL Log Table中的记录已经被删除，Post DDL阶段什么也不做。

### DROP INDEX

DROP INDEX将DDL Log当作Redo操作来使用。

```sql
ALTER TABLE t2 DROP INDEX ind1;
```

该DDL对DDL Log Table的操作如下：

![](/assets/img/atomic-ddl-08.png)

1. 通过DDL Trx，在DDL Log Table中插入一条FREE指令。
2. 更新Metadata后提交DDL Trx。
3. Post DDL阶段，根据DDL Log Table中的Free指令删除Index的B-Tree.

### RENAME TABLE

RENAME TABLE将DDL Log当作UNDO操作来使用。

```sql
RENAME TABLE t2 TO t20;
```

‍该DDL对DDL Log Table的操作如下：

![](/assets/img/atomic-ddl-09.png)

1. 重命名文件前，通过DDL Log Trx(1909)，在DDL Log Table中插入一条RENAME SPACE指令。这是Undo记录，所以这条指令是将t20.ibd重命名为t2.ibd.
2. 通过DDL Trx将DDL Log Trx中的RENAME SPACE记录删除。
3. 通过DDL Log Trx(1909)，在DDL Log Table中插入一条RENAME TABLE指令。这是Undo记录，所以这条指令是将t20重命名为t2.  
   RENAME时需要修改一些内存结构，另外修改innodb_table_stats中的表名是通过单独的事务更新的，无法回滚。所以这里记录一个RENAME TABLE指令，通过反向命名实现回滚。
4. 通过DDL Trx将DDL Log Trx中的RENAME TABLE记录删除。
5. DDL Trx(1908)事务提交后, DDL Log Table中的记录已经被删除，Post DDL阶段什么也不做。

### ALTER TABLE

ALTER TABLE有很多种，最为复杂的是需要重建表的情形。在这种情况下，DDL Log即是Redo，也是Undo。

```sql
ALTER TABLE t2 ADD COLUMN c3 int;
```

该DDL对DDL Log Table的操作如下：

![](/assets/img/atomic-ddl-10.png)

ALTER TABLE集合了CREATE TABLE, RENAME TABLE和DROP TABLE的操作：

1. 创建临时表db1/#sql-ib1066-677171833，记录删除该临时表的Undo指令。
2. 重命名db1/t2为db1/#sql-ib1071-677171834，记录反向命名的Undo指令。
3. 重命名db1/#sql-ib1066-677171833为db1/t2，记录反向命名的Undo指令。
4. 删除表db1/t2为db1/#sql-ib1071-677171834，记录删除文件的Redo指令。

### DDL Log用法总结

- DDL的执行过程分为DDL Trx阶段和Post DDL阶段。
- DDL Trx 阶段向DDL Log Table中插入Redo或/和Undo指令。
- Post DDL阶段执行DDL Log Table中的指令。
- 当作Redo使用时，通过DDL Trx向DDL Log Table插入Redo指令。Post DDL阶段执行这些指令。
- 当作Undo使用时，则通过DDL Log Trx向DDL Log Table插入Undo指令。然后通过DDL Log Trx删除这些Undo指令。如果DDL Trx提交了，Post DDL阶段什么也不做。如果DDL Trx回滚了，Post DDL阶段按照DDL Log Table中的指令进行归滚操作。
- DDL Log Table中的指令都是幂等的，可以多次执行不影响正确性。

## DDL Crash Recovery

Crash可能发生在Post DDL完成之前，所以重启后做Recovery时，Server需要检查DDL Log Table,并按DDL Log Table中的记录完成所有Post DDL操作。

## Binlog Crash Safe

![](/assets/img/atomic-ddl-11.png)

现在每个DDL都是一个事务了，事务过程中失败了可以回滚。事务提交则意味着DDL执行成功。当binlog开启时DDL的事务也会采用两阶段提交。MySQL-8.0对Binlog的Query_log_event做了扩展，DDL事务的Xid会存储到Query_log_event中了。

![](/assets/img/atomic-ddl-12.png)

这样在做Crash Recovery的时候就可以根据DDL的Query_log_event中的Xid来决定提交或回滚Prepare状态的DDL事务了。如果想了解Crash Recovery的原理，可以参考《[MySQL的CrashSafe和Binlog的关系](http://mp.weixin.qq.com/s?__biz=MzIyMTQ1NDE0MQ==&mid=2247483833&idx=1&sn=d96f2994b9dc4d566bbc7ac5a92b3aad&chksm=e83dcbebdf4a42fd8ece0252dbfa46a65d8c5bca4297a7ca4c93ea60f596ea145629b9abedfc&scene=21#wechat_redirect)》.

## CREATE TABLE ... SELECT

CRETE TABLE ... SELECT 在Binlog中会被分成了CREATE TABLE和INSERT两部分(Row Format)， 如下所示：

```text
CREATE TABLE t1 // Query_log_event

BEGIN           // Query_log_event
                // Table_map_log_event
INSERT          // Write_rows_log_event

COMMIT          // Xid_log_event
```

MySQL-8.0之前，CREATE部分和INSERT部分是两个独立的事务，所以没办法实现原子性。因此当开启GTID后，CREATE TABLE ... SELECT是被禁止的。

在实现了Atomic DDL后，MySQL-8.0对CREATE TABLE ... SELECT做了改进。现在CREATE TABLE部分和INSERT部分可以在同一个事务中执行，并且保证原子性。因此开启GTID时，也可以使用CREATE TABLE ... SELECT语句了。

```text
BEGIN                   // Query_log_event
CREATE TABLE t1 （...
...）START TRANSACTION  // Query_log_event

                        // Table_map_log_event
INSERT                  // Write_rows_log_event

COMMIT                  // Xid_log_event
```

下图是一个实际的例子：

![](/assets/img/atomic-ddl-13.png)

可以看到, MySQL对CREATE TABLE语句做了扩展，从而实现CREATE TABLE和DML在同一个事务中执行。

```text
CREATE TABLE ... START TRANSACTION;
```

DDL在执行完毕时，会自动COMMIT当前的事务。有了START TRANSACTION这个扩展后，CREATE TABLE语句就不会自动结束当前事务了，所以后续的DML就和CREATE TABLE在同一个事务中执行了。

从这个改进可以看到，在实现了Atomic DDL之后。让DDL支持事务就不难了，今后也许更多DDL可以和DML放到同一个事务中执行。

## 参考

- Atomic DDL: 

https://dev.mysql.com/doc/refman/8.0/en/atomic-ddl.html

- WL#13355: Make CREATE TABLE...SELECT atomic and crash safe: 

https://dev.mysql.com/worklog/task/?id=13355

- WL#9536: https://dev.mysql.com/worklog/task/?id=9536  
   InnoDB_New_DD: Support crash-safe DDL
