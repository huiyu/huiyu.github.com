---
layout: post
title: "Linux top命令"
categories: Linux
tags: [Linux, top, 性能调优, 进程, 线程]
---

top命令是Linux下常用的性能分析工具，能够实时显示系统以及其中各个进程的资源占用状况。

## 使用方式

``` shell
top [args]
```

参数列表:

* -b 批处理
* -c 显示完整的控制命令
* -i 忽略失效过程
* -s 保密模式
* -S 累积模式
* -i <time> 设置间隔时间
* -u <user> 指定用户名
* -p <pid> 指定进程
* -n <count> 循环显示次数



## 信息说明

### 系统总体统计信息

前五行是系统整体情况的统计区域：

``` shell
top - 15:55:51 up 22:05,  3 users,  load average: 0.00, 0.00, 0.00
Tasks: 183 total,   1 running, 182 sleeping,   0 stopped,   0 zombie
Cpu(s):  0.3%us,  0.3%sy,  0.0%ni, 99.3%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Mem:   3918912k total,  3649032k used,   269880k free,   116492k buffers
Swap:  2064380k total,    14684k used,  2049696k free,  3009132k cached
```

第一行表示系统概览：

* `15:55:51` 当前系统时间
* `up` 系统运行时间
* `3 users` 当前3个用户登录
* `load average: 0.00, 0.00, 0.00` 1分钟、5分钟、15分钟的系统平均负载

第二行表示系统进程情况，目前有183个进程，1个在运行，182个在休眠，0个暂停，0个僵死

第三行表示CPU使用率：

* `0.3%us`用户态
* `0.3%sy`内核态
* `0.0%ni`优先级变化的进程CPU占比
* `0.0%id`空闲CPU百分比
* `0.0%wa`等待CPU百分比
* `0.0%hi`硬终端（Hardware IRQ）占用CPU百分比
* `0.0%si`软中断（Software IRQ）占用CPU百分比
* `0.0%st`丢失时间（Steal Time）

第四行表示内存使用情况：

* `3918912k total`物理内存总量
* `3649032k used`物理内存使用总量
* `269880k free`物理内存空闲总量
* `116492k buffers`物理内存缓冲区总量

第五行表示交换空间使用情况：

* `2064380k total`交换空间总量
* `14684k used`交换空间使用总量
* `2049696k`交换空间空闲总量
* `3009132k`交换空间缓冲区总量



### 进程信息说明

第六行以下显示进程的状态监控信息：

``` shell
  PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
```

* `PID`进程id
* `USER`进程所有者
* `PR`进程优先级，值越小优先级越高
* `NI`nice值，进程的修正优先级，此时`优先级=PR+NI`
* `RES`进程使用的、未被换出的物理内存大小，单位kb
* `SHR`共享内存大小，单位kb
* `S`进程状态。D=不可中断的睡眠状态 R=运行 S=睡眠 T=跟踪/停止 Z=僵尸进程
* `%CPU`上次更新到现在的CPU时间占用百分比
* `%MEM`进程使用的物理内存百分比
* `COMMAND`进程名称（命令名）



## 交互命令

* h：显示帮助
* k：终止进程
* i：忽略显示限制和僵死进程
* q：退出top程序
* r：重新安排一个进程的优先级
* S：切换到累计模式
* s：改变两次刷新的延迟时间（单位为秒），如果有效数，就换算成ms。默认是5s。
* f/F：从当前现实中添加/删除项目
* o/O：改变显示项目的顺序
* I：Irix Mode开关，Irix模式关闭的话，top会以Solaris Mode运行，此时一个任务的CPU使用率将除以CPU核数
* m：显示或隐藏内存信息
* t：显示或隐藏cpu信息
* c：显示完整命令
* M：根据驻留内存大小进行排序
* P：根据CPU使用百分比大小进行排序
* T：根据时间/累计时间进行排序
* W：将当前设置写入~/.toprc文件中

## 参考资料

* [Top命令](http://www.cnblogs.com/peida/archive/2012/12/24/2831353.html)
* [高效地使用top](http://www.oschina.net/translate/using-top-more-efficiently)