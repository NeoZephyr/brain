## WAL

mysql 在更新时，如果每一次更新都写进磁盘，需要磁盘找到对应记录，然后再更新，IO 成本特别高。因此 mysql 采用 WAL 技术，WAL 的全称是 Write-Ahead Logging，关键在于先写日志，再写磁盘

当有数据更新时，先将记录写到 redo log 中，并更新内存。存储引擎会在适当的时候，将这个操作记录更新到磁盘里面，这通常是在空闲的时候进行

由于 redo log 大小固定，如果写入数据过多，需要存储引擎将 redo log 中的一部分操作记录更新到磁盘里面，从而腾出空间记录新的操作记录

redo log 能保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为 crash-safe

## 写入机制

![[Pasted image 20230305214915.png]]

redo log 存在三种状态：
1. 存在 redo log buffer 中，物理上是在 MySQL 进程内存中
2. 写到磁盘（write），但是没有持久化（fsync），物理上是在文件系统的 page cache 里面
3. 持久化到磁盘，对应的是 hard disk

日志写到 redo log buffer 是很快的，wirte 到 page cache 也差不多，但是持久化到磁盘的速度就慢多。innodb 通过参数 `innodb_flush_log_at_trx_commit` 控制 redo log 的写入策略：

1. 设置为 0 的时候，表示每次事务提交时都只是把 redo log 留在 redo log buffer 中
2. 设置为 1 的时候，表示每次事务提交时都将 redo log 直接持久化到磁盘
3. 设置为 2 的时候，表示每次事务提交时都只是把 redo log 写到 page cache

`innodb_flush_log_at_trx_commit` 这个参数设置成 1 的时候，表示每次事务的 redo log 都直接持久化到磁盘。建议设置为 1，保证 MySQL 异常重启之后数据不丢失

```sql
show variables like "innodb_flush_log_at_trx_commit";
```

InnoDB 有一个后台线程，每隔 1 秒，就会把 redo log buffer 中的日志，调用 write 写到文件系统的 page cache，然后调用 fsync 持久化到磁盘

注意，事务执行中间过程的 redo log 也是直接写在 redo log buffer 中的，这些 redo log 也会被后台线程一起持久化到磁盘。也就是说，一个没有提交的事务的 redo log，也是可能已经持久化到磁盘的。实际上，除了后台线程每秒一次的轮询操作外，还有两种场景会让一个没有提交的事务的 redo log 写入到磁盘中：

1. redo log buffer 占用的空间即将达到 innodb_log_buffer_size 一半的时候，后台线程会主动写盘。注意，由于这个事务并没有提交，所以这个写盘动作只是 write，而没有调用 fsync，也就是只留在了文件系统的 page cache
2. 并行的事务提交的时候，顺带将这个事务的 redo log buffer 持久化到磁盘。假设一个事务 A 执行到一半，已经写了一些 redo log 到 buffer 中，这时候有另外一个线程的事务 B 提交，如果 innodb_flush_log_at_trx_commit 设置的是 1，事务 B 要把 redo log buffer 里的日志全部持久化到磁盘。这时候，就会带上事务 A 在 redo log buffer 里的日志一起持久化到磁盘

时序上 redo log 先 prepare，再写 binlog，最后再把 redo log commit。如果把 innodb_flush_log_at_trx_commit 设置成 1，那么 redo log 在 prepare 阶段就要持久化一次（有一个崩溃恢复逻辑是要依赖于 prepare 的 redo log，再加上 binlog 来恢复的）

每秒一次后台轮询刷盘，再加上崩溃恢复这个逻辑，InnoDB 就认为 redo log 在 commit 的时候就不需要 fsync 了，只会 write 到文件系统的 page cache 中就够了。可以看出，一个事务完整提交前，需要等待两次刷盘，一次是 redo log（prepare 阶段），一次是 binlog

## 组提交机制

日志逻辑序列号（log sequence number，LSN）是单调递增的，用来对应 redo log 的一个个写入点。每次写入长度为 length 的 redo log，LSN 的值就会加上 length。LSN 也会写到 InnoDB 的数据页中，来确保数据页不会被多次执行重复的 redo log

