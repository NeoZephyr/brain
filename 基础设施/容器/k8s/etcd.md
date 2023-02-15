```bash
# 健康检查
etcdctl endpoint health

etcdctl member list --write-out=table

etcdctl --endpoints=localhost:2379 put /key val
etcdctl --endpoints=localhost:2379 get /key

# 按前缀查询
etcdctl --endpoints=localhost:2379 get --prefix /

# 只显示键值
etcdctl --endpoints=localhost:2379 get --prefix / --keys-only --debug
```

## Raft 一致性

[Raft 演示](http://thesecretlivesofdata.com/raft/)

初始启动时，节点处于 follower 状态，并且被设定一个 election timeout。如果在这一时间周期内没有收到来自 leader 的心跳，节点将发起选举：将自己切换为 candidate 之后，向集群中的其他 follower 节点发送请求，询问是否选举自己成为 leader

当收到集群中过半数的接受投票之后，成为 leader。如果没有达成一致，则随机选择一个等待时间间隔再次发起投票

每成功选举一次，新 leader 的任期都会比之前 leader 的任期大 1

leader 节点依靠定时向 follower 发送心跳保持地位

leader 接到客户端的请求之后，先把日志追加到本地 log 中，然后通过心跳同步给其他 follower。当 leader 收到 n/2 + 1 个 follower 的 ACK 信息后，将该日志设置为已提交并追加到本地磁盘中

## 调优

减少网络延迟
减少磁盘 IO 延迟（使用 ssd）
保持合理的日志文件大小（使用快照）
合理的存储配额
自动压缩历史版本
定期消除碎片化
优化运行参数（调整心跳周期、选举超时时间，降低 leader 选举的可能性）
备份存储