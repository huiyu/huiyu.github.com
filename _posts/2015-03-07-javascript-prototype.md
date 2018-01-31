---
layout: post
title: "Javascript原型链和继承"
summary: ""
categories: Javascript
tags: [Javascript, OOP, 原型链]
---

## 创建对象

### 简单创建对象

创建对象的最简单方式就是创建一个Object的实例：

```javascript
var person = new Object();
person.name = "Jack";
person.age = 20;
person.sayName = function() {
  console.log(this.name);
};
```

可以通过为对象字面量模式创建，这种方式跟上面是等价的：

```javascript
var person = {
  name: "Jack",
  age: 20;
  sayName: function() {
    console.log(this.name);
  }
};
```

### 工厂方法模式

虽然Object构造函数和对象字面量都可以创建单个对象，但是这些方法并**不具备重用性**，在创建对个对象时，会产生大量的重复代码。

工厂方法模式是常见的一种设计模式，抽象了创建具体对象的过程：

```javascript
function = createPerson(name, age) {
  var o = new Object();
  o.name = name;
  o.age = age;
  o.sayName = function() {
    console.log(this.name);
  };
}

var person = createPerson("Jack", 20);
```

### 构造函数模式

**工厂方法模式解决了创建多个同样类型对象的问题，但是并没有解决对象识别的问题（如何知道一个对象的类型）**。

下面的模式称为构造函数模式（有时候也称为伪类模式）：

```javascript
function Person(name, age) {
  this.name = name;
  this.age = age;
  this.sayName = function() {
    console.log(this.name);
  };
};

var person = new Person("Jack", 20);
```

在这里，`Person`函数替代了`createPerson`函数，并有以下不同之处：

- 没有显式地创建对象
- 直接将属性和方法赋给了`this`
- 没有`return`语句

此外，`Person`使用了一个大写字母P。这是惯例，构造函数始终应该一个大写字母开头，非构造函数应该以一个小写字母开头。

要用构造函数创建`Person`实例，必须使用`new`操作符，这种方式实际上会有四个步骤：

- 创建一个新对象
- 将构造函数的作用域赋给新对象（`this`指向了这个新对象）
- 执行构造函数中的代码（为这个新对象添加属性）
- 返回新对象

此时`person`对象的构造函数指向`Person`：

```javascript
console.log(person.constructor == Person); // true
```

此时`person`对象是`Person`的一个实例：

```javascript
console.log(person instanceOf Object); // true
console.log(person instanceOf Person); // true
```

构造函数与其他函数的唯一区别在于调用方式的不同。但是构造函数**同时也是函数**，不存在任何特殊语法。构造函数也可以不通过`new`关键字调用，但是就产生不了我们期待的结果：

```javascript
var person = Person("Jack", 20);

// person的值为undefined，因为Person函数没有返回任何对象
person.name; // 报错
person.age; // 报错
person.sayName(); // 报错

// 那么这些属性哪去了呢？
// 实际上绑定到了全局对象里了，在浏览器环境下是window对象
// 实际上是全局环境在调用Person方法
console.log(name); // Jack
console.log(age); // 20
sayName(); // Jack
window.sayName(); // Jack
```

构造函数虽然已经很好用了，但是还是有点问题。每一个方法都要在**每个实例上重新创建一遍**。实际上下面两种方式是等同的：

```javascript
this.sayName() = function() {console.log(this.name);}
this.sayName() = new Function() {consle.log(this.name);}
```

### 原型模式

我们创建的每一个函数都有一个`prototype`属性。这个属性是一个指针，指向一个对象，这个对象的属性和方法由这个函数创建的所有对象所共享。

```javascript
function Person(name, age) {
  this.name = name;
  this.age = age;
}

Person.prototype.sayName = function() {
  console.log(this.name);
};

var person = new Person("Jack", 20);
person.sayName(); // Jack
```

#### 理解原型对象

无论什么时候，只要创建一个新函数，就会为该函数创建一个`prototype`属性，这个属性指向函数的原型对象。**在默认情况下，所有的原型对象会自动获得一个`constructor`属性，这个属性包含一个指向`prototype`属性所在函数的指针**。比如`Person`函数，`Person.prototype.constructor`指向`Person`自身。

创建了自定义的构造函数后，其原型对象默认只会取得`constructor`属性，其他属性则完全从Object继承。当调用构造函数创建一个新实例后，该实例的内部将包含一个指针（内部属性）指向构造函数的原型对象。ECMA-262第五版中管这个指针叫`[[Prototype]]`。虽然标准没有定义如何访问`[[Prototype]]`，但是Firefox、Safari和Chrome在每个对象上都支持`__proto__`。这里要说明的是，这个**连接关系存在于实例和构造函数的原型对象之间，而不是实例和构造函数之间**。

