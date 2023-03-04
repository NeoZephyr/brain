主 Reactor 组中的 Reactor 管理的 Channel 类型为 NioServerSocketChannel，主要用来监听端口，接收客户端连接，为客户端创建初始化 NioSocketChannel，然后采用 round-robin 轮询的方式从 Reactor 组中选择一个 Reactor 与该客户端 NioSocketChannel 进行绑定

从 Reactor 组中的 Reactor 管理的 Channel 类型为 NioSocketChannel，每个连接对应一个。从 Reactor 负责监听处理绑定在其上的所有 NioSocketChannel 上的 IO 事件

## ServerBootstrap

ServerBootstrap 主要负责对主从 Reactor 线程组相关的配置进行管理，其中带 child 前缀的配置方法是对从 Reactor 线程组的相关配置管理。从 Reactor 线程组中的 Reactor 负责管理的客户端 NioSocketChannel 相关配置存储在 ServerBootstrap 结构中

父类 AbstractBootstrap 则是主要负责对主 Reactor 线程组相关的配置进行管理，以及主 Reactor 线程组中的 Reactor 负责处理的服务端 ServerSocketChannel 相关的配置管理

![[ServerBootstrap.png]]

### 配置主从线程组

```java
public class ServerBootstrap extends AbstractBootstrap<ServerBootstrap, ServerChannel> {
	private volatile EventLoopGroup childGroup;
	private volatile ChannelHandler childHandler;

	public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup) {
	    super.group(parentGroup);
	    ObjectUtil.checkNotNull(childGroup, "childGroup");

	    if (this.childGroup != null) {
	        throw new IllegalStateException("childGroup set already");
	    }
	    this.childGroup = childGroup;
	    return this;
	}
}
```

### 配置服务端 Channel

```java
public abstract class
	AbstractBootstrap<B extends AbstractBootstrap<B, C>, C extends Channel>
	implements Cloneable {

	public B channel(Class<? extends C> channelClass) {
	    return channelFactory(new ReflectiveChannelFactory<C>(
	            ObjectUtil.checkNotNull(channelClass, "channelClass")
	    ));
	}

	public B channelFactory(ChannelFactory<? extends C> channelFactory) {
	    ObjectUtil.checkNotNull(channelFactory, "channelFactory");
	    if (this.channelFactory != null) {
	        throw new IllegalStateException("channelFactory set already");
	    }

	    this.channelFactory = channelFactory;
	    return self();
	}
}
```

```java
public class ReflectiveChannelFactory<T extends Channel>
	implements ChannelFactory<T> {

	private final Constructor<? extends T> constructor;

	public ReflectiveChannelFactory(Class<? extends T> clazz) {
	    ObjectUtil.checkNotNull(clazz, "clazz");
	    try {
	        this.constructor = clazz.getConstructor();
	    } catch (NoSuchMethodException e) {
	        throw new IllegalArgumentException("Class " +
				StringUtil.simpleClassName(clazz) +
				" does not have a public non-arg constructor", e);
	    }
	}

	public T newChannel() {
	    try {
	        return constructor.newInstance();
	    } catch (Throwable t) {
	        throw new ChannelException("Unable to create Channel from class " +
				constructor.getDeclaringClass(), t);
	    }
	}
}
```

### 配置 ChannelOption

无论是服务端的 NioServerSocketChannel 还是客户端的 NioSocketChannel 它们的相关底层 Socket 选项 ChannelOption 配置全部存放于一个 Map 类型的数据结构中

```java
public abstract class
	AbstractBootstrap<B extends AbstractBootstrap<B, C>, C extends Channel>
	implements Cloneable {

	private final Map<ChannelOption<?>, Object> options =
		new ConcurrentHashMap<ChannelOption<?>, Object>();

	public <T> B option(ChannelOption<T> option, T value) {
	    ObjectUtil.checkNotNull(option, "option");

	    if (value == null) {
	        options.remove(option);
	    } else {
	        options.put(option, value);
	    }
	    return self();
	}
}
```

```java
public class ServerBootstrap
	extends AbstractBootstrap<ServerBootstrap, ServerChannel> {

	private final Map<ChannelOption<?>, Object> childOptions =
		new ConcurrentHashMap<ChannelOption<?>, Object>();

	public <T> ServerBootstrap childOption(ChannelOption<T> childOption,
										   T value) {
	    ObjectUtil.checkNotNull(childOption, "childOption");

		if (value == null) {
	        childOptions.remove(childOption);
	    } else {
	        childOptions.put(childOption, value);
	    }
	    return this;
	}
}
```

相关的底层 Socket 选项，netty 全部枚举在 ChannelOption 类中

```java
public class ChannelOption<T> extends AbstractConstant<ChannelOption<T>> {
	public static final ChannelOption<Integer> SO_BACKLOG =
		valueOf("SO_BACKLOG");
	public static final ChannelOption<Boolean> SO_KEEPALIVE =
		valueOf("SO_KEEPALIVE");
}
```

