基于 TCP 的自定义应用层协议通常有两种定义模式：

1.  二进制模式：采用长度字段标识独立数据包的边界
2.  文本模式：采用特定分隔符标识流中的数据包的边界

![[Pasted image 20230215235616.png]]

# Socket 编程模型

-   阻塞 I/O

![[Pasted image 20230215235628.png]]

当用户空间应用线程，向操作系统内核发起 I/O 请求后，内核会尝试执行这个 I/O 操作，并等所有数据就绪后，将数据从内核空间拷贝到用户空间，最后系统调用从内核空间返回。而在这 个期间内，用户空间应用线程将阻塞在这个 I/O 系统调用上

-   非阻塞 I/O

![[Pasted image 20230215235643.png]]

当用户空间线程向操作系统内核发起 I/O 请求后，内核会执行这个 I/O 操作，如果这个时候数据尚未就绪，就会立即将未就绪的状态以错误码形式返回

在非阻塞模型下，请求发起者通常会通过轮询的方式，去一次次发起 I/O 请求，直到读到所需的数据为止。不过，这样的轮询是对 CPU 计算资源的极大浪费，因此，非阻塞 I/O 模型单独应用于实际生产的比例并不高

-   I/O 多路复用

![[Pasted image 20230215235659.png]]

应用线程首先将需要进行 I/O 操作的 Socket，添加到多路复用函数中，然后阻塞等待 select 系统调用返回。当内核发现有数据到达时，对应的 Socket 具备了通信条件，这时 select 函数返回。然后用户线程会针对这个 Socket 再次发起网络 I/O 请求。由于数据已就绪，这次网络 I/O 操作将得到预期的操作结果

I/O 多路复用模型中，一个应用线程可以同时处理多个 Socket。同时，I/O 多路复用模型由内核实现可读 / 可写事件的通知，避免了非阻塞模型中轮询，带来的 CPU 计算资源浪费的问题

## Go 语言 socket 模型

Go 选择了为开发人员提供阻塞 I/O 模型，只需在 Goroutine 中以最简单、最易用的阻塞 I/O 模型的方式，进行 Socket 操作就可以了。Go 使用了开销更小的 Goroutine 作为基本执行单元，这让每个 Goroutine 处理一个 TCP 连接成为可能，并且在高并发下依旧表现出色

不过，网络 I/O 操作都是系统调用，Goroutine 执行 I/O 操作的话，一旦阻塞在系统调用上，就会导致 M 也被阻塞，为了解决这个问题，Go 设计者将这个复杂性隐藏在 Go 运行时中，在运行时中实现了网络轮询器 netpoller，只阻塞执行网络 I/O 操作的 Goroutine，但不阻塞执行 Goroutine 的线程，也就是 M

对于 Go 程序的用户层来说，看到的 goroutine 采用了阻塞 I/O 模型进行网络 I/O 操作，Socket 都是阻塞的。但实际上，是通过 Go 运行时中的 netpoller I/O 多路复用机制，模拟出来的，对应的、真实的底层操作系统 Socket，实际上是非阻塞的。只是运行时拦截了针对底层 Socket 的系统调用返回的错误码，并通过 netpoller 和 Goroutine 调度，让 Goroutine 阻塞在用户层所看到的 Socket 描述符上

比如：当用户层针对某个 Socket 描述符发起 read 操作时，如果这个 Socket 对应的连接上还没有数据，运行时就会将这个 Socket 描述符加入到 netpoller 中监听，同时发起此次读操作的 Goroutine 会被挂起。直到 Go 运行时收到这个 Socket 数据可读的通知，Go 运行时才会重新唤醒等待在这个 Socket 上准备读数据的那个 Goroutine

Go 语言在网络轮询器中采用了 I/O 多路复用的模型。考虑到最常见的多路复用系统调用 select 有比较多的限制，比如：监听 Socket 的数量有上限（1024）、时间复杂度高等，Go 运行时选择了在不同操作系统上，使用操作系统各自实现的高性能多路复用函数，这样可以最大程度提高 netpoller 的调度和执行性能

### 监听与接收连接

```go
func handleConn(c net.Conn) {
    defer c.Close()
    
    for {}
}

func main() {
    l, err := net.Listen("tcp", ":8888")
    
    if err != nil {
        fmt.Println("listen error:", err)
        return
    }
    
    for {
        c, err := l.Accept()
        
        if err != nil {
            fmt.Println("accept error:", err)
            break
        }
        
        go handleConn(c)
    }
}
```

### 建立 TCP 连接

```go
conn, err := net.Dial("tcp", "localhost:8888")
conn, err := net.DialTimeout("tcp", "localhost:8888", 2 * time.Second)
```

Dial 函数向服务端发起 TCP 连接，这个函数会一直阻塞，直到连接成功或失败后，才会返回。DialTimeout 带有超时机制，如果连接耗时大于超时时间，这个函数会返回超时错误。 对于客户端来说，连接的建立还可能会遇到几种特殊情形：

1.  网络不可达或对方服务未启动
2.  服务的 listen backlog 队列满，这会导致客户端的 Dial 调用阻塞，直到服务端进行一次 accept，从 backlog 队列中腾出一个槽位，客户端的 Dial 才会返回成功。在 Ubuntu Linux 下，backlog 队列的长度值与系统中 net.ipv4.tcp_max_syn_backlog 的设置有关。在极端情况下，如果服务端一直不执行 accept 操作，客户端最终还是会收到超时错误
3.  若网络延迟较大，Dial 将阻塞并超时

