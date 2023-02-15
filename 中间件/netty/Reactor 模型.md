## Netty 示例

### 代码模版

```java
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup();
final EchoServerHandler serverHandler = new EchoServerHandler();
try {
    ServerBootstrap b = new ServerBootstrap();
    b.group(bossGroup, workerGroup)
            .channel(NioServerSocketChannel.class)
            .option(ChannelOption.SO_BACKLOG, 100)
            .handler(new LoggingHandler(LogLevel.INFO))
            .childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                public void initChannel(SocketChannel ch) throws Exception {
                    ChannelPipeline p = ch.pipeline();
                    p.addLast(new LoggingHandler(LogLevel.INFO));
                    p.addLast(serverHandler);
                }
            });

    ChannelFuture f = b.bind(8080).sync();
    f.channel().closeFuture().sync();
} finally {
    bossGroup.shutdownGracefully();
    workerGroup.shutdownGracefully();
}
```

```java
@Sharable  
public class EchoServerHandler extends ChannelInboundHandlerAdapter {  
  
    @Override  
    public void channelRead(ChannelHandlerContext ctx, Object msg) {  
        ctx.write(msg);  
    }  
  
    @Override  
    public void channelReadComplete(ChannelHandlerContext ctx) {  
        ctx.flush();  
    }  
  
    @Override  
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {  
        cause.printStackTrace();  
        ctx.close();  
    }  
}
```

### 模版注释

创建主从 Reactor Group，在 Netty 中，EventLoopGroup 就是 Reactor Group 的实现类
```java
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup();
```

![[Excalidraw/reactor 模型|960]]

主 Reactor Group 中的 Reactor 数量取决于服务端要监听的端口个数，通常只会监听一个端口，所以主 Reactor Group 只会有一个 Reactor 线程来处理最重要的事情：绑定端口地址，接收客户端连接，为客户端创建对应的 SocketChannel，将客户端 SocketChannel 分配给一个固定的子 Reactor

子 Reactor Group 里有多个 Reactor 线程，Reactor 线程的个数可以通过系统参数 `-D io.netty.eventLoopThreads` 指定。默认的 Reactor 个数为 `CPU 核数 * 2`。子 Reactor 线程主要用来轮询客户端 SocketChannel 上的 IO 就绪事件，处理 IO 就绪事件，执行异步任务

一个客户端 SocketChannel 只能分配给一个固定的子 Reactor。一个子 Reactor 负责处理多个客户端 SocketChannel，这样可以将服务端承载的全量客户端连接分摊到多个子 Reactor 中处理，同时也能保证客户端 SocketChannel 上的 IO 处理的线程安全性

> netty 有两种 Channel 类型：一种是服务端用于监听绑定端口地址的 NioServerSocketChannel，一种是用于客户端通信的 NioSocketChannel。每种 Channel 类型实例都会对应一个 Pipeline，用于编排对应 channel 实例上的 IO 事件处理逻辑。Pipeline 中组织的就是 ChannelHandler 用于编写特定的IO处理逻辑

ServerBootstrap 启动类方法带有 child 前缀的均是设置客户端 NioSocketChannel 属性的

ChannelInitializer 是当 SocketChannel 成功注册到绑定的 Reactor 上后，用于初始化该 SocketChannel 的 Pipeline。它的 initChannel 方法会在注册成功后执行

> serverBootstrap.handler 设置服务端 NioServerSocketChannel 中对应 Pipieline 中的 ChannelHandler
> 
> serverBootstrap.childHandler 用于设置客户端 NioSocketChannel 中对应 Pipieline 中的 ChannelHandler
>
> serverBootstrap.option 设置服务端 ServerSocketChannel 中的 SocketOption

## 主从 Reactor 组

Netty 中 EventLoop 线程组的实现类为 NioEventLoopGroup，在创建 bossGroup 和 workerGroup 的时候用到了 NioEventLoopGroup 的两个构造函数

```java
public class NioEventLoopGroup extends MultithreadEventLoopGroup {
	
	public NioEventLoopGroup() {  
	    this(0);  
	}

	public NioEventLoopGroup(int nThreads) {  
	    this(nThreads, (Executor) null);  
	}

	public NioEventLoopGroup(
		int nThreads, Executor executor,
		final SelectorProvider selectorProvider,
		final SelectStrategyFactory selectStrategyFactory) {  
	    super(nThreads, executor, selectorProvider, selectStrategyFactory, RejectedExecutionHandlers.reject());  
	}
}
```