### NioServerSocketChannel 配置 ChannelHandler

```java
public abstract
	class AbstractBootstrap<B extends AbstractBootstrap<B, C>, C extends Channel>
	implements Cloneable {

	public B handler(ChannelHandler handler) {
		this.handler = ObjectUtil.checkNotNull(handler, "handler");
		return self();
	}
}
```

向 NioServerSocketChannel 中的 Pipeline 添加 ChannelHandler 分为两种方式：

1. 显式添加：通过 ServerBootstrap#handler 的方式添加。如果需要添加多个 ChannelHandler，可以通过 ChannelInitializer 向 pipeline 中进行添加
2. 隐式添加：隐式添加主要添加的就是主 Reactor 组的核心组件 ServerBootstrapAcceptor，主要负责在客户端连接建立好后，初始化客户端 NioSocketChannel，在从 Reactor 组中选取一个 Reactor，将客户端 NioSocketChannel 注册到 Reactor 中的 selector 上

### NioSocketChannel 配置 ChannelHandler

```java
public class ServerBootstrap
	extends AbstractBootstrap<ServerBootstrap, ServerChannel> {

	public ServerBootstrap childHandler(ChannelHandler childHandler) {
	    this.childHandler = ObjectUtil.checkNotNull(
		    childHandler, "childHandler");
	    return this;
	}
}
```

在 Netty 的 IO 线程模型中，由单个从 Reactor 线程负责执行客户端 NioSocketChannel 中的 Pipeline，一个从 Reactor 线程负责处理多个 NioSocketChannel 上的 IO 事件，如果 Pipeline 中的 ChannelHandler 添加的太多，就会影响 Reactor 线程执行其他 NioSocketChannel 上的 Pipeline，从而降低 IO 处理效率

所以 Pipeline 中的 ChannelHandler 不易添加过多，并且不能在 ChannelHandler 中执行耗时的业务处理任务

## ChannelInitializer

![[ChannelInitializer.png]]

ChannelInitializer 继承于 ChannelHandler，本身就是一个ChannelHandler，那为什么不直接添加 ChannelHandler 而是选择用 ChannelInitializer 呢？这里主要有两点原因：

NioSocketChannel 是在服务端 accept 连接后，在 NioServerSocketChannel 中被创建出来的。但是此时正处于配置 ServerBootStrap 阶段，服务端还没有启动，更没有客户端连接上来，此时 NioSocketChannel 还没有被创建出来，所以也就没办法向 NioSocketChannel 的 pipeline 添加 ChannelHandler

NioSocketChannel 中 Pipeline 里可以添加任意多个 ChannelHandler，但是 Netty 框架无法预知用户到底需要添加多少个 ChannelHandler，所以 Netty 框架提供了回调函数 `ChannelInitializer#initChannel`，使用户可以自定义 ChannelHandler 的添加行为

当 NioSocketChannel 注册到对应的从 Reactor 上后，紧接着就会初始化 NioSocketChannel 中的 Pipeline，此时 Netty 框架会回调 `ChannelInitializer#initChannel` 执行用户自定义的添加逻辑

```java
public abstract class ChannelInitializer<C extends Channel>
	extends ChannelInboundHandlerAdapter {

	// ChannelInitializer 实例是被所有的 Channel 共享的
	// 通过 Set 集合保存已经初始化的 pipeline，避免重复初始化同一 pipeline
	private final Set<ChannelHandlerContext> initMap = Collections.newSetFromMap(  
        new ConcurrentHashMap<ChannelHandlerContext, Boolean>());

	// 自定义的初始化逻辑
	protected abstract void initChannel(C ch) throws Exception;

	public final void channelRegistered(ChannelHandlerContext ctx)
		throws Exception {
		if (initChannel(ctx)) {
			ctx.pipeline().fireChannelRegistered();
			removeState(ctx);
		} else {
			ctx.fireChannelRegistered();
		}
	}

	public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
		if (ctx.channel().isRegistered()) {
			if (initChannel(ctx)) {
				removeState(ctx);
			}
		}
	}

	private boolean initChannel(ChannelHandlerContext ctx) throws Exception {
	
		if (initMap.add(ctx)) {
			try {
				initChannel((C) ctx.channel());
			} catch (Throwable cause) {
				exceptionCaught(ctx, cause);
			} finally {
				ChannelPipeline pipeline = ctx.pipeline();
				if (pipeline.context(this) != null) {
					pipeline.remove(this);
				}
			}
			return true;
		}
		return false;
	}
}
```

## 服务端启动

