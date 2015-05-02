---
layout: post
title: Java一段泛型方法的声明
categories: Java
tags: [Java, 泛型]
---

在阅读`java.util.Collections`的源码时看到这样一段泛型方法声明：
```java
public static <T extends Comparable<? super T>> void sort(List<T> list) {
  // 略
}
```
看到这里有点疑问，这中间的`<T extends Comparable<? super T>>`泛型声明为什么不写成`<T extends Comparable<T>>`？

动手做点实验，先声明一个方法：
```java
public static <T extends Comparable<T>> void sort(List<T> list) {}
```
然后来调用一下：
```java
List<Integer> integerList = new ArrayList<>();
sort(integerList);
```

编译一下好像没什么问题。换一个通配符试试：
```java
List<? extends Integer> integerList = new ArrayList<>();
sort(integerList);
```

再编译一下，报错了，看下错误:
```
Test.java:8: 错误: 无法将类 Test中的方法 sort应用到给定类型;
        sort(integerList);
        ^
  需要: List<T>
  找到: List<CAP#1>
  原因: 推断类型不符合等式约束条件
    推断: Integer
    等式约束条件: Integer,CAP#1
  其中, T是类型变量:
    T扩展已在方法 <T>sort(List<T>)中声明的Comparable<T>
  其中, CAP#1是新类型变量:
    CAP#1从? extends Integer的捕获扩展Integer
```

读了下错误就明白了，`CAP#1`代表类型`<? extends Integer>`，此时`sort(List<T>)`方法会检查参数`T`是否满足`<T extends Comparable<T>>`的条件，也就是`<CAP#1 extends Comparable<CAP#1>>`，而作为`Integer`的子类来说，只能保证其子类`CAP#1` *IS-A* `Comparable<Integer>`无法保证`CPA#!` *IS-A* `Comparable<CAP#1>`，因此检查失败，编译出错。




