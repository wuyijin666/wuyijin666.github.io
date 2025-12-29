---
title: IO多路复用之epoll的原理与使用 + Reactor模型解析
date: 2025-11-19 23:35:59
categories:
  - 读书笔记
  - Linux高性能服务器编程
tags:
  - Linux
  - 服务器编程
  - 网络编程
---
## IO多路复用之epoll的原理与使用 + Reactor模型解析
### IO复用：在复用什么？
IO复用使得程序可以同时监听多个文件描述符，**复用的是线程/进程**，而非连接本身。让一个线程（进）同时监听成百上千个网络连接（文件描述符），只在某个连接真正可读/可写时，才处理该连接。

### IO复用使用场景
1. 客户端程序同时处理多个socket(非阻塞connect技术)
2. 客户端程序同时处理用户输入与网络连接（聊天室）
3. TCP服务器程序同时处理监听socket和连接socket
4. 服务器同时处理TCP UDP请求（回射服务器）
5. 服务器同时监听多个端口，或者处理多种服务
   
需要注意的是，IO复用虽然能同时监听多个网络连接，但是**其本身是阻塞的**，并且当多个文件描述符就绪时，如果不采用额外的措施，程序就只能依次按顺序处理文件描述符。这使得服务器程序看上去像是串行在工作。若要实现并发，要采用多进程/多线程等编程手段。

(之后，我会去聊聊 多线程/多进程的协作方式  参考blog：多线程/多进程的协作方式 )
### epoll原理：
1. 内核事件表：
   首先，epoll使用一组函数来完成任务，而不是单个函数。其次，epoll把用户关心的文件描述符事实上的事件都放入内核中的一个事件表里，从而无需像select poll那样，每次调用都需要重复传入文件描述符集或事件集。但epoll需要一个额外的文件描述符，来**唯一标识内核中的这个事件表**。这个文件描述符使用**epoll_create()** 函数创建:
   ```c
   #include <sys/epoll.h>
   int epoll_create(int size);
   ```
   size参数：告诉内核，事件表需要多大。该函数返回的文件描述符将用作其他所有epoll系统调用的第一个参数，以指定要访问的内核事件表。
   epoll_ctl()用来操作epoll的内核事件表。
   ```c
   #include <sys/epoll.h>
   int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
   ```
   fd参数：要操作的文件描述符
   op参数：操作类型
   - 操作类型：
   EPOLL_CTL_ADD：添加文件描述符到内核事件表
   EPOLL_CTL_MOD：修改文件描述符在内核事件表中的属性
   EPOLL_CTL_DEL：从内核事件表中删除文件描述符
   
   event参数：指定事件
   ```c
   struct epoll_event {
     _uint32 events;  /* epoll 事件 */
     epoll_data data; /* 用户数据 */
   }
   ```
   events参数：描述事件类型， epoll有两个额外的事件类型-EPOLLET和EPOLLONESHOT，它们对于epoll的高效运作非常关键。 后面讲一下EPOLLONESHOT

2. epoll_wait()
   epoll系列系统调用的主要接口。**它在一段超时时间内等待一组文件描述符上的事件。**
   原型：
   ```c
   #include <sys/epoll.h>
   int epoll_wait(int epfd, struct epoll_event *events, int maxevents,
   int timeout)
   ```
   该函数成功时返回已就绪的文件描述符个数，失败时返回-1，并设置errno
   重点参数解析：
   - maxevents参数：指定最多监听的事件数
   - epoll_wait()如果检测到事件，就将所有就绪的事件从内核事件表（由epfd参数指定）中复制到events参数指定的数组中。
   - 重点：events参数：这个数组只用于输出epoll_wait()检测到的就绪事件，不想poll select的数组参数那样，既用于传入用户注册的事件，也用于输出内核检测到的就绪事件。(简单说，就是epoll的events数组里放的正好就是就绪了的文件描述符，都是能用的) 这就**极大提高了应用程序索引就绪文件描述符的效率**。
  来看下 poll 和 epoll在使用上的区别 
  ```c
  /* 如何索引poll返回的就绪文件描述符 */
  int ret = poll(fds, MAX_EVENT_NUMBER, -1);
  /* 必须遍历所有的已注册的文件描述符并找到其中的就绪者 */
  for i = 0; i < MAX_EVENT_NUMBER; i++ {
    if (fds[i].revents & POLLIN){ // 判断第i个可读的就绪事件
        int socketfd = fds[i].fd;
        // 处理socketfd
        ...
    }
  } 

  /* 如何索引epoll返回的就绪文件描述符 */
  int ret = epoll_wait(epfd, events, MAX_EVENT_NUMBER, -1);
  /* 仅遍历就绪ret个文件描述符 */
  for i = 0; i < ret; i++ {
        int socketfd = events[i].data.fd;
        // 处理socketfd
        ...
  } 
 ```
3.  LT vs ET模式
   LT是默认的工作模式，**当往epoll的内核事件表中注册一个文件描述符上的EPOLLET事件时**，表示epoll将以ET模式操作文件描述符。**(ET模式下事件被触发的次数少于LT)**
   - LT模式：只要还有数据可读/写，就一直提醒你处理
   - ET模式：只在数据刚到/状态变化时提醒一次，就提醒你处理一次，没处理完也不会再喊你
   **！注意：Q: 为何ET用非阻塞socket**
   A：如果文件描述符是阻塞的，那么读或写操作将会因为没有后续事件而一直处于阻塞状态（饥渴状态）