图中，是三个并发事务 (trx1, trx2, trx3) 在 prepare 阶段，都写完 redo log buffer，持久化到磁盘的过程，对应的 LSN 分别是 50、120 和 160

![[Pasted image 20230305215004.png]]

1. trx1 是第一个到达的，会被选为这组的 leader
2. 等 trx1 要开始写盘的时候，这个组里面已经有了三个事务，这时候 LSN 也变成了 160
3. trx1 去写盘的时候，带的就是 LSN=160，因此等 trx1 返回时，所有 LSN 小于等于 160 的 redo log，都已经被持久化到磁盘
4. 这时候 trx2 和 trx3 就可以直接返回了

所以，一次组提交里面，组员越多，节约磁盘 IOPS 的效果越好。但如果只有单线程压测，那就只能一个事务对应一次持久化操作了。在并发更新场景下，第一个事务写完 redo log buffer 以后，接下来这个 fsync 越晚调用，组员可能越多，节约 IOPS 的效果就越好。为了让一次 fsync 带的组员更多，MySQL 有一个优化：拖时间

![[Pasted image 20230305215027.png]]

图中，写 binlog 是一个动作。但实际上，写 binlog 是分成两步的：先把 binlog 从 binlog cache 中写到磁盘上的 binlog 文件；调用 fsync 持久化。MySQL 为了让组提交的效果更好，把 redo log 做 fsync 的时间拖到了第一步之后。也就是说，流程如下图所示

![[Pasted image 20230305215050.png]]

这样，binlog 也可以组提交了。在图中把 binlog fsync 到磁盘时，如果有多个事务的 binlog 已经写完了，也是一起持久化的，这样也可以减少 IOPS 的消耗。不过通常情况下第 3 步执行得会很快，所以 binlog 的 write 和 fsync 间的间隔时间短，导致能集合到一起持久化的 binlog 比较少，因此 binlog 的组提交的效果通常不如 redo log 的效果那么好。如果你想提升 binlog 组提交的效果，可以通过设置以下两个参数来实现：

`binlog_group_commit_sync_delay` 参数，表示延迟多少微秒后才调用 fsync
`binlog_group_commit_sync_no_delay_count` 参数，表示累积多少次以后才调用 fsync

这两个条件是或的关系，也就是说只要有一个满足条件就会调用 fsync。所以，当 `binlog_group_commit_sync_delay` 设置为 0 的时候，`binlog_group_commit_sync_no_delay_count` 也无效了

WAL 机制主要得益于两个方面：redo log 和 binlog 都是顺序写，磁盘的顺序写比随机写速度要快；组提交机制，可以大幅度降低磁盘的 IOPS 消耗。如果 MySQL 出现了性能瓶颈，而且瓶颈在 IO 上，可以考虑以下三种方法：

1. 设置 `binlog_group_commit_sync_delay` 和 `binlog_group_commit_sync_no_delay_count` 参数，减少 binlog 的写盘次数。这样做，可能会增加语句的响应时间
2. 将 sync_binlog 设置为大于 1 的值（比较常见是 100~1000）。这样做，主机掉电时会丢 binlog 日志
3. 将 innodb_flush_log_at_trx_commit 设置为 2。这样做，主机掉电的时候会丢数据

不建议把 `innodb_flush_log_at_trx_commit` 设置成 0。因为把这个参数设置成 0，表示 redo log 只保存在内存中，这样的话 MySQL 本身异常重启也会丢数据，风险太大。而 redo log 写到文件系统的 page cache 的速度也是很快的，所以将这个参数设置成 2 跟设置成 0 其实性能差不多，但这样做 MySQL 异常重启时就不会丢数据了，相比之下风险会更小

一般在以下情况将 sync_binlog 与 innodb_flush_log_at_trx_commit 都不设置为 1

1. 业务高峰期，把主库设置成非双 1
2. 备库延迟，为了让备库尽快赶上主库
3. 用备份恢复主库的副本，应用 binlog 的过程
4. 批量导入数据的时候