```java
public abstract
	class AbstractBootstrap<B extends AbstractBootstrap<B, C>, C extends Channel> 
	implements Cloneable {

	public ChannelFuture bind(int inetPort) {  
	    return bind(new InetSocketAddress(inetPort));  
	}

	public ChannelFuture bind(SocketAddress localAddress) {  
	    validate();  
	    return doBind(ObjectUtil.checkNotNull(localAddress, "localAddress"));  
	}

	private ChannelFuture doBind(final SocketAddress localAddress) {
		final ChannelFuture regFuture = initAndRegister();
		final Channel channel = regFuture.channel();

		if (regFuture.cause() != null) {
			return regFuture;
		}

		if (regFuture.isDone()) {
			ChannelPromise promise = channel.newPromise();
			doBind0(regFuture, channel, localAddress, promise);
			return promise;
		} else {
			final PendingRegistrationPromise promise =
				new PendingRegistrationPromise(channel);

			// 如果此时注册还没有完成，则添加注册成功的回调函数
			regFuture.addListener(new ChannelFutureListener() {
				@Override
				public void operationComplete(ChannelFuture future)
					throws Exception {
					Throwable cause = future.cause();

					if (cause != null) {
						promise.setFailure(cause);
					} else {
						promise.registered();
						doBind0(regFuture, channel, localAddress, promise);
					}
				}
			});
			return promise;
		}
	}
}
```

Netty 服务端的启动流程总体如下：

1. 创建服务端 NioServerSocketChannel 并初始化
2. 将服务端 NioServerSocketChannel 注册到主 Reactor 线程组中
3. 注册成功后，初始化 NioServerSocketChannel 中的 pipeline，然后在 pipeline 中触发 channelRegister 事件
4. 绑定端口地址，然后向 NioServerSocketChannel 对应的 Pipeline 中触发传播 ChannelActive 事件，在 ChannelActive 事件回调中向主 Reactor 注册 OP_ACCEPT 事件，开始等待客户端连接

### initAndRegister

```java
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
        channel = channelFactory.newChannel();
        init(channel);
    } catch (Throwable t) {
	    // ...
    }

    ChannelFuture regFuture = config().group().register(channel);

    // ...

    return regFuture;
}
```

#### Init 过程

##### 创建

```java
public class
	NioServerSocketChannel extends AbstractNioMessageChannel
	implements io.netty.channel.socket.ServerSocketChannel {

	private static final SelectorProvider DEFAULT_SELECTOR_PROVIDER =
		SelectorProvider.provider();

	// 创建 JDK NIO 原生 ServerSocketChannel
	private static ServerSocketChannel newSocket(SelectorProvider provider) {
	    try {
		    return provider.openServerSocketChannel();
	    } catch (IOException e) {
		    throw new ChannelException("Failed to open a server socket.", e);
	    }
	}

	private final ServerSocketChannelConfig config;

	public NioServerSocketChannel() {
	    this(newSocket(DEFAULT_SELECTOR_PROVIDER));
	}

	public NioServerSocketChannel(ServerSocketChannel channel) {
		// 设置感兴趣的 IO 事件，并将 JDK NIO 原生 ServerSocketChannel 封装起来
	    super(null, channel, SelectionKey.OP_ACCEPT);
	    config = new NioServerSocketChannelConfig(this, javaChannel().socket()); 
	}
}
```

配置类 NioServerSocketChannelConfig 中封装了对 Channel 底层的一些配置行为，以及 JDK 中的 ServerSocket。以及创建 NioServerSocketChannel 接收数据用的 Buffer 分配器 AdaptiveRecvByteBufAllocator

![[NioServerSocketChannel.png]]

AbstractNioMessageChannel 类主要是对 NioServerSocketChannel 底层读写行为的封装和定义，比如 accept 接收客户端连接

```java
public abstract class AbstractNioChannel extends AbstractChannel {

	// JDK NIO 原生 ServerSocketChannel
	private final SelectableChannel ch;

	// 感兴趣的 IO 事件
	protected final int readInterestOp;

	protected AbstractNioChannel(Channel parent,
								 SelectableChannel ch, int readInterestOp) {
	    super(parent);
	    this.ch = ch;
	    this.readInterestOp = readInterestOp;

	    try {
		    // 设置 JDK NIO 原生 ServerSocketChannel 为非阻塞模式
	        ch.configureBlocking(false);
	    } catch (IOException e) {
	        // ... 
	    }
	}
}
```

```java
public abstract class AbstractChannel
	extends DefaultAttributeMap implements Channel {

	private final Channel parent;
	private final ChannelId id;
	private final Unsafe unsafe;
	private final DefaultChannelPipeline pipeline;

	protected AbstractChannel(Channel parent) {
	    this.parent = parent;
	    id = newId();
	    unsafe = newUnsafe();
	    pipeline = newChannelPipeline();
	}
}
```

