---
layout: post
title: "Java调优工具箱"
categories: "Java"
tags: [Java, JVM, 性能调优, 工具, 垃圾回收]
---

本文介绍了Java的一些命令行工具，不包括一些图形化的工具如jconsole等。这些命令都可以通过`man`命令来查看详细使用说明。

## jps

jps（java process info）用于输出Java进程信息。

### 语法格式

语法格式如下：

```bash
jps [options] [<hostid>]
```

不指定hostid就默认为当前主机或服务器。

### 参数说明

```shell
-q 不输出类名、Jar名和传入main方法的参数
-m 输出传入main方法的参数
-l 输出main类或Jar的全限名
-v 输出传入JVM的参数
-V 输出通过flags文件传入JVM的参数(.hotspotrc或者通过-XX:Flags=<filename>参数指定的文件)
```

### 参考示例

```shell
$ jps -l -m
38835 org.jetbrains.idea.maven.server.RemoteMavenServer
41367 Main -port 9000
41371 sun.tools.jps.Jps -l -m
```

## jinfo

 jinfo（Java Configuration Info）可以输出并修改运行时的java 进程的options。用处比较简单，用于输出JAVA系统参数及命令行参数。

### 语法格式

```
jinfo [ option ] pid
jinfo [ option ] executable core
jinfo [ option ] [ server-id@ ] remote-hostname-or-IP
```

### 参数说明

```shell
-flags    输出命令行参数和系统参数
-sysprops 输出系统properties
```

### 参考示例

如需要查看进程`MaxHeapSize`可以通过以下命令：

```shell
$ jinfo -flag MaxHeapSize 40563
-XX:MaxHeapSize=4294967296
```

命令`jinfo -flags 40563`可以打印所有参数。

## jstack

jstack(Java Stack Trace)主要用于查看某个java进程内的线程堆栈信息。

### 语法格式

语法格式如下：

```shell
jstack [ option ] pid
jstack [ option ] executable core
jstack [ option ] [server-id@]remote-hostname-or-IP
```

### 参数说明

参数选项说明如下：

```shell
-l long listings，会打印出额外的锁信息，在发生死锁时可以用jstack -l pid来观察锁持有情况
-m mixed mode，不仅会输出Java堆栈信息，还会输出C/C++堆栈信息（比如Native方法）
```

### 参考案例

