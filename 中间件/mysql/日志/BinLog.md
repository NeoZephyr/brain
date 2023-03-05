## 完整性

statement 格式的 binlog，最后会有 COMMIT；row 格式的 binlog，最后会有一个 XID event

在 MySQL 5.6.2 版本以后，引入了 binlog-checksum 参数，用来验证 binlog 内容的正确性。对于 binlog 日志由于磁盘原因，可能会在日志中间出错的情况，MySQL 可以通过校验 checksum 的结果来发现

## 写入逻辑

事务执行过程中，先把日志写到 binlog cache，事务提交的时候，再把 binlog cache 写到 binlog 文件中。一个事务的 binlog 是不能被拆开的，不论这个事务多大，都要确保一次性写入。每个线程都拥有自己的 binlog cache，大小由参数 binlog_cache_size 控制

```sql
show variables like "%binlog_cache_size%";
```

事务提交的时候，执行器把 binlog cache 里的完整事务写入到 binlog 中，并清空 binlog cache

![[Pasted image 20230305214511.png]]

write 操作会把日志写入到文件系统的 page cache，并没有把数据持久化到磁盘，所以速度比较快。fsync 操作将数据持久化到磁盘。一般情况下，fsync 才占磁盘的 IOPS。write 和 fsync 的时机，是由参数 sync_binlog 控制的：

1. sync_binlog=0，表示每次提交事务都只 write，不 fsync
2. sync_binlog=1，表示每次提交事务都会执行 fsync
3. sync_binlog=N（N>1），表示每次提交事务都 write，但累积 N 个事务后才 fsync

在出现 IO 瓶颈的场景里，将 sync_binlog 设置成一个比较大的值，可以提升性能。在实际的业务场景中，考虑到丢失日志量的可控性，一般不建议将这个参数设成 0，比较常见的是将其设置为 100~1000 中的某个数值

```sql
show variables like "sync_binlog";
```

建议设置为 1，保证 MySQL 异常重启之后 binlog 不丢失

binlog 的默认是保持时间由参数 expire_logs_days 配置

## 日志格式

```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `t_modified` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `a` (`a`),
  KEY `t_modified`(`t_modified`)
) ENGINE=InnoDB;

insert into t values(1,1,'2018-11-13');
insert into t values(2,2,'2018-11-12');
insert into t values(3,3,'2018-11-11');
insert into t values(4,4,'2018-11-10');
insert into t values(5,5,'2018-11-09');
```

执行以下语句

```sql
delete from t where a>=4 and t_modified<='2018-11-10' limit 1;
```

### statement

记录 SQL 语句的原文

```sql
show binlog events in 'master.000001';
```

![[Pasted image 20230305214615.png]]

第二行的 BEGIN 跟第四行的 COMMIT 对应，表示中间是一个事务，第三行就是执行真实的语句。可以看到，在真实执行的 delete 命令之前，还有一个 use 'test' 命令。这条命令是 MySQL 根据当前要操作的表所在的数据库，自行添加的。这样做可以保证日志传到备库去执行的时候，不论当前的工作线程在哪个库里，都能够正确地更新到 test 库的表 t

![[Pasted image 20230305214628.png]]

通过 show warnings 语句可以看到，运行这条 delete 命令产生了一个 warning，原因是当前 binlog 设置的是 statement 格式，并且语句中有 limit，所以这个命令可能是 unsafe 的。这是因为 delete 带 limit，很可能会出现主备数据不一致的情况。比如：

如果 delete 语句使用的是索引 a，那么会根据索引 a 找到第一个满足条件的行，也就是说删除的是 a=4 这一行；但如果使用的是索引 t_modified，那么删除的就是 t_modified='2018-11-09' 这一行，对应到 a=5

### row

![[Pasted image 20230305214640.png]]

与 statement 格式的 binlog 相比，前后的 BEGIN 和 COMMIT 是一样的。但是，row 格式的 binlog 里没有 SQL 语句的原文，而是替换成了两个 event：Table_map 和 Delete_rows

1. Table_map event，用于说明接下来要操作的表是 test 库的表 t
2. Delete_rows event，用于定义删除的行为

从图中可以看到，这个事务的 binlog 是从 8900 这个位置开始的，可以借助 mysqlbinlog 工具解析和查看 binlog 中的内容，并且指定 start-position 参数从 8900 的位置开始日志解析

