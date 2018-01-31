---
layout: post
title: "Google FlumeJava 论文笔记"
categories: 论文
tags: [分布式系统, 论文, MapReduce, Pipeline, Java]
---

Google在2010年发表了一篇论文，介绍了其并行编程框架FlumeJava。FlumeJava在MapReduce基础上提供了数据管道和更高层次的抽象操作，并在此基础上优化了执行器。相比于MapReduce，FlumeJava更简单易用，后来的Spark等大量的并行编程框架都参考了FlumeJava的设计。

## 核心抽象和基本操作

### PCollection抽象

`PCollection<T>`表示一个不可变集合，可以是有序，也可以无序，既可以由一个内存中的Java集合对象创建，也可以通过读取文件来创建。

### PTable抽象

`PTable<K,V>`表示一个不可变map，是`PCollection<Pair<K,V>>`的子类。

### parallelDo操作

`parallelDo`操作将一个`PCollection<T>`转换成另一个`PCollection<S>`。

```java
PCollection<String> lines = ...;
PCollection<String> words = lines.parallelDo(new DoFn<String, String>() {
  void process(String line, EmitFn(String> emitFn)) {
    for (String word: splitIntoWords(line)) {
      emitFn.emit(word);
    }
  }
});
```

上述代码将lines转换成words，其中`DoFn`函数接口的`process`方法有两个函数，一个参数表示要处理的数据，第二个参数是一个回调函数，只有调用它的`emit`方法的数据才会被添加到输出`PCollection<String>`中。`mapFn`和`filterFn`都可以通过`parallelDo`操作衍生出来。

`parallelDo`可以同时表示MapReduce的map过程和reduce过程。由于该函数会被并行执行，因此`DoFn`函数不应该依赖Java闭包函数外的全局状态，也就是说`DoFn`应该是纯函数式的调用。

### groupByKey操作

`groupByKey`操作将`PTable<K,V>`转换成`PTable<K, Collection<V>>`，`Coolection<V>`是一个纯Java无序集合，包含对应key下的所有value值。`groupByKey`实质上等同于MapReduce的shuffle操作。

### combineValues操作

`combineValues`操作接受一个`PTable<K,Collection<V1>>`，将其转换成`PTable<K,V2>`。`combineValues`操作实际上是`parallleDo`操作的一个特例。

### flatten操作

`flatten`操作接受一个`PCollection<T>`列表，将其转换成一个单一的`PCollection<T>`。

## 衍生操作

FlumeJava有一系列其他操作，是由基本操作派生而来。

### count操作

`count`操作将`PCollection<T>`转成一个`PTable<T, Integer>`，其中`PTable<T, Integer>`表示每个类型T到T数量映射：

```java
PTable<String, Integer> wordCounts = words.count();
```

`count`操作实际上是由`parallelDo`、`groupBy`和`combineValues`组合而成：

```java
PTable<String, Integer> wordCounts = words
	.parallelDo(new DoFn<String, Pair<String, Integer>() {
    	void process(String line, EmitFn<Pair<String, Integer>> emitFn) {
    		emitFn.emit(Pair.of(line, 1));
	    }
	})
	.groupByKey()
  	.combineValues(SUM_INTS);
```

### join操作

`join`操作接受`PTable<K, V1>`和`PTable<K, V2>`，返回一个新的`PTable<K,Tuple2<Collection<V1>,Collection<V2>>>`。

`join`操作的计算方式如下：

1. 使用`parallelDo`操作将`PTable<K,V1>`和`PTable<K,V2>`各自转换为`PTable<K,TaggedUnion2<V1,V2>>`。
2. 使用`flatten`合并得到一个`PTable<K,TaggedUnion2<V1,V2>>`。
3. 使用`groupByKey`得到一个`PTable<K, Collection<TaggedUnion2<V1,V2>>>`。
4. 使用`parallelDo`将每个`Collection<TaggedUnion2<V1,V2>>`转换成一个`Collection<V1>`和一个`Collection<V2>`。