jstack在JVM性能调优当中用的非常多，主要用于排查死锁、CPU消耗过高等问题。[线上性能问题初步排查方法](http://ifeve.com/find-bug-online/)给出了一个jstack使用的具体案例。

jstack打印的线程可能非常多，[JVM内部运行线程介绍](http://ifeve.com/jvm-thread/)介绍了常见的线程。

## jmap

jmap（Java Memory Map）用来查看堆内存使用状况，一般结合jhat（Java Heap Analysis Tool）使用。

### 语法格式

```shell
jmap [ option ] pid
jmap [ option ] executable core
jmap [ option ] [ server-id@ ] remote-hostname-or-IP
```

### 参数说明

- `-dump:[live,]format=b,file=<filename>`

  使用hprof二进制形式,输出jvm的heap内容dump到文件，live子选项是可选的，假如指定live选项,那么只输出活的对象到文件。

- `-finalizerinfo`

  打印正等候回收的对象的信息。

- `-heap`

  打印heap的概要信息，GC使用的算法，heap的配置及wise heap的使用情况。

- `-histo[:live]`

  打印每个class的实例数目，内存占用，类全名信息。VM的内部类名字开头会加上前缀*， 如果live子参数加上后，只统计活的对象数量。

- `-permstat`

  打印classload和jvm heap长久层的信息。包含每个classloader的名字，活泼性，地址，父classloader和加载的class数量。另外，内部String的数量和占用内存数也会打印出来。

### 参考示例

比如以下命令就可以将进程heap内容dump到文件中，然后使用jhat、[MAT](http://www.eclipse.org/mat/)等工具查看。

```shell
$ jmap -dump:format=b,file=dump.out 42860
Dumping heap to /Users/yuhui/Notes/dump.out ...
Heap dump file created
```

## jhat

jhat用于离线分析Java Heap，但是不如MAT之类的工具直观。

### 语法格式

```shell
jhat [ options ] <heap-dump-file>
```

### 参数说明

- `-stack false/true`

  关闭对象分配调用栈跟踪(tracking object allocation call stack)。 如果分配位置信息在堆转储中不可用，则必须将此标志设置为 false. 默认值为 true。

- `-refs false/true`

  关闭对象引用跟踪(tracking of references to objects)。 默认值为 true，默认情况下 返回的指针是指向其他特定对象的对象,如反向链接或输入引用(referrers or incoming references)，会统计/计算堆中的所有对象。

- `-port <port-number>`

  设置 jhat HTTP server 的端口号。 默认值 7000。

- `-J<flag>`

  因为 jhat 命令实际上会启动一个JVM来执行, 通过 -J 可以在启动JVM时传入一些启动参数。例如, -JXmx512m 则指定运行 jhat 的Java虚拟机使用的最大堆内存为 512 MB. 如果需要使用多个JVM启动参数,则传入多个 -Jxxxxxx。

### 参考示例

```shell
$ jhat dump.out
Reading from dump.out...
Dump file created Sat Aug 12 12:44:13 CST 2017
Snapshot read, resolving...
Resolving 4554 objects...
Chasing references, expect 0 dots
Eliminating duplicate references
Snapshot resolved.
Started HTTP server on port 7000
Server is ready.
```

然后就可以访问http://localhsot:7000来查看堆内存信息了。

## jstat

jstat（Java Virtual Machine statistics monitoring tool）用于统计JVM。

### 语法格式

```shell
jstat [ generalOption | outputOptions vmid [ interval [ s|ms ] [ count ] ] ]
```

### 参数说明

- `generalOption`

  单个的常用的命令行选项，如-help，-options, 或 -version。

- `outputOptions`

  一个或多个输出选项，由单个的`statOption`选项组成，可以和-t, -h, and -J等选项配合使用。

- `statOption`

  根据jstat统计的维度不同，可以使用如下表中的选项进行不同维度的统计，不同的操作系统支持的选项可能会不一样，可以通过-options选项，查看不同操作系统所支持选项，如：

  | 选项                                       | 说明                                       |
  | ---------------------------------------- | ---------------------------------------- |
  | [class](http://docs.oracle.com/javase/1.5.0/docs/tooldocs/share/jstat.html#class_option) | 用于查看类加载情况的统计                             |
  | [compiler](http://docs.oracle.com/javase/1.5.0/docs/tooldocs/share/jstat.html#compiler_option) | 用于查看HotSpot中即时编译器编译情况的统计                 |
  | [gc](http://docs.oracle.com/javase/1.5.0/docs/tooldocs/share/jstat.html#gc_option) | 用于查看JVM中堆的垃圾收集情况的统计                      |
  | [gccapacity](http://docs.oracle.com/javase/1.5.0/docs/tooldocs/share/jstat.html#gccapacity_option) | 用于查看新生代、老生代及持久代的存储容量情况                   |
  | [gccause](http://docs.oracle.com/javase/1.5.0/docs/tooldocs/share/jstat.html#gccause_option) | 用于查看垃圾收集的统计情况（这个和-gcutil选项一样），如果有发生垃圾收集，它还会显示最后一次及当前正在发生垃圾收集的原因。 |
  | [gcnew](http://docs.oracle.com/javase/1.5.0/docs/tooldocs/share/jstat.html#gcnew_option) | 用于查看新生代垃圾收集的情况                           |
  | [gcnewcapacity](http://docs.oracle.com/javase/1.5.0/docs/tooldocs/share/jstat.html#gcnewcapacity_option) | 用于查看新生代的存储容量情况                           |
  | [gcold](http://docs.oracle.com/javase/1.5.0/docs/tooldocs/share/jstat.html#gcold_option) | 用于查看老生代及持久代发生GC的情况                       |
  | [gcoldcapacity](http://docs.oracle.com/javase/1.5.0/docs/tooldocs/share/jstat.html#gcoldcapacity_option) | 用于查看老生代的容量                               |
  | [gcpermcapacity](http://docs.oracle.com/javase/1.5.0/docs/tooldocs/share/jstat.html#gcpermcapacity_option) | 用于查看持久代的容量                               |
  | [gcutil](http://docs.oracle.com/javase/1.5.0/docs/tooldocs/share/jstat.html#gcutil_option) | 用于查看新生代、老生代及持代垃圾收集的情况                    |
  | [printcompilation](http://docs.oracle.com/javase/1.5.0/docs/tooldocs/share/jstat.html#printcompilation_option) | HotSpot编译方法的统计                           |

- -h n

  用于指定每隔几行就输出列头，如果不指定，默认是只在第一行出现列头。

- -JjavaOption

  用于将给定的*javaOption*传给java应用程序加载器，例如，“-J-Xms48m”将把启动内存设置为48M。如果想查看可以传递哪些选项到应用程序加载器中，可以相看如下的文档：

- Linux and Solaris：<http://docs.oracle.com/javase/1.5.0/docs/tooldocs/solaris/java.html>

- Windows： <http://docs.oracle.com/javase/1.5.0/docs/tooldocs/windows/java.html>

- -t n

  用于在输出内容的第一列显示时间戳，这个时间戳代表的时JVM开始启动到现在的时间（注：在IBM JDK5中是没有这个选项的）。

### 统计维度说明

不同统计维度（`statOption`）输出说明如下：

- `-class` 类加载情况统计

  | 列名       | 说明            |
  | -------- | ------------- |
  | Loaded   | 加载了的类的数量      |
  | Bytes    | 加载了的类的大小，单为Kb |
  | Unloaded | 卸载了的类的数量      |
  | Bytes    | 卸载了的类的大小，单为Kb |
  | Time     | 花在类的加载及卸载的时间  |

- `-compile` JIT编译器ing看的统计

  | 列名           | 说明              |
  | ------------ | --------------- |
  | Compiled     | 编译任务执行的次数       |
  | Failed       | 编译任务执行失败的次数     |
  | Invalid      | 编译任务非法执行的次数     |
  | Time         | 执行编译花费的时间       |
  | FailedType   | 最后一次编译失败的编译类型   |
  | FailedMethod | 最后一次编译失败的类名及方法名 |


- `-gc` 垃圾回收情况的统计

  | 列名   | 说明                                       |
  | ---- | ---------------------------------------- |
  | S0C  | 新生代中Survivor space中S0当前容量的大小（KB）         |
  | S1C  | 新生代中Survivor space中S1当前容量的大小（KB）         |
  | S0U  | 新生代中Survivor space中S0容量使用的大小（KB）         |
  | S1U  | 新生代中Survivor space中S1容量使用的大小（KB）         |
  | EC   | Eden space当前容量的大小（KB）                    |
  | EU   | Eden space容量使用的大小（KB）                    |
  | OC   | Old space当前容量的大小（KB）                     |
  | OU   | Old space使用容量的大小（KB）                     |
  | PC   | Permanent space当前容量的大小（KB）               |
  | PU   | Permanent space使用容量的大小（KB）               |
  | YGC  | 从应用程序启动到采样时发生 Young GC 的次数               |
  | YGCT | 从应用程序启动到采样时 Young GC 所用的时间(秒)            |
  | FGC  | 从应用程序启动到采样时发生 Full GC 的次数                |
  | FGCT | 从应用程序启动到采样时 Full GC 所用的时间(秒)             |
  | GCT  | T从应用程序启动到采样时用于垃圾回收的总时间(单位秒)，它的值等于YGC+FGC |

- `-gccapacity` 新生代、老生代及持久代的存储容量情况

  | 列名    | 说明                              |
  | ----- | ------------------------------- |
  | NGCMN | 新生代的最小容量大小（KB）                  |
  | NGCMX | 新生代的最大容量大小（KB）                  |
  | NGC   | 当前新生代的容量大小（KB）                  |
  | S0C   | 当前新生代中survivor space 0的容量大小（KB） |
  | S1C   | 当前新生代中survivor space 1的容量大小（KB） |
  | EC    | Eden space当前容量的大小（KB）           |
  | OGCMN | 老生代的最小容量大小（KB）                  |
  | OGCMX | 老生代的最大容量大小（KB）                  |
  | OGC   | 当前老生代的容量大小（KB）                  |
  | OC    | 当前老生代的空间容量大小（KB）                |
  | PGCMN | 持久代的最小容量大小（KB）                  |
  | PGCMX | 持久代的最大容量大小（KB）                  |
  | PGC   | 当前持久代的容量大小（KB）                  |
  | PC    | 当前持久代的空间容量大小（KB）                |
  | YGC   | 从应用程序启动到采样时发生 Young GC 的次数      |
  | FGC   | 从应用程序启动到采样时发生 Full GC 的次数       |

- `-gccause` 这个选项用于查看垃圾收集的统计情况（这个和-gcutil选项一样），如果有发生垃圾收集，它还会显示最后一次及当前正在发生垃圾收集的原因，它比**-gcutil**会多出最后一次垃圾收集原因以及当前正在发生的垃圾收集的原因。

  | 列名   | 说明                                       |
  | ---- | ---------------------------------------- |
  | LGCC | 最后一次垃圾收集的原因，可能为“unknown GCCause”、“System.gc()”等 |
  | GCC  | 当前垃圾收集的原因                                |

- `-gcnew` 新生代垃圾收集的情况

  | 列名   | 说明                                       |
  | ---- | ---------------------------------------- |
  | S0C  | 当前新生代中survivor space 0的容量大小（KB）          |
  | S1C  | 当前新生代中survivor space 1的容量大小（KB）          |
  | S0U  | S0已经使用的大小（KB）                            |
  | S1U  | S1已经使用的大小（KB）                            |
  | TT   | Tenuring threshold，要了解这个参数，我们需要了解一点Java内存对象的结构，在Sun JVM中，（除了数组之外的）对象都有两个机器字（words）的头部。第一个字中包含这个对象的标示哈希码以及其他一些类似锁状态和等标识信息，第二个字中包含一个指向对象的类的引用，其中第二个字节就会被垃圾收集算法使用到。在新生代中做垃圾收集的时候，每次复制一个对象后，将增加这个对象的收集计数，当一个对象在新生代中被复制了一定次数后，该算法即判定该对象是长周期的对象，把他移动到老生代，这个阈值叫着tenuring threshold。这个阈值用于表示某个/些在执行批定次数youngGC后还活着的对象，即使此时新生的的Survior没有满，也同样被认为是长周期对象，将会被移到老生代中。 |
  | MTT  | Maximum tenuring threshold，用于表示TT的最大值。   |
  | DSS  | Desired survivor size (KB).可以参与这里：http://blog.csdn.net/yangjun2/article/details/6542357 |
  | EC   | Eden space当前容量的大小（KB）                    |
  | EU   | Eden space已经使用的大小（KB）                    |
  | YGC  | 从应用程序启动到采样时发生 Young GC 的次数               |
  | YGCT | 从应用程序启动到采样时 Young GC 所用的时间(单位秒)          |

- `-gcnewcapacity`新生代的存储容量情况

  | 列名    | 说明                         |
  | ----- | -------------------------- |
  | NGCMN | 新生代的最小容量大小（KB）             |
  | NGCMX | 新生代的最大容量大小（KB）             |
  | NGC   | 当前新生代的容量大小（KB）             |
  | S0CMX | 新生代中SO的最大容量大小（KB）          |
  | S0C   | 当前新生代中SO的容量大小（KB）          |
  | S1CMX | 新生代中S1的最大容量大小（KB）          |
  | S1C   | 当前新生代中S1的容量大小（KB）          |
  | ECMX  | 新生代中Eden的最大容量大小（KB）        |
  | EC    | 当前新生代中Eden的容量大小（KB）        |
  | YGC   | 从应用程序启动到采样时发生 Young GC 的次数 |
  | FGC   | 从应用程序启动到采样时发生 Full GC 的次数  |

- `gcold`老生代及持久代发生GC的情况

  | 列名   | 说明                                      |
  | ---- | --------------------------------------- |
  | PC   | 当前持久代容量的大小（KB）                          |
  | PU   | 持久代使用容量的大小（KB）                          |
  | OC   | 当前老年代容量的大小（KB）                          |
  | OU   | 老年代使用容量的大小（KB）                          |
  | YGC  | 从应用程序启动到采样时发生 Young GC 的次数              |
  | FGC  | 从应用程序启动到采样时发生 Full GC 的次数               |
  | FGCT | 从应用程序启动到采样时 Full GC 所用的时间(单位秒)          |
  | GCT  | 从应用程序启动到采样时用于垃圾回收的总时间(单位秒)，它的值等于YGC+FGC |

- `gcoldcapacity`老生代的存储容量情况

  | 列名    | 说明                                      |
  | ----- | --------------------------------------- |
  | OGCMN | 老生代的最小容量大小（KB）                          |
  | OGCMX | 老生代的最大容量大小（KB）                          |
  | OGC   | 当前老生代的容量大小（KB）                          |
  | OC    | 当前新生代的空间容量大小（KB）                        |
  | YGC   | 从应用程序启动到采样时发生 Young GC 的次数              |
  | FGC   | 从应用程序启动到采样时发生 Full GC 的次数               |
  | FGCT  | 从应用程序启动到采样时 Full GC 所用的时间(单位秒)          |
  | GCT   | 从应用程序启动到采样时用于垃圾回收的总时间(单位秒)，它的值等于YGC+FGC |

- `gcpermcapacity`持久代的存储容量情况

  | 列名    | 说明                                      |
  | ----- | --------------------------------------- |
  | PGCMN | 持久代的最小容量大小（KB）                          |
  | PGCMX | 持久代的最大容量大小（KB）                          |
  | PGC   | 当前持久代的容量大小（KB）                          |
  | PC    | 当前持久代的空间容量大小（KB）                        |
  | YGC   | 从应用程序启动到采样时发生 Young GC 的次数              |
  | FGC   |                                         |
  | FGCT  | 从应用程序启动到采样时 Full GC 所用的时间(单位秒)          |
  | GCT   | 从应用程序启动到采样时用于垃圾回收的总时间(单位秒)，它的值等于YGC+FGC |

- `gcutil`新生代、老生代及持代垃圾收集的情况

  | 列名   | 说明                                      |
  | ---- | --------------------------------------- |
  | S0   | Heap上的 Survivor space 0 区已使用空间的百分比      |
  | S1   | Heap上的 Survivor space 1 区已使用空间的百分比      |
  | E    | Heap上的 Eden space 区已使用空间的百分比            |
  | O    | Heap上的 Old space 区已使用空间的百分比             |
  | P    | Perm space 区已使用空间的百分比                   |
  | YGC  | 从应用程序启动到采样时发生 Young GC 的次数              |
  | YGCT | 从应用程序启动到采样时 Young GC 所用的时间(单位秒)         |
  | FGC  | 从应用程序启动到采样时发生 Full GC 的次数               |
  | FGCT | 从应用程序启动到采样时 Full GC 所用的时间(单位秒)          |
  | GCT  | 从应用程序启动到采样时用于垃圾回收的总时间(单位秒)，它的值等于YGC+FGC |

- `printcomplilation`JIT编译信息统计

  | 列名       | 说明                                       |
  | -------- | ---------------------------------------- |
  | Compiled | 编译任务执行的次数                                |
  | Size     | 方法的字节码所占的字节数                             |
  | Type     | 编译类型                                     |
  | Method   | 指定确定被编译方法的类名及方法名，类名中使名“/”而不是“.”做为命名分隔符，方法名是被指定的类中的方法，这两个字段的格式是由HotSpot中的“-**XX:+PrintComplation**”选项确定的。 |

### 参考示例

- 查看gc情况

```shell
$ jstat -gcutil 49488
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
  0.00   0.00   4.00   0.00  17.28  19.75      0    0.000     0    0.000    0.000
```

- 加载类情况

```shell
$ jstat -class 49488
Loaded  Bytes  Unloaded  Bytes     Time
   416   856.3        0     0.0       0.04
```

- JIT编译情况

```shell
$ jstat -compiler 49488
Compiled Failed Invalid   Time   FailedType FailedMethod
      15      0       0     0.00          0
```