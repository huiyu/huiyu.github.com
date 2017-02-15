---
layout: post
title: "Java8 CompletableFuture"
categories: Java
tags: [Java, Future, 并发, 异步]
---

Java的Future接口抽象了一个异步计算过程，提供了一些方法可以检查或等待执行结果。Future功能较弱，仅支持两种用法，要么检查计算是否完成，要么阻塞地等待计算完成。Guava的ListenableFuture扩展了Future接口，可以在计算完成执行回调函数。Java8添加了CompletableFuture类，除了支持函数回调以外，还实现了新的CompletionStage接口。CompletionStage代表了异步计算中的一个阶段或者步骤，使用者可以自由组合各种CompletionStage，完成各种强大的计算任务。

## 创建异步任务

创建CompletableFuture异步任务非常简单：

```java
CompletableFuture.supplyAsync(this::downloadImage);
```

工厂方法`supplyAsync`接收一个函数式对象`Supplier`作为参数，包含了想要异步执行的代码。默认情况下，`supplyAsync`会将`Supplier`运行在`ForkJoinPool.commonPool()`中。此外`supplyAsync`还有一个重载版本接受额外的`Executor`对象作为`supplier`的运行容器。

除了`supplyAsync`外，还有以下几种方式创建`CompletableFuture`：

* 工厂方法`runAsync`接受一个`Runnable`对象以及一个可选的`Executor`对象，用于执行无需返回值的计算作业
* CompletableFuture的无参构造器，返回一个处于未完成的CompletableFuture对象，该对象需要手动完成，否则将永远处于未完成状态
* 工厂方法`completedFuture`直接将执行结果包装成一个处于完成状态的CompletableFuture对象

## 获取结果

CompletableFuture实现了Future接口，因此可以通过`get()`方法获取执行状态，`get()`方法会阻塞当前线程直到完成或取消。除此之外，CompletableFuture还支持回调方法：

```java
CompletableFuture.supplyAsync(this::downloadImage)
                 .thenAccept(this::notify);
```

`thenAccept`接收一个`Consumer<T>`对象，当CompletableFuture完成时将回调`Consumer.accept`方法。这里传入`thenAccept`的对象`Consumer`的`accept`方法是被同步调用的，也就是执行CompletableFuture和执行`accept`方法是同一个线程。`thenAcceptAsync`是`thenAccept`的异步版本，将会异步地调用`Consumer.accept`方法。

`thenAccept`是众多回调方法之一，每种回调方法都会有同步和异步两种，而异步方法（以`Async`结尾）与`supplyAsync`类似，也都会有两个重载版本，区别也是异步代码时运行在`ForkJoinPool.commonPool`中还是在给定的`Executor`对象中。

## 转换结果

`thenAccept`方法是有返回值的，返回值也是一个`CompletableFuture`对象。事实上，CompletableFuture所有的回调方法都会返回一个CompletableFuture对象，这正是CompletableFuture强大的地方：可以通过链式调用形成新的CompletableFuture。

```java
CompletableFuture.supplyAsync(this::findImage)
                 .thenApplyAsync(this::downloadImage)
                 .thenAccept(this::notify);
```

## 组合其他CompletableFuture

CompletableFuture之间可以互相组合形成新的CompletableFuture。比如下载通常是非常耗时并且会阻塞当前线程，因此可以将`downloadImage`方法包装成一个异步方法：

```java
CompletableFuture<Result> downloadImageAsync(String url) {
  return CompletableFuture.supplyAsync(() -> downloadImage(url));
}
```
然后就可以通过`thenCompose`将两个CompletableFuture组合起来：

```java
CompletableFuture.supplyAsync(this::findImage)
                 .thenCompose(this::downloadImageAsync);
```

除了`thenCompose`，CompletableFuture还提供了一系列方法用于组合CompletableFuture，详情可以参见[API文档](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html)

## 错误处理

有时候异步运行的代码会出错，CompletableFuture提供了`exceptionally`方法用于注册错误处理的回调函数：

```java
CompletableFuture.supplyAsync(this::downloadImage)
                 .thenApplyAsync(ex -> new Result(Status.FAIL))
                 .thenAccept(this::notify);
```

`exceptionally`提供了一种错误处理和修正的机制，它的返回值会作为结果传递给`thenAccept`。如果想同时处理正常结束和异常结束，那么可以使用`whenComplete`：

```java
CompletableFuture.supplyAsync(this::downloadImage)
                 .whenComplete((result, ex) -> {
                   if (ex != null)
                     result = new Result(Status.FAIL);
                   notify(result);
                 })
```

## 参考资料
* [Java 8: Writing asynchronous code with CompletableFuture](http://www.deadcoderising.com/java8-writing-asynchronous-code-with-completablefuture/)
* [使用Java 8的CompletableFuture实现函数式的回调](http://www.infoq.com/cn/articles/Functional-Style-Callbacks-Using-CompletableFuture)