- ET（边缘触发）模式下，如果文件描述符仍保持阻塞属性，会出现两个致命问题：
	1.	数据可能“卡死”在内核缓冲区
ET 只在状态变化时通知一次。假设内核缓冲区里有 2048 字节，应用只读了 1024 字节就返回，剩余数据会一直留在缓冲区。由于 ET 不会再次触发可读事件，应用就再也收不到后续通知，数据就此“饿死”。
	2.	单次读写操作可能永久阻塞
若描述符是阻塞的，当应用试图一次性读完所有数据时，最后一次 read/write 会因缓冲区暂时空/满而挂起，线程被牢牢锁死，无法继续处理其他就绪 fd，导致整个事件循环停滞。
4. EPOLLONESHOT事件
   假设使用ET模式，还是有可能一个socket上的事件被多次触发，这在并发程序中会造成问题。比如说，一个线程/进程先处理了一个socket，后续该socket又有新的数据可是读写（EPOLLIN再次被触发），此时另外的线程会被唤醒来读取这些新数据。从而变成多个线程同时操作同一个socket的局面，这显然不是我们愿意看到的。
   EPOLLONESHOT事件**用于解决这个问题**， **一个socket在任一时刻都只能被一个线程处理。**
   简单概括下工作原理，我们注册EPOLLONSHOT事件的文件描述符，操作系统最多触发其上注册的一个可读、可写或者异常事件，且触发一次。除非用epoll_ctl()重置该文件描述符上注册的EPOLLONESHOT事件。但同时，如果该socket之后还有新的任务需要被重新处理，你应该在之前的线程中立即重置该socket上的EPOLLONESHOT事件。

   - 总结：一个socket可以被多个线程不同时处理， 但我们保证同一时间，一个socket只能被一个线程处理。从而保证连接的完整性，避免很多的竞态条件。

go这里，更多采用net + Goroutine来实现类Reactor模型的Echo Server，底层自动使用epoll(Linux)
简单看个epoll echo server 的实现
```go

var wg sync.WaitGroup

func handleConn(c net.Conn) {
    defer c.Close()
    defer wg.Done()

    reader := bufio.NewReader(c)
    for {
        message, err := reader.ReadString('\n')
        if err != nil {
            //客户端断开连接
            log.Printf("client %s disconnected", c.RemoteAddr())
            return
        }
        log.Printf("Received from %s: %q", c.RemoteAddr(), message)
        // Echo back to client
        err := c.Write([]byte(message))
        if err != nil {
            return
        }
    }
}

func main() { 
    listen, err := net.Listen("tcp", ":8080")
    if err != nil {
        log.Fatal("Listen failed:", err)
    }
    defer listen.Close()

    for{
        conn, err := listen.Accept()
        if err != nil {
           log.Fatal("Accept failed:", err)
           continue
        }
        log.Printf("New client connected: %s", conn.RemoteAddr())

        wg.Add(1)
        go handleConn(conn)

    }
}
```


TODO 有空会去分析下go runtime库下的netpoll，对linux epoll的封装

### select poll epoll 对比
- 找就绪文件描述符方式不同：select poll 轮询， epoll 回调
- LT ET选择模式不同：select poll LT， epoll 可选择ET模式，并且还支持EPOLLONESHOT事件，该事件能进一步减少可读、可写或异常等事件被触发的次数
- 但是当活动连接比较多的时候，epoll效率不一定比select poll高，原因在于此时回调函数被触发的过于频繁（回调函数将该就绪的文件描述符上对应的事件插入到内核就绪事件队列）。
- 适用场景：epoll适用于大并发场景，连接数比较多，但活动连接数比较少

![](/images/23.jpg)


## 两种高效的事件处理模式
### Reactor模型思想：
总结：事件驱动 + 非阻塞IO + **少量连接支持大量并发**
IO分为两个阶段：
- 监听：用IO多路复用（epoll / hqueue等）把所有待处理的连接/文件描述符统一监控，一旦某个描述符就绪（可读/可写/异常），立即触发事件。
- 分发：事件循环（Reactor线程）拿到就绪事件后，把它分发到预先注册的事件处理器（Handler），由Handler完成实际的数据处理。
- ! 简单理解 : Reactor 负责 “发现事件并派活” ->  Handler 负责 “干活”，处理完立即返回，避免阻塞事件循环 -> 如果业务逻辑耗时，可把任务扔进线程池异步处理，让主线程继续监听新事件。

### Proactor 

### 区别：
- 核心区别在于：数据处理阶段
- Reactor: 同步非阻塞 Proactor: 异步非阻塞
- Reactor: 就绪通知 Proactor: 完成通知
- Reactor: IO操作由应用完成  Proactor: IO操作由内核处理
- Reactor: Linux(epoll) 和 Windows(IOCP)均可实现  Proactor: 需要操作系统支持异步IO Windows(IOCP) 
- 使用场景： Reactor: 跨平台 短链接 业务逻辑轻 Proactor: 大文件/视频 极致吞吐 Windows高性能服务 可接受较高复杂度