Netty 中的 Channel 创建是有层次的，这里的 parent 属性用来保存上一级的 Channel。这里的 NioServerSocketChannel 是顶级 Channel，所以它的 parent 为 null。客户端 NioSocketChannel 是由 NioServerSocketChannel 创建的，所以它的 parent 为对应的 NioServerSocketChannel

为 Channel 分配全局唯一的 ChannelId。ChannelId 由机器 Id，进程 Id，序列号，时间戳，随机数构成

```java
protected ChannelId newId() {
	return DefaultChannelId.newInstance();
}
```

NioServerSocketChannel 的底层操作类 Unsafe，封装对 Channel 底层的各种操作。Unsafe 接口定义的操作行为只能由 Netty 框架的 Reactor 线程调用，用户线程禁止调用

```java
interface Unsafe {

	// 分配接收数据用的 Buffer
	RecvByteBufAllocator.Handle recvBufAllocHandle();

	// channel 向 Reactor 注册
	void register(EventLoop eventLoop, ChannelPromise promise);

	// 服务端绑定端口地址
	void bind(SocketAddress localAddress, ChannelPromise promise);

	// 客户端连接服务端
	void connect(SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise);

	// 关闭 channel
	void close(ChannelPromise promise);

	// 读数据
	void beginRead();

	// 写数据
	void write(Object msg, ChannelPromise promise);

	void flush();
}
```

为 NioServerSocketChannel 分配独立的 pipeline 用于 IO 事件编排。pipeline 其实是一个 ChannelHandlerContext 类型的双向链表。头结点 HeadContext，尾结点 TailContext。ChannelHandlerContext 中包装着 ChannelHandler，以及 ChannelHandler 上下文信息，用于事件传播

```java
protected DefaultChannelPipeline newChannelPipeline() {
	return new DefaultChannelPipeline(this);
}
```

```java
public class DefaultChannelPipeline implements ChannelPipeline {
	final AbstractChannelHandlerContext head;
	final AbstractChannelHandlerContext tail;
	
	protected DefaultChannelPipeline(Channel channel) {
	    this.channel = ObjectUtil.checkNotNull(channel, "channel");
	    succeededFuture = new SucceededChannelFuture(channel, null);
	    voidPromise =  new VoidChannelPromise(channel, true);

	    tail = new TailContext(this);
	    head = new HeadContext(this);

	    head.next = tail;
	    tail.prev = head;
	}
}
```

##### 初始化

```java
void init(Channel channel) {
	// 设置 ServerSocketChannelOption
	setChannelOptions(channel,
					  options0().entrySet().toArray(newOptionArray(0)),
					  logger);
	// 设置 attributes
	setAttributes(channel,
				  attrs0().entrySet().toArray(newAttrArray(0)));

	ChannelPipeline p = channel.pipeline();
	final EventLoopGroup currentChildGroup = childGroup;
	final ChannelHandler currentChildHandler = childHandler;
	final Entry<ChannelOption<?>, Object>[] currentChildOptions =
		childOptions.entrySet().toArray(newOptionArray(0));
	final Entry<AttributeKey<?>, Object>[] currentChildAttrs =
		childAttrs.entrySet().toArray(newAttrArray(0));

	p.addLast(new ChannelInitializer<Channel>() {
		@Override
		public void initChannel(final Channel ch) {
			final ChannelPipeline pipeline = ch.pipeline();
			ChannelHandler handler = config.handler();

			if (handler != null) {
				pipeline.addLast(handler);
			}

			ch.eventLoop().execute(new Runnable() {
				@Override
				public void run() {
					pipeline.addLast(
						new ServerBootstrapAcceptor(
							ch,
							currentChildGroup,
							currentChildHandler,
							currentChildOptions, currentChildAttrs));
				}
			});
		}
	});
}
```

Netty 自定义的 SocketChannel 类型均继承 AttributeMap 接口以及 DefaultAttributeMap 类，用于向 Channel 添加用户自定义的一些信息

![[NioServerSocketChannel|600]]

> 为什么不直接将 ChannelHandler 添加到 pipeline 中，而是又使用到了 ChannelInitializer？其实原因有两点：
> 1. 为了保证线程安全地初始化 pipeline，所以初始化的动作需要由 Reactor 线程进行
> 2. 初始化 Channel 中 pipeline 的动作，需要等到 Channel 注册到对应的 Reactor 中才可以进行初始化

#### Register 过程

```java
ChannelFuture regFuture = config().group().register(channel);
```

##### Reactor 启动

从 ServerBootstrap 获取主 Reactor 线程组 NioEventLoopGroup，将 NioServerSocketChannel 注册到 NioEventLoopGroup 中

```java
public abstract class MultithreadEventLoopGroup
	extends MultithreadEventExecutorGroup implements EventLoopGroup {

	@Override
	public EventLoop next() {
	    return (EventLoop) super.next();
	}

	@Override
	public ChannelFuture register(Channel channel) {
	    return next().register(channel);
	}
}
```

