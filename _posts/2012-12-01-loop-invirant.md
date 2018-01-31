---
layout: post
title: "循环不变式"
categories: 算法
tags: [算法, 循环不变式]
---

在编写循环时，找到让每次循环都成立的逻辑表达式很重要。这种逻辑表达式称为循环不变式（loop invariant）。循环不变式相当于用数学归纳法证明的“断言”。

所谓循环不变式有三条性质：
* 初始化：循环的第一次迭代之前不变式为真
* 保持：如果循环的某一次迭代前为真，那么下次迭代后，断言依然为真
* 终止：最后在循环终止时，产生预期的结果


来看实例：
```java
public int sum(int[] a) {
    int sum = 0;
    for (int i = 0; i < a.length; i++)
        sum += a[i];
        
    return sum;
}
```
我们从数学归纳法角度来看这个程序，可以得出以下循环不变式M：

    M(i):  sum等于数组前i个元素之和
    
然后我们将程序加上循环不变式M的注释；

```java
public int sum(int[] a) {
    int sum = 0;
    // M(0)
    for (int i = 0; i < a.length; i++) {
        // M(i)
        sum += a[i];
        // M(i+1)
    }
    // M(a.length)
    return sum;
}
```
* 初始化：`M(0)`等于`0`为真
* 保持：当`M(i)`为真，也就是`sum`为`a[0..i-1]`之和时，下一次迭代后`sum+=a[i]`，也就是`sum=a[0..i]`之和，显然`M(i+1)`为真
* 终止：循环终止条件是`i < a.length`，此时`i=a.length`，因此`M(i)=M(a.length)`满足预期结果（数组`a`求和，也就是求`a`前`a.length`个元素之和）


再来看第二个例子，插入排序：

```java
public <E extends Comparable<? super E>> sort(E[] a) {
    for (int i = 1; i < a.length; i++) {
        E pivot = a[i];
        int j = i - 1;
        while (j >= 0 && less[pivot, a[j]) {
            a[j + 1] = a[j];
            --j;
        }
        a[j] = pivot;
    }
}
```

这里有两层循环，我们这里只分析第一层循环，得出以下循环不变式M：

    M(i)：数组a的前i个元素，都是排好序的，也就是说数组a的子数组a[0..i-1]都是排好序的

让我们来看下插入排序如何满足循环不变式M

* 初始化：在第一次循环之前，`i=1`，子数组`a[0..i-1]`实际上只包含一个元素也就是`a[0]`，显而易见`M(1)`为真
* 保持：假设`M(i)`为真，也就是`a[0..i-1]`是排好序的。非形式化地，语句5~8将`a[0..i-1]`中的大于`pivot`的元素右移动一位。然后将`pivot`元素放入位置`j`，此时`a[0..j-1]`是排好序的且都小于`pivot`，`a[j+1..i]`也是排好序的且大于`pivot`，显然`a[0..i]`是排好序的，`M(i+1)`为真得证。
* 终止：导致循环终止的条件是`i<a.length`，因此退出循环时`i=a.length`。`M(i)`为真得出`M(a.length)`为真，也就是`a[0..a.length-1]`是排好序的，也就是数组`a`是排好序的，满足预期结果！

## 参考资料
* [算法导论（第二版）](http://book.douban.com/subject/1885170/)
* [编程珠玑 - 编写正确的程序](http://book.douban.com/subject/3227098/)
* [图灵社区 - 循环不变式](http://www.ituring.com.cn/article/20305)
* [知乎 - 如何正确的理解循环不变式](https://www.zhihu.com/question/26700198)
