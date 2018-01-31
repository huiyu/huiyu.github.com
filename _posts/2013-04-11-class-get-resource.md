---
layout: post
title: "Class.getResource和ClassLoader.getResource"
categories: Java
tags: [Java, Class, ClassLoader, Resource]
---

Java中经常用到Class.getResource和ClassLoader.getResource这两个方法，它们在取资源时有所区别。

## Class.getResource(String path)

当`path`以"/"开头时，从`Classpath`根路径找资源， 否则从该类`Class`所在包路径下查找资源。

```java
package test;

public class GetResourseTest {
  public static void main(String[] args) {
    // 获取到的是GetResourceTest类所在的test包路径
    GetResourseTest.class.getResource("");
    // 获取到的是Classpath根目录，取决于你的环境
    GetResourseTest.class.getResource("/");
  }
}
```

## ClassLoader.getResource(String path)

当`path`以"/"开头时，总是返回`null`，否则`ClassLoader.getResource(path)`等同于`Class.getResource("/" + path)`

```java
package test;

public class GetResourseTest {
  public static void main(String[] args) {
    // 等同于Class.getResource("/")
    GetResourseTest.class.getClassLoader().getResource("");
    // null
    GetResourseTest.class.getClassLoader().getResource("/");
  }
}
```

## 参考资料
* [Stackoverflow: What is the difference between Class.getResource() and ClassLoader.getResource()?](http://stackoverflow.com/questions/6608795/what-is-the-difference-between-class-getresource-and-classloader-getresource)