```java
public abstract class MultithreadEventExecutorGroup
	extends AbstractEventExecutorGroup {

	@Override
	public EventExecutor next() {
	    return chooser.next();
	}
}
```

```java
public abstract class SingleThreadEventLoop
	extends SingleThreadEventExecutor implements EventLoop {

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

> 注册之后，Channel 生命周期内的所有 IO 事件都由这个 Reactor 负责处理，如 accept、connect、read、write 等 IO 事件
> 
> 一个 channel 只能绑定到一个 Reactor 上，一个 Reactor 负责监听多个 channel

通过 NioServerSocketChannel 中的 Unsafe 类执行底层具体的注册动作

```java
protected abstract class AbstractUnsafe implements Unsafe {
	@Override
	public final void register(EventLoop eventLoop,
							   final ChannelPromise promise) {

		// 检查 NioServerSocketChannel 是否已经完成注册
	    if (isRegistered()) {
	        promise.setFailure(new IllegalStateException("..."));
	        return;
	    }

		// 验证 EventLoop 是否与 Channel 的类型匹配
		// NioEventLoop 对应 NioServerSocketChannel
		if (!isCompatible(eventLoop)) {
	        promise.setFailure(new IllegalStateException("..."));
	        return;
	    }

		// 在 channel 上设置绑定的 Reactor
	    AbstractChannel.this.eventLoop = eventLoop;

		// channel 向 Reactor 注册的动作必须要是在 Reactor 线程中执行
	    if (eventLoop.inEventLoop()) {
	        register0(promise);
	    } else {
	        try {
		        // 将注册动作封装成异步任务
	            eventLoop.execute(new Runnable() {
	                @Override
	                public void run() {
	                    register0(promise);
	                }
	            });
	        } catch (Throwable t) {
	            closeForcibly();
	            closeFuture.setClosed();
	            safeSetFailure(promise, t);
	        }
	    }
	}
}
```

NioEventLoopGroup 的创建过程中需要一个构造器参数 executor，用于启动 Reactor 线程，类型为 ThreadPerTaskExecutor。Reactor 线程的启动时机，就是在向 Reactor 提交第一个异步任务的时候启动的。所以，向主 Reactor 提交用于注册 Channel 的异步任务，会触发主 Reactor 线程的启动

```java
public abstract class SingleThreadEventExecutor
	extends AbstractScheduledEventExecutor
	implements OrderedEventExecutor {

	private static final int ST_NOT_STARTED = 1;
	private static final int ST_STARTED = 2;
	private static final int ST_SHUTTING_DOWN = 3;
	private static final int ST_SHUTDOWN = 4;
	private static final int ST_TERMINATED = 5;

	private volatile int state = ST_NOT_STARTED;

	private static final AtomicIntegerFieldUpdater<SingleThreadEventExecutor>
		STATE_UPDATER = AtomicIntegerFieldUpdater
			.newUpdater(SingleThreadEventExecutor.class, "state");

	@Override
	public void execute(Runnable task) {	  
	    boolean inEventLoop = inEventLoop();

		// 将异步任务添加到 Reactor 中的 taskQueue 中
	    addTask(task);

	    if (!inEventLoop) {
		    // 启动 Reactor 线程
	        startThread();

	        if (isShutdown()) {
		        // ...
	        }
	    }
	}

	private void startThread() {
	    if (state == ST_NOT_STARTED) {
	        if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) { 
	            boolean success = false;

	            try {
	                doStartThread();
	                success = true;
	            } finally {
	                if (!success) {
	                    STATE_UPDATER.compareAndSet(
		                    this, ST_STARTED, ST_NOT_STARTED);
	                }
	            }
	        }
	    }
	}

	// ThreadPerTaskExecutor 用于启动 Reactor 线程
	private final Executor executor;

	private void doStartThread() {
	    executor.execute(new Runnable() {
	        @Override  
	        public void run() {
	            thread = Thread.currentThread();

	            if (interrupted) {
	                thread.interrupt();
	            }

	            boolean success = false;
	            updateLastExecutionTime();

	            try {
	                SingleThreadEventExecutor.this.run();
	                success = true;
	            } catch (Throwable t) {
	                logger.warn("...");
				} finally {
					// ...
	            }
	        }
	    });
	}
}
```

```java
public final class ThreadPerTaskExecutor implements Executor {
    private final ThreadFactory threadFactory;

    public ThreadPerTaskExecutor(ThreadFactory threadFactory) {
        this.threadFactory = threadFactory;
    }