### 继承结构

![[Pasted image 20230211110830.png]]

NioEventLoopGroup 是 Netty 中的 Reactor 线程组的实现，负责管理和创建多个 Reactor 线程，所以 Mutithread 前缀的类定义的行为自然是对 Reactor 线程组内多个 Reactor 线程的创建和管理工作

### 参数说明

nThreads：表示当前要创建的 Reactor 线程组内包含多少个 Reactor 线程。不指定 nThreads 参数的话采用默认的 Reactor 线程个数，用 0 表示

Executor：负责启动 Reactor 线程，进而 Reactor 才可以开始工作。Reactor 线程组负责创建 Reactor 线程，在创建的时候会将 executor 传入

RejectedExecutionHandler：当向 Reactor 添加异步任务添加失败时，采用的拒绝策略。

SelectorProvider：Reactor 中的 IO 模型为 IO 多路复用模型，对应于 JDK NIO 中的实现为 `java.nio.channels.Selector`，每个 Reator 中都包含一个 Selector，用于轮询注册在该 Reactor 上的所有 Channel 上的 IO 事件。SelectorProvider 就是用来创建 Selector 的

SelectStrategyFactory：Reactor 最重要的事情就是轮询注册其上的 Channel 上的 IO 就绪事件，这里的 SelectStrategyFactory 用于指定轮询策略，默认为 DefaultSelectStrategyFactory.INSTANCE

### MultithreadEventLoopGroup

```java
public abstract class MultithreadEventLoopGroup extends MultithreadEventExecutorGroup implements EventLoopGroup {

	private static final int DEFAULT_EVENT_LOOP_THREADS;  
  
	static {  
	    DEFAULT_EVENT_LOOP_THREADS = Math.max(
		    1,
			SystemPropertyUtil.getInt(
				"io.netty.eventLoopThreads",
				NettyRuntime.availableProcessors() * 2
			)
		);
	}

	protected MultithreadEventLoopGroup(
		int nThreads, Executor executor, Object... args) {  
	    super(
		    nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads,
			executor,
			args
		);
	}
}
```

MultithreadEventLoopGroup 类主要的功能就是用来确定 Reactor 线程组内 Reactor 的个数。默认的 Reactor 的个数存放于字段 DEFAULT_EVENT_LOOP_THREADS 中。从 static 静态代码块中可以看出默认 Reactor 的个数的获取逻辑：

- 通过系统变量 `-D io.netty.eventLoopThreads` 指定   
- 如果不指定，那么默认的就是 `NettyRuntime.availableProcessors() * 2`

### MultithreadEventExecutorGroup

```java
public abstract class MultithreadEventExecutorGroup
	extends AbstractEventExecutorGroup {  

	private final EventExecutor[] children;
    private final Set<EventExecutor> readonlyChildren;
    private final AtomicInteger terminatedChildren = new AtomicInteger();
    private final Promise<?> terminationFuture = new DefaultPromise(GlobalEventExecutor.INSTANCE);
    private final EventExecutorChooserFactory.EventExecutorChooser chooser;

	protected MultithreadEventExecutorGroup(int nThreads, ThreadFactory threadFactory, Object... args) {
		this(
			nThreads,
			threadFactory == null ? null : new ThreadPerTaskExecutor(threadFactory),
			args
		);
	}

	protected MultithreadEventExecutorGroup(int nThreads, Executor executor, Object... args) {  
	    this(
		    nThreads, executor,
			DefaultEventExecutorChooserFactory.INSTANCE, args
		);  
	}
}
```

主 Reactor 会创建客户端连接 NioSocketChannel，并将其绑定到子 Reactor 组中的一个固定 Reactor。具体要绑定到哪个子 Reactor 上，由 EventExecutorChooserFactory 来创建的绑定策略决定。默认为 DefaultEventExecutorChooserFactory

```java
protected MultithreadEventExecutorGroup(
	int nThreads, Executor executor,
	EventExecutorChooserFactory chooserFactory, Object... args) {

    if (executor == null) {
	    // 用于创建 Reactor 线程
        executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
    }

    children = new EventExecutor[nThreads];

	// 循环创建 Reactor
    for (int i = 0; i < nThreads; i ++) {
        boolean success = false;
        try {
            children[i] = newChild(executor, args);
            success = true;
        } catch (Exception e) {
            // 抛出异常
        } finally {
            if (!success) {
	            // shutdown 已经创建的，并等待 terminate
	            // ...
            }
        }
    }

	// 创建 channel 到 Reactor 的绑定策略
    chooser = chooserFactory.newChooser(children);

	// 为每个 Reactor 添加 terminationListener
}
```