虽然不是所有实现都可以访问到`[[Prototype]]`，但是可以通过`isPrototypeOf`方法来确定是否存在这种关系：

```javascript
Person.prototype.isPrototypeOf(person); // true
```

ECMAScript 5新增了一个方法`Object.getPrototypeOf`，这个方法返回`[[Prototype]]`的值：

```javascript
Object.getPrototypeOf(person) == Person.prototype; // true
Object.getPrototypeOf(person).sayName; // sayName方法
```

每当程序在访问某个对象的属性时，都会先搜索对象实例本身。如果在实例中找到了给定名字的属性，那么就返回该属性。否则就继续搜索`[[Prototype]]`指针指向的原型对象，如果在原型对象中找到了该属性，那么就返回该属性的值。

使用`hasOwnProperty`方法可以检测一个属性是存在于实例中，还是存在于原型中。

```javascript
person.hasOwnProperty("name"); // true
person.hasOwnProperty("sayName"); // false
```

#### in操作符

in操作符可以用于检测对象是否存在某属性，无论该属性存在于对象实例中还是在原型中。

```javascript
"name" in person; // true
"sayHello" in person; // true
```

in操作符合`hasOwnProperty`可以组合使用，判断属性是对象属性还是原型属性。

此外，还可以通过for-in循环访问对象所有可访问的属性：

```javascript
for (var p in person) {
  console.log(p);
}

---
输出：
name
age
sayName
```

要去的对象上所有的**实例**属性，可以使用ECMAScript5的`Object.keys()`方法。该方法接受一个对象作为参数，返回一个包含所有实例属性的字符串数组。

```javascript
Object.keys(person); // ["name", "age"]
```

如果要在结果中包含不可枚举的`constructor`属性，那么可以用`Object.getOwnPropertyNames()`：

```javascript
Object.getOwnPropertyNames(Person.prototype);// ["constructor", "sayName"]
```

#### 更简洁的原型语法

每添加一个属性和方法都要敲一遍`Person.prototype`，这里可以用**对象字面量**来重写整个原型对象：

```javascript
Person.prototype = {
  sayName: function() {
    console.log(this.name);
  }
};
```

但是这样有一个问题，`constructor`属性不在指向`Person`。此时`instanceOf`关键词判断仍然为`true`，但是`constructor`此时等于`Object`而不是`Person`。如果`constructor`很重要，可以像下面这样特地设回来：

```javascript
Person.prototype = {
  constructor: Person,
  sayName: function() {
    console.log(this.name);
  }
};
```

但是这样会导致一个新的问题，原生的`constructor`是不可枚举的，如果你使用兼容ECMAScript5的引擎，可以试一试`Object.defineProperty`方法：

```javascript
Person.prototype = {
  constructor: Person,
  sayName: function() {
    console.log(this.name);
  }
};
// 重设构造函数
Object.defineProperty(Person.prototype, "constructor", {
  enumerable: false,
  value: Person
});
```

#### 原型的动态性

由于在原型中查找值是一次搜索过程，因此对原型对象所做的任何修改都能够立即反映到实例上，即时是先创建实例后修改了原型也照样如此。

```java
var friend = new Person();

// 添加新方法
Person.prototype.sayHi = function() {
  console.log("hi")
};

friend.sayHi(); // hi
```

#### 动态原型模式

其他OO语言的开发人员在看到构造函数和原型各自独立时，会感到十分困惑。动态原型模式正是致力于解决合格问题：

```javascript
function Person(name, age) {
  this.name = name;
  this.age = age;
  
  if (typeof this.sayName != 'function') {
    Person.prototype.sayName = function() {
      console.log(this.name);
    };
  }
};
```

这是在`sayName`方法不存在的情况下才初始化原型。这种方法可以说十分完美，而且不需要用if检查每个原型属性和方法，**只需要检查其中一个即可**。

#### 寄生构造函数模式

在前述几种模式都不适用的情况下，可以采用寄生（parastic）构造函数模式。这种模式时创建一个函数，该函数仅仅是封装创建对象的代码，然后再返回新创建的对象。从表面上看很像构造函数，本质上其实是一个工厂函数。

```javascript
function Person(name, age) {
  var o = new Object();
  o.name = name;
  o.age = age;
  o.sayName = function() {
    console.log(this.name);
  };
  return o;
}

var person = new Person("Jack", 20);
person.sayName(); // "Jack"
```