## 全双工通信

一旦客户端调用 Dial 成功，就在客户端与服务端之间建立起了一条全双工的通信通道

![[Pasted image 20230215235715.png]]

## 读操作

Dial 连接成功后，会返回一个 net.Conn 接口类型的变量值

```go
type TCPConn struct {
    conn
}
```

如果客户端未发送数据，服务端会阻塞在 Socket 的读操作上，执行该这个操作的 Goroutine 也会被挂起。Go 运行时会监视这个 Socket，直到它有数据读事件，才会重新调度这个 Socket 对应的 Goroutine 完成读操作

如果有部分数据就绪，且数据数量小于一次读操作期望读出的数据长度，那么读操作将会成功读出这部分数据，并返回，而不是等待期望长度数据全部读取后，再返回。

如果连接上有数据，且数据长度大于等于一次读操作期望读出的数据长度，那么将会成功读出这部分数据，并返回

有些场合，对 socket 的读操作的阻塞时间有限制，可以通过 net.Conn 提供的 SetReadDeadline 方法，设置读操作的超时时间，当超时后仍然没有数据可读的情况下，读操作会解除阻塞并返回超时错误。如果要取消超时设置，可以使用 SetReadDeadline(time.Time{}) 实现

```go
func handleConn(c net.Conn) {
    defer c.Close()

    for {
        var buf = make([]byte, 128)
        c.SetReadDeadline(time.Now().Add(time.Second))
        n, err := c.Read(buf)

        if err != nil {
            log.Printf("conn read %d bytes, error: %s", n, err)
            
            if nerr, ok := err.(net.Error); ok && nerr.Timeout() {
                continue
            }
            return
        }
        log.Printf("read %d bytes, content is %s\n", n, string(buf[:n]))
    }
}
```

### 写操作

当发送方将对方的接收缓冲区，以及自身的发送缓冲区都写满后，再调用 Write 方法就会出现阻塞的情况

```go
func main() {
    conn, err := net.Dial("tcp", ":8888")
    
    if err != nil  {
        log.Println("dial error:", err)
        return
    }
    
    defer conn.Close()
    
    log.Println("dial ok")
    
    data := make([]byte, 65535)
    var total int
    
    for {
        n, err := conn.Write(data)
        
        if err != nil {
            total += n
            log.Printf("write %d bytes, error: %s\n", n, err)
            break
        }
        total += n
        log.Printf("write %d bytes, % bytes in total\n", n, total)
    }
    
    log.Printf("write %d bytes in total\n", total)
}
```

服务端在前 10 秒中并不读取数据，因此当客户端写到一定量后就会发生阻塞

```go
func handleConn(c net.Conn) {
    defer c.Close()
    time.Sleep(time.Second * 10)
    
    for {
        time.Sleep(5 * time.Second)
        var buf = make([]byte, 60000)
        log.Println("start to read from conn")
        n, err := c.Read(buf)
        
        if err != nil {
            log.Printf("conn read %d bytes, error: %s", n, err)
            
            if nerr, ok := err.(net.Error); ok && nerr.Timeout() {
                continue
            }
        }
        
        log.Printf("read %d bytes, content is %s\n", n, string(buf[:n]))
    }
}
```

Write 操作存在写入部分数据的情况：杀掉服务端之后，客户端又写入部分数据，然后才返回 broken pipe 错误。由于这部分数据并未真正被服务端接收到，程序需要考虑妥善处理这些数据，以防数据丢失

写入超时：可以调用 SetWriteDeadline 方法，给 Write 操作增加一个期限

```go
conn.SetWriteDeadline(time.Now().Add(time.Microsecond * 10))
```

在 Write 方法写入超时时，依旧存在数据部分写入的情况。另外，和 SetReadDeadline 一样，只要通过 SetWriteDeadline 设置了写超时，无论后续 Write 方法是否成功，如果不重新设置写超时或取消写超时，后续对 Socket 的写操作都将以超时失败告终

### 并发读写

对于 Read 操作而言，由于 TCP 是面向字节流，conn.Read 无法正确区分数据的业务边界，因此多个 Goroutine 对同一个 conn 进行读操作的意义不大，Goroutine 读到不完整的业务包，反倒增加了业务处理的难度

对与 Write 操作而言，存在多个 Goroutine 并发写的情况

net.conn只是 *netFD 的外层包裹结构，最终读写都会落在其中的 fd 字段上。netFD 在不同平台上有着不同的实现

```go
type conn struct {
    fd *netFD
}
```

### 关闭

当客户端需要断开与服务端的连接时，客户端会调用 net.Conn 的 Close 方法关闭与服务端通信的 Socket。如果客户端主动关闭了 Socket

有数据关闭：在客户端关闭连接时，Socket 中还有服务端尚未读取的数据。在这种情况下，服务端的成功将剩余数据读取出来，最后一次读操作将得到 io.EOF 错误码，表示客户端已经断开了连接

无数据关闭：服务端调用的 Read 方法将直接返回 io.EOF

因为 Socket 是全双工的，客户端关闭 Socket 后，如果服务端 Socket 尚未关闭，这个时候服务端向 Socket 的写入操作依然可能会成功，因为数据会成功写入己方的内核 socket 缓冲区中