## Reactor

![[reactor 构成]]

### ThreadPerTaskExecutor

```java
public final class ThreadPerTaskExecutor implements Executor {  
    private final ThreadFactory threadFactory;  
  
    public ThreadPerTaskExecutor(ThreadFactory threadFactory) {  
        if (threadFactory == null) {  
            throw new NullPointerException("threadFactory");  
        }  
        this.threadFactory = threadFactory;  
    }  
  
    @Override  
    public void execute(Runnable command) {  
        threadFactory.newThread(command).start();  
    }  
}
```

ThreadPerTaskExecutor 做的事情很简单，就是来一个任务就创建一个线程执行。而创建的这个线程正是 netty 的核心引擎 Reactor 线程。在 Reactor 线程启动的时候，Netty 会将 Reactor 线程要做的事情封装成 Runnable，丢给 exexutor 启动。

Reactor 线程的核心就是一个死循环不停的轮询 IO 就绪事件，处理 IO 事件，执行异步任务

### 创建 Reactor

```java
public class NioEventLoopGroup extends MultithreadEventLoopGroup {
	protected EventLoop newChild(Executor executor, Object... args) throws Exception {  
	    EventLoopTaskQueueFactory queueFactory =
			args.length == 4 ? (EventLoopTaskQueueFactory) args[3] : null;  
	    return new NioEventLoop(
		    this,
			executor,
			(SelectorProvider) args[0],
	        ((SelectStrategyFactory) args[1]).newSelectStrategy(),
	        (RejectedExecutionHandler) args[2],
			queueFactory
		);
	}
}
```

之前提到的众多构造器参数，在这里会通过可变参数 args 传入到 Reactor 类 NioEventLoop 的构造器中

Netty 为了极致的压榨 Reactor 的性能，除了处理 IO 事件以外，还会让它做一些异步任务的执行工作。这些异步任务需要一个队列来保存，EventLoopTaskQueueFactory 就是用来创建这个队列的

### NioEventLoop

```java
public final class NioEventLoop extends SingleThreadEventLoop {

	private Selector selector;
    private Selector unwrappedSelector;
    private SelectedSelectionKeySet selectedKeys;

    private final SelectorProvider provider;

    NioEventLoop(NioEventLoopGroup parent, Executor executor, SelectorProvider selectorProvider, SelectStrategy strategy, RejectedExecutionHandler rejectedExecutionHandler, EventLoopTaskQueueFactory queueFactory) {  
        super(
	        parent, executor, false,
	        newTaskQueue(queueFactory),
			newTaskQueue(queueFactory),
			rejectedExecutionHandler
		);

        if (selectorProvider == null) {
            throw new NullPointerException("selectorProvider");
        }
        if (strategy == null) {
            throw new NullPointerException("selectStrategy");
        }
        provider = selectorProvider;
        final SelectorTuple selectorTuple = openSelector();
        selector = selectorTuple.selector;
        unwrappedSelector = selectorTuple.unwrappedSelector;
        selectStrategy = strategy;
    }
}
```

#### SelectorProvider

```java
private SelectorTuple openSelector() {
    final Selector unwrappedSelector;
    try {
        unwrappedSelector = provider.openSelector();
    } catch (IOException e) {
        throw new ChannelException("failed to open a new selector", e);
    }

    if (DISABLE_KEY_SET_OPTIMIZATION) {
        return new SelectorTuple(unwrappedSelector);
    }

	// ...
}
```

openSelector 是 NioEventLoop 类中用于创建 IO 多路复用的 Selector，并对创建出来的 JDK NIO 原生的 Selector 进行性能优化

SelectorProvider 是在前面介绍的 NioEventLoopGroup 类构造函数中通过调用`SelectorProvider.provider()` 被加载，并通过 `NioEventLoopGroup#newChild` 方法中的可变长参数 `Object... args` 传递到 `NioEventLoop` 中的 `provider` 字段中

