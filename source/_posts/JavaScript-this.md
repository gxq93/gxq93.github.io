---
title: JavaScript的this
date: 2016-11-22 14:21:41
tags: [JavaScript]
categories: 技术
---
最近在学 JavaScript，就记录一些学习的心得吧。
# this
开始了解 JavaScript 的时候，对于 this 理解很费劲，毕竟在 ES6 之前没有 class，习惯了普通面向对象语言的人就很不习惯，虽然现在网上资料很多，但是有时候资料多也不是什么好事，看了[MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this)的文档，觉得上面的描述清晰多了，接下来本文主要大致的描述一下 this 的使用场景。

<!--more-->

首先 this 作为函数关键字和其他语言不太一样，而对他自己来说，在严格模式和非严格模式中意义也不同。

this 的指向在函数定义的时候是确定不了的，只有函数执行的时候才能确定 this 到底指向谁，实际上 this 的最终指向的是那个调用它的对象，而每次函数调用时这个 this 的值也可能是不一样的。ES5 使用 bind 方法来设置 this 的值无论这个函数什么时候调用,ECMAScript 2015 中引入箭头函数来决定 this 的词法作用域(它设置 this 的值在封闭的执行上下文中)。

## 全局上下文
在全局上下文中不管是不是严格模式，this 都表示全局对象。
在严格版中的默认的 this 不再是 window，而是 undefined。
``` js
console.log(this.document === document); /* true */

/* In web browsers, the window object is also the global object: */
console.log(this === window); /* true */

this.a = 37;
console.log(window.a); /* 37 */
```
## 函数上下文
在函数中，this 的值取决于这个函数是怎么调用的。
### 简单的调用
``` js
function f1() {
  return this;
}
/* In a browser: */
f1() === window; /* the window is the global object in browsers */

/* In Node: */
f1() === global
```
### 对象方法
如果 this 作为一个对象方法，他指向调用方法的对象。
``` js
var o = {
  prop: 37,
  f: function() {
    return this.prop;
  }
};

console.log(o.f()); /* logs 37 */
```
注意,这种行为是不受或者函数是如何定义的。在前面的示例中,我们定义了内联函数 f 在 o 的定义。然而,我们可以简单地定义了函数在对象外,后来将它连接到 o.f. 这样做结果相同:
``` js
var o = {prop: 37};

function independent() {
  return this.prop;
}

o.f = independent;

console.log(o.f()); /* logs 37 */
```
或者这样：
``` js
o.b = {g: independent, prop: 42};
console.log(o.b.g()); /* logs 42 */
```
甚至这样：
``` js
var o = {f:function(){ return this.a + this.b; }};
var p = Object.create(o);
p.a = 1;
p.b = 4;

console.log(p.f()); /* 5 */
```
只要记住 this 就是指向函数调用对象就行。
同样，在 setter 和 getter 方法中原理也是一样的：
``` js
function sum() {
  return this.a + this.b + this.c;
}

var o = {
  a: 1,
  b: 2,
  c: 3,
  get average() {
    return (this.a + this.b + this.c) /* 3 */;
  }
};

Object.defineProperty(o, 'sum', {
    get: sum, enumerable:true, configurable:true});

console.log(o.average, o.sum); /* logs 2, 6 */
```
### 构造函数
当一个函数作为构造函数(new 关键字)，this 会被绑定到构造的新对象。
``` js
function C() {
  this.a = 37;
}

var o = new C();
console.log(o.a); /* logs 37 */


function C2() {
  this.a = 37;
  return {a:38};
}

o = new C2();
console.log(o.a); /* logs 38 */
```
### 作为DOM事件处理者
如果函数作为 DOM 事件的处理者，this 指向执行的元素。
``` js
<button onclick="alert(this.tagName.toLowerCase());">
  Show this
</button>
```
这里的 this 指的时 button 这个元素
``` js
<button onclick="alert((function(){return this})());">
  Show inner this
</button>
```
在这里，因为没有设置 this 的值，所以 this 指向全局对象(window)

# call函数
``` js
var o1 = {
    a: 3,
    fn: function() {
        console.log(this.a);
    }
}
var o2 = o1.fn;
o2(); /*undefined */
```
在上述情况下你会发现 o2 函数中的 this 已经不是指向 o1 的 a，这很好理解，因为函数调用对象不是 o1，但是有时候我们不得不将这个对象保存到另外的一个变量中，那么就可以通过以下方法：
``` js
var o1 = {
    a: 3,
    fn: function(){
        console.log(this.a); /* 3 */
    }
}
var o2 = o1.fn;
o2.call(o1);
```
通过在 call 方法中给第一个参数添加要把 b 添加到哪个环境中，简单来说，this 就会指向那个对象。

