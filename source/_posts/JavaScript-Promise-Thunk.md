---
title: JavaScript的Promise和Thunk
date: 2016-12-16 11:17:31
tags: [JavaScript]
categories: 技术
---
这篇文章主要介绍解决 JavaScript 异步编程中 callback hell 问题的两种方案，而这两种方案也都包含了函数式编程的思想在里面。

<!--more-->

# Promise
## 介绍
所谓 Promise，简单说就是一个容器，里面保存着某个未来才会结束的事件（通常是一个异步操作）的结果。

Promise 对象是一个返回值的代理，这个返回值在 promise 对象创建时未必已知。它允许你为异步操作的成功返回值或失败信息指定处理方法。 这使得异步方法可以像同步方法那样返回值：异步方法会返回一个包含了原返回值的 promise 对象来替代原返回值。

## 创建
``` JavaScript
new Promise(executor);
new Promise(function(resolve, reject) { ... });
```
``executor``:
带有 resolve 、reject 两个参数的一个函数。这个函数在创建 Promise 对象的时候会立即得到执行（在 Promise 构造函数返回 Promise 对象之前就会被执行），并把成功回调函数（resolve）和失败回调函数（reject）作为参数传递进来。调用成功回调函数（resolve）和失败回调函数（reject）会分别触发 promise 的成功或失败。这个函数通常被用来执行一些异步操作，操作完成以后可以选择调用成功回调函数（resolve）来触发 promise 的成功状态，或者，在出现错误的时候调用失败回调函数（reject）来触发 promise 的失败。
Example:
``` JavaScript
let p = new Promise((resolve, reject) => {
    setTimeout(() => {
        Math.random() > 0.5 ? resolve('success') : reject('fail');
    }, 1000)
});

p.then((result) => {
    console.log(result);
}, (err) => {
    console.log(err);
});
```
## 描述
Promise 对象有以下几种状态:

* pending: 初始状态, 既不是 fulfilled 也不是 rejected
* fulfilled: 成功的操作
* rejected: 失败的操作

pending 状态的 promise 对象既可转换为带着一个成功值的 fulfilled 状态，也可变为带着一个失败信息的 rejected 状态。当状态发生转换时，promise.then 绑定的方法（函数句柄）就会被调用。(当绑定方法时，如果 promise 对象已经处于 fulfilled 或 rejected 状态，那么相应的方法将会被立刻调用，所以在异步操作的完成情况和它的绑定方法之间不存在竞争条件。)上述例子我们可以看到，then 方法可以接受两个函数作为参数，第一个函数是用来处理 resolve 的结果，第二个是可选的，用来处理 reject 的结果。也就是说，我们在创建 p 这个 Promise 对象的时候，通过函数 resolve 传递出去的结果可以被 p 的第一个 then 方法中的第一个函数捕获然后作为它的参数。通过函数 reject 传递出去的结果可以被 p 的第一个 then 方法中的第二个函数捕获然后作为它的参数。

因为 Promise.prototype.then 和 Promise.prototype.catch 方法返回 promises 对象, 所以它们可以被链式调用--一种被称为 composition 的操作。

{% asset_img promises.png %}

## API
``Promise.prototype.catch(onRejected)``
添加一个否定(rejection) 回调到当前 promise, 返回一个新的 promise。如果这个回调被调用，新 promise 将以它的返回值来resolve，否则如果当前 promise 进入 fulfilled 状态，则以当前 promise 的肯定结果作为新 promise 的肯定结果。这个方法其实是 then 方法的一种特例，这个特例就是：
``.then(null, rejection)``
相当于我们不使用 then 方法的第一个函数，只是用第二个函数；catch 函数比较简单，就是用来捕获之前的 then 方法里面的异常，我们可以简单的来看一个例子：
``` JavaScript
let p = new Promise((resolve, reject) => {
    resolve();
});
p.then(() => {
    console.log('progress...');
}).then(() => {
    throw new Error('fail');
}).catch((err) => {
    console.log(err);
});
```

``Promise.prototype.then(onFulfilled, onRejected)``
添加肯定和否定回调到当前 promise, 返回一个新的 promise, 将以回调的返回值来 resolve。

``Promise.all(iterable)``
这个方法返回一个新的 promise 对象，该 promise 对象在 iterable 里所有的 promise 对象都成功的时候才会触发成功，一旦有任何一个 iterable 里面的 promise 对象失败则立即触发该 promise 对象的失败。这个新的 promise 对象在触发成功状态以后，会把一个包含 iterable 里所有 promise 返回值的数组作为成功回调的返回值，顺序跟 iterable 的顺序保持一致；如果这个新的 promise 对象触发了失败状态，它会把 iterable 里第一个触发失败的 promise 对象的错误信息作为它的失败错误信息。Promise.all 方法常被用于处理多个 promise 对象的状态集合。Example:
``` JavaScript
let arr = [1, 2, 3].map(
    (value) => {
        return new Promise((resolve, reject) => {
            setTimeout(() => {
                resolve(value);
            }, value * 1000);
        });
    }
);

console.log(arr);

let promises = Promise.all(arr)
.then((result) => {
    console.log(result);
}).catch((err) => {
    console.log(err);
});

Log:
[ Promise { <pending> },
  Promise { <pending> },
  Promise { <pending> } ]
[ 1, 2, 3 ] /* 3s后输出的结果 */
```