```java
public abstract class SelectorProvider {
	public static SelectorProvider provider() {
	    synchronized (lock) {
	        if (provider != null)
	            return provider;
	        return AccessController.doPrivileged(
	            new PrivilegedAction<>() {
	                public SelectorProvider run() {
	                        if (loadProviderFromProperty())
	                            return provider;
	                        if (loadProviderAsService())
	                            return provider;
	                        provider = sun.nio.ch.DefaultSelectorProvider.create();
	                        return provider;
	                    }
	                });  
	    }
	}
}
```

SelectorProvider 的加载方式有三种，优先级如下：

1. 通过系统变量 `-D java.nio.channels.spi.SelectorProvider` 指定 `SelectorProvider` 的自定义实现类全限定名
```java
private static boolean loadProviderFromProperty() {  
    String cn = System.getProperty("java.nio.channels.spi.SelectorProvider");

    if (cn == null)
        return false;

	@SuppressWarnings("deprecation")
    Object tmp = Class.forName(
	    cn,
		true,
		ClassLoader.getSystemClassLoader()
	).newInstance();
    provider = (SelectorProvider) tmp;
    return true;
}
```

2. 通过 SPI 方式加载。在工程目录 `META-INF/services` 下定义名为 `java.nio.channels.spi.SelectorProvider` 的 SPI 文件，文件中第一个定义的 SelectorProvider 实现类全限定名就会被加载
```java
private static boolean loadProviderAsService() {
    ServiceLoader<SelectorProvider> sl =
        ServiceLoader.load(
	        SelectorProvider.class,
			ClassLoader.getSystemClassLoader());
    Iterator<SelectorProvider> i = sl.iterator();
    for (;;) {
        try {
            if (!i.hasNext())
                return false;
            provider = i.next();
            return true;
        } catch (ServiceConfigurationError sce) {
            // ...
        }
    }
}
```

3. 系统默认实现 `sun.nio.ch.DefaultSelectorProvider`
```java
public class DefaultSelectorProvider {

	private DefaultSelectorProvider() {}

	public static SelectorProvider create() {
        return new KQueueSelectorProvider();
    }
}
```

#### Selector 优化

NioEventLoop 中有一个 Selector 优化开关 `DISABLE_KEY_SET_OPTIMIZATION`，通过系统变量 `-D io.netty.noKeySetOptimization` 指定，默认是开启的

1. 判断由 SelectorProvider 创建出来的 Selector 是否是 JDK NIO 原生的 Selector 实现。因为 Netty 优化针对的是 JDK NIO 原生 Selector。判断标准为 `sun.nio.ch.SelectorImpl` 类是否为 SelectorProvider 创建出 Selector 的父类。如果不是则直接返回。不在继续下面的优化过程

```java
abstract class SelectorImpl extends AbstractSelector {

	// 所有注册到该 Selector 上的 Channel
    private final Set<SelectionKey> keys;

	// 就绪队列
    private final Set<SelectionKey> selectedKeys;

	// 向外部线程返回所有注册在该 Selector 上的 SelectionKey
    private final Set<SelectionKey> publicKeys;

	// 用于向外部线程返回 IO 就绪的 SelectionKey
    private final Set<SelectionKey> publicSelectedKeys;
	private boolean inSelect;
}
```

2. 创建 SelectedSelectionKeySet 通过反射替换掉 `sun.nio.ch.SelectorImpl` 类中 selectedKeys 和 publicSelectedKeys 的默认 HashSet 实现

对 HashSet 类型的 `sun.nio.ch.SelectorImpl#selectedKeys` 集合有两种操作：
- 插入操作：在 Selector 监听到 IO 就绪的 SelectionKey 后，会将 IO 就绪的 SelectionKey 插入 `sun.nio.ch.SelectorImpl#selectedKeys` 集合中，这时 Reactor 线程会从 `java.nio.channels.Selector#select(long)` 阻塞调用中返回
- 遍历操作：Reactor 线程返回后，会从 Selector 中获取 IO 就绪的 SelectionKey 集合，Reactor 线程遍历 selectedKeys，获取 IO 就绪的 SocketChannel，并处理 SocketChannel 上的 IO 事件    

由于 Hash 冲突的情况存在，导致对哈希表进行插入和遍历操作的性能不如对数组进行插入和遍历操作的性能好。而且，数组可以利用 CPU 缓存的优势来提高遍历的效率。所以 Netty 为了优化对 `sun.nio.ch.SelectorImpl#selectedKeys` 集合的插入，遍历性能，自己用数组这种数据结构实现了 `SelectedSelectionKeySet`，用它来替换原来的 HashSet 实现

