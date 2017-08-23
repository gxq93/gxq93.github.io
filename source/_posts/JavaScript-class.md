---
title: JavaScript的构造函数与类
date: 2016-11-28 14:24:33
tags: [JavaScript]
categories: 技术
---
JavaScript 虽然有些地方跟其他语言不同，但是他仍然是面向对象语言，因此对象的使用是整个体系的基础。本文列举了几种 JavaScript 创建对象的方式，并讲讲 ES6 中 class 的概念。

<!--more-->

# 对象创建方法
## 最简单的new方法
``` js
var obj = new Object();
obj.name = "guxinqi";
obj.bar = "JavaScript";
obj.sayWhat = function() {
    console.log(this.name);
}
```
## 使用字面量创建
作为脚本语言，字面量定义对象更加的直观
``` js
var obj = {
  name: "guxinqi",
  bar: "JavaScript",
  sayWhat: function() {
    console.log(this.name);
  }
}
```
## 工厂模式
工厂模式很好的防止了大量重复代码的产生
``` js
function createObj(name,bar) {
  var o = new Object();
  o.name = name;
  o.bar = bar;
  o.sayWhat = function() {
    console.log(this.name);
  }
}
var o1 = createObj("Bob", "JavaScript");
var o2 = createObj("Harry", "Objective-C");
```

## 构造函数
工厂模式解决了多个相似对象的创建问题，但是问题又来了，这些对象都是 Object 创建出来的，怎么区分它们的对象具体类型呢？这时候我们就需要切换到另一种模式了，构造函数模式。JavaScript 中的构造函数和其它语言中的构造函数是不同的。 通过 new 关键字方式调用的函数都被认为是构造函数。在构造函数内部 this 指向新创建的对象 Object，**如果被调用的函数没有显式的 return 表达式，则隐式的会返回 this 对象－也就是新创建的对象。**
``` js
function Obj(name,bar) {
  this.name = name;
  this.bar = bar;
  this.sayWhat = function() {
    console.log(this.name);
  }
}
var o1 = new Obj("Bob", "JavaScript");
var o2 = new Obj("Harry", "Objective-C");
```
这里我们使用一个大写字母开头的构造函数替代了上例中的 createObj，注意按照约定构造函数的首字母要大写。在这里我们创建一个新对象，然后将构造函数的作用域赋给新对象，调用构造函数中的方法。

我们可以发现上述两个不同实例调用构造函数的方法 sayWhat 也不是同一个函数实例：
``` js
console.log(o1.sayWhat == o2.sayWhat); /* false */
```
我们可以把这个函数放到构造函数外部，防止多次定义方法函数。

# Class
ES6 提供了更接近传统语言的写法，引入了 Class（类）这个概念，作为对象的模板。通过 class 关键字，可以定义类。基本上，ES6 的 class 可以看作只是一个语法糖，它的绝大部分功能，ES5 都可以做到，新的 class 写法只是让对象原型的写法更加清晰、更像面向对象编程的语法而已。

上面例子用 class 来改写，可以这样：
``` js
class Obj {
  constructor(name, bar) {
    this.name = name;
    this.bar = bar;
  }

  sayWhat() {
    return '(' + this.name + ', ' + this.bar + ')';
  }
}
```
上面代码定义了一个“类”，可以看到里面有一个 constructor 方法，这就是构造方法，而 this 关键字则代表实例对象。也就是说，ES5 的构造函数 Obj，对应 ES6 的 Obj 类的构造方法。

Obj 类除了构造方法，还定义了一个 sayWhat 方法。注意，定义“类”的方法的时候，前面不需要加上 function 这个关键字，直接把函数定义放进去了就可以了。另外，方法之间不需要逗号分隔，加了会报错。

**ES6的类，完全可以看作构造函数的另一种写法。**

使用的时候，也是直接对类使用new命令，跟构造函数的用法完全一致。

`` var o1 = new Obj("Bob", "JavaScript")``

构造函数的 prototype 属性，在 ES6 的“类”上面继续存在。事实上，类的所有方法都定义在类的 prototype 属性上面。
## constructor方法
constructor 方法是类的默认方法，通过 new 命令生成对象实例时，自动调用该方法。一个类必须有 constructor 方法，如果没有显式定义，一个空的 constructor 方法会被默认添加。

constructor 方法默认返回实例对象（即 this），完全可以指定返回另外一个对象。
``` js
class Foo {
  constructor() {
    return Object.create(null);
  }
}

new Foo() instanceof Foo  /* false */
```
类的构造函数，不使用 new 是没法调用的，会报错。这是它跟普通构造函数的一个主要区别，后者不用 new 也可以执行。

## this的指向
类的方法内部如果含有 this，它默认指向类的实例。但是，必须非常小心，一旦单独使用该方法，很可能报错。

一个比较简单的解决方法是，在构造方法中绑定 this，这样就不会找不到 print 方法了。另一种解决方法是使用箭头函数。
``` js
class Logger {
  constructor() {
    this.someFunc = this.someFunc.bind(this);
  }
  /* 或者 */
  constructor() {
    this.someFunc = (...) => {
      /* .... */
    };
  }
}
```
## 继承
Class 之间可以通过 extends 关键字实现继承，这比 ES5 的通过修改原型链实现继承，要清晰和方便很多。

``class SubObj extends Obj {}``

ES6 的继承机制完全不同，实质是先创造父类的实例对象 this（所以必须先调用 super 方法），然后再用子类的构造函数修改 this。如果子类没有定义 constructor 方法，这个方法会被默认添加，代码如下。也就是说，不管有没有显式定义，任何一个子类都有 constructor 方法。
``` js
constructor(...args) {
  super(...args);
}
```
另一个需要注意的地方是，在子类的构造函数中，只有调用 super 之后，才可以使用 this 关键字，否则会报错。这是因为子类实例的构建，是基于对父类实例加工，只有 super 方法才能返回父类实例。

## super关键字
super 这个关键字，既可以当作函数使用，也可以当作对象使用。在这两种情况下，它的用法完全不同。

第一种情况，super 作为函数调用时，代表父类的构造函数。ES6 要求，子类的构造函数必须执行一次 super 函数。

作为函数时，super() 只能用在子类的构造函数之中，用在其他地方就会报错。

第二种情况，super 作为对象时，指向父类的原型对象。(这里先不展开讲)

## Class的getter和setter
``` js
class MyClass {
  constructor() {
    /* ... */
  }
  get prop() {
    return 'getter';
  }
  set prop(value) {
    console.log('setter: '+value);
  }
}

let inst = new MyClass();

inst.prop = 123;
/* setter: 123 */

inst.prop
/* 'getter' */
```
上面代码中，prop 属性有对应的存值函数和取值函数，因此赋值和读取行为都被自定义了。

## Class的静态属性和实例属性
静态属性指的是 Class 本身的属性，即 Class.propname，而不是定义在实例对象（this）上的属性。
``` js
class Foo {
}

Foo.prop = 1;
Foo.prop /* 1 */
```
目前，只有这种写法可行，因为 ES6 明确规定，Class 内部只有静态方法，没有静态属性。