#### 稳妥构造函数

```javascript
function Person(name, age) {
  var o = new Object();
  o.sayName = function() {
    console.log(name);
  }
  return o;
}
```

以这种模式创建的对象中，除了使用`sayName`外，没有任何其他办法来访问`name`属性。这种方式提供的安全性，使得它非常适合一些安全执行环境。

## 原型继承

继承是面向对象语言中的一个重要概念。传统的面向对象语言支持两种方式的继承：接口继承和实现继承。在ECMAScript中无法实现接口继承，只支持实现继承，并且是通过原型链来实现的。

### 原型链

实现原型链有一种基本模式：

```javascript
function Parent() {
  this.property = true;
}

Parent.prototype.getProperty = function() {
  return this.property;
}

function Child() {
  this.property = false;
}
// 继承了Parent
Child.prototype = new Parent();

var child = new Child();
child.getProperty();
```

实现的本质是重写原型对象，代之以一个新类型的实例。

有两种方式可以确定原型和实例之间的关系，第一种方式是通过`instanceof`操作符，还有一种是通过`isPrototypeOf`方法，两者都可以达到目的。

```javascript
child instanceof Object; // true
child instanceof Parent; // true
child instanceof Child;  // true

Object.prototype.isPrototypeOf(child); // true
Parent.prototype.isPrototypeOf(child); // true
Child.prototype.isPrototypeOf(child);  // true
```

原型链虽然很强大，但是也存在一些问题。最主要的问题是包含引用类型的原型。在通过原型来实现继承时，子类原型实际上是父类的实例。**那么父类的实例属性就顺理成章地成为了子类的原型属性**。

```javascript
function Parent() {
  this.colors = ["red", "blue"];
}

function Child() {}

// 继承了Parent
Child.prototype = new Parent();

var child1 = new Child();
child1.colors.push("green");
console.log(child1.colors); // ["red", "blue", "green"]

var child2 = new Child();
console.log(child2.colors); // ["red", "blue", "green"]
```

此外原型链的另外一个问题是，在创建子类的实例时，不能向超类的构造函数中传递参数。准确的说，是没有办法在不影响所有对象实例的情况下，给超类的构造函数传递参数。因此综上两个原因，实践中很少会单独使用原型链。

### 借用构造函数

为了解决原型链中引用类型继承的问题，开发人员开始使用一种叫做**借用构造函数（constructor stealing）**的技术（也叫伪造对象或经典继承）。这种技术的基本思想非常简单，就是在子类型的构造函数内部调用父类的构造函数。

```javascript
function Parent() {
  this.colors = ["red","blue"];
}

function Child() {
  // 继承了Parent
  Parent.call(this);
}
```

相对原型链而言，借用构造函数有一个很大的优势，既可以在子类构造函数中向超类构造函数传递参数。

```javascript
function Parent(name) {
  this.name = name;
}

function Child() {
  Parent.call(this, "Jack");
}
```

但是，借用构造函数也有问题：

- 借用构造函数无法避免构造函数模式固有的问题，也就是每个函数都在子类的构造函数中重新定义了一遍，函数复用无从谈起。
- 在超类原型中定义的方法，子类也不可见。
- `instanceOf`和`isPrototypeOf`无法识别借用构造函数创建的子类对象。

综上在实践中也很少单独使用借用构造函数。

### 组合继承

组合继承（combination inheritance），也称伪经典继承，指的是将原型链和借用构造函数技术组合到一起。思路是使用原型链实现对原型属性的方法的继承，而通过借用构造函数来实现对实例属性的继承。这样即通过原型上定义方法实现了函数复用，又能够保证每个实例都有自己的属性：

```javascript
function Parent(name) {
  this.name = name;
  this.colors = ["red", "blue"];
}

Parent.prototype.sayName = function() {
  console.log(this.name);
};

function Child(name, age) {
  // 继承属性
  Parent.call(this, name);
  this.age = age;
};

// 继承方法
Child.prototype = new Parent();
Child.prototype.constructor = Child;

// 子类定义方法
Child.prototype.sayAage = function() {
  console.loog(this.age);
}
```

> `Child.prototype.constructor = Child`这里属于一个惯例。constructor属性本身没什么用，是JavaScript语言设计的历史遗留物。由于constructor属性是可以变更的，所以未必指向对象的构造函数。不过从编程习惯上，我们应该尽量让对象的construcotr指向其构造函数，以维持这个惯例。

组合继承避免了原型链和借用构造函数的缺陷，融合了它们的有点，成为JavaScript中最常见的继承模式。并且`instanceOf`和`isPrototypeOf`也能够识别基于组合继承创建的对象。