```sql
mysqlbinlog -vv data/master.000001 --start-position=8900;
```

![[Pasted image 20230305214701.png]]

可以得到以下信息：
1. server id 1，表示这个事务是在 server_id=1 的这个库上执行的
2. 如果把参数 binlog_checksum 设置成了 CRC32，则每个 event 都有 CRC32 的值
3. Table_map event 显示了接下来要打开的表，map 到数字 226。如果操作多张表，则每个表都有一个对应的 Table_map event、都会 map 到一个单独的数字，用于区分对不同表的操作
4. 由于 -vv 参数是为了把内容都解析出来，可以看到各个字段的值（比如，@1=4、 @2=4 这些值）
5. binlog_row_image 的默认配置是 FULL，因此 Delete_event 里面，包含了删掉的行的所有字段的值。如果把 binlog_row_image 设置为 MINIMAL，则只会记录必要的信息，在这个例子里，就是只会记录 id=4 这个信息
6. 最后的 Xid event，用于表示事务被正确地提交

可以看到，当 binlog_format 使用 row 格式的时候，binlog 里面记录了真实删除行的主键 id，这样 binlog 传到备库去的时候，就肯定会删除 id=4 的行，不会有主备删除不同行的问题

### mixed

因为有些 statement 格式的 binlog 可能会导致主备不一致，所以要使用 row 格式。但 row 格式很占空间。比如：用一个 delete 语句删掉 10 万行数据，用 statement 的话就是一个 SQL 语句被记录到 binlog 中，占用几十个字节的空间。但如果用 row 格式的 binlog，就要把这 10 万条记录都写到 binlog 中。这样做，不仅会占用更大的空间，同时写 binlog 也要耗费 IO 资源，影响执行速度

因为以上原因，MySQL 采取折中方案，也就是有了 mixed 格式的 binlog。mixed 格式的意思是，MySQL 自己会判断这条 SQL 语句是否可能引起主备不一致，如果有可能，就用 row 格式，否则就用 statement 格式

现在越来越多的场景要求把 MySQL 的 binlog 格式设置成 row。这么做的理由有很多，比如恢复数据

## 恢复数据

1. 执行错了 delete 语句，由于 row 格式的 binlog 会把被删掉的行的整行信息保存起来，因此可以直接把 binlog 中记录的 delete 语句转成 insert，这样被错删的数据就可以恢复了
2. 执行错了 insert 语句，由于 row 格式下，insert 语句的 binlog 里会记录所有的字段信息，可以精确定位到刚刚被插入的那一行。因此直接把 insert 语句转成 delete 语句就可以了
3. 执行错了 update 语句，binlog 里面会记录修改前整行的数据和修改后的整行数据。所以只需要把这个 event 前后的两行信息对调一下，再去数据库里面执行，就能恢复这个更新操作

MariaDB 的 [Flashback](https://mariadb.com/kb/en/flashback/) 工具就是基于上面介绍的原理来回滚数据的

```sql
insert into t values(10,10, now());
```

如果把 binlog 格式设置为 mixed

![[Pasted image 20230305214754.png]]

可以看到，MySQL 用的居然是 statement 格式。看上去，如果这个 binlog 过了 1 分钟才传给备库的话，那主备的数据就可能不一致。再用 mysqlbinlog 工具来看看：

![[Pasted image 20230305214801.png]]

从图中可以看到，binlog 在记录 event 的时候，多记了一条命令：SET TIMESTAMP=1546103491，从而约定了接下来的 now() 函数的返回时间。因此，不论这个 binlog 在多久之后被备库执行，这个 insert 语句插入的行，值都是固定的，从而确保了 MySQL 主备数据的一致性

如果在重放 binlog 数据的时候，用 mysqlbinlog 解析出日志，然后把里面的 statement 语句直接拷贝出来执行，就会有风险的。因为有些语句的执行结果是依赖于上下文命令的。所以，正确的做法是用 mysqlbinlog 工具解析出来，然后把解析结果整个发给 MySQL 执行。类似下面的命令：

```sql
mysqlbinlog master.000001 --start-position=2738 --stop-position=2973 | mysql -h127.0.0.1 -P13000 -u$user -p$pwd;
```

这个命令的意思是，将 master.000001 文件里面从第 2738 字节到第 2973 字节中间这段内容解析出来，放到 MySQL 去执行