``Promise.race(iterable)``
当 iterable 参数里的任意一个子 promise 被成功或失败后，父 promise 马上也会用子 promise 的成功返回值或失败详情作为参数调用父 promise 绑定的相应句柄，并返回该 promise 对象。Example:
``` JavaScript
let promises = Promise.race(arr)/* 上例的arr */
    .then((result) => {
        console.log(result);
    }).catch((err) => {
        console.log(err);
    });

Log:
1 /* 是最先改变状态的那个Promise实例resolve的值 */
```

``Promise.resolve(value)``
用成功值 value 完成一个 Promise 对象。如果该 value 为可继续的（thenable，即带有 then 方法），返回的 Promise 对象会“跟随”这个 value，采用这个 value 的最终状态；否则的话返回值会用这个 value 满足（fullfil）返回的 Promise 对象。Example:
``` JavaScript
let arr = [null, 0, 'hello',
    { then: function() { console.log(' a thenable obj')}}
];

arr.map((value) => {
        return Promise.resolve(value);
    });

console.log(arr);

Log:
[ null, 0, 'hello', { then: [Function: then] } ]
 a thenable obj /* Promise.resolve方法会将具有then方法的对象转换为一个Promise对象，然后就立即执行then方法。*/
```

``Promise.reject(reason)``
调用 Promise 的 rejected 句柄，并返回这个 Promise 对象。Example:
``` JavaScript
let p = Promise.reject('fail');
p.catch((err) => {
    console.log(err);
}); /* fail */
```

# Thunk
## 介绍
* thunk 是一个被封装了同步或异步任务的函数。
* thunk 有唯一一个参数 callback，是 CPS 函数。
* thunk 运行后返回新的 thunk 函数，形成链式调用。
* thunk 自身执行完毕后，结果进入 callback 运行。
* callback 的返回值如果是 thunk 函数，则等该 thunk 执行完毕将结果输入新 thunk 函数运行；如果是其它值，则当做正确结果进入新的 thunk 函数运行。

## 创建
``` JavaScript
/* 正常版本的readFile（多参数版本）*/
fs.readFile(fileName, callback);

/* Thunk版本的readFile（单参数版本）*/
var readFileThunk = Thunk(fileName);
readFileThunk(callback);

var Thunk = function (fileName){
  return function (callback){
    return fs.readFile(fileName, callback);
  };
};
```
上面代码中，fs 模块的 readFile 方法是一个多参数函数，两个参数分别为文件名和回调函数。经过转换器处理，它变成了一个单参数函数，只接受回调函数作为参数。这个单参数版本，就叫做 Thunk 函数。
任何函数，只要参数有回调函数，就能写成 Thunk 函数的形式。下面是一个简单的 Thunk 函数转换器。
``` JavaScript
var Thunk = function(fn){
  return function (){
    var args = Array.prototype.slice.call(arguments);
    return function (callback){
      args.push(callback);
      return fn.apply(this, args);
    }
  };
};
```
使用上面的转换器，生成 fs.readFile 的 Thunk 函数。
``` JavaScript
var readFileThunk = Thunk(fs.readFile);
readFileThunk(fileA)(callback);
```

## 描述
Thunk 意为形实替换程序（有时候也称为延迟计算，suspended computation）。它指的是在计算的过程中，一些函数的参数或者一些结果通过一段程序来代表，这被称为 thunk。可以简单地把 Thunk 看做是一个未求得完全结果的表达式与求得该表达式结果所需要的环境变量组成的函数，这个表达式与环境变量形成了一个无参数的闭包（parameterless closure），所以 Thunk 中有求得这个表达式所需要的所有信息，只是在不需要的时候不求而已。

JavaScript 语言是传值调用，它的 Thunk 函数含义有所不同。在 JavaScript 语言中，Thunk 函数替换的不是表达式，而是多参数函数，将其替换成单参数的版本，且只接受回调函数作为参数。


# Promise和Thunk的区别
Thunk 的编程思维与原生 Promise 是一致的，原生 Promise 能实现的异步业务组合，Thunk 都能实现。区别有以下几点：

1.原生 Promise 出现在 ES6，Thunk 没有使用特别的 JS 特性，ES3 下都能完美运行。
2.Promise 封装出来的是个对象，异步业务隐藏在 promise 对象中，promise 对象的方法或属性值可能被改写（入侵）；Thunk 封装出来是一个 thunk 函数，异步业务隐藏在函数里，对外而言就是一个黑盒，不会受到外部的入侵。
3.Promise 通常自称符合函数式编程，但 Thunk 更符合函数式编程，而且是标准的 CPS 风格。
4.功能同样强大，但 Thunk 的 API 更简洁一致，Thunk 封装也更简洁。
5.Thunk 拥有完美的 debug 模式，Promise 好像没有？
6.Thunk 的性能是原生 Promise 的4倍。