    @Override
    public void execute(Runnable command) {
        threadFactory.newThread(command).start();
    }
}
```

Reactor 线程的核心工作就是轮询所有注册其上的 Channel 中的 IO 就绪事件，处理对应 Channel 上的 IO 事件，执行异步任务。Netty 将这些核心工作封装在 io.netty.channel.nio.NioEventLoop#run 方法中

##### 注册任务执行

Reactor 线程启动完成，后面的工作全部都由这个 Reactor 线程来负责执行。用户启动线程在向 Reactor 提交完 NioServerSocketChannel 的注册任务 register0 后，就逐步退出调用堆栈，回退到最开始的启动入口处 `ChannelFuture f = b.bind(PORT).sync()`

此时 Reactor 中的任务队列中只有一个任务 register0，Reactor 线程启动后，会从任务队列中取出任务执行

```java
protected abstract class AbstractUnsafe implements Unsafe {
	private void register0(ChannelPromise promise) {
	    try {
		    // 查看注册操作是否已经取消，或者对应channel已经关闭
			if (!promise.setUncancellable() || !ensureOpen(promise)) {
	            return;
	        }
	        boolean firstRegistration = neverRegistered;

			// 执行真正的注册操作
	        doRegister();
	        neverRegistered = false;
	        registered = true;

			// 回调 pipeline 中添加的 ChannelInitializer 的 handlerAdded 方法
			pipeline.invokeHandlerAddedIfNeeded();

			// 设置 regFuture 为 success，触发 operationComplete 回调
	        safeSetSuccess(promise);
	        pipeline.fireChannelRegistered();

			// 此时绑定操作作为异步任务在 Reactor 的任务队列中，所以这里返回 false
			if (isActive()) {
	            if (firstRegistration) {
	                pipeline.fireChannelActive();
	            } else if (config().isAutoRead()) {
		            beginRead();
	            }
	        }
	    } catch (Throwable t) {
	        closeForcibly();
	        closeFuture.setClosed();
	        safeSetFailure(promise, t);
	    }
	}
}
```

```java
volatile SelectionKey selectionKey;

protected abstract class AbstractNioUnsafe
	extends AbstractUnsafe implements NioUnsafe {

	protected void doRegister() throws Exception {
	    boolean selected = false;

	    for (;;) {
	        try {
	            selectionKey = javaChannel().register(
		            eventLoop().unwrappedSelector(), 0, this
		        );
	            return;
	        } catch (CancelledKeyException e) {
		        // ...
	        }
	    }
	}
}
```

> SelectionKey 可以理解为 Channel 在 Selector 上的特殊表示形式，SelectionKey 中封装了 Channel 感兴趣的 IO 事件集合 interestOps，以及 IO 就绪的事件集合 readyOps，同时也封装了对应的 JDK NIO Channel 以及注册的 Selector

注册的时候，还将 Netty 自定义的 NioServerSocketChannel 附着在 SelectionKey 的 attechment 属性上，完成 Netty 自定义 Channel 与 JDK NIO Channel 的关系绑定。这样在每次对 Selector 进行 IO 就绪事件轮询时，Netty 都可以从 JDK NIO Selector 返回的 SelectionKey 中获取到自定义的 Channel 对象

##### HandlerAdded 事件回调

当 NioServerSocketChannel 注册到 Reactor 上的 Selector 后，pipeline 中只有在初始化 NioServerSocketChannel 时添加的 ChannelInitializer

```java
public class DefaultChannelPipeline implements ChannelPipeline {

	public final ChannelPipeline addLast(EventExecutorGroup group,
										 String name,
										 ChannelHandler handler) {
	    final AbstractChannelHandlerContext newCtx;

	    synchronized (this) {
	        newCtx = newContext(group, filterName(name, handler), handler);
	        addLast0(newCtx);

			if (!registered) {
	            newCtx.setAddPending();

				// 还没有注册完成，把 handler 回调任务加入到 pending 队列
	            callHandlerCallbackLater(newCtx, true);
	            return this;
	        }

	        EventExecutor executor = newCtx.executor();

	        if (!executor.inEventLoop()) {
		        // 提交任务到 executor 去执行
	            callHandlerAddedInEventLoop(newCtx, executor);
	            return this;
	        }
	    }

		// 如果注册了，调用 handler add 回调
	    callHandlerAdded0(newCtx);
	    return this;
	}

	// 将会调用 handler 的 handlerAdded 方法
	private void callHandlerAdded0(final AbstractChannelHandlerContext ctx) {
	    try {
	        ctx.callHandlerAdded();
	    } catch (Throwable t) {
		    // ...
	    }
	}

	private void callHandlerCallbackLater(AbstractChannelHandlerContext ctx,
											boolean added) {
	    assert !registered;

		// PendingHandlerAddedTask 中调用 callHandlerAdded0
	    PendingHandlerCallback task = added ?
			new PendingHandlerAddedTask(ctx) :
			new PendingHandlerRemovedTask(ctx);
	    PendingHandlerCallback pending = pendingHandlerCallbackHead;

	    if (pending == null) {
	        pendingHandlerCallbackHead = task;
	    } else {
	        while (pending.next != null) {
	            pending = pending.next;
	        }
	        pending.next = task;
	    }
	}