```java
final class SelectedSelectionKeySet extends AbstractSet<SelectionKey> {

    SelectionKey[] keys;
    int size;

    SelectedSelectionKeySet() {
        keys = new SelectionKey[1024];
    }

	@Override
	public boolean add(SelectionKey o) {
	    if (o == null) {
	        return false;
	    }

	    keys[size++] = o;

	    if (size == keys.length) {
	        increaseCapacity();
	    }

	    return true;
	}

	@Override  
	public Iterator<SelectionKey> iterator() {
	    return new Iterator<SelectionKey>() {
	        private int idx;

	        @Override
	        public boolean hasNext() {
	            return idx < size;
	        }

	        @Override
	        public SelectionKey next() {
	            if (!hasNext()) {
	                throw new NoSuchElementException();
	            }
	            return keys[idx++];
	        }

	        @Override
	        public void remove() {
	            throw new UnsupportedOperationException();
	        }
	    };
	}

	private void increaseCapacity() {
	    SelectionKey[] newKeys = new SelectionKey[keys.length << 1];
	    System.arraycopy(keys, 0, newKeys, 0, size);
	    keys = newKeys;
	}
}
```

- 初始化 `SelectionKey[] keys` 数组大小为 1024，当数组容量不够时，扩容为原来的两倍大小
- 通过数组尾部指针，在向数组插入元素的时候可以直接定位到插入位置。操作一步到位，不用像哈希表那样还需要解决 Hash 冲突

3. Netty 通过反射的方式用 SelectedSelectionKeySet 替换掉 selectedKeys 和 publicSelectedKeys 这两个集合中原来 HashSet 的实现

4. 将与 `sun.nio.ch.SelectorImpl` 类中 selectedKeys 和 publicSelectedKeys 关联好的 Netty 优化实现 SelectedSelectionKeySet，设置到 `io.netty.channel.nio.NioEventLoop#selectedKeys` 字段中保存。后续 Reactor 线程就会直接从 `io.netty.channel.nio.NioEventLoop#selectedKeys` 中获取 IO 就绪的 SocketChannel

5. 用 SelectorTuple 封装 unwrappedSelector 和 wrappedSelector 返回给 NioEventLoop 构造函数

#### 异步任务队列

```java
public final class NioEventLoop extends SingleThreadEventLoop {
	private static Queue<Runnable> newTaskQueue(
		EventLoopTaskQueueFactory queueFactory) {

		if (queueFactory == null) {
	        return newTaskQueue0(DEFAULT_MAX_PENDING_TASKS);
	    }
	    return queueFactory.newTaskQueue(DEFAULT_MAX_PENDING_TASKS);
	}

	private static Queue<Runnable> newTaskQueue0(int maxPendingTasks) {
	    return maxPendingTasks == Integer.MAX_VALUE ?
			PlatformDependent.<Runnable>newMpscQueue() :
			PlatformDependent.<Runnable>newMpscQueue(maxPendingTasks);
	}
}
```

NioEventLoop 的父类 SingleThreadEventLoop 提供了一个静态变量 DEFAULT_MAX_PENDING_TASKS 来指定任务队列的大小。可以通过系统变量 `-D io.netty.eventLoop.maxPendingTasks` 进行设置，默认为 `Integer.MAX_VALUE`

> Reactor 内的异步任务队列的类型为 MpscQueue，它是由 JCTools 提供的一个高性能无锁队列，适用于多生产者单消费者的场景
> 
> Reactor 可以线程安全的处理注册其上的多个 SocketChannel 上的 IO 数据，保证线程安全的核心原因正是因为这个 MpscQueue，它可以支持多个业务线程在处理完业务逻辑后，线程安全的向 MpscQueue 添加异步写任务，然后由单个 Reactor 线程来执行这些写任务

#### 继承结构

![[NioEventLoop.png]]

#### SingleThreadEventLoop

Reactor 负责执行的异步任务分为三类：
1. 普通任务：Netty 最主要执行的异步任务，存放在普通任务队列 taskQueue 中。在 NioEventLoop 构造函数中创建
2. 定时任务：存放在优先级队列中
3. 尾部任务：存放于尾部任务队列 tailTasks 中，在普通任务执行完后，Reactor 线程会执行尾部任务