call 方法除了第一个参数以外还可以添加多个参数，如下：
``` js
var o1 = {
    a: 3,
    fn: function(b,c){
        console.log(this.a); /* 3 */
        console.log(b + c);  /* 3 */
    }
}
var o2 = o1.fn;
o2.call(o1,1,2);
```
# apply函数
apply 方法和 call 方法有些相似，它也可以改变 this 的指向：
``` js
var o1 = {
    a: 3,
    fn: function(){
        console.log(this.a); /* 3 */
    }
}
var o2 = o1.fn;
o2.apply(o1);
```
同样 apply 也可以有多个参数，但是不同的是，第二个参数必须是一个数组，如下：
``` js
var o1 = {
    a: 3,
    fn: function(b,c){
        console.log(this.a); /* 3 */
        console.log(b + c);  /* 3 */
    }
}
var o2 = o1.fn;
o2.apply(o1,[1,2]]);
```
**注意如果 call 和 apply 的第一个参数写的是 null，那么 this 指向的是 window 对象**

**call 和 apply 的区别只在于这两个函数接受的参数形式不同。**

# bind函数
bind 方法和 call、apply 方法有些不同，实际上 bind 方法返回的是一个修改过后的函数
``` js
var o1 = {
    a: 3,
    fn: function(){
        console.log(this.a);
    }
}
var o2 = o1.fn;
console.log(o2.bind(o1)); //function() { console.log(this.a); }
```
因此我们可以用这个函数来完成你想要的需求:
``` js
var o1 = {
    a: 3,
    fn: function(){
        console.log(this.a); /* 3 */
    }
}
var o2 = o1.fn;
var o3 = o2.bind(o1);
o3();
```
同样 bind 也可以有多个参数，并且参数可以执行的时候再次添加，但是要注意的是，参数是按照形参的顺序进行的。
``` js
var o1 = {
    a: 3,
    fn: function(b,c,d){
        console.log(this.a); /* 3 */
        console.log(b+c+d);  /* 6 */
    }
}
var o2 = o1.fn;
var o3 = o2.bind(o1,1);
o3(2,3);
```
call 和 apply 都是改变上下文中的 this 并立即执行这个函数，bind 方法可以让对应的函数想什么时候调就什么时候调用，并且可以将参数在执行的时候添加，这是它们的区别，根据自己的实际情况来选择使用。

# 胖箭头函数
胖箭头函数(Fat arrow functions)，又称箭头函数，是一个来自 ECMAScript 2015(又称ES6) 的全新特性。有传闻说，箭头函数的语法 =>，是受到了 CoffeeScript 的影响，并且它与 CoffeeScript 中的 => 语法一样，共享 this 上下文。
>反正我就觉得他只是闭包的另一种写法

举个例子：
``` js
function getVerifiedToken(selector) {
  return getUsers(selector)
    .then(function (users) { return users[0]; })
    .then(verifyUser)
    .then(function (user, verifiedToken) { return verifiedToken; })
    .catch(function (err) { log(err.stack); });
}
```
如果用箭头函数：
``` js
function getVerifiedToken(selector) {
  return getUsers(selector)
    .then(users => users[0])
    .then(verifyUser)
    .then((user, verifiedToken) => verifiedToken)
    .catch(err => log(err.stack));
}
```
这个就有点像 swift 的闭包了，就是把 in 换做了 =>。

以下是值得注意的几个要点：
* function 和 {} 都消失了，所有的回调函数都只出现在了一行里。
* 当只有一个参数时，() 也消失了（rest 参数是一个例外，如 (...args) => ...）。
* 当 {} 消失后，return 关键字也跟着消失了。单行的箭头函数会提供一个隐式的 return（这样的函数在其他编程语言中常被成为 lamda 函数）。

这里再着重强调一下上述的最后一个要求。仅仅当箭头函数为单行的形式时，才会出现隐式的 return。当箭头函数伴随着 {} 被声明，那么即使它是单行的，它也不会有隐式 return：
``` js
const getVerifiedToken = selector => {
  return getUsers()
    .then(users => users[0])
    .then(verifyUser)
    .then((user, verifiedToken) => verifiedToken)
    .catch(err => log(err.stack));
}
```
不幸的是，空对象 {} 和空白函数代码块 {} 长得一模一样。。以上的例子中，emptyObject 的 {} 会被解释为一个空白函数代码块，所以 emptyObject() 会返回 undefined。如果要在箭头函数中明确地返回一个空对象，则你不得不将 {} 包含在一对圆括号中 ({})：
``` js
const emptyObject = () => ({});
emptyObject(); /* {} */
```
下面是一个更完整的例子：
``` js
function () { return 1; }
() => { return 1; }
() => 1

function (a) { return a * 2; }
(a) => { return a * 2; }
(a) => a * 2
a => a * 2

function (a, b) { return a * b; }
(a, b) => { return a * b; }
(a, b) => a * b

function () { return arguments[0]; }
(...args) => args[0]

() => {} /* undefined */
() => ({}) /* {} */
```

**箭头函数没有属于自己的 this 和 arguments**

因此箭头函数就不需要担心 this 的上下文问题，不管嵌套多少层，箭头函数中的 this 总能指向正确的上下文，因为函数体内的 this 指向的对象，就是定义时所在的对象，而不是使用时所在的对象。

**箭头函数不能作为 generator 函数使用。**

最后注意，React class 中的 this 指向组件本身，因此在事件绑定中应该将 this 绑定到事件函数中。