	final void invokeHandlerAddedIfNeeded() {
	    assert channel.eventLoop().inEventLoop();

	    if (firstRegistration) {
	        firstRegistration = false;
	        callHandlerAddedForAllHandlers();
	    }
	}

	private void callHandlerAddedForAllHandlers() {
	    final PendingHandlerCallback pendingHandlerCallbackHead;

	    synchronized (this) {
	        assert !registered;

	        registered = true;
	        pendingHandlerCallbackHead = this.pendingHandlerCallbackHead;
	        this.pendingHandlerCallbackHead = null;
	    }

		PendingHandlerCallback task = pendingHandlerCallbackHead;

	    while (task != null) {
	        task.execute();
	        task = task.next;
	    }
	}
}
```

在 ChannelInitializer 的自定义 initChannel 方法中，向主 reactor 提交添加 ServerBootstrapAcceptor 的任务
```java
p.addLast(new ChannelInitializer<Channel>() {
    @Override
    public void initChannel(final Channel ch) {
        final ChannelPipeline pipeline = ch.pipeline();
        ChannelHandler handler = config.handler();
        if (handler != null) {
            pipeline.addLast(handler);
        }

        ch.eventLoop().execute(new Runnable() {
            public void run() {
                pipeline.addLast(new ServerBootstrapAcceptor(
                        ch,
						currentChildGroup, currentChildHandler,
						currentChildOptions, currentChildAttrs)
				);
            }
        });
    }
});
```

##### ChannelFutureListener 回调

```java
public abstract
	class AbstractBootstrap<B extends AbstractBootstrap<B, C>, C extends Channel>
	implements Cloneable {

	private ChannelFuture doBind(final SocketAddress localAddress) {
	    final ChannelFuture regFuture = initAndRegister();

	    if (regFuture.isDone()) {
	        ChannelPromise promise = channel.newPromise();
	        doBind0(regFuture, channel, localAddress, promise);
	        return promise;
	    } else {
	        final PendingRegistrationPromise promise =
				new PendingRegistrationPromise(channel);

	        regFuture.addListener(new ChannelFutureListener() {
	            @Override
	            public void operationComplete(ChannelFuture future) {
	                Throwable cause = future.cause();
	                if (cause != null) {
		                promise.setFailure(cause);
	                } else {
	                    doBind0(regFuture, channel, localAddress, promise);
	                }
	            }
	        });
	        return promise;
	    }
	}

	private static void doBind0(
        final ChannelFuture regFuture, final Channel channel,
        final SocketAddress localAddress, final ChannelPromise promise) {

		channel.eventLoop().execute(new Runnable() {
	        @Override
	        public void run() {
	            if (regFuture.isSuccess()) {
	                channel.bind(
		                localAddress,
		                promise
		            ).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
	            } else {
	                promise.setFailure(regFuture.cause());
	            }
	        }
	    });
	}
}
```

### bind

```java
public abstract class AbstractChannel
	extends DefaultAttributeMap implements Channel {

	public ChannelFuture bind(SocketAddress localAddress,
							  ChannelPromise promise) {
	    return pipeline.bind(localAddress, promise);
	}
}
```

```java
public class DefaultChannelPipeline implements ChannelPipeline {
	public final ChannelFuture bind(SocketAddress localAddress,
									ChannelPromise promise) {
	    return tail.bind(localAddress, promise);
	}
}
```

调用 pipeline.bind 方法，在 pipeline 中传播 bind 事件，触发回调 pipeline 中所有 ChannelHandler 的 bind 方法

事件在 pipeline 中的传播具有方向性：
- inbound 事件从 HeadContext 开始逐个向后传播直到 TailContext
- outbound 事件则是反向传播，从 TailContext 开始反向向前传播直到 HeadContext

此时，channel 的 pipeline 的结构如下（添加 acceptor 任务排在在 bind 任务的前面）：

![[pipeline|640]]

> inbound 事件只能被 pipeline 中的 ChannelInboundHandler 响应处理 outbound 事件只能被 pipeline 中的 ChannelOutboundHandler 响应处理

这里的 bind 事件在 Netty 中被定义为 outbound 事件，在 pipeline 中是反向传播。先从 TailContext 开始反向传播直到 HeadContext。但是，bind 的核心逻辑的实现在 HeadContext 中

```java
abstract class AbstractChannelHandlerContext
	implements ChannelHandlerContext, ResourceLeakHint {

	public ChannelFuture bind(SocketAddress localAddress) {
	    return bind(localAddress, newPromise());
	}

	public ChannelFuture bind(final SocketAddress localAddress,
							  final ChannelPromise promise) {
	    final AbstractChannelHandlerContext next =
			findContextOutbound(MASK_BIND);
	    EventExecutor executor = next.executor();

	    if (executor.inEventLoop()) {
	        next.invokeBind(localAddress, promise);
	    } else {
	        safeExecute(executor, new Runnable() {
	            @Override
	            public void run() {
	                next.invokeBind(localAddress, promise);
	            }
	        }, promise, null);
	    }
	    return promise;
	}

	private void invokeBind(SocketAddress localAddress, ChannelPromise promise) {
	    if (invokeHandler()) {
	        try {
	            ((ChannelOutboundHandler) handler())
		            .bind(this, localAddress, promise);
	        } catch (Throwable t) {
	            notifyOutboundHandlerException(t, promise);
	        }
	    } else {
	        bind(localAddress, promise);
	    }
	}
}
```

```java
final class HeadContext extends AbstractChannelHandlerContext  
        implements ChannelOutboundHandler, ChannelInboundHandler {

	private final Unsafe unsafe;

	HeadContext(DefaultChannelPipeline pipeline) {
	    super(pipeline, null, HEAD_NAME, HeadContext.class);
	    unsafe = pipeline.channel().unsafe();
	    setAddComplete();
	}

	public void bind(ChannelHandlerContext ctx,
					SocketAddress localAddress,
					ChannelPromise promise) {
	    unsafe.bind(localAddress, promise);
	}
}
```

```java
protected abstract class AbstractUnsafe implements Unsafe {
	public final void bind(final SocketAddress localAddress,
						   final ChannelPromise promise) {

	    boolean wasActive = isActive();

	    try {
	        doBind(localAddress);
	    } catch (Throwable t) {
	        safeSetFailure(promise, t);
	        closeIfClosed();
	        return;
	    }

	    if (!wasActive && isActive()) {
		    // 提交触发 channelActive 事件传播的任务
	        invokeLater(new Runnable() {
	            public void run() {
	                pipeline.fireChannelActive();
	            }
	        });
	    }

		// 回调注册在 promise 上的 ChannelFutureListener
	    safeSetSuccess(promise);
	}
}
```

```java
public class NioServerSocketChannel
	extends AbstractNioMessageChannel
	implements io.netty.channel.socket.ServerSocketChannel {

	protected void doBind(SocketAddress localAddress) throws Exception {
	    if (PlatformDependent.javaVersion() >= 7) {
	        javaChannel().bind(localAddress, config.getBacklog());
	    } else {
	        javaChannel().socket().bind(localAddress, config.getBacklog());
	    }
	}
}
```

#### channelActive 事件处理

channelActive 事件在 Netty 中定义为 inbound 事件，所以它在 pipeline 中的传播为正向传播，从 HeadContext 一直到 TailContext 为止

在 channelActive 事件回调中需要触发向 Selector 指定需要监听的 OP_ACCEPT 事件，这块的逻辑主要在 HeadContext 中实现

```java
final class HeadContext extends AbstractChannelHandlerContext
	implements ChannelOutboundHandler, ChannelInboundHandler {

	private final Unsafe unsafe;

	HeadContext(DefaultChannelPipeline pipeline) {
	    super(pipeline, null, HEAD_NAME, HeadContext.class);
	    unsafe = pipeline.channel().unsafe();
	    setAddComplete();
	}

	public void channelActive(ChannelHandlerContext ctx) {
		// 继续向后传播 channelActive 事件
	    ctx.fireChannelActive();
	    readIfIsAutoRead();
	}

	private void readIfIsAutoRead() {
	    if (channel.config().isAutoRead()) {
		    // 如果是 autoRead，则触发 read 事件传播
		    // 在 read 回调函数中触发 OP_ACCEPT 注册
	        channel.read();
	    }
	}

	public void read(ChannelHandlerContext ctx) {
	    unsafe.beginRead();
	}
}
```

```java
public abstract class AbstractChannel extends DefaultAttributeMap
	implements Channel {

	private final DefaultChannelPipeline pipeline;

	public Channel read() {
		// 触发 read 事件，一直传播到 HeadContext
	    pipeline.read();
	    return this;
	}
}
```

```java
protected abstract class AbstractUnsafe implements Unsafe {
	public final void beginRead() {
		doBeginRead();
	}
}
```

```java
public abstract class AbstractNioChannel extends AbstractChannel {
	protected void doBeginRead() throws Exception {
	    final SelectionKey selectionKey = this.selectionKey;
	    final int interestOps = selectionKey.interestOps();

		// NioServerSocketChannel 初始化时设置 readInterestOp 为 OP_ACCEPT
	    if ((interestOps & readInterestOp) == 0) {
	        selectionKey.interestOps(interestOps | readInterestOp);
	    }
	}
}
```

![[netty.png]]