> 对 Netty 的运行状态做一些统计数据，例如任务循环的耗时、占用物理内存的大小等都可以向尾部队列添加一个收尾任务完成统计数据的实时更新

```java
public abstract class SingleThreadEventLoop extends SingleThreadEventExecutor implements EventLoop {

    private final Queue<Runnable> tailTasks;

    protected SingleThreadEventLoop(EventLoopGroup parent, Executor executor,
                                    boolean addTaskWakesUp,
									Queue<Runnable> taskQueue,
									Queue<Runnable> tailTaskQueue,
                                    RejectedExecutionHandler rejectedExecutionHandler) {
        super(parent, executor, addTaskWakesUp, taskQueue, rejectedExecutionHandler);
        tailTasks = ObjectUtil.checkNotNull(tailTaskQueue, "tailTaskQueue");
    }

    @Override
    public ChannelFuture register(Channel channel) {
        return register(new DefaultChannelPromise(channel, this));
    }

    @Override
    public ChannelFuture register(final ChannelPromise promise) {
        ObjectUtil.checkNotNull(promise, "promise");
        promise.channel().unsafe().register(this, promise);
        return promise;
    }
}
```

#### SingleThreadEventExecutor

```java
public abstract class SingleThreadEventExecutor extends AbstractScheduledEventExecutor implements OrderedEventExecutor {

    private final Queue<Runnable> taskQueue;
    private volatile Thread thread;
    private final Executor executor;

    protected SingleThreadEventExecutor(EventExecutorGroup parent,
										Executor executor,
                                        boolean addTaskWakesUp,
										Queue<Runnable> taskQueue,
                                        RejectedExecutionHandler rejectedHandler) {
        super(parent);
        this.addTaskWakesUp = addTaskWakesUp;
        this.maxPendingTasks = DEFAULT_MAX_PENDING_EXECUTOR_TASKS;
        this.executor = ThreadExecutorMap.apply(executor, this);
        this.taskQueue = ObjectUtil.checkNotNull(taskQueue, "taskQueue");
        rejectedExecutionHandler = ObjectUtil.checkNotNull(rejectedHandler, "rejectedHandler");
    }
}
```

### Channel 到 Reactor 的绑定策略

Channel 到 Reactor 的绑定策略。默认为 `DefaultEventExecutorChooserFactory.INSTANCE`

```java
public final class DefaultEventExecutorChooserFactory implements EventExecutorChooserFactory {

    public static final DefaultEventExecutorChooserFactory INSTANCE = new DefaultEventExecutorChooserFactory();

    private DefaultEventExecutorChooserFactory() {}

    @Override
    public EventExecutorChooser newChooser(EventExecutor[] executors) {
        if (isPowerOfTwo(executors.length)) {
            return new PowerOfTwoEventExecutorChooser(executors);
        } else {
            return new GenericEventExecutorChooser(executors);
        }
    }

    private static boolean isPowerOfTwo(int val) {
        return (val & -val) == val;
    }
}
```

Netty 中的绑定策略就是采用 round-robin 轮询的方式来挨个选择 Reactor 进行绑定。如果 Reactor 的个数 reactor.length 恰好是 2 的次幂，那么就可以用位操作 & 运算来代替 % 运算，因为位运算的性能更高

### 注册 terminated 回调函数

```java
private final AtomicInteger terminatedChildren = new AtomicInteger();
private final Promise<?> terminationFuture =
	new DefaultPromise(GlobalEventExecutor.INSTANCE);

final FutureListener<Object> terminationListener = new FutureListener<Object>() {  
    @Override
    public void operationComplete(Future<Object> future) throws Exception {
        if (terminatedChildren.incrementAndGet() == children.length) {
            terminationFuture.setSuccess(null);
        }
    }
};

for (EventExecutor e: children) {
    e.terminationFuture().addListener(terminationListener);
}
```

创建 Reactor 关闭的回调函数 terminationListener，在 Reactor 关闭时回调。terminationListener 回调的逻辑很简单：
- 通过 `AtomicInteger terminatedChildren` 变量记录已经关闭的 Reactor 个数，用来判断 NioEventLoopGroup 中的 Reactor 是否已经全部关闭
- 如果所有 Reactor 均已关闭，设置 NioEventLoopGroup 中的 terminationFuture 为 success。表示 Reactor 线程组关闭成功