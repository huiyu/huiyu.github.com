---
layout: post
title: Netty实现断线重连
categories: Java
summary: "讨论Netty客户端断线重连的问题"
tags: [Java, Netty, 重连]
---

服务器端网络编程中，重连机制非常重要。要实现重连要考虑两个状态下的重连：

1. 建立连接失败时
2. 传输过程中失去连接时

Netty提供了`ChannelFutureListener`接口来监听网络中的各种事件，可以对这些网络异常进行监听以及处理。

Netty基本的编程框架不再赘述，来看以下代码：

```java
public void connect() {
    try {
        ChannelFuture future = bootstrap.connect(hostname, port).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    doReconnect();
                }
            }
        }).sync();

        future.channel().closeFuture().addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                doReconnect();
            }
        });
    } catch (Exception e) {
        System.err.println(e.getMessage());
    }
}
```

我们知道，Netty中的操作都是异步事件驱动的，当客户端发起连接时，会返回一个`ChannelFuture`对象，我们可以通过注册一个监听器来获取异常状态并对其进行处理。

由于这里建立连接和失去连接是两种状态，因此就对应两个`ChannelFuture`，我们分别向其注册`ChannelFutureListener`，第一个`Listener`在操作失败，也就是建立连接失败时执行重连；第二个`Listener`注册至`closeFuture`，也就是关闭连接状态触发时执行重连。

```java
private synchronized void doReconnect() throws InterruptedException {
    Thread.sleep(1000L);
    connect();
}
```

方法`doReconnect`非常简单，只需要简单调用一下connect即可。

对于失去连接状态的重连处理，也可以使用ChannelHandler处理：

```java
bootstrap = new Bootstrap().group(workerGroup)
                           .channel(NioSocketChannel.class)
                           .handler(new ChannelInitializer<SocketChannel>() {
                               @Override
                               protected void initChannel(SocketChannel channel) throws Exception {
                                   ChannelPipeline pipeline = channel.pipeline();
                                   pipeline.addFirst(new ChannelInboundHandlerAdapter() {
                                       @Override
                                       public void channelInactive(ChannelHandlerContext ctx) throws Exception {
                                           doReconnect();
                                       }
                                   });
                               }
                           });
```

这样的效果和`closeFuture().addListner()`是一样的。

[Netty权威指南](http://book.douban.com/subject/25897245/)里面有另外一种实现方式，试了一下也是可以的：

```java
private ScheduledExecutorService executor = Executors.newScheduledThreadPool(1);

public void connect() {
    try {
        ChannelFuture future = bootstrap.connect(hostname, port).sync();
        future.channel().closeFuture().sync();
    } catch (Exception e) {
        System.err.println(e.getMessage());
    } finally {
        executor.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    doReconnect();
                } catch (Exception e) {
                    System.err.println(e.getMessage());
                }
            }
        });
    }
}
```
`future.channel().closeFunture().sync()`会阻塞当前线程，直至连接关闭，因此在`finally`语句中发起重连操作也是可以实现重连目的。但是这种方式的缺点是需要声明额外的线程池，并且会阻塞当前线程，违背了netty非阻塞IO的设计初衷。

## 参考资料

1. [Netty TCP client with reconnect handling](http://tterm.blogspot.jp/2014/03/netty-tcp-client-with-reconnect-handling.html)
2. [What's the best way to reconnect after connection closed in Netty](http://stackoverflow.com/questions/19739054/whats-the-best-way-to-reconnect-after-connection-closed-in-netty)
3. [Netty自动重连](http://www.dozer.cc/2015/05/netty-auto-reconnect.html)
4. [Netty权威指南](http://book.douban.com/subject/25897245/)
