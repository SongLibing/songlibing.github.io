---
title: MySQL 大事务提交优化
date: 2026-07-15 10:00:00 +0800
categories: [数据库, MySQL]
tags: [MySQL, Binlog, 大事务, 提交优化]
toc: true
---

> 本文也有英文版：[English version](/posts/mysql-large-transaction-commit-optimization-en/)。
{: .prompt-tip }

在使用和运维MySQL的过程中你一定碰到过下面这种奇怪的慢SQL。

![](/assets/img/bigtxn-commit-1.webp)

- 平时执行很快的INSERT语句，竟然执行了`1.3s`，并且慢SQL记录里也没有看到长时间的锁等待。
- 多语句事务的所有语句都已经执行完了，但是COMMIT语句竟然执行了`1.3s`。

当这种情况出现时，最有可能的就是有大事务在提交。以下是一个模拟测试的结果，我们用Sysbench来模拟正常的业务，然后在后台每5秒执行一个大的UPDATE，可以看到大的UPDATE会严重的影响业务的性能。

![](/assets/img/bigtxn-commit-2.webp)

## 根因分析

![](/assets/img/bigtxn-commit-3.webp)

上图展示了两个事务的执行过程：

- 事务执行分为两个阶段，执行阶段和提交阶段。
- 事务执行阶段，当语句更新数据时，会生成Binlog Events。这些Binlog events会被存储到Binlog Cache中，Binlog Cache有两部分，一部分是内存的Buffer，一部分是一个临时文件，当Buffer 写满后，就会将Binlog Events 写入到临时文件中。
- 当事务提交时，会将Binlog Cache 中的Binlog Events 全部拷贝到Binlog文件中。
- `将Binlog Events写入Binlog文件的过程必须要串行执行，只有一个事务写完了，另外一个事务才能执行。`因此当`Trx_n`在写Binlog文件时，`Trx_m`就必须等待。
- 图中`Trx_n`是一个大事务，产生了大量的Binlog Events。`拷贝Binlog Events到Binlog 文件所需要的时间是和事务产生的Binlog Events大小线性相关的，Binlog Events越大，拷贝的时间就越长。`
- `Trx_m`是一个小事务，虽然执行阶段很快就完成了，但是在提交时，遇到了大事务`Trx_n`在提交，因此必须要等待`Trx_n`拷贝完Binlog Events才能继续。`Trx_m`在提交阶段花了大量的时间在等待`Trx_n`写Binlog文件，这就是小事务变慢的原因。

## 问题的严重性

从前面我们的模拟测试我们可以看到，大事务的提交对业务的稳定性是有非常大的影响的。实际的使用场景中可能要严重的多，并且也很普遍。

- GB量级的事务会造成实例长时间的不可写。由于存储的IO带宽是一定的，大事务写binlog的时间就取决于事务的大小。在运维的过程中，我们见到的最大的事务产生了`104GB`的Binlog Events。
- GB量级的事务会造成IO吞吐上升变慢,甚至IO打满，这也会导致查询变慢。
- 几百MB的事务虽然不会造成长时间的影响，但仍有会让业务DML慢上几百毫秒。这种程度的慢，对于延迟敏感型的业务来说，可能也是不能接受的。
- 除此之外，以上的几种情况都可能引起活跃连接数的上升。如果活跃连接不能及时消化，导致CPU打高，可能造成恶性循环，最终雪崩，产生更大的问题。

## 大事务写Binlog优化

我们在AliSQL上对大事务写Binlog的过程做了优化，彻底消除了大事务提交对稳定性的影响。RDS-5.7，RDS-8.0都已经默认的开启了这个优化。去年我们将AliSQL的这个优化捐赠给了MariaDB，这个功能已经在MariaDB-11.7[1]上发布。

### 优化方案

下面我来介绍一下MariaDB-11.7上的实现方案，MySQL和MariaDB代码上虽然差别已经比较大了，但是根本的逻辑还是一样的，所以实现方案也是一样的。

![](/assets/img/bigtxn-commit-4.webp)

