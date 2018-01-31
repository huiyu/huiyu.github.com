---
layout: post
title: "限流问题总结"
categories: "算法"
tags: [算法, Guava, Java, 并发, 分布式系统, 网络]
---

速率控制（rate limiting）或流量整形（traffic shaping）典型应用就是限制网络流量的突发，使报文以比较匀速的方式进行传输。常见算法有漏桶算法（leaky bucket）和令牌桶算法（token bucket）。

## Leaky Bucket算法

Leaky bucket算法将流量缓冲区抽象成一个漏水的桶，水（请求）首先会加入桶里，然后以恒定速率漏出。如果桶满了（异常流量），要么就丢掉，要么就通知上游降速。

Leaky Bucket是限制流量的很朴素的算法，实现算法只需要给定两个参数：

- 桶漏水的速率
- 桶的大小

实际应用中有很多变化，主要在于漏水的速率以及桶的的大小，以及处理异常流量的策略。

<img src="{{site.baseurl}}/images/rate-limiting/leaky-bucket.jpg" alt="Leaky Bucket" style="height:300px;"/>

## Token Bucket算法

Token bucket算法也很简单，相当于leaky bucket的镜像算法：

- 每隔`1/r`秒添加一个令牌。
- 桶内最多有`b`个令牌，多余的令牌将被丢弃。
- 当n字节请求到达时，那么就从桶内删除n个令牌，并对请求放行。
- 当n字节请求到达，但桶内不足n个令牌，那么该请求将会被限制。

<img src="{{site.baseurl}}/images/rate-limiting/token-bucket.jpg" alt="Token Bucket" style="height:256px;"/>

Leaky bucket和token bucket算法都能实现速度控制的功能，但是两者有点区别。Leaky bucket强制限制数据的传输速率，而token bucket允许某种程度的突发传输。

## Guava RateLimter分析

