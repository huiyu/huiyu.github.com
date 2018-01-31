---
layout: post
title: "Kotlin中的let、run、with、apply和also函数"
categories: Kotlin
tags: [Kotlin, Lambda, 函数式]
---

Kotlin中的`standard.kt`提供了一些非常有用的函数（`let`、`run`、`with`、`apply`和`also`）用于处理对象的上下文作用域问题。

## 高阶函数和lambda表达式

首先回顾一下Kotlin中lambda表达式的语法。

```kotlin
{ x: Int, y: Int -> x + y }
```

Lambda表达式总是被一对大括号括起来，大括号里面依次是参数列表、`->`和函数体。函数体的最后一句就是表达式的值。

Kotlin中约定，如果函数调用时最后一个参数是lambda表达式，就可以将表达式放在函数调用的圆括号外面。如下面的语句：

```kotlin
listOf(1, 2, 3).reduce({ x: Int, y: Int -> x + y }) // -> 6
```

等价于：

```kotlin
listOf(1, 2, 3).reduce() { x: Int, y: Int -> x + y }
```

如果lambda是唯一参数，圆括号也可以省略：

```kotlin
listOf(1, 2, 3).reduce { x: Int, y: Int -> x + y }
```

这种将函数作为参数或返回值的函数，就是所谓的高阶函数。`standard.kt`中提供的这些函数实际上就是高阶函数的封装。

## 函数上下文

`standard.kt`提供的函数大同小异，区别在于函数上下文中的`this`、`it`和返回值不同。

以`let`函数为例，默认调用函数的对象作为闭包的it参数，lambda表达式的返回值就是`let`函数的返回值。

```kotlin
class MyClass {
  fun test() = "test".let {
    println(this) // MyClass对象
    println(it)   // "test"
    42            // 返回值
  }
}
```

* `this`表示函数的调用者（kotlin中叫`receiver`），上述场景中表示`MyClass`实例对象，如果`let`不在任何对象里，那么调用`this`会导致编译出错。
* `it`表示调用`let`方法的对象，上述场景中是`"test"`字符串。
* 最后一句表达式的值就是整个`let`函数的值。

将上述代码中的`let`替换，可以得出以下表格，其中标记为`*`的函数表示没有任何调用者，比如`run*`表示是`val result = run { ... } `的调用方式。

| 函数    | this      | it        | 返回值       |
| ----- | --------- | --------- | --------- |
| let   | MyClass对象 | "test"字符串 | 42        |
| run   | "test"字符串 | N\A       | 42        |
| run*  | MyClass对象 | N\A       | 42        |
| with* | "test"字符串 | N\A       | 42        |
| apply | "test"字符串 | N\A       | "test"字符串 |
| also  | MyClass对象 | "test"字符串 | "test"字符串 |

## 参考资料

- [Kotlin Reference](https://kotlinlang.org/docs/reference/operator-overloading.html)
- [Kotlin实战](https://book.douban.com/subject/27093660/)
- [The difference between Kotlin's functions: let, apply,  with, run, and also](https://medium.com/@tpolansk/the-difference-between-kotlins-functions-let-apply-with-run-and-else-ca51a4c696b8)
- [Stackoverflow: What's "receiver" in kotlin](https://stackoverflow.com/questions/45875491/what-is-a-receiver-in-kotlin)