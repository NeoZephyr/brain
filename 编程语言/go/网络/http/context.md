## 库函数

```bash
go doc context | grep "^func"
```

## 结构定义

```bash
go doc context | grep "^type"
```

## Context 原理

在树形逻辑链条上，一个节点其实有两个角色：一是下游树的管理者；二是上游树的被管理者。那么就对应需要有两个能力：一个是能让整个下游树结束的能力，也就是函数句柄 CancelFunc；另外一个是在上游树结束的时候被通知的能力，也就是 Done() 方法。同时因为通知是需要不断监听的，所以 Done() 方法需要通过 channel 作为返回值让使用方进行监听

![[Pasted image 20230215235532.png]]

其实每个连接的 Context 都是基于 baseContext 复制来的。对应到代码中就是，在为某个连接开启 Goroutine 的时候，为当前连接创建了一个 connContext，这个 connContext 是基于 server 中的 Context 而来，而 server 中 Context 的基础就是 baseContext

生成最终的 Context 的流程中，net/http 设计了两处可以注入修改的地方，都在 Server 结构里面，一处是 BaseContext，另一处是 ConnContext。BaseContext 是整个 Context 生成的源头，如果不希望使用默认的 context.Backgroud()，可以替换这个源头。而在每个连接生成自己要使用的 Context 时，会调用 ConnContext，它的第二个参数是 net.Conn，能让我们对某些特定连接进行设置，比如要针对性设置某个调用 IP