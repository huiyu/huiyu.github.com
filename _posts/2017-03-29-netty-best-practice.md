---
layout: post
title: "Netty最佳实践"
categories: Java
tags: [Java, Netty, IO, 网络]
---

Netty某种意义上已经成为高性能Java网络编程的代名词，本文总结一些Netty编程的最佳实践以作备忘。

## 引导配置

### Netty Native

Netty native是由Twitter从tomcat移植过来的，主要有以下特性：

* 更好性能、更低CPU占用、更少同步和GC。
* 内置Twitter OpenSSL based SSLEngine，比Java原生SSL更快，更少GC。
* 支持TCP_CORK参数。
* 只支持Linux。

使用netty native非常简单，可以参照[官方文档](https://netty.io/wiki/native-transports.html)。据Norman Maurer说谷歌已经在生产环境中使用netty native了，所以稳定性应该是有保证的。

### Channel选项

Netty支持的Channel选项都在ChannelOption类里，常见的有：

* SO_REUSEADDR：是否重用地址。
* SO_KEEPALIVE：服务器是否保持连接。
* TCP_NODELAY：禁用Nagle算法，不会将小包合并成大包。
* TCP_CORK：避免发送小包，提升网络吞吐。
* SO_LINGER：定义close连接的超时时间。
* SO_TIMEOUT：定义阻塞操作的超时时间。
* SO_BACKLOG：定义等待连接队列的最大长度，参见[What does ChannelOption.SO_BACKLOG do?](https://stackoverflow.com/questions/14075688/what-does-channeloption-so-backlog-do)
* SO_SNDBUF：发送缓冲区大小。
* SO_RCVBUF：接受缓冲区大小。

详情可以参见这篇[文章](http://blog.csdn.net/zlxfogger/article/details/44922993)以及java.net.SocketOptions类。

## EventLoopGroup

### 永远不要阻塞EventLoop

将如数据库操作，远程调用等阻塞任务，放入业务线程池里进行执行，即使付出一些一些上下文切换的代价。阻塞EventLoop可以造成IO事件堆积，使得服务不能即使响应请求。

### 共享EventLoopGroup

可以的话尽量共享EventLoopGroup，可以减少线程数量，提升利用率。

### EventLoop as ScheduledExecutorService

EventLoop同时实现了ScheduledExecutorService接口，因此可以直接使用它来进行一些定时任务如超时控制等。

## ByteBuf

### 堆外内存

使用堆外内存是高性能java服务器的标配，这也是netty在DirectByteBuffer的基础上封装了DirectByteBuf，并实现了内存池算法。

DirectByteBuf的分配非常慢，因此尽量使用PooledByteBuffer：

```java
Bootstrap b = new Bootstrap();
b.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
ServerBootstrap b = new ServerBootstrap();
b.childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
```

> 池化的DirectByteBuf在版本4.1.x后默认开启

此外，池化的DirectByteBuf的释放是采用了引用计数法，一般由pipeline中最后一个Handler来进行释放，一定要注意各种异常情况下是否正确释放，并关注日志中是否有Leak信息。

### 遍历ByteBuf

避免以下方式遍历ByteBuf：

```java
ByteBuf buf = ...;
int index = -1;
for (int i = buf.readerIndex(); index = -1 && i < buf.writerIndex(); i++) {
  if (buf.getByte(i) == '\n') {
    index = i;
  }
}
```

`getByte`操作不是简单地读一个字节，它会检查引用计数、是否越界等。当需要遍历ByteBuf的时候，使用`forEachByte(ByteBufProcessor)`方法，使用`ByteBufProcessor`更快，因为它避免了越界检查，以及更易被JIT内联。

```java
buf.forEachByte(new new ByteBufProcessor() {
  @Override
  public boolean process(byte value) {
    return value != '\n';
  }
});
```

### ASCIIString

在一些场景下（比如存储头部信息）用于替换String（utf8 based），里面是一个byte数组而不是char数组，减少了内存占用和提升数据传输速度。Netty在内部的一些常量定义上使用了ASCIIString，比如HttpHeaderNames。

### ByteBuf Payload

各种协议都有各自的playload，此时可以通过实现ByteBufHolder接口（直接继承DefaultByteBufHolder）来实现将payload存储在ByteBuf中的消息对象。ByteBufHolder提供了一些高级特性：

* 引用计数
* 缓冲区池化

具体可以参见Netty的一些内置实现，如WebSocketFrame等。

### 避免复制ByteBuf

尽量避免ByteBuf的复制操作（copy），使用duplicat和slice，但是要注意使用duplicate和slice返回ByteBuf，对其内容的修改会影响原ByteBuf，并且refCnt不变（需要手动调用retain方法）。

在某些场景下可以利用CompositeByteBuf来组合和重用ByteBuf。

## Pipelining

### 减少writeAndFlush

通常我们会这样写：

```java
public class HttpHandler extends SimpleChannelInboundHandler<HttpRequest> {
  @Override
  public void channelRead(ChannelHandlerContext ctx, HttpRequest req) {
    ChannelFuture future = ctx.writeAndFlush(createResponse(req));
  }
}
```

这里`ChannelHandlerContext`类有三个方法：

* `write(msg)`将消息传递所有的Pipeline，没有系统调用。
* `flush()`将所有消息冲刷到网卡中，存在系统调用。
* `writeAndFlush(msg)`等同于`write(msg)`和`flush()`。

系统调用还是比较昂贵的，所以尽量少直接调用`writeAndFlush`：

```java
public class HttpHandler extends SimpleChannelInboundHandler<HttpRequest> {
  @Override
  public void channelRead(ChannelHandlerContext ctx, HttpRequest req) {
    ChannelFuture future = ctx.write(createResponse(req));
  }
  
  @Override
  public void channelReadComplete(ChannelHandlerContext ctx) {
    ctx.flush();
  }
}
```

这里`channelReadComplete(ctx)`当[没有更多数据可读](https://stackoverflow.com/questions/28473252/how-does-netty-determine-when-a-read-is-complete)的时候会被调用。

### 共享ChannelHandler

如果ChannelHandler是无状态的，那么可以设置为Sharable的（通过`@ChannelHandler.Shareable`注解），这样可以减少对象数量。

### 避免直接调用Channel的write方法

```java
public class YourHandler extends ChannelInboundHandlerAdapter {
  @Override
  public void channelActive(ChannelHandlerContext ctx) {
    // BAD (most of the times)
    ctx.channel().writeAndFlush(msg); 
    // GOOD
    ctx.writeAndFlush(msg); 
   }
}
```

如上，`ctx.channel().writeAndFlush(msg)`会将消息穿过整个pipeline，而`ctx.writeAndFlush(msg)`只会将消息传递给下一个Handler。

### 避免过快写入

过快的写入可能会使远端服务器来不及消费而造成OOM。可以通过一些手段进行控制：

```java
public class GracefulWriteHandler extends ChannelInboundHandlerAdapter {
  @Override
  public void channelActive(ChannelHandlerContext ctx) {
    writeIfPossible(ctx.channel());
  }
  @Override
  public void channelWritabilityChanged(ChannelHandlerContext ctx) {
    writeIfPossible(ctx.channel());
  }

  private void writeIfPossible(Channel channel) {
    while(needsToWrite && channel.isWritable()) { 
      channel.writeAndFlush(createMessage());
    }
  }
}
```

这里使用`channel.isWritable()`方法来探测状态是否可写从而避免OOM。

此外还可以通过设置水位（watermark）来进行流量规整：

```java
// server
ServerBootstrap bootstrap = new ServerBootstrap();
bootstrap.childOption(ChannelOption.WRITE_BUFFER_HIGH_WATER_MARK, 32 * 1024);
bootstrap.childOption(ChannelOption.WRITE_BUFFER_LOW_WATER_MARK, 8 * 1024);
// client
Bootstrap bootstrap = new Bootstrap();
bootstrap.option(ChannelOption.WRITE_BUFFER_HIGH_WATER_MARK, 32 * 1024);
bootstrap.option(ChannelOption.WRITE_BUFFER_LOW_WATER_MARK, 8 * 1024);
```

关于watermark的具体作用可参见[Pipelining and flow control](http://adolgarev.blogspot.tw/2013/12/pipelining-and-flow-control.html?view=flipcard)。

### VoidChannelPromise

Channel的writeAndFlush(msg)方法默认会返回一个ChannelFuture，无论你需不需要。如果你对该ChannelFuture不感兴趣，可以通过以下方式避免创建该实例，减少一些GC压力：

```java
Channel.write(msg, Channel.voidPromise())
```

## 总结

一些netty的最佳实践大多都是拾人牙慧，详情可见参考资料。性能方面netty通常不会是瓶颈，瓶颈往往是在数据库等其他操作上，因此还是要做好profile，找出关键路径。

## 参考资料

* [Netty实战](https://book.douban.com/subject/27038538/)
* [Netty Best Practtice](http://normanmaurer.me/presentations/2014-facebook-eng-netty/slides.html#1.0)
* [Netty高性能编程备忘录](http://calvin1978.blogcn.com/articles/netty-performance.html)