### top操作

`top`操作，接受一个整型N和一个比较函数，得出`PCollection`中Top N元素组成的`PCollection`子集。该操作由`parallelDo`、`groupByKey`和`combineValues`操作组成。

这里论文里没有详细说明是怎么实现的，但是从Apache Crunch的源码可以大致推算计算步骤：

1. 使用`parallelDo`函数对`PCollection<T>`中的每一个分片执行Top N过滤，仅保留每个分片的N条记录，得到`PTable<0,T>`，这里的数字0是为了`groupByKey`。
2. 对`PTable<0,T>`执行`groupByKey`，得到`PTable<0,Collection<Pair<0, T>>`。
3. 对`PTable<0, Collection<Pair<0,T>>`应用`combineValues`，对`Collection<Pair<0,T>>`执行Top N过滤，最终得到`PTable<0, T>`。
4. 将`PTable<0, T>`执行`parallelDo`映射成`PTable<T>`。

## 惰性求值

延迟求值（Deferred Evaluation）或惰性求值（Lazy Evaluation）是程序设计语言中的概念，在这里用于优化求值。`PCollection`有两种类型的状态，deferred表示未求值，materialized表示已求值。

从前面可以知道，一个`PCollection`可以由其他`PCollection`通过操作转换而来，当一个`PCollection`执行`parallelDo`操作时，并不会真正执行其中的`DoFn`函数，而是创建一个defferred状态的`PCollection`，保存了原始`PCollection`和需要执行的`parallelDo`操作。执行多个FlumeJava的操作组成了一个有向无环图，称之为execution plan。只有执行了`FlumeJava.run()`时，才会按照execution plan执行所有的操作，将`PCollection`从`defferred`状态转换成`materialized`状态。

## PObject抽象

为了在执行过程中观察`PCollection`的内容，FlumeJava提出了`PObject`抽象。PObject是一个单一Java对象容器，和`PCollection`类似，也有`defferred`和`materialized`两种状态。当一个pipline执行后，`PObject`中的内容可以通过`getValue()`获取。PObject的工作模式和Java中的Future对象非常类似。

比如`PCollection<T>.asSequencetialCollection()`操作会产生一个`PObject<Collection<T>>`，当运行`FlumeJava.run()`后，可以通过`getValue()`获取一个内存的Java Collection对象：

```java
PTable<String,Integer> wordCounts = ...;
PObject<Collection<Pair<String, Integer>>> result = wordCounts.asSequentialCollection();
...
FlumeJava.run();
for (Pair<String, Integer> count : result.getValue()) {
  System.out.println(count.first + ": " + count.second);
}
```

## 优化器

### ParallelDo融合

最简单的一种优化策略是ParallelDo融合。ParallelDo融合有两种方式，一种叫做producer-consumer融合，另一种叫sibling融合。

所谓producer-consumer融合就是指当一个parallelDo执行一个函数$f$的结果作为另一个parallelDo执行$g$函数的输入，那么这两个parallelDo会被合并成一个parallelDo来执行，同时输出函数$f$和复合函数$f \circ g$的执行结果，换句话说数据源PCollection在这种情况下只会被遍历一遍。如果函数$f$的执行结果没有被其他操作引用，那么生成$f$结果的代码也会被优化器移除。

ParallelDo sibling融合是指当多个ParallelDo操作读取的是同一个PCollection时，它们的读取会合并成一个parallelDo来执行，同样地在这种情况下PCollection只会被遍历一遍。

如下图所示一个执行计划的优化过程，优化器将A、B、C、D四个ParallelDo操作融合成了一个ParallelDo操作。其中中间结果A.0不被需要所以就被丢弃了，而A.1被其他操作引用而得以保留。

<img src="{{ site.baseurl }}/images/flumejava/paralleldo-fusion.png" height="500" />

### MSCR操作

FlumeJava的一个核心优化，就是将ParallelDo、GroupByKey、CombineValues和Flatten操作的组合，转换成一个MapReduce。为了在这两种抽象中建立联系，FlumeJava引入了一个中间层，叫做MSCR（MapShuffleCombineReduce）操作。

