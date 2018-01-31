---
layout: post
title: "Kotlin运算符重载"
categories: Kotlin
tags: [Kotlin, 运算符, 重载]
---

操作符重载是一种语法糖，使得语言的表达更为灵活和优雅。

## 重载算术运算符

### 重载二元算数运算符

下面是一个例子，实现平面坐标系中的两个点的加法。使用了operator关键字声明了plus函数后，就可以使用+号来求和了。

```kotlin
data class Point(val x Int, val y: Int) {
  operator fun plus(other: Point): Point 
  	= Point(x + other.x, y + other.y)
}

>>> val p1 = Point(10, 20)
>>> val p2 = Point(30, 20)
>>> println(p1 + p2)
Point(x=40, y=60)
```

除了将运算符声明为一个成员函数外，也可以定义为一个扩展函数：

```kotlin
operator fun Point.plus(other: Point): Point
  = Point(x + other.x, y + other.y)
```

在在kotlin中效果是一样的，但是Java调用起来会有所不同，前者相当于`p1.plus(p2)`，扩展函数相当于定义了一个静态方法。

Kotlin中不能自定义自己的运算符。Kotlin限定了你能重载哪些运算符，以及你需要在你的类中定义的对应名字的函数：

| 表达式     | 翻译为          |
| ------- | ------------ |
| `a * b` | `a.times(b)` |
| `a / b` | `a.div(b)`   |
| `a % b` | `a.mod(b)`   |
| `a + b` | `a.plus(b)`  |
| `a - b` | `a.minus(b)` |

> Kotlin运算符不支持交换，也就是a * b 不等于 b * a

### 重载复合赋值运算符

当定义`plus`这样的运算符函数时，`kotlin`不止支持+运算符，还支持+=这些**复合赋值运算符**。

```kotlin
>>> var point = Point(1, 2)
>>> point += Point(3, 4)
>>> print(point)
Point(x=4, y=6)
```

`point += Point(3, 4)`相当于`point = point + Point(3, 4)`。

有时候定义+=运算可以修改使用它的变量所引用的对象，但不会重新分配引用。如果定义了一个返回值是Unit，名为plusAssign的函数，Kotlin将会将+=运算符翻译成它。其他二元算数运算符也有类似命名的函数：如minusAssign、timesAssign等。

当在代码中用到+=时，理论上plus和plusAssign都有可能被调用，此时编译器会报错：

```kotlin
data class Point(var x: Int, var y: Int) {
  operator fun plus(other: Point): Point = Point(x + other.x, y + other.y)
  operator fun plusAssign(other: Point) {
    x += other.x
    y += other.y
  }

>>> var Point(1, 2)
>>> point += Point(3, 4) // 编译器报错
```

此时有一种解决方案是用val替换var，这样`plusAssign`就不再适用。但是最佳实践就是在设计新类时就保持**可变性一致**，尽量不要同时给一个类添加plus和plusAssign方法。如果一个类是不可变的，那么就应该只用plus方法；如果一个类是可变的，比如构建器（Builder），那么只需要提供plusAssign方法。

### 重载一元运算符

重载一元运算符的方式与之前相同，用预定义的一个名称来声明函数，并用operator标记。

```kotlin
data class Point(val x: Int, val y: Int) {
  operator fun unaryMinus(): Point
  	= Point(-x, -y)
}

>>> val p = Point(1, 2)
>>> print(-p)
Point(x=-1, y=-2)
```

一元运算符的函数没有任何参数，下表列举了所有可以重载的一元操作符：

| 表达式          | 翻译为              |
| ------------ | ---------------- |
| `+a`         | `a.unaryPlus()`  |
| `-a`         | `a.unaryMinus()` |
| `!a`         | `a.not()`        |
| `++a`, `a++` | `a.inc()`        |
| `—a`,`a—`    | `a.dec()`        |

## 重载比较运算符

### 等号运算符equals

在Kotlin中，`==`和`!=`运算符都会被转换成equals函数的调用。和其他运算符不同，`==`和`!=`可以用于可空运算数，它们会检查运算数是否为null。

