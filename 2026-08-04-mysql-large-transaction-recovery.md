---
title: MySQL 大事务的 Recovery 优化
date: 2026-08-04 10:00:00 +0800
categories: [数据库, MySQL]
tags: [MySQL, 大事务, Recovery, Binlog, InnoDB]
toc: true
published: false # TODO: 检查无误后删除这一行即可发布
---

> 本文也有英文版：[English version](/posts/mysql-large-transaction-recovery-en/)。
{: .prompt-tip }

你有没有碰到过 mysqld 进程启动了很长时间也起不来的情况？这时候我们可以用 `perf top` 命令查看一下 MySQL 进程主要在干什么事情。如果你查看到的信息如下图所示，启动过程中 MySQL 的 `主线程(mysqld_main函数开始的线程)` 绝大多数的时间都花在了回滚事务上，那么很可能是遇到了大事务回滚。

![](/assets/img/bigtxn-recovery-1.webp)

这种情况最常见的一个场景是一个大事务在写 Binlog 时把磁盘空间占满了，导致了实例的宕机重启。我曾经遇到的最大的 Binlog 文件超过了 `114GB`。由于 Binlog Cache 的临时文件在写完 Binlog 后才被清理，所以这个事务总共占用了 `228GB` 的空间。MySQL 的参数 `binlog_error_action` 用来控制写 Binlog 文件失败的行为，默认的配置是 `ABORT_SERVER`，就是关闭进程。用户也可以配置这个参数为 `IGNORE_ERROR`，意思是当写 Binlog 失败时关闭 Binlog 文件，后续的事务不再产生 Binlog。这种情况显然会导致主备的数据不一致，因此除非不得已不要这样设置。

## 根因分析

为什么在 MySQL 进程启动时，主线程要做事务回滚的操作呢？这源自于 Binlog 的 Crashsafe 机制，这里只做一个概括的介绍。事务的 DML 执行时会产生 Binlog Events，当事务提交时这些 Binlog Events 会被写入到 Binlog 文件并持久化。为了保证 MySQL 宕机重启后数据和 Binlog 的一致性，MySQL 设计了一个 Crashsafe 的机制。该机制对普通事务采用了两阶段提交（2PC），也称为内部 XA（Internal XA）。

![](/assets/img/bigtxn-recovery-2.webp)

如上图所示，在内部 XA 机制下一个事务的提交过程分为三个步骤：

1. 存储引擎 `Prepare` 事务。事务状态由 `ACTIVE` 变为 `PREPARED`，并将事务的状态和 `XID` 持久化到 Redo 中。
2. 事务会产生一个 `Xid_event`，同 DML 的 Binlog Events 一同写入到 Binlog 文件中，并持久化。
3. 提交事务。

当异常宕机时，事务可能处于以下几种状态之一：

- `Active`：在两阶段提交里，此类事务从未被写入 Binlog。
- `Prepared 但未写入 Binlog（或仅部分写入）`：事务已处于 Prepared 状态，但其 XID 未出现在 Binlog 文件中。
- `Prepared 且已写入 Binlog`：事务已处于 Prepared 状态，且其 XID 已出现在 Binlog 文件中。
- `Committed`：事务已经写入 Binlog 并且提交。

对于 `Committed` 的事务，设计上已经保证了它的 Binlog Events 一定写入了 Binlog 文件。因此 Binlog 和数据是一致的，启动时无需任何操作。对于 `Active` 的事务，Binlog Events 肯定没有写入 Binlog 文件，`InnoDB有一个后台回滚线程会自动将其回滚`。`Prepared` 的事务则需根据最后一个 Binlog 文件中的 XID 信息进行处理：如果该事务的 `XID` 出现在了 Binlog 文件中，则需要提交该事务来保证 Binlog 和数据的一致性；反之则回滚该事务。

![](/assets/img/bigtxn-recovery-3.webp)

处理 `Prepared` 事务的过程称为 `Binlog Recovery`，`必须在MySQL向用户提供服务之前完成`。事务提交通常很快，但回滚往往耗时与其执行时间相当。如果一个事务执行用了 1 小时，回滚很可能也需要 1 小时，MySQL 在此期间将不可用。

为什么必须要在提供服务前回滚所有事务呢？这和 `XID` 的实现有关系。XID 是用 `MySQL` 前缀加上 `query_id` 构成的，`query_id` 是一个全局的计数器，系统重启后会重新从 1 开始计数。如果在启动后，不对之前的 Prepared 事务进行提交或者回滚，那么就可能出现两个 Prepared 的事务有相同 XID 的情况。在恢复时，就无法区分哪个事务该提交，哪个事务该回滚。

## 异步回滚 Prepared 事务

在 AliSQL 中，我们设计了一套异步回滚的机制来解决这个问题。

![](/assets/img/bigtxn-recovery-4.webp)

如上图所示，这个设计中将 Prepared 的事务回滚分为两个部分：

1. 主线程将事务状态设为 `Active` 并持久化该状态。
2. 利用 InnoDB 的后台回滚线程异步回滚事务的所有操作。

Binlog Recovery 在完成第一部分后即可立即对外提供服务。由于第一步的执行非常快，Binlog Recovery 可以在很短的时间内完成。

在宕机重启时，`Active` 的事务会被 InnoDB 通过后台线程直接回滚掉，不需要 `XID` 来辅助决策。所以恢复时，只要将要回滚的事务的状态从 `Prepared` 改成 `Active`，就能避免两个 Prepared 事务有相同 `XID` 的问题。这里关键是要对 `Active` 的状态做持久化，保证在宕机重启后事务的状态仍然是 `Active`，这样 InnoDB 就会自动将这个事务回滚掉。

> 社区版的 InnoDB 原本对 Prepared 事务的回滚就是先设置成 `Active` 状态，然后再根据 Undo 记录进行回滚。`Active` 的状态会记录到 Redo 中，只是没有对 Redo 做持久化。然而 InnoDB 默认每秒会做一次 Redo 的持久化，所以在改成 `Active` 后，很快就会被持久化。因此当碰到了大事务回滚造成实例无法启动的情况时，**即使是在社区版本，我们只要强制重启 MySQL 进程，大事务就会转变成后台回滚，不再阻塞实例的启动**。
{: .prompt-tip }

这个功能的源码贡献给了 MariaDB，已经合并到 MariaDB-11.7 中，详情参考 [MDEV-33853](https://jira.mariadb.org/browse/MDEV-33853)。

## 结论

通过异步回滚的设计，在 `Binlog Recovery` 阶段只需要将 Prepared 的事务的状态设置为 `Active`，真正耗时的事务回滚则由 InnoDB 的后台回滚线程异步的执行。通过这个优化，原本需要几十分钟甚至几个小时的启动过程，被缩短到秒级完成。
