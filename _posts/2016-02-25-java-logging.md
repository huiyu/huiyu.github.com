---
layout: post
title: "Java日志框架体系"
categories: "Java"
tags: [Java, 日志, Facade]
---

日志在程序开发中起到非常重要的作用。除了JDK自带的`java.util.logging`外，Java世界中还有各种不同的日志框架，这些框架之间还存在着依赖和关联，这里对这些日志框架做一个总结。

## 日志体系

常用的日志框架主要分成两类，一种是日志的门面（facade）框架，另外一种是具体的日志实现。

门面框架主要有：
* slf4j
* apache commons logging(原jakarta commons logging，JCL)

具体日志实现有：
* java.util.logging
* log4j
* log4j2
* logback

一般在项目中不直接使用具体的日志实现，而使用门面（facade）框架。日志门面框架提供了一套独立于具体日志框架实现的API，应用程序通过使用这些独立的API就能够实现与具体日志框架的解耦，这跟JDBC是类似的。最早的日志门面接口是commons-logging，但目前最受欢迎的是slf4j。

## SLF4J桥接

SLF4J不依赖于特定的类加载机制，因此没有遭受类加载问题或者Apache Commons Logging (JCL)观察到的内存泄露问题。事实上，每个SLF4J绑定在编译时硬连接来使用一个指定的日志框架。比如说， slf4j-log4j12-1.7.19.jar在编译时绑定使用log4j。在你的代码中，除  slf4j-api-1.7.19.jar之外，只能有一个你选择的绑定 到正确的class path 路径上。不要在class path 放置多个绑定。下面是一个说明图表：

![concrete-bindings.png]({{ site.baseurl | append: "/images/slf4j-bridging.png"}})

* 如果在classpath中没有发现具体的绑定，SLF4J将默认一个无操作的实现（输出到/dev/null）。
* slf4j-nop.jar：也是将日志全部输出到/dev/null，区别是没有警告信息。
* logback-classic（logback-core.jar）：将slf4j绑定至logback。Logback完全实现了slf4j接口（作者是同一人）。Logback的[ch.qos.logback.classic.Logger](http://logback.qos.ch/apidocs/ch/qos/logback/classic/Logger.html)类实现了SLF4J的[org.slf4j.Logger](http://www.slf4j.org/apidocs/org/slf4j/Logger.html)接口。
* slf4j-log4j12.jar：将slf4j绑定到[log4j version 1.2](http://logging.apache.org/log4j/1.2/index.html)，要求classpath下有log4j.jar。
* slf4j-jdk14.jar：将slf4j绑定至`java.util.logging`上。
* slf4j-simple.jar：简单实现，将所有日志输出到`System.err`上。
* slf4j-jcl-1.7.25.jar：将日志输出到apache commons logging。

切换slf4j的绑定十分简单，只在classpath下更换相应的jar包即可。

> 注意图中的adaptation layer起得就是桥接器作用，将具体日志api转换成slf4j的api。所谓的natvie implementation就是直接实现了slf4j接口，没有转换这一层。

## SLF4J反向桥接

slf4j还提供了另外一类的桥接器，将具体日志框架的API转调到slf4j的api上，sl4j称之为[Bridging legacy APIs](https://www.slf4j.org/legacy.html)，作用主要是为了处理一些遗留代码。

![legacy.png]({{ site.baseurl | append: "/images/slf4j-bridging-legacy.png" }})

上图是slf4j提供的三种能从别的日志框架API转回slf4j的三种情形。以左上角的第一种情形为例，当slf4j底层桥接到logback框架的时候，上层允许log4j和jul两种具体日志框架反向桥接回slf4j。还有jcl虽然不是什么日志框架的具体实现，但是它的API仍然是能够被转调回slf4j的。要想实现转调，方法就是图上列出的用特定的桥接器jar**替换**掉原有的日志框架jar。

几乎所有其他日志框架的API，包括jcl的API，都能够随意的反向桥接回slf4j。但是有一个唯一的限制就是反向桥接回slf4j的日志框架不能跟slf4j当前桥接的日志框架相同。这个限制就是为了防止A-to-B.jar跟B-to-A.jar同时出现在类路径中，从而导致A和B一直不停地互相递归调用，最后堆栈溢出。常见的堆栈溢出情形主要有两对：log4j-over-slf4j和slf4j-log4j12，以及jcl-over-slf4j.jar和slf4j-jcl.jar。目前这个限制并不是通过技术保证的，仅仅靠开发者自己保证，这也是为什么slf4j官网上要强调所有合理的方式只有上图的三种情形。

## 参考资料
* [SLF4J user manual](https://www.slf4j.org/manual.html)
* [SLF4J: Bridging legacy APIs](https://www.slf4j.org/legacy.html)
* [Apache Commons Logging](https://commons.apache.org/proper/commons-logging/guide.html)
* [log4j-over-slf4j与slf4j-log4j12共存stack overflow异常分析](http://blog.csdn.net/kxcfzyk/article/details/38613861)