一般情况下，把生产库改成非双 1 配置，是设置 innodb_flush_logs_at_trx_commit=2、sync_binlog=1000

## 刷脏页

innodb 在数据更新时，只做了写日志这一个磁盘操作，即 `redo log`，然后更新内存就返回了。可以看出，内存页与数据页内容可能不一致，即出现脏页。而将内存中的数据更新到磁盘的过程，就叫刷脏页，也即是 flush 操作。一般在以下情况，会进行刷脏页操作：

1. 因为 `redo log` 空间大小固定，当写满时，此时会停止所有更新操作，通过刷脏页留出空间继续写
2. 内存不足，需要淘汰一些内存页，如果淘汰的是脏页，就需要将脏页写回磁盘。这样保证了每个数据页只有两个状态：在内存中的数据页，是最新的数据，可以直接返回；内存中没有数据，就可以肯定磁盘上面是最新的数据
3. mysql 空闲的时候刷脏页
4. mysql 正常关闭，会将内存中的脏页都刷新到磁盘上

第一种情况要尽量避免的，因为这种情况下，整个系统不能再接受更新了，此时更新次数跌为 0
第二种是比较常见的现象。innodb 用缓冲池（buffer pool）管理内存，缓冲池中的内存页有以下三种状态：
1. 还未使用的内存页
2. 使用了仍是干净的数据页
3. 使用了已经是脏页

innodb 的策略是尽量使用内存，但是每当要读入的数据页不在内存中的时候，就必须要到缓冲池中申请一个数据页。此时需要把最久不使用的数据页从内存中淘汰掉：如果要淘汰的是一个干净页，就直接释放出来复用；但如果是脏页，就必须先刷到磁盘，变成干净页后才能复用

根据以上分析可以得知，尽管刷脏页是常见现象，但是出现以下情况，会明显影响性能：
1. 一个查询要淘汰的脏页个数太多，会导致查询的响应时间明显变长
2. 日志写满，更新全部堵住，写性能跌为 0

可以通过控制脏页比例来避免明显影响性能的刷脏页情况：

1. 正确地告诉 innoDB 所在主机的 IO 能力

```sql
-- 建议设置成磁盘的 IOPS
-- 设置太小可能会导致刷脏页特别慢，进而导致内存脏页太多，redo log 写满，影响了查询和更新性能
show variables like "innodb_io_capacity";
```

```bash
fio -filename=$filename -direct=1 -iodepth 1 -thread -rw=randrw -ioengine=psync
-bs=16k -size=500M -numjobs=10 -runtime=10 -group_reporting -name=mytest
```

2.  根据脏页比例及 `redo log` 写入速度控制刷脏页速度

如果刷脏页太慢，会导致内存脏页太多，redo log 写满。因此，InnoDB 刷盘速度需要参考这两个因素

![[Pasted image 20230305215224.png]]

```sql
-- 脏页比例上限，默认为 75%
show variables like "innodb_max_dirty_pages_pct";
```

查看脏页比例

```sql
select VARIABLE_VALUE into @a from global_status
where VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty';

select VARIABLE_VALUE into @b from global_status
where VARIABLE_NAME = 'Innodb_buffer_pool_pages_total';

select @a/@b;
```

如果一个查询在执行过程中需要 flush 掉一个脏页，这个查询就比较慢。但 mysql 有一个机制：在刷脏页的时候，如果这个数据页旁边的数据页也是脏页，那么就会一起刷掉

在 innodb 中，`innodb_flush_neighbors` 参数为 1 表示脏页刷新具有传递性，为 0 则表示刷新不会影响邻居脏页

```sql
show variables like "innodb_flush_neighbors";
```

刷新邻居脏页的特性在机械硬盘时代，可以减少很多随机 IO。如果使用的是 SSD 这种 IOPS 比较高的设备，可以将 `innodb_flush_neighbors` 设置为 0，这样就能更快的执行刷脏页操作，减少 sql 语句的响应时间。在 mysql8.0 中，`innodb_flush_neighbors` 参数的默认值已经设置为 0 了