[Guava](https://github.com/google/guava)提供了一个RateLimiter类，实现了Token Bucket算法。使用RateLimiter非常简单，详情可以参见[API文档](https://google.github.io/guava/releases/19.0/api/docs/index.html?com/google/common/util/concurrent/RateLimiter.html)，这里主要分析一下RateLimiter的设计。

首先实现token bucket算法并不需要一个类似bucket的容器来存储token。只需要记住上一次请求的时间，就可以算出当前有多少个token。假设参数`permitsPerSecond`每秒钟允许的请求数（Guava里的permit等同于token），上一次请求的时间为t，那么当前有多少permit就可以通过以下计算得出：

```java
permitsPerSecond * toSeconds(now() - t)
```

但是这样的实现存在一些问题：

1. 假设在一个请求q来之前有很长时间的空闲，那么说明RateLimiter其实是未被充分利用的，但是它只会记得最后一次请求q的时间。
2. 如果在长时间的空闲之后突然来了一个大请求，那么RateLimiter的放行请求或导致服务器准备不足。
3. 在RateLimiter刚启动时的请求会被阻塞。

来看Guava RateLimiter是怎么解决这些问题的。

针对过去时间内的空闲资源，RateLimiter抽象出了`storedPermits`的概念，为0时表示没有空闲，最大能到`maxStoredPermits`，表示完全空闲。

第2个问题，其实就是RateLimiter不能将`storedPermits`一股脑的全部放行，这里RateLimiter抽象出了一个`storedPermitsToWaitTime(storedPermits, permitsToTake)`方法和`coolDownIntervalMicros`方法，前者给定当前存储的permit数和需要请求的permit数，返回要等待的时间；后者调整两个permit之间的间隔用于控制permit产生的速率。这里Guava提供了两种实现，后面会详细分析。

对于RateLimiter启动时的请求阻塞问题，RateLimiter没有记住最后一次请求的时间，取而代之的是记住下一次允许请求的时间。假如RateLimiter启动时请求n个permit，那么RateLimiter会直接放行，然后计算下一次允许的时间：

```java
now + (n / permitsPerSeconds)
```

理解了以上再来看RateLimiter的代码就会简单很多。RateLimiter抽象类主要有两类方法：

* `aquire`方法主要用于请求一定数量的permit，调用该方法会阻塞直至成功获取，并返回阻塞的时间。
* `tryAquire`方法同样适用于请求一定数量的permit，区别在于不会阻塞，成功获取就返回true，失败则返回false。

这里以aquire为例，分析一下其调用链。如下图所示，acquire首先调用reserve方法预约permit并计算所需等待时间，然后执行等待后返回等待时间。`reserve`方法则将具体计算委托给`SmoothRateLimiter`类的`reserveEarliestAvailable`方法。

![RateLimiter Acquire Call Chain]({{site.baseurl}}/images/rate-limiting/ratelimiter-call-chain.png)

再来看`reserveEarliestAvailable`方法的实现：

```java
@Override
final long reserveEarliestAvailable(int requiredPermits, long nowMicros) {
  // 刷新storePermits和nextFreeTicketMicros
  resync(nowMicros);
  // 下一次允许请求的时间
  long returnValue = nextFreeTicketMicros;
  // 计算消耗的permits
  double storedPermitsToSpend = min(requiredPermits, this.storedPermits);
  // 计算所需permits和目前能提供的permits之间的缺口
  double freshPermits = requiredPermits - storedPermitsToSpend;
  // 计算等待permites缺口所需时间，在等待freshPermits的基础上加上storedPermitsToWaitTime
  long waitMicros =
      storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend)
          + (long) (freshPermits * stableIntervalMicros);
  // 更新nextFreeTicketMicros
  this.nextFreeTicketMicros = LongMath.saturatedAdd(nextFreeTicketMicros, waitMicros);
  // 更新storePermits
  this.storedPermits -= storedPermitsToSpend;
  return returnValue;
}

void resync(long nowMicros) {
  // 如果nextFreeTicket落后于当前时间，则刷新当前时间
  if (nowMicros > nextFreeTicketMicros) {
    // 计算nextFreeTicket到nowMicros之间产生的新的permits
    double newPermits = (nowMicros - nextFreeTicketMicros) / coolDownIntervalMicros();
    // 更新storedPermits，不能大于maxPermits
    storedPermits = min(maxPermits, storedPermits + newPermits);
    // 更新nextFreeTicketMicros
    nextFreeTicketMicros = nowMicros;
  }
}
```

这里`storedPermitsToWaitTime`和`coolDownIntervalMicros`是`SmoothRateLimiter`类的两个抽象方法。

SmoothRateLimiter有两个实现：SmoothBursty和SmoothWarmingUp。SmoothBursty的实现非常直接，`storedPermitsToWaitTime`方法直接返回0，`coolDownIntervalMicros`返回的是默认间隔`stableIntervalMicros`，也就是说对于长时间等待后的突然请求，SmoothBursty没有起到任何缓冲作用。

而SmoothWarmingUp的原理如下所示（来自SmoothRateLimiter源码注释），这个示意图的意思是说，当超过thresholdPermits这个阈值后，需要等待更长时间的恢复（warmup）。

```
          ^ throttling
          |
    cold  +                  /
 interval |                 /.
          |                / .
          |               /  .   ← "warmup period" is the area of the trapezoid
          |              /   .     between thresholdPermits and maxPermits
          |             /    .
          |            /     .
          |           /      .
   stable +----------/  WARM .
 interval |          .   UP  .
          |          . PERIOD.
          |          .       .
        0 +----------+-------+--------------→ storedPermits
          0 thresholdPermits maxPermits
```

当采用SmmothBursty实现时，相当于上图中只有一条水平的stable interval线，那么`storedPermitsToWaitTime`实现是`permits * stableInterval`，相当于**计算一个矩形的面积**。

当采用SmoothWarmingUp实现时，`storedPermitsToWaitTime`函数计算分成两部分：

* 超过thresholdPermits的**直角梯形部分的面积**。
* 如果请求的permits大于超过thresholdPermits的部分能提供，那么计算超过部分的**矩形面积**。

面积计算如下所示：

```
          ^ throttling
          |
    cold  +                  /
 interval |                 /.
          |                / .
          |               /  .    
          |              /   .     
          |             /    .
          |            /.|   .
          |           /..|   .  
   stable +----------/...|   .
 interval |      |...|...|   .  ← 总面积由两块组成，左半部分是矩形，右半部分是楔形
          |      |...|...|   .
          |      |...|...|   .
        0 +----------+---+---+--------------→ storedPermits
          0          storePermits
```

SmoothWarmingUp的`storedPermitsToWaitTime`函数实现实际上就是在计算这块复合区域的面积，这里就不再复制代码了。

同样地，当请求的permits数超过thresholdPermits时，发放新的pertmis数量是由`coolDownIntervalMicros`来控制，如果warmup时间比较长，相对就恢复比较慢，反之就会比较快：

```java
@Override
double coolDownIntervalMicros() {
  return warmupPeriodMicros / maxPermits;
}
```

SmoothWarmingUp通过这两个函数就相当于实现了一个自动的阀门控制，与TCP协议基于滑动窗口的拥塞控制有异曲同工之妙。

## 分布式限流

Guava的RateLimiter较为完美地解决了单节点上限流的问题，在分布式环境上限流类问题就略微复杂一些，主要有两种方式。

如果是只是较为粗略地限制请求数量，不要求非常精确的限流的话，可以考虑在n个节点中各自处理限流。比如n个节点负载均衡，每个节点都要进行限流处理，要求总请求不超过每秒m次，那么如果节点都是被访问量比较均匀的话，那么每个节点就可以各自启动一个`permitPerSecond = n/m`的RateLimiter来处理。

如果需要精确限流的话，比如限制第三方api调用次数之类，那么就需要考虑一个集中化的RateLimiter服务了。如果考虑到RateLimiter服务本身的高可用，那么就需要将状态数据，如调用次数等存储到一个中间存储介质，如Redis、Zookeeper等，这里的实现就比较灵活了，Github上也有相应开源代码可以供借鉴。

## 参考资料

- [维基百科：Traffic shaping](https://en.wikipedia.org/wiki/Traffic_shaping)
- [维基百科：Leaky bucket](https://en.wikipedia.org/wiki/Leaky_bucket)
- [维基百科：Token bucket](https://en.wikipedia.org/wiki/Token_bucket)
- [Guava RateLimiter](https://google.github.io/guava/releases/19.0/api/docs/index.html?com/google/common/util/concurrent/RateLimiter.html)
- [聊聊高并发系统之限流特技](http://mp.weixin.qq.com/s?__biz=MzIwODA4NjMwNA==&mid=2652897781&idx=1&sn=ae121ce4c3c37b7158bc9f067fa024c0#rd)
- [服务化体系之－限流](http://calvin1978.blogcn.com/articles/ratelimiter.html)
