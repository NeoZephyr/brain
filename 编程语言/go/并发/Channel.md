```go
var ch chan int
ch = make(chan int, 5)
```

如果 channel 类型变量在声明时没有被赋予初值，那么它的默认值为 nil。channel 类型变量赋初值的唯一方法就是使用 make 函数

## 发送与接收

Goroutine 对无缓冲 channel 的接收和发送操作是同步的。对同一个无缓冲 channel，只有对它进行接收操作的 Goroutine 和对它进行发送操作的 Goroutine 都存在的情况下，通信才能得以进行，否则单方面的操作会让对应的 Goroutine 陷入挂起状态

对带缓冲 channel 的发送操作在缓冲区未满、接收操作在缓冲区非空的情况下是异步的

## 关闭

channel 的一个使用惯例，那就是发送端负责关闭 channel。一旦向一个已经关闭的 channel 执行发送操作，这个操作就会引发 panic

## select

通过 select，可以同时在多个 channel 上进行发送 / 接收操作。当 select 语句中没有 default 分支，而且所有 case 中的 channel 操作都阻塞了的时候，整个 select 语句都将被阻塞，直到某一个 case 上的 channel 变成可发送，或者某个 case 上的 channel 变成可接收，select 语句才可以继续进行下去

### 无缓冲 channel

#### 信号传递

1. 1 对 1 通知信号

```go
type signal struct{}

func worker() {
	println("worker is working...")
	time.Sleep(1 * time.Second)
}

func spawn(f func()) <-chan signal {
	c := make(chan signal)
	go func() {
		println("worker start to work...")
		f()
		c <- signal(struct{}{})
	}()
	return c
}

println("start a worker...")
c := spawn(worker)
<-c
println("worker work done...")
```

2. 1 对 n 通知信号，关闭一个无缓冲 channel 会让所有阻塞在这个 channel 上的接收操作返回

```go
type signal struct{}

func worker(i int) {
	fmt.Printf("worker %d: is working...\n", i)
	time.Sleep(1 * time.Second)
}

func spawn(f func(i int), num int, groupSignal <-chan signal) <-chan signal {
	c := make(chan signal)
	var wg sync.WaitGroup

	for i := 0; i < num; i++ {
		wg.Add(1)
		go func(i int) {
			<-groupSignal
			fmt.Printf("worker %d: start to work...\n", i)
			f(i)
			wg.Done()
		}(i)
	}

	go func() {
		wg.Wait()
		c <- signal(struct{}{})
	}()

	return c
}

groupSignal := make(chan signal)
c := spawn(worker, 5, groupSignal)
close(groupSignal)
<-c
```

#### 替代锁机制

无缓冲 channel 具有同步特性，在某些场合可以替代锁

### 带缓冲 channel

#### 消息队列

#### 计数信号量

带缓冲 channel 中的当前数据个数代表的是，当前同时处于活动状态的 Goroutine 的数量，而带缓冲 channel 的容量，就代表了允许同时处于活动状态的 Goroutine 的最大数量。向带缓冲 channel 的一个发送操作表示获取一个信号量，而从 channel 的一个接收操作则表示释放一个信号量

```go
active := make(chan struct{}, 3)
jobs := make(chan int, 10)

go func() {
    for i := 0; i < 8; i++ {
        jobs <- (i + 1)
    }
    close(jobs)
}()

var wg sync.WaitGroup

for j := range jobs {
    wg.Add(1)
    go func(j int) {
        active <- struct{}{}
        fmt.Printf("handle job: %d\n", j)
        time.Sleep(2 * time.Second)
        <-active
        wg.Done()
    }(j)
}

wg.Wait()
```

### 超时机制

Go 语言标准库提供的 timer 由 Go 运行时自行维护的，而不是操作系统级的定时器资源，它的使用代价要比操作系统级的低许多。但即便如此，也要及时调用 timer 的 Stop 方法回收 Timer 资源

```go
select {
    case <-c:
    fmt.Println("...")
    case <-time.After(5 * time.Second):
    fmt.Println("after 5 second")
}
```

### 心跳机制

time.NewTicker 创建了一个 Ticker 类型实例，这个实例包含一个 channel 类型的字段 C，这个字段会按一定时间间隔持续产生事件，就像心跳一样。和 timer 一样，在使用完 ticker 之后，也要调用它的 Stop 方法，避免心跳事件在 ticker 的 channel 中持续产生

```go
var c chan int

heartbeat := time.NewTicker(1 * time.Second)
defer heartbeat.Stop()

for {
    select {
        case <-c:
        fmt.Println("...")
        case <-heartbeat.C:
        fmt.Println("beat...")
    }
}
```

## channel 状态

当 ch 为无缓冲 channel 时，len(ch) 总是返回 0
当 ch 为带缓冲 channel 时，len(ch) 返回当前 channel 中尚未被读取的元素个数

```go
func tryRecv(c <-chan int) (int, bool) {
	select {
	case i := <-c:
		return i, true
	default:
		return 0, false
	}
}

func trySend(c chan<- int, i int) bool {
	select {
	case c <- i:
		return true
	default:
		return false
	}
}
```

如果一个 channel 类型变量的值为 nil，就称为 nil channel。对 nil channel 的读写都会发生阻塞

```go
ch1, ch2 := make(chan int), make(chan int)

go func() {
    time.Sleep(100 * time.Microsecond)
    ch1 <- 100
    close(ch1)
}()

go func() {
    time.Sleep(300 * time.Microsecond)
    ch2 <- 300
    close(ch2)
}()

for {
    select {
        case x, ok := <-ch1:
        if !ok {
            // 从一个已关闭的 channel 接收数据将永远不会被阻塞，而且获取的是对应类型的零值
            // 因此，需要置空
            ch1 = nil
        } else {
            fmt.Println(x)
        }
        case x, ok := <-ch2:
        if !ok {
            ch2 = nil
        } else {
            fmt.Println(x)
        }
    }

    if ch1 == nil && ch2 == nil {
        break
    }
}
```