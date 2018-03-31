---
layout:     post
title:      Promise基础
subtitle:   Promise与Deferred相关知识的再阅读与提炼
date:       2016-02-28
author:     elson
header-img: 
catalog: true
tags:
    - javascript
    - 异步编程
---


>**阅读资料**
>[promise迷你书](http://liubin.org/promises-book/)
>[We have a problem with promises](http://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html)   [(中文版看这里)](http://efe.baidu.com/blog/promises-anti-pattern/)
> [化解使用 Promise 时的竞态条件](http://efe.baidu.com/blog/defusing-race-conditions-when-using-promises/)
>[阮老师的jQuery的deferred对象详解](http://www.ruanyifeng.com/blog/2011/08/a_detailed_explanation_of_jquery_deferred_object.html)



## 解决的是异步处理问题，传统是用回调函数

> 这和回调函数方式相比有哪些不同之处呢？ 在使用promise进行一步处理的时候，我们必须按照接口规定的方法编写处理代码。
也就是说，除promise对象规定的方法(这里的 then 或 catch)以外的方法都是不可以使用的， 而不会像回调函数方式那样可以自己自由的定义回调函数的参数，而必须严格遵守固定、统一的编程方式来编写代码。
这样，基于Promise的统一接口的做法， 就可以形成基于接口的各种各样的异步处理模式。
所以，promise的功能是可以将复杂的异步处理轻松地进行模式化， 这也可以说得上是使用promise的理由之一。

## Promise的基础用法
```javascript
function getURL(url) {
    return new Promise(function (resolve, reject) {
        var xhr = new XMLHttpRequest();
        xhr.open('get', url, true);

        xhr.onload = function () {
            if (xhr.status === 200) {
                resolve(xhr.responseText);
            }
            else {
                reject(new Error(xhr.statusText));
            }
        };
        xhr.onerror = function () {
            reject(new Error(xhr.statusText));
        };

        xhr.send();
    });
}

getURL("xxx").then(function (responseText) {
    console.log(responseText);
}).catch(function (error) {
    console.error(error);
});
```

> 静态方法Promise.resolve(value) 可以认为是 new Promise() 方法的快捷方式。
会让这个promise对象立即进入确定（即resolved）状态，但如果还有其他异步操作呢？

> Promise.resolve(thenable object)
然后会将thenable对象转换成promise对象并返回

```javascript
var promise = Promise.resolve($.ajax('/json/comment.json'));// => promise对象
promise.then(function(value){
   console.log(value);
});
```

为了避免同时使用同步、异步调用可能引起的混乱问题，Promise在规范上规定 Promise的then只能使用异步调用方式 。

```javascript
var promise = new Promise(function (resolve){
    console.log("inner promise"); // 1
    resolve(42);
});
promise.then(function(value){
    console.log(value); // 3
});
console.log("outer promise"); // 2

// why not 213
// new Promise里面的那个回调函数是同步的？
// 回调函数 != 异步 http://liubin.org/promises-book/#mixed-onready.js
```

## 几个快捷方法的对应关系

| 快捷方法 | 对应于 | 备注 |
------------- | ------------- | -------------
| Promise.resolve | new Promise(function (resolve) { resolve() }) | |
| Promise.reject | new Promise(function (resolve, reject) { reject() }) | |
| .catch | .then(undefined, onRejected) | IE8下的保留字不能用做属性名，只能用["catch"] |

### 1. `.catch`和`.then`没有本质区别，但要分场合使用
``` javascript
function throwError(value) {
    // 抛出异常
    throw new Error(value);
}

// onRejected不会被调用
Promise.resolve(42).then(throwError, onRejected);
// 改成这样则会被调用
Promise.resolve(42).then(throwError).then(null, onRejected);
```

> .then 方法中的onRejected参数所指定的回调函数，实际上针对的是其promise对象或者之前的promise对象，而不是针对 .then 方法里面指定的第一个参数，即onFulfilled所指向的对象，所以捕获不到onFulfilled抛出的错误。这也是 then 和 catch 表现不同的原因。

### 2. `.then`和`.done`的区别
在其他类库里提供了`done`方法，可以用来代替`then`，但是ES6 Promises和Promises/A+规范中并没有对`done`做出规定。
``` javascript
if (typeof Promise.prototype.done === 'undefined') {
    Promise.prototype.done = function (onFulfilled, onRejected) {
        this.then(onFulfilled, onRejected).catch(function (error) {
            setTimeout(function () {
                throw error;
            }, 0);
        });
    };
}
```
从实现代码看出：

1. 封装了`catch`方法的执行，通过`setTimeout callback throw error`把错误抛出来
解决了`Promise.resolve(42).then(throwError);`忘记了使用catch进行异常处理，而导致错误被吞，不会报错的问题

```javascript
function JSONPromise(value) {
    return new Promise(function (resolve) {
        resolve(JSON.parse(value));
    });
}

// 因为promise内部有try catch机制，错误被内部catch捕获了，但没有处理，不会抛出
var string = "{}";
JSONPromise(string).then(function (object) {
    conole.log(object); // console拼写错误
});
```

2. `done`并不返回promise对象，因此在done之后不能使用`catch`等方法组成方法链


## promise的then链式操作

```javascript
function doubleUp(value) {
    return value * 2;
}
function increment(value) {
    return value + 1;
}
function output(value) {
    console.log(value);// => (1 + 1) * 2
}

var promise = Promise.resolve(1);
promise
    .then(increment)
    .then(doubleUp)
    .then(output)
    .catch(function(error){
        // promise chain中出现异常的时候会被调用
        console.error(error);
    });
```
> 每个方法中 return 的值不仅只局限于字符串或者数值类型，也可以是对象或者promise对象等复杂类型。
> return的值会由 Promise.resolve(return的返回值); 进行相应的包装处理，因此不管回调函数中会返回一个什么样的值，最终 then 的结果都是返回一个新创建的promise对象。

### 每次then调用之后都返回一个新的Promise对象
> 从代码上乍一看， aPromise.then(...).catch(...) 像是针对最初的 aPromise 对象进行了一连串的方法链调用。
然而实际上不管是 then 还是 catch 方法调用，都返回了一个新的promise对象。
> 也就是说， `Promise#then` 不仅仅是注册一个回调函数那么简单，它还会将回调函数的返回值进行变换，创建并返回一个promise对象。

``` javascript
var aPromise = Promise.resolve(100);
aPromise.then(function (value) { return value * 2});
aPromise.then(function (value) { return value * 2});
aPromise.then(function (value) { console.log(value);}); // 100

// 反模式
function badAsyncCall() {
    var promise = Promise.resolve();
    promise.then(function() {
        // 任意处理
        return newVar;
    });
    return promise;
}

// 2: 对 `then` 进行 promise chain 方式进行调用
var bPromise = new Promise(function (resolve) {
    resolve(100);
});
bPromise.then(function (value) {
    return value * 2;
}).then(function (value) {
    return value * 2;
}).then(function (value) {
    console.log("2: " + value); // => 100 * 2 * 2
});

// 模式
function anAsyncCall() {
    var promise = Promise.resolve();
    return promise.then(function() {
        // 任意处理
        return newVar;
    });
}
```

## Promise.all() & Promise.race()
### 1. Promise.all 
接收一个**promise对象的数组**作为参数，当这个数组里的所有promise对象全部变为resolve或reject状态的时候，它才会去调用 `.then` 方法。

```javascript
Promise.all([req.comment(), req.posts()]).then(function (results) {
    console.log(results); // [comment, posts]
});
```
- 得到的数据数组的顺序和传入all的顺序一致
- 传递给 `Promise.all` 的promise并不是一个个的顺序执行的，而是同时开始、并行执行的

### 2. Promise.race
只要有一个promise对象进入 FulFilled 或者 Rejected 状态的话，就会继续进行后面的处理

```javascript
// task1(延迟10ms) task2(延迟20ms)
// 执行时间相同
Promise.race([req.task1(10), req.task2(20)]).then(function (value) {
    console.log(value); // 10 输出最先完成的
});
```
虽然只要有一个Promise不再处于pending态就会进行后续操作，但是并不会取消传进去的其他Promise对象的执行

> 在 ES6 Promises 规范中，也没有取消（中断）promise对象执行的概念，我们必须要确保promise最终进入resolve or reject状态之一。也就是说Promise并不适用于 状态 可能会固定不变的处理。也有一些类库提供了对promise进行取消的操作。

## Deferred(jQuery) 与 Promise
上面用Promise对XHR进行了封装，以下用基于Promise实现的Deferred对象进行的改写

```javascript
function Deferred() {
    this.promise = new Promise(function (resolve, reject) {
        this._resolve = resolve;
        this._reject = reject;
    }.bind(this)); 
}
Deferred.prototype.resolve = function (value) {
    this._resolve.call(this.promise, value);
};
Deferred.prototype.reject = function (error) {
    this._reject.call(this.promise, error);
};

function getURL(url) {
    var deferred = new Deferred();
    var xhr = new XMLHttpRequest();
    xhr.open("get", url, true);
    xhr.onload = function () {
        if (xhr.status === 200) {
            deferred.resolve(xhr.responseText);
        }
        else {
            deferred.reject(new Error(xhr.statusText));
        }
    };
    xhr.onerror = function () {
        deferred.reject(new Error(xhr.statusText));
    };
    xhr.send();
    return deferred.promise;
}

// run
getURL("xxx").then(function (value) {
    console.log(value);
}).catch(function (error) {console.error(error)});

```
### 异同
0. deferred包含了promise，并具有一些改变状态的特权方法；而promise上则没有resolve这些方法（通过参数传进了构造函数）
1. 在Promise一般都会在构造函数中编写主要处理逻辑，对resolve、reject方法进行调用
2. Deferred则不需要将处理逻辑写成一大块代码用Promise构造函数括起来，只需要先创建deferred对象，可以在任何时机对 resolve、reject 方法进行调用。

> 换句话说，Promise代表了一个对象，这个对象的状态现在还不确定，但是未来一个时间点它的状态要么变为正常值（FulFilled），要么变为异常值（Rejected）；而Deferred对象表示了一个处理还没有结束的这种事实，在它的处理结束的时候，可以通过Promise来取得处理结果。