一个MSCR操作有M个输入通道（每个通道执行map操作）和R个输出通道（每个通道执行一个可选shuffle、一个可选的combine以及一个reduce操作）。每个输入通道m接收一个$PCollection \langle T_m \rangle $作为输入，然后执行一个ParallelDo（map）操作得到一个拥有R个键值对的$PTable\langle K_r,V_r \rangle$，这里每个输入通道都将元素发射到一个或多个输出通道上。每个操作通道r可以选择：

1. 将它的输入执行Flatten，然后依次执行GroupByKey（shuffle）、可选的CombineValues（combine）、ParallelDo（reduce）得到$O_r$个输出，这种称之为分组输出通道（grouping output channel）。
2. 将它的输入直接写入输出，这种称之为传递通道（pass-through output channel）。

如下图所示是拥有三个输入通道，两个分组输出通道，以及一个传递通道（pass-through channel）的MSCR。

<img src="{{ site.baseurl }}/images/flumejava/mscr-example.png" height="500"/>

MSCR通过以下几个方面泛化了MapReduce：

* 允许多个reducer和combiner
* 允许每个reducer产生多个输出
* 允许reducer接收不同key类型的输入
* 允许传递通道（pass-through）形式的输出


### MSCR融合

一个MSCR操作是由一组**相互关联**的GroupByKey操作产生的。一组GroupByKey操作被认为是互相关联的，仅当它们消费同一个输入或同一个（融合了）ParallelDo产生的输入。

MSCR操作的输入和输出通道就是从这一组相互关联的GroupByKey操作上派生的。每一个ParallelDo操作如果被一个或多个GroupByKey消费（通过Flatten），就会产生一个输入通道，此外其他不经过ParallelDo处理的输入也会形成一个输入通道。每个GroupByKey都会形成一个分组输出通道。输入通道产生结果如果需要输出，则会形成一个传递通道。

如下图所示一个执行计划如何进行MSCR融合的。图中三个GroupByKey操作是相关联的，所以它们将融合成一个MSCR操作。其中M2、M3和M4会形成三个输入通道，Op1没有经过ParallelDo，但是被GBK1消费，也会形成一个输入通道。这里GBK1、GBK2和GBK3会形成三个分组通道，Op1和M4产生的输出形成两个传递通道。

![]({{ site.baseurl }}/images/flumejava/mscr-fusion.png)

这样所有的GroupBy操作都会融合成MSCR操作。此外所有其他的ParallelDo也会形成一个只包含一个ParallelDo和一个传递通道的的MSCR。**优化后的执行计划将只会包含MSCR操作**。

### 优化策略

优化器按照以下步骤优化执行计划：

1. 下沉Flatten。一个Flatten操作可以被下推到ParallelDo下面，也就是$h(f(a) + g(b))$可以转换成$h(f(a)) + h(g(b))$。下沉Flatten可以为ParallelDo融合创造条件。
2. 上提CombineValues。如果CombineValues紧跟一个GoupByKey，那么就会被识别成MSCR的一部分。
3. 插入Fusion块标记。如果两个GroupByKey通过一个ParalellDo相连，形成了producer-consumer链。那么优化器必须选择ParallelDo操作是和上一个GroupByKey融合还是下一个GroupByKey融合。优化器会估计中间PCollection大小来选择具体的融合方向。
4. 融合ParallelDo。
5. 融合MSCR。

## 总结

FlumeJava构建在MapReduce之上，提供了更高层次的抽象以及更好的pipeline支持。并且通过惰性求值以及操作融合等优化措施接近手动优化后的MapReduce pipline。论文中还介绍了一个Site-Data案例、执行器的一些执行策略，这里就不多展开了。

## 参考资料
* [FlumeJava: Easy, Efficient Data-Parallel Pipelines](https://research.google.com/pubs/pub35650.html)
* [Apache Crunch](https://github.com/cloudera/crunch)