### 原型式继承

[Douglas Crockford](http://www.crockford.com/)介绍了一种实现继承的方法，这种方法并没有使用严格意义上的构造函数。他的想法是借助原型可以基于已有对象创建新对象，不必因此自定义类型。为了达到这个目的，他给出如下函数：

```javascript
function object(o) {
  function F(){}
  F.prototype = o;
  return new F();
}
```

在`object()`函数内部，先创建了一个临时性的构造函数，然后将传入的对象作为这个构造函数的原型，最后返回了这个临时类型的一个新实例。本质上`object()`函数对每一个传入的对象进行了浅复制：

```javascript
var person = {
  name: "Jack",
  friends: ["Van", "Jeff"]
};

var anotherPerson = object(person);
anotherPerson.name = "Greg";
anotherPerson.friends.push("Rob");

console.log(anotherPerson.friends);
```

ECMAScript5通过增加Object.create()方法来规范原型式继承。这个方法接受两个参数，一个用作新对象原型的对象和（可选的）一个为新对象定义额外属性的对象。

```javascript
var person = {
  name: "Jack",
  friends: ["Van", "Jeff"]
};

var anotherPerson = Object.create(person, {
  name: {
    value: "Greg"
  }
});

console.log(anotherPerson.name); // Greg
```

只想简单的让一个对象与另一个对象保持类似的情况下，原型式继承是完全可以胜任的。不过包含引用类型的属性始终会被共享，跟使用原型链模式是一样的。

### 寄生式继承

寄生式继承和原型式继承紧密相关，同样是由[Douglas Crockford](http://www.crockford.com/)推广的。寄生式继承的思路与寄生构造函数和工厂模式类似：

```javascript
function craeteAnother(original) {
  var clone = object(original); // 调用函数创建一个新对象
  clone.sayHi = function() {    // 增强这个对象
    console.log("hi");
  }
  return clone;                 // 返回这个对象
}
```

### 寄生组合式继承

组合继承是JavaScript最常用的继承模式，不过也有不足。组合继承最大的问题就是无论什么情况下，都会调用两次父类的构造函数：一次是在创建子类原型的时候，另外一次在子类型构造函数内部。来看之前的组合继承的例子：

```javascript
function Parent(name) {
  this.name = name;
  this.colors = ["red", "blue"];
}

Parent.prototype.sayName = function() {
  console.log(this.name);
};

function Child(name, age) {
  // 继承属性
  Parent.call(this, name);               // 第二次调用Parent()
  this.age = age;
};

// 继承方法
Child.prototype = new Parent();          // 第一次调用Parent()
Child.prototype.constructor = Child;

// 子类定义方法
Child.prototype.sayAage = function() {
  console.loog(this.age);
}
```

实际上是第二次调用时在新对象上创建的`name`和`colors`覆盖了原型上的两个属性。

所谓寄生组合式继承，即通过借用构造函数来继承属性，通过原型链的混成形式来继承方法。其基本思路就是：不必为了指定子类的原型而调用父类的构造函数，我们所需要的无非就是父类原型的一个副本而已。本质上，就是使用寄生式继承来继承父类的原型，然后再将结果指定给子类的原型。寄生组合式继承的基本模式如下：

```javascript
function inheritPrototype(child, parent) {
  var prototype = object(parent.prototype);  // 创建父类原型的一个副本
  prototype.constructor = child;             // 指定construcotr
  child.prototype = prototype;               // 将父类原型赋给子类原型
}
```

这相当于就替换了之前的`Child.prototype = new Parent();`和`Child.prototype.constructor = Child;`这两句语句。

```javascript
function Parent(name) {
  this.name = name;
  this.colors = ["red", "blue"];
}

Parent.prototype.sayName = function() {
  console.log(this.name);
}

function Child(name, age) {
  Parent.call(this, name);
  this.age = age;
}

inheritPrototype(Child, Parent);

Child.prototype.sayAge = function() {
  console.log(this.age);
}
```

这个例子的高效率体现在它只调用了一次`Parent`的构造函数，并且因此避免了在`Parent.prototype`上创建不必要的、多余的属性。与此同时，原型链还能保持不变，`instanceof`和`isPrototypeOf`也能正常使用。开发人员普遍认为寄生组合式继承是引用类型最理想的继承范式。

## 参考资料

- [JavaScript高级程序设计](https://book.douban.com/subject/10546125/?utm_campaign=douban_search_top_right&utm_medium=pc_web&utm_source=douban)