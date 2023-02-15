## zookeeper 特点

1. 一个 leader，多个 follower
2. 集群中只要有半数以上的节点存活，zookeeper 集群就能正常工作
3. 全局一致性，每个 server 保存一份相同的数据副本
4. 更新请求顺序进行，来自同一个 client 的更新请求按其发送顺序依次执行
5. 数据更新原子性，一次数据更新要么成功，要么失败
6. 实时性，在一定时间范围内，客户端能读到最新数据

## zookeeper 结构

zookeeper 的数据模型可以看成一棵树，每个节点都是一个 ZNode。每一个 ZNode 默认能够存储 1MB 的数据

## zookeeper 应用场景

统一命名服务
统一配置管理
统一集群管理
服务器节点动态上下线

## 客户端操作

查看
```sh
ls -R /
ls / watch
```

创建
```sh
create /app1
create /app2
```

```sh
# 临时节点
create -e /master "m"
```

```sh
# 顺序节点
create -s /master "m"
```

获取节点信息
```sh
get /app1

# 监听节点值变化
get /app1 watch
```

修改节点值
```sh
set /app1 "app1"
```

删除节点
```sh
delete /app1

# 递归删除节点
rmr /app1/master
```

节点状态
```sh
stat /app1

# 监控
stat -w /app1
```

## stat 结构

每次修改 zookeeper 状态都会收到一个 zxid 形式的时间戳，也就是 zookeeper 事务 id。事务 id 是 zookeeper 中所有修改总的次序。如果 zxid1 小于 zxid2，那么 zxid1 在 zxid2 之前发生

1. czxid：创建节点的事务 zxid
2. ctime：znode 被创建的毫秒数
3. mzxid：znode 最后更新的事务 zxid
4. mtime：znode 最后修改的毫秒数
5. pzxid：znode 最后更新的子节点 zxid
6. cversion：znode 子节点变化号，znode 子节点修改次数
7. dataversion：znode 数据变化号
8. aclVersion：znode 访问控制列表的变化号
9. ephemeralOwner：临时节点拥有者的 session id
10. dataLength：znode 的数据长度
11. numChildren：znode 子节点数量

条件更新
```sh
# -v 匹配版本
set -s -v 0 /count 1
set -s -v 1 /count 2

# -w 监听
get -s -w /count
```

```java
RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);

CuratorFramework client = CuratorFrameworkFactory.newClient(connectString, retryPolicy);
CuratorFramework client = CuratorFrameworkFactory.buidler().connectString(connectString).retryPolicy(retryPolicy).build();

client.start();
```

```java
// 同步版本
client.create().withMode(CreateMode.PERSISTENT).forPath(path, data);

// 异步版本
client.create().withMode(CreateMode.PERSISTENT).inBackground().forPath(path, data);

// 使用 watch
client.getData().watched().forPath(path);
```

```java
private CuratorFramework client;
private String connectString = "localhost:2181";
private RetryPolicy retryPolicy;

public void setUp() {
  retryPolicy = new ExponentialBackoffRetry(1000, 3);
  client = CuratorFrameworkFactory.newClient(connectString, retryPolicy);
  client.start();
}

public void tearDown() {
  client.close()
}

public void testSync() {
  String path = "sync";
  byte[] data = {'1'};
  client.create().withMode(CreateMode.PERSISTENT).forPath(path, data);
  byte[] actualData = client.getData().forPath(path);
  client.delete().forPath(path);
  client.close();
}

public void testAsync() {
  String path = "async";
  byte[] data = {'2'};
  CountDownLatch latch = new CountDownLatch(1);

  client.getCuratorListenable().addListener((CuratorFramework c, CuratorEvent event) -> {
    switch (event.getType) {
      case CREATE:
        // print event.getPath
        c.getData().inBackground().forPath(event.getPath());
        break;
      case GET_DATA:
        c.delete().inBackground().forPath(path);
        break;
      case DELETE:
        latch.countDown();
        break;
    }
  });

  client.create().withMode(CreateMode.PERSISTENT).inBackground().forPath(path, data);
  latch.await();
  client.close();
}

public void testWatch() {
  String path = "watch";
  byte[] data = {'3'};
  byte[] newData = {'4'};
  ClientDownLatch latch = new CountDownLatch(1);

  client.getCuratorListenable().addListener((CuratorFramework c, CuratorEvent event) -> {
    switch (event.getType()) {
      case WATCHED:
        WatchedEvent we = event.getWatchedEvent();

        if (we.getType() == Watcher.Event.EventType.NodeDataChanged && we.getPath().equals(path)) {
          byte[] actualData = c.getData().forPath(path);
        }

        latch.countDown();
        break;
    }
  });

  client.create().withMode(CreateMode.PERSISTENT).inBackground().forPath(path, data);
  byte[] actualData = client.getData().watched().forPath(path);
  client.setData().forPath(path, newData);
  latch.await();
  c.delete().forPath(path);
}

public void testWatchAndCallback() {
  String path = "/callback";
  byte[] data = {'5'};
  byte[] newData = {'6'};
  CountDownLatch latch = new CountDownLatch(2);

  client.getCuratorListenable().addListener((CuratorFramework c, CuratorEvent event) -> {
    switch (event.getType()) {
      case CREATE:
        client.getData().watched().forPath(path);
        client.setData().forPath(path, newData);
        latch.countDown();
        break;
      case WATCHED:
        WatchedEvent we = event.getWatchedEvent();

        if (we.getType() == Watcher.Event.EventType.NodeDataChanged && we.getPath().equals(path)) {
          byte[] actualData = c.getData().forPath(path);
        }

        latch.countDown();
    }
  });

  client.create().withMode(CreateMode.PERSISTENT).inBackground().forPath(path, data);
  latch.await();
  client.delete().forPath(path);
}
```

zookeeper 配置
```
# 保存快照文件的目录
dataDir
# 事务日志文件目录
dataLogDir
```

```sh
echo ruok | ncat localhost 2181
echo conf | ncat localhost 2181
echo stat | ncat localhost 2181

# 临时节点信息
echo dump | ncat localhost 2181

# 查看 watch
echo wchc | ncat localhost 2181

echo srvr | ncat localhost 2181
```

客户端写请求 -> observer/follower -> forward 到 leader -> propose 到 follower -> follower accept -> leader 发送 commit

observer 不参与提交和选举的投票过程，只会接受 leader 节点的 inform 消息。可以通过向集群中添加 observer 节点提高集群读性能
observer -> 跨数据中心部署

配置 observer
```
server.4=127.0.0.1:5550:5555:observer
```

动态配置
```java
String digest = DigestAuthenticationProvider.generateDigest("super:jingguo");
```

```
export SERVER_JVMFLAGS
export SERVER_JVMFLAGS=-Dzookeeper.DigestAuthenticationProvider.superDigest=<digest>
```

动态配置的配置文件
```
reconfigEnabled=true
dataDir=/data/zookeeper
syncLimit=5
tickTime=2000
initLimit=10
dynamicConfigFile=conf/dyn.cfg
```
```
server.1=ip1:2222:2223:participant;ip1:2181
server.2=ip2:2222:2223:participant;ip2:2181
server.3=ip3:2222:2223:participant;ip3:2181
```
```
zkServer.sh start zoo_reconf.cfg
```
```
zkCli.sh -server ip1:2181
addauth digest super:jingguo
config
reconfig -remvoe 3
config
```

```sh
zkTxnLogToolkit.sh log.1
```