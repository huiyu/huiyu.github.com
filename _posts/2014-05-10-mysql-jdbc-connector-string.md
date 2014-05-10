---
layout: post
title: "MySQL JDBC连接字符串"
summary: ""
categories: Java
tags: [Java, MySQL, 数据库, JDBC]
---

MySQL官方提供的JDBC驱动：

```java
com.mysql.jdbc.Driver
```

格式如下：

```
jdbc:mysql://[host:port]/[database][?参数名1][=参数值1][&参数名2][=参数值2]...
```

几个重要参数如下：

| 参数名称                  | 参数说明                                     | 缺省值   | 最低版本   |
| --------------------- | ---------------------------------------- | ----- | ------ |
| user                  | 数据库用户名（用于连接数据库）                          |       | 所有版本   |
| password              | 用户密码（用于连接数据库）                            |       | 所有版本   |
| useUnicode            | 是否使用Unicode字符集，如果参数characterEncoding设置为gb2312或gbk，本参数值必须设置为true | false | 1.1g   |
| characterEncoding     | 当useUnicode设置为true时，指定字符编码。比如可设置为gb2312或gbk | false | 1.1g   |
| autoReconnect         | 当数据库连接异常中断时，是否自动重新连接？                    | false | 1.1    |
| autoReconnectForPools | 是否使用针对数据库连接池的重连策略                        | false | 3.1.3  |
| failOverReadOnly      | 自动重连成功后，连接是否设置为只读？                       | true  | 3.0.12 |
| maxReconnects         | autoReconnect设置为true时，重试连接的次数            | 3     | 1.1    |
| initialTimeout        | autoReconnect设置为true时，两次重连之间的时间间隔，单位：秒   | 2     | 1.1    |
| connectTimeout        | 和数据库服务器建立socket连接时的超时，单位：毫秒。 0表示永不超时，适用于JDK 1.4及更高版本 | 0     | 3.0.1  |
| socketTimeout         | socket操作（读写）超时，单位：毫秒。 0表示永不超时            | 0     | 3.0.1  |

通常URL可以设置为：

```
jdbc:mysql://localhost:3306/test?user=root&password=root&useUnicode=true&characterEncoding=utf-8&autoReconnect=true&failOverReadOnly=false
```