这个优化方案说起来是比较简单和清晰的：既然Binlog Cache已经将Binlog Events写到了文件里，那我们就直接把这个`文件直接转成(rename)一个Binlog文件`。这样就不需要拷贝Binlog Events，没有额外的IO产生。并且Rename操作的时间是恒定的和Binlog Cache的大小无关，可以彻底解决大事务造成的问题。下面我们来看一下实现逻辑。

### #binlog\_cache\_files 目录

Binlog Cache里的文件是一个系统临时文件，不能直接转成一个普通文件。因此我们在binlog所在的目录创建了一个目录`#binlog_cache_files`，Binlog Cache创建的文件从系统临时文件变成了普通的文件，放在这个目录里。

```shell
$ls var/mysqld.1/data/#binlog_cache_files                                
ML_140413554102520
```

### 头部保留空间

Binlog cache的文件里只包含事务的Binlog Events，如果要转成Binlog文件，则需要保留一定的空间用来写Binlog文件头的Binlog Events，比如Format\_description\_event等。

![](/assets/img/bigtxn-commit-5.webp)

预留的空间是按4KB对齐的，因此至少会预留4KB。对于大多数情况来说，4KB的空间已经足够。但是在一些场景下Gtid\_list\_log\_event(类似于MySQL的Previous\_gtid\_event用来记录这个Binlog之前生成的Gtid集合)可能会非常大。为了避免在这种情况下这个功能无法使用，在生成新的Binlog文件时会根据Binlog文件头部Events实际占用的空间大小来调整保留的空间。当下一个事务开始时，会调整Binlog Cache文件的保留空间。Binlog头部的Binlog Events通常占用不到4KB的空间，因此在写完头部的Binlog Events后，可能会剩余一些空间。如何处理这些剩余空间呢？得益于MariaDB Gtid\_log\_event可以在末尾填充0的机制，这些剩余空间会被填充到对应的Gtid\_log\_event中。在将Binlog Cache文件转成Binlog文件后，其结构如下所示：

![](/assets/img/bigtxn-commit-6.webp)

### Rename 过程

![](/assets/img/bigtxn-commit-7.webp)

Rename 的主要过程如下：

- 持久化Binlog Cache文件，此时还没有进入Rename过程，不会阻塞其他事务提交。
- 执行Rotate的过程，关闭原来的Binlog文件，产生一个新的Binlog文件。
- 将新文件的Header的内容拷贝到Binlog Cache头部。
- 生成Gtid\_log\_event.
- 删除生成的Binlog文件，将Binlog Cache文件Rename成新的Binlog文件。

## 优化效果

我们仍然用sysbench模拟业务执行，然后在后台每5秒执行一个大的UPDATE，这个UPDATE会产生512MB的Binlog Events。测试结果如下：

![](/assets/img/bigtxn-commit-8.webp)

在有大事务提交优化的情况下, sysbench的TPS已经比较平稳，不会出现剧烈的抖动。看起来每5秒仍然会有一个小幅的抖动，这个抖动是因为执行大的UPDATE本身要占用一定的CPU，而不是事务提交导致的。

此外我们也模拟了不同大小的事务，对业务SQL造成的延迟情况。结果如下图所示：

![](/assets/img/bigtxn-commit-9.webp)

- 在没有大事务提交优化的情况下，当大事务超过64MB后sysbench的最大延迟开始明显增加，并且随着事务变大，迅速增加。
- 当开启了大事务提交优化后, 无论事务多大sysbench的最大延迟始终保持稳定，保持正常业务延迟水平。当事务达到1024GB时，因为多了一次Binlog Rotate，所以我们看到延迟略有增加。

## 总结

在MySQL Binlog复制架构中，大事务是一个比较典型的问题导火索，会导致稳定性、复制延迟等问题。通过将Binlog Cache的临时文件直接转成Binlog文件的方法，可以避免对于Binlog Events的拷贝，消除额外的IO，让大事务的提交始终保持高效和稳定。因此彻底解决了大事务提交导致的各种稳定性问题。

#### 引用链接

`[1]` MariaDB-11.7: *https://mariadb.com/resources/blog/binlog-commit-optimization-for-large-transaction/*