# Promise和Thunk在Redux中的应用
在 Redux 中如果要进行异步操作的时候，至少要 dispatch 两个以上 Action,用户触发第一个 Action，这个跟同步操作一样，没有问题；如何才能在操作结束时，系统自动送出第二个 Action 呢？

redux-thunk 中间件支持 dispatch function，以此让 action creator 控制反转。被 dispatch 的 function 会接收 dispatch 作为参数，并且可以异步调用它。这类的 function 就称为 thunk。

Example:
``` JavaScript
class AsyncApp extends Component {
  componentDidMount() {
    const { dispatch, selectedPost } = this.props
    dispatch(fetchPosts(selectedPost))
  }

  const fetchPosts = postTitle => (dispatch, getState) => {
    dispatch(requestPosts(postTitle));
    return fetch(`/some/API/${postTitle}.json`)
      .then(response => response.json())
      .then(json => dispatch(receivePosts(postTitle, json)));
    };
  };

  /* 使用方法一 */
  store.dispatch(fetchPosts('reactjs'));
  /* 使用方法二 */
  store.dispatch(fetchPosts('reactjs')).then(() =>
    console.log(store.getState())
  );
```
上面代码中，fetchPosts 是一个Action Creator（动作生成器），返回一个函数。这个函数执行后，先发出一个 Action（requestPosts(postTitle)），然后进行异步操作。拿到结果后，先将结果转成 JSON 格式，然后再发出一个 Action（receivePosts(postTitle, json)）。

上面代码中，有几个地方需要注意:
1.fetchPosts 返回了一个函数，而普通的 Action Creator 默认返回一个对象。
2.返回的函数的参数是 dispatch 和 getState 这两个 Redux 方法，普通的 Action Creator 的参数是 Action 的内容。
3.在返回的函数之中，先发出一个 Action（requestPosts(postTitle)），表示操作开始。
4.异步操作结束之后，再发出一个 Action（receivePosts(postTitle, json)），表示操作结束。

既然 Action Creator 可以返回函数，当然也可以返回其他值。另一种异步操作的解决方案，就是让 Action Creator 返回一个 Promise 对象。这就需要使用 redux-promise 中间件。这个中间件使得 store.dispatch 方法可以接受 Promise 对象作为参数。这时，Action Creator 有两种写法。

写法一，返回值是一个 Promise 对象:
``` JavaScript
const fetchPosts =
  (dispatch, postTitle) => new Promise(function (resolve, reject) {
     dispatch(requestPosts(postTitle));
     return fetch(`/some/API/${postTitle}.json`)
       .then(response => {
         type: 'FETCH_POSTS',
         payload: response.json()
       });
});
```

写法二，Action 对象的 payload 属性是一个 Promise 对象。这需要从 redux-actions 模块引入 createAction 方法，并且写法也要变成下面这样:
``` JavaScript
import { createAction } from 'redux-actions';

class AsyncApp extends Component {
componentDidMount() {
  const { dispatch, selectedPost } = this.props
  /* 发出同步 Action */
  dispatch(requestPosts(selectedPost));
  /* 发出异步 Action */
  dispatch(createAction(
    'FETCH_POSTS',
    fetch(`/some/API/${postTitle}.json`)
      .then(response => response.json())
  ));
}
```

上面代码中，第二个 dispatch 方法发出的是异步 Action，只有等到操作结束，这个 Action 才会实际发出。注意，createAction 的第二个参数必须是一个 Promise 对象。

从 redux-promise 源码可以看出，如果 Action 本身是一个 Promise，它 resolve 以后的值应该是一个 Action 对象，会被 dispatch 方法送出（action.then(dispatch)），但 reject 以后不会有任何动作；如果 Action 对象的 payload 属性是一个 Promise 对象，那么无论 resolve 和 reject，dispatch 方法都会发出 Action。

redux-thunk源码：
``` JavaScript
function createThunkMiddleware(extraArgument) {
  return ({ dispatch, getState }) => next => action => {
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }

    return next(action);
  };
}

const thunk = createThunkMiddleware();
thunk.withExtraArgument = createThunkMiddleware;
```

redux-promise源码：
``` JavaScript
export default function promiseMiddleware({ dispatch }) {
  return next => action => {
    if (!isFSA(action)) {
      return isPromise(action)
        ? action.then(dispatch)
        : next(action);
    }

    return isPromise(action.payload)
      ? action.payload.then(
          result => dispatch({ ...action, payload: result }),
          error => {
            dispatch({ ...action, payload: error, error: true });
            return Promise.reject(error);
          }
        )
      : next(action);
  };
}
```
