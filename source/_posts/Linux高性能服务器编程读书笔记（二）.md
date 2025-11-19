---
title: Linux高性能服务器编程读书笔记（二）
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
### epoll原理：
使用：

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
1. LT vs ET
2. 为何ET用非阻塞socket

### Reactor模型思想：
Reactor 事件驱动
Proactor 