对于Point类，因为已经被标记成数据类，equals方法将会由编译器自动生成。也可以手动实现，代码可以是这样：

```kotlin
class Point(val x: Int, val y: Int) {
  override fun equals(other: Any?): Boolean {
    if (other === this) return true
    if (other !is Point) return false
    return other.x == x && other.y == y
  }
}
```

### 排序运算符compareTo

Kotlin中比较运算符（<，>，<=和>=）将会转换成compareTo方法。

有两种定义`compareTo`方法的方式，一种是使用`operator`关键字，另一种是遵照Java规范实现`Comparatble`接口（实际上两者是一样的，因为`Comparable`接口的`compareTo`方法已经被`operator`修饰）。建议实现`Comparable`接口，可以使得代码被相关Java函数中使用。

```kotlin
data class Person(val firstName: String, val lastName: String) : Comparable<Person> {
  override fun compareTo(other: Person): Int
      = compareValuesBy(this, other, Person::firstName, Person::lastName)
}
>>> val p1 = Person("Jack", "Ma")
>>> val p2 = Person("Bruce", "Lee")
>>> println(p1 > p2)
true
```

## 集合与区间的约定

### 通过下标来访问元素

在Kotlin中可以使用Java中数组下标方式来访问集合元素：

```kotlin
val value = map[key]
map[key] = newValue
```

在Kotlin中下标运算符是一个约定。使用下标运算符读取元素会转换成get方法的调用，写入元素将调用set方法。

```kotlin
data class Point(var x: Int, var y: Int) {
  
  operator fun get(index: Int): Int = when (index) {
    0 -> x
    1 -> y
    else ->
      throw IndexOutOfBoundsException("Invalid coordinate $index")
  }

  operator fun set(index: Int, value: Int) = when (index) {
    0 -> x = value
    1 -> y = value
    else ->
      throw IndexOutOfBoundsException("Invalid coordinate $index")
  }

}
>>> val p = Point(1, 2)
>>> println(p[0])
1
>>> p[0] = 3
>>> println(p)
Point(x=3, y=2)
```

注意get、set的索引参数可以是任意类型，而不只是Int。

### 重载in运算符

集合支持的另一个操作符是in运算符，用于检查某个对象是否属于集合，相应的函数叫做`contains`。

```kotlin
data class Line(val p1: Point, val p2: Point) {
  operator fun contains(p: Point): Boolean {
    return (p1.x <= p.x && p.x <= p2.x) && run {
      val ratio1 = (p1.x - p.x).toDouble() / (p1.y - p.y).toDouble()
      val ratio2 = (p2.x - p.x).toDouble() / (p2.y - p.y).toDouble()
      return ratio1 == ratio2
    }
  }
}

>>> val line = Line(Point(1, 1), Point(4, 4))
>>> println(Point(2, 2) in line)
>>> println(Point(2, 3) in line)
>>> println(Point(0, 0) in line)
true
false
false
```

### rangeTo

Kotlin中创建一个区间可以使用`..`语法，比如`1..10`表示`[1, 10]`的区间。`..`运算符是`rangeTo`函数的一个语法糖。

你可以自己定义这个操作符，但是如果实现了`Comparable`接口，就不需要自己实现`rangeTo`方法。

如可以使用`LocalDate`定义一个日期的区间：

```kotlin
>>> val now = LocalDate.now()
>>> val vacation = now..now.plusDays(10)
>>> println(now.plusWeeks(1) in vacation)
true
```

### 在for循环中使用iterator

for循环中也可以使用in，但是和区间检查的in是不同的：它用于迭代。`for (x in list){...}`将会转换成`list.iterator()`的调用，然后就跟Java中一样，在它上面反复调用`hasNext()`和`next()`方法。

```kotlin
>>> for (c in "abc") { ... }
```

也可以自定义`iterator()`方法，实现方式和Java中类似。

## 参考资料

- [Kotlin Reference](https://kotlinlang.org/docs/reference/operator-overloading.html)
- [Kotlin实战](https://book.douban.com/subject/27093660/)

