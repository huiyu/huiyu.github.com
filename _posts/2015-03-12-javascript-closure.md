---
layout: post
title: "Javascript作用域和闭包"
summary: ""
categories: Javascript
tags: [Javascript, 作用域, 闭包, 函数式]
---

# Javascript作用域和闭包

## 作用域

- Javascript只有函数作用域
- 其他花括号，如循环语句for、while，和判断语句if等不创造新的作用域

## 执行环境

当JS执行时构建的用于保存变量和变量值的存储系统。

## 闭包

> 闭包（Closure）或者词法闭包（Lexical  Closure）或者函数闭包（function closure），是引用了自用变量的函数。当这个被引用的自由变量将和这个函数一同存在，即时离开创造它的环境也不例外。

## this

JS中，函数中的变量只有执行的时候才会被绑定：

```javascript
var fn = function(one, two) {
  log(one, two);
};

fn(1,2);
```

如果不调用`fn`，那么变量one和two永远不会绑定到具体的对象中。this关键字也是如此。

this绑定的是调用函数的那个对象：

```javascript
var obj = {
  fn: function() {
    log(this);
  }
}

obj.fn(); // obj调用了fn，所以绑定的是fn
```


```javascript
var fn = function(one, two) {
  log(this, one, two);
}

var r = {};
r.method = fn;

// this是指自己
r.method(1, 2); 
// 这种调用方式跟上面完全一样
r['method'](1,2); 

// 如果直接调用，那么this绑定的就是全局对象，在浏览器环境中就是window对象
fn(1, 2); 

// call的第一个参数是指定调用函数的对象
fn.call(r, 1, 2); 
// apply也是如此
fn.apply(r, [1,2]); 

// 回调函数中的fn绑定还是要看它是怎么调用的
// setTimeout中的调用方式还是自有函数调用，因此this绑定的是global对象
setTimeout(fn, 1000)；

// 这种方式跟上面的没有任何区别，这里的.不是函数调用，而是访问对象属性
setTimeout(r.method, 1000);

// 默认绑定为全局对象，新标准已经移除这种方式
console.log(this);
```
