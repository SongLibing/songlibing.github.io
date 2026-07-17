---
title: MySQL DROP TABLE 的性能问题与优化
date: 2026-04-10 14:56:21 +0800
categories: [数据库, MySQL]
tags: [MySQL, InnoDB, DROP TABLE, 性能优化]
toc: true
---

以下信息是在测试一个 SQL 时，AliSQL 的 Performance Agent 功能采集的 buffer pool 相关的监控信息。你能从这些监控信息看出是什么样的 SQL 在执行吗？

![](https://mmbiz.qpic.cn/sz_mmbiz_png/ibf4J5w9SFz6sT26TSXzbgQTWbziaOmDGpQxiaEotib6cEUKicUvOUZ0w1HvicBuSzH8totUzoQf6icwc6VPLaR5rA5ZQ/640?wx_fmt=png&from=appmsg)

图中：bp = buffer pool, pg = page。  
- bp_pg_total 是 buffer pool 的总 page 数，一般不会变化的，除非修改了 buffer pool 的大小。  

- bp_pg_data 指的是 buffer pool 中数据文件的 page 的数量，包括 undo page。  

- bp_pg_dirty 指的是数据页中的脏页(被修改了，但还没有写入数据文件的页)的数量。  

- bp_pg_free 是 buffer pool 中空闲的 page 数量，这部分是没有被使用的部分。  

- bp_wait_free 当从存储上 读一个 page 到 buffer pool 时，如果没有空闲的 page 也没有净页(没有被修改的) 可释放，则需要等待一个脏页被刷盘后释放。bp_wait_free 记录的是每秒等待的次数。  

- bp_reads 是每秒从存储上读取的 page 的数量。  

- bp_read_req 是每秒读取的 page 总数量，包括从 buffer pool 和磁盘读取的 page 数。  

 以上的信息取自于 InnoDB 提供的 buffer pool 状态变量（`Status`），Performance Agent 每 1 秒采集一次，将这些信息存储到了文件中。![](https://mmbiz.qpic.cn/sz_mmbiz_png/ibf4J5w9SFz4QXmTF6QtNyqiamvDMyesOibEbDU6Pvh3rxHZX51PPWVIOuic4TUVzBEeoKUc7cLicjyHpDYiaPIfcXvg/0?wx_fmt=png&from=appmsg)

## 现象分析
### bp_reads 和 bp_read_req
上面的监控信息可以看到 bp_reads 非常的少，这说明被访问的数据页几乎都在 buffer pool 中，因此不需要从存储上读取。`当需要判断问题是否是由 IO 引起的时候，这两个指标非常重要`。

### bp_pg_free

刚开始的时候 bp_pg_free 的数量是 `1024`，这个值是由参数 `innodb_lru_scan_depth` 控制的，默认是 1024。当 free page 保持在这个值附近时，说明 buffer pool 的空间已经用完了。为了避免因没有 free page 而导致的等待，后台的 page cleaner 线程会扫描 LRU 链表最后的 1024 个 page，如果能释放则释放掉这些 page。

当一个正在执行的 SQL 需要从磁盘读一个 page 时，首先会尝试从 free list 获取一个空闲页。如果此时没有空闲页，当前线程则会扫描 LRU 链表去释放一个净页，如果没有找到合适的净页，则会尝试释放一个脏页。释放脏页就要刷脏，时间会比较长，会导致 SQL 变慢，活跃连接增加，严重时实例可用性受损。下图是这种情形下的一个函数调用栈，可以看到实例超过`59%`的cpu时间都是在获取一个free page(`buf_LRU_get_free_block`)，其中`45%`都是因为没有free page或者clean page而不得通过刷脏来获取一个free page(`buf_flush_single_page_from_LRU`)。![](https://mmbiz.qpic.cn/sz_mmbiz_png/ibf4J5w9SFz4QXmTF6QtNyqiamvDMyesOibznO60CaxtZZWyk4iaFia5T44MJZDtXF1F6IynsiaWsK3ffmSj62vlticEA/0?wx_fmt=png&from=appmsg)

对于规格比较大的实例，可以将 `innodb_lru_scan_depth` 调大来提高实例的稳定性。注意这个参数控制的是单个 `buffer pool instance` 的空闲页数量，如果实例设置了 8 个或者更多的 buffer pool instance，则空闲页的总数量是 `8 * innodb_lru_scan_depth`。

### bp_pg_free + bp_pg_data < bp_pg_total

- 在`00:18:46`时 `空闲页` 与 `数据页` 之和比 buffer pool 的 `总页数` 少了大约 9 万个页。
- 从`00:18:46`到`00:21:15`这段时间，空闲页的数量持续的增加，但是数据页却没有减少。
- 直到`00:21:15` `空闲页` 与 `数据页` 之和才接近了`总页数`。

那9万多个页那里去了呢？除了数据页之外，`行锁和自适应哈希索引（AHI）也会占用 buffer pool 的内存`。AHI 占据的内存比例通常不会特别大，因为只有已经加载到 buffer pool 中的页上的数据可能会产生 AHI 记录。当一个页被淘汰时，它的 AHI 记录也会被清理掉。而行锁是在事务提交时才会释放和页的淘汰无关。比如全表更新这种操作，很可能产生大量的行锁记录。InnoDB 限制这些额外占用的 buffer pool 内存不能超过 buffer pool 总大小的 75%（`buf_LRU_buf_pool_running_out()`）。如果超过了，在申请行锁时则会报 `ERR_LOCK_TABLE_FULL` 错误。

```
The total number of locks exceeds the lock table size
```

从上面描述的现象看，很像是在持续的释放 AHI 记录或者是行锁占用的 page。

### 数据页突然大量减少
`00:21:16` 这一秒减少了大约 `62 万`个数据页，这些数据页都变成了空闲页。短时间内释放这么多的数据页，通常和 tablespace 的删除、重建相关，因为这些操作会把当前 tablespace 的所有页从 buffer pool 中清除掉。因此可能是以下几种操作：

- 删除表、表分区的操作
- 清空表、表分区的操作
- 重建表的操作，包括OPTIMIZE TABLE和许多Inplace/Copy DDL
- Undo空间的缩容和删除

Undo空间的缩容和删除不会有 AHI 或者行锁的释放行为，重建表前期往往有大量的IO操作，所以`最可能的还是删除或者清空表或者表分区`。如果是这两个操作之一的话，那么这个实例应该是个 MySQL 5.7 或者 MySQL 8.0.23 以前的实例。

MySQL 8.0.23 实现了`快速清空、删除表空间`的功能。简单地说就是给 tablespace 添加了`版本信息`，每个 page 都记录有当前的 tablespace 版本。Drop/truncate table 时，只是将 tablespace 的`版本加 1`，不会清除这个 tablespace 的数据页。这些遗留下来的页称为 `stale page`，当从 buffer pool 获取一个页时，就要检测当前页是否是 `stale page`。如果是就直接放到free list中。详细信息可以参考 [WL#14100: InnoDB: Faster truncate/drop table space](https://dev.mysql.com/worklog/task/?id=14100)。

## DROP TABLE 带来的问题
以上的监控信息是一个 RDS MySQL 5.7 的 `DROP TABLE` 语句产生的，这个语句执行了大约 `150` 秒。它导致了主实例长时间的不可写，因此发生了`主备切换`。这里最大疑问是
- 为什么`单个表`的 DROP TABLE 语句会导致`所有的表`都不可写？
- 为什么会持续这么长的时间？

导致 HA 切换的原因如下：  
- 这个表上有大量的 `AHI 记录`，DROP TABLE 时需要清理这些 AHI 的记录，花费了 `149` 秒的时间。  
- DROP TABLE 清理 AHI 记录时持有 `dict_sys::mutex`。dict_sys::mutex 是内部字典系统的全局锁，因此所有访问字典系统的行为都会被阻塞。  
- DML 操作时，`before_dml hook` 中需要获取当前表的外键信息，这个操作需要访问字典系统，因此被 `DROP TABLE` 阻塞。HA 写操作探测失败，触发了主备切换。DML当时的函数栈如下图所示![](https://mmbiz.qpic.cn/sz_mmbiz_png/ibf4J5w9SFz4AGQ1X31Waz40pkkbH5CDuWvKgicUFyErmKEkNjRchibnFRiaGetx1JwSFmJnCQGyU4LeLd3mxOKY9g/0?wx_fmt=png&from=appmsg)

## before_dml hook 优化
事务的钩子系统是binlog复制为了实现对事务的控制设计的，一共有5个钩子：
- before_dml
- before_commit
- before_rollback
- after_commit
- after_rollback

Plugin 如果需要在相应的阶段做一些事情，需要实现一个`事务观察者(Trans_observer)`，将其注册到钩子系统里。`semi-sync` 和 `group replication` 都会注册一个事务观察者，当安装了 semi-sync plugin 或者 group replication plugin 后，before_dml hook 就会被执行，plugin 内部会根据开启的状态进行不同的处理。before_dml hook 在调用plugin相应函数前，需要初始化一些参数的信息，`外键的信息`也是在调用 plugin 相应函数前获取的。所以哪怕semi-sync 或者 group replication `没有开启`，也会触发上面例子里的问题。如果要避免这个问题，就必须要`卸载(UNINSTALL PLUGIN)`了 semi-sync plugin. 

before_dml hook 目前仅在 `Group Replication` 中有用，Group Replication 不支持 `CASCADE` 操作，会在 `before_dml hook` 中做外键级联操作的检查。这个操作的效率是非常低，没有开启 Group Replication 的情况下，是没有必要做检查的，因此`AliSQL-5.7上去掉了这个hook`。`MySQL-8.0` 上由于采用了新的 Data Dictionary 机制，这个 hook 里已经不需要持有 `dict_sys::mutex` ，所以不会被阻塞。

## dict_sys::mutex
`dict_sys::mutex` 是用来保护字典系统的。字典是InnoDB的内部表和内存中的缓存结构，用来存储表的元信息。比如表结构信息、索引信息、外键约束信息等等。读取或者修改字典表和缓存结构的内容需要持有dict_sys::mutex，因此字典是完全串行的访问模式，访问效率非常低。

字典的访问还有以下几种情况：
- 访问的表不在 `Table Cache` 中，需要创建新的 `TABLE` 对象。
- DML操作的表有外键时，做外键检查时需要持有 dict_sys::mutex;
- DDL 修改表的元数据信息。
- 通过 information_schema 获取表的元数据信息。
- 无主键表上会有一个 InnoDB 内建的 `row_id` 字段，获取全局 `row_id` 时也需要持有 dict_sys::mutex。

所以需要持有 dict_sys::mutex 的情形是非常多的，对于 DROP TABLE 过程还是要做更多的优化，尽量减少持有 dict_sys::mutex 带来的全局影响。MySQL-5.7上 DROP TABLE 在持有 dict_sys::mutex 时主要做的事情包括:
- 获取 `dict_sys::mutex`
- 删除 Dictionary 里的元数据信息。
- 删除所有索引（包括聚簇索引）。 
- 删除 ibd 文件
- 释放 `dict_sys::mutex`

## Adaptive Hash Index 清理优化
 从上面的原因来看，关闭 InnoDB 的 Adaptive Hash Index 就应该能解决这个问题了吧？确实对于上面的案例来说，关闭 AHI 后就能够解决问题。InnoDB 中的 AHI 功能本身的实现是不够好的，关于 AHI 导致的稳定性的 bug 有一大堆，因此`关闭 AHI 也是强烈推荐的做法`。
 
 但是`关闭 AHI 就万事大吉了吗？` 上面测试中表的大小是`10GB`，如果你拿一个`1TB`的表去做`DROP TABLE`操作，你会发现即使关闭了AHI，它仍然会阻塞所有的 DML 很长的时间。通过下面的 CPU 热点图，我们可以看到大量的时间仍然耗费在了清理 AHI 相关的函数上(`btr_search_drop_page_hash_when_freed()`)。![](https://mmbiz.qpic.cn/sz_mmbiz_png/ibf4J5w9SFz7FZjEndCKDMKHgDUQt2dVB9fc16bW8mWY26JLCicYcYQmian5RzQBCTRfkulOu3UFrbR3DrzhIzGeQ/0?wx_fmt=png&from=appmsg)

清理 AHI 记录是融合在删除索引的过程中的。删除索引时会读取 Index B+Tree 上的 `File Segment` 中记录的 `Extent` 信息，逐个Extent 释放。为了释放 AHI 的记录，会检查 Extent 中的每一个 Page 是否有 AHI 的记录， 如果有就释放。具体执行分为三步：
1. 检查 page 是否在 buffer pool 中。
2. 如果 page 在 buffer pool 中则检查 page block 的 index 是否为 NULL。为 NULL 则表示这个 page 上没有 AHI 记录。
3. 如果 page 上有 AHI 记录，则读取每一行记录计算 hash 检查是否有对应的 AHI 记录，如果有则删除 AHI 上的记录。

第3步非常耗时，上面例子中 `10GB` 的表清空时，绝大部分的时间消耗在第3步。虽然关闭了 AHI 后不会有第3步的执行，但是第1步、第2步的检查还在。当表非常大时由于检测的数据页数量非常的庞大，在第1步上花费的时间就会变长。上面的CPU热点图里我们可以看到 `buf_page_get_gen`  这个函数及下层函数总共花费了`46.24%`的 CPU 时间，`buf_page_get_gen` 本身则占了`41.24%`。

AliSQL上对这个问题做了优化，逻辑很简单：
- 将清理 AHI 的工作放到删除表之前，这一步不需要`dict_sys::mutex`的保护。
- 在删除索引时，会判断这个索引上是否有 AHI 的记录，如果没有则跳过每个 page 的检查。由于第一步已经提前清理了表上 AHI 记录(或者 AHI 是关闭的)，这一步会跳过每个 page 的检查。

## 删除 IBD 文件优化
在 DROP TABLE 的过程，实际删除 IBD 文件的过程被`dict_sys::mutex`保护。删除一个文件花费的时间和文件的大小成正比，比如删除一个100GB的文件，我的测试环境里大约需要`20`秒。

在 AliSQL 中我们实现一套异步删除大文件的机制：
- 删除一个超过一定大小的 IBD 文件时，这个文件不会立刻删除，而是被 rename 成一个临时文件。
- 一个后台线程以 `512MB` 为单位，逐步的 truncate 文件，直到文件长度为 0，才删除文件。这样可以避免瞬时大量的 os 内核 inode 操作，让删除更平滑。

## 清理 Page 优化
在真正的删除 table space 文件之前，MySQL-5.7 需要将这个 table space 的所有 page 从 buffer pool 中清理掉。由于 MySQL buffer pool 中没有单个表的 page 链表，所以 MySQL-5.7 采用了 `LRU Scan` 的方法，就是扫描 buffer pool 中的所有数据页，将扫到的属于当前 table space 的 page 全部释放掉。在 LRU scan 的过程中需要持有 `buf_pool::mutex`, 其他的 buffer pool 的读写都会被阻塞。

这个设计最坑的地方在于，`删除任意大小的表都需要执行 LRU scan`。这对`大内存`的实例非常的不友好，对一个 `128GB` buffer pool 的实例来说，大约有`800万`个 page, 即便分为 8 个 buffer pool instance，一次的 LRU scan 也要扫描 `100万` 个 page。在运维中大家对大表的处理一般都会比较谨慎。对于非常小的表可能会忽视了 `LRU scan` 产生的影响。如果在繁忙的系统上做了小表的删除、清理操作，就会导致实例不稳定。![](https://mmbiz.qpic.cn/sz_mmbiz_png/ibf4J5w9SFz6lkVcFM5Z3v2jDQGKY2zCSicywuWriaotOdwv29lIyxmYAZlldu7jj7Q9QiaCgFia5HxnQBQ63aKkG4w/0?wx_fmt=png&from=appmsg)

除了 `LRU scan`， DDL 也常常需要做 `Flush List Scan`，扫描脏页链表，把DDL产生的脏页都刷盘。这个扫描也需要 `buf_pool::mutex` 的保护，其他的 buffer pool 的读写都会被阻塞。对于一个写繁忙的实例来说，flush list 会更长，从而 flush list scan 持锁的时间就会变长。`更要命的当碰到一个需要刷盘的 page 时，flush list scan 就会被中断。当刷完这个 page 后，需要从头再来一遍。直到最后一遍，一个需要刷盘的 page 都没有发现才结束`。详情可以参考[BUG#102289](https://bugs.mysql.com/bug.php?id=102289) 。MySQL-8.0 对 `flush list scan` 做了个优化，就是提前 `pin` 住下一个要检查的 page，然后才去刷当前页。当刷完这个页后，可以接着从被 `pin`住的页继续处理。这样只需要执行一次完整的 flush scan 就可以了。如果下一个页正在做 IO 操作, pin 就会失败，这时还是要从头再来一遍。

AliSQL 中设计了基于 `space_id` 分区的 `page list` 和 ` flush list `，总共分了 `2048` 个分区。当要清除数据页或刷脏时，只需要扫描一个分区就可以了，不再需要进行 `LRU scan` 或 `flush list scan`。

以上的几个优化同样也适用于`重建表`的场景。OPTIMIZE TABLE, 以及许多的 Online DDL, Copy DDL 内部其实都是重建了一个新表，把原来的表删除掉。

## TRUNCATE TABLE 的问题
MySQL-5.7 上 TRUNCATE TABLE 主要的执行过程如下:
- 获取 `dict_sys::mutex`
- 删除所有索引（包括聚簇索引）。  
- 重建所有索引
- 释放 `dict_sys::mutext`
- 获取 `fil_sys::mutex`
- Truncate ibd 文件
- 释放 `fil_sys::mutex`

TRUNCATE TABLE 也会删除所有的索引，因此删除 AHI 记录的优化对 TRUNCATE TABLE 同样有效。但是 TRUNCATE TABLE 不会删除 IBD 文件，只是将 IBD 多余的空闲空间给 truncate 掉，因此无法进行异步的删除。

`DROP TABLE` 时 table space 会被删除，因此只有释放 table space 内存结构时持有 `fil_sys::mutex`，删除文件时不需要持有 `fil_sys::mutex`，所以持锁时间很短。但是清空一个大表时 truncate 文件过程中会持有`fil_sys::mutex`。Truncate 大文件时花费的时间很长，这段时间内`所有的 DML 会被阻塞`。DML被阻塞时的堆栈如下图所示：![](https://mmbiz.qpic.cn/sz_mmbiz_png/ibf4J5w9SFz7FZjEndCKDMKHgDUQt2dVB9wZazbiaNk3qBT1jsW6yh6tyhcHtWS6wibC7sAT60lk2llSt3AWnCN1Q/0?wx_fmt=png&from=appmsg)
 `set_named_space` 函数在获取正在修改的 table space 时需持有 `fil_sys::mutex`，进而被 `TRUNCATE TABLE` 阻塞。
 
在MySQL-5.7 中，所有数据页的操作都需要调用 `set_named_space` 函数。 这个函数是 InnoDB redo 重放机制中的一部分。Redo 中的所有数据页上的操作只记录了 space id。在重放这些记录前需要将 space id 和 文件名一一对应起来，这样才知道去操作哪个文件。

MySQL-5.7 的实现是把这个对应关系写到 redo 中去，因此在做数据页修改时就需要获取当前数据页所在的table space。MySQL-8.0 去掉了这部分逻辑，取而代之从 Data Dictionary 中获取 space id 和 文件名的对应关系。因此在 MySQL-8.0 中, `fil_sys::mutex` 使用的频率就非常低了，一般的数据页操作都不会涉及到 table space。实际上 MySQL-8.0 并不存在 `fil_sys::mutex` , MySQL-8.0对`fil_sys` 做了分区，总共分了 `64` 个 `fil_shard` 。一般情况下只需要获取一个分区的 mutex 就可以了。


## TRUNCATE TABLE 实操建议
鉴于上述的问题，我们建议使用 `RENAME + DROP` 的方式来替代 `TRUNCATE TABLE`。
``` sql
# 重建一张相同的表
CREATE TABLE t1_new LIKE t1;
# 检查重建表的表定义是否符合预期
SHOW CREATE TABLE t1_new;
# 新表和老表交换
RENAME TABLE t1 TO t1_bak, t1_new TO t1;
# 删除原表
DROP TABLE t1_bak;
```

如果你使用的是社区的 MySQL-5.7，实例的稳定性又要求特别高。那么建议通过 `DISCARD TABLESPACE` 的方式来完成表的删除。

```
# 创建一个t1.bak.ibd的硬连接
ln t1.bak.ibd hard_link_t1.bak.ibd

ALTER TABLE t1_bak DISCARD TABLESPACE;
DROP TABLE t1_bak;

# 分批truncate hard_link_t1.bak.ibd
# 最后 rm hard_link_t1.bak.ibd
```
用 Discard 的好处是discard table space期间不需要删除索引，可以避免释放 AHI 带来的问题。这种方法执行删除，唯一不能避免的就是`LRU Scan`。

## MySQL-8.0
MySQL-8.0 针对需要删除表的场景做了很大的优化：
- 删除表时不再需要从table space 中删除B+tree了。 
- AHI 清理变成了 LRU scan 的方式，如果 index 上没有任何 AHI 记录则不会做 LRU scan，在关闭AHI的情况下，没有任何效率问题。
-  采用多版本的 table space 机制，在删除表时不再需要清理数据页，避免了 LRU scan。
- 实现了 `Atomic DDL` 机制，table space 的物理删除过程放到  post_ddl 阶段，避免了长时间持有 `dict_sys::mutex`。想了解 Atomic DDL 的，可以参考《Atomic DDL 揭秘》
- TRUNCATE TABLE 内部通过 DROP + CREATE 的方式实现，避免了长时间持有 `fil_sys::mutext`。

目前遗留的问题主要有两个：
- 大文件删除可能造成的实例抖动
- 个别地方仍然需要 LRU scan 和 flush list scan 可能造成实例的抖动。
AliSQL-8.0 则集成了 AliSQL-5.7 的优化，通过大文件异步删除和基于 space id 分区的设计避免了这两个问题。

通过以上的优化，MySQL-8.0 上删除表、清空表和需要重建表的 Online/Copy DDL要比 MySQL-5.7 触发问题的概率要低很多。之所以大家体感 MySQL-8.0 非常不稳定，那是因为前几年 MySQL-8.0 由于采用了 Rolling Release 的策略，每个小版本都有新功能引入，所以导致版本极不稳定。但是从 `MySQL-8.0.34` 之后就不再有新功能的合并只做Bugfix，之后就越来越稳定了。

