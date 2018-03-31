---
layout:     post
title:      Promise 原理探究
subtitle:   你真的了解Promise吗？看看这几道题，能不能轻松答出来？
date:       2018-03-30
author:     elson
header-img: 
header-bg-color: 337ab7
catalog: true
tags:
    - javascript
    - 异步编程
---

## 前言：你真的了解Promise吗
你真的了解Promise吗？对我而言，除了知道如何使用then解决回调地狱以外，其他的还真的一知半解。虽然ES6的generator和ES7的async await提供了更先进的异步编程解决方案，但是它们还是离不开Promise，比如generator的co库的实现以及await后面必须返回promise。因此有必要深入了解一下Promise的原理。

看看下面这几道题，能不能轻松答出来？
> 假设 doSomething() 和 doSomethingElse() 返回一个 promise 对象，这些 promise 对象都代表了一个异步操作。
> 
> 那么：doSomething/doSomethingElse/finalHandler这三者的执行时序是怎样的？finalHandler分别会接受到什么值？
> 
> 出处：https://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html

``` javascript
// ① 
doSomething().then(function () {
    return doSomethingElse();
}).then(finalHandler);

// ② 
doSomething().then(function () {
    doSomethingElse();
}).then(finalHandler);

// ③ 
doSomething().then(doSomethingElse()).then(finalHandler);

// ④ 
doSomething().then(doSomethingElse).then(finalHandler);
```
如果满脸黑人问号的话，下面我们通过实现一个简易的Promise——`MyPromise`来分析一下上面几道题。
> ps. `MyPromise`仅实现核心功能，不保证完全满足promise规范。

## 一、雏形(v1)
Promise最基本的用法：调用`resolve`时，then的回调才会被执行，并得到resolve时的值。
``` javascript
new MyPromise(resolve => {
    resolve(2111);
}).then(val => {
    console.log(val); // 2111
});
```
### 1. 实现分析
从后往前看，首先MyPromise实例拥有`then`方法，而传入`then`的回调一定是晚于`resolve`执行的，因此这里通过闭包将then的回调存起来，等待被调用。

对于`resolve`而言，它的作用就是从闭包中取出then的回调进行调用，并透传参数值。

这里需要将callback的调用时机通过settimout放到下一事件循环中，让then方法先调用，否则会报`TypeError: callback is not a function`
``` javascript
function MyPromise(cb) {
    let callback = null;
    this.then = cb => { // 具有then方法
        callback = cb; // 调用`then`时，闭包保存回调
    }

    function resolve(val) {
        setTimeout(() => { // 下一事件循环再执行
            callback(val); // 取出闭包中的回调执行
        }, 0);
    }
	if (typeof cb == 'function') {
		cb(resolve); // 执行初始化cb，并传入resolve函数
	}
}
```

### 2. 问题
``` javascript
var pms = new MyPromise(function (resolve, reject) {
    resolve(2111);
});

// 异步调用then时
// TypeError: callback is not a function
setTimeout(function () {
	pms.then(function (val) {
	    console.log(val); 
	});
}, 100);
```
原因：`resolve`和`then`的调用顺序紊乱。
当resolve调用callback时，then的回调仍未被保存到callback中。

## 二、引入状态流转(v2)
通过状态流转，管理调用时序。这也是Promise规范规定的一个Promise有且仅有的三种状态之一：pending/fulfilled(resolved)/rejected (rejected这一版先不引入)
- 状态初始值为pending
- then早于resolve调用时：此时状态值仍是pending，因此可以保存onResolve回调，等待resolve调用
- resolve早于then调用时：保存决议值，状态流转为resolved；等待then调用

``` javascript
function MyPromise(cb) {
    var cachedResolved = null;
    var state = 'pending';
    var resolvedVal;

    this.then = function (onResolved) {
        // 状态判断，pending时保存onResolved，否则直接调用onResolved
        if (state == 'pending') {
            cachedResolved = onResolved;
            return;
        }

        if (typeof onResolved == 'function') {
            onResolved(resolvedVal);
        }
    }

    function resolve(newVal) {
        resolvedVal = newVal; // 保存决议值
        state = 'resolved'; // 流转状态
		
        // 如then早被调用已保存了的回调，则可以直接执行
        if (typeof cachedResolved == 'function') {
            cachedResolved(resolvedVal);
        }
    }

    if (typeof cb == 'function') {
		cb(resolve);
	}
}
```

## 三、增加reject及错误处理(v3)
除了resolve函数能够流转状态之外，还有一个reject函数，调用后将会把当前promise的状态流转成rejected，用作异常处理。

### 1. 实现源码
``` javascript
function MyPromise(fn) {
	this.state = 'pending'; // 状态
    this.resolvedVal = ''; // 决议值
    this.rejectedReason = null; // 拒绝值
    this.onResolved = null; // resolve后的注册回调
    this.onRejected = null; // reject后的注册回调
	
	this.resolve = function (val) {
		try {
			// 每个promise的状态只能流转一次
			if (this.state != 'pending') {
				return;
			}
			this.state = 'resolved';
			this.resolvedVal = val;
			if (typeof this.onResolved == 'function') {
                console.log('triggered by resolve');
				this.onResolved(val);
			}
		} catch(err) {
			this.reject(err);
		}
	}
	
	this.reject = function (err) {
		if (this.state != 'pending') {
			return;
		}
		this.state = 'rejected';
		this.rejectedReason = err;
        
        if (typeof this.onRejected == 'function') {
            console.log('triggered by reject');
            this.onRejected(err);
        }
    }
    
    typeof fn == 'function' && fn(this.resolve.bind(this), this.reject.bind(this));
}

MyPromise.prototype.then = function (onResolved, onRejected) {
    // 未决议时 先存起来 等待决议
    if (this.state == 'pending') {
        this.onResolved = onResolved;
        this.onRejected = onRejected;
        return;
    }
    
    // 根据状态选择最终要调用的函数
    let finalCallback = this.state == 'resolved' ? onResolved : onRejected;

    if (typeof finalCallback == 'function') {
        console.log('triggered by then');
        finalCallback.call(this, this.value)
    }
};
```
### 2. 执行用例
``` javascript
new MyPromise((resolve, reject) => {
    setTimeout(() => {
        resolve(6666);
        reject('oops');
    }, 500);
}).then((val) => {
    console.log(val);  // 6666
}, (err) => {
    console.error(err); // 不会执行
});
```
### 3. 实现分析
对比v2，先看下`then`方法的改造：除了`onResolved`以外，新增了一个`onRejected`参数，用于处理reject状态的回调。

此时函数内根据三个不同状态，做出不同处理：
- pending：此时状态仍未流转，因此分别缓存`onResolved`和`onRejected`，提供给`resolve`和`reject`函数后续调用。
- fulfilled(resolved)：调用`onResolved`
- rejected：调用`onRejected`
``` javascript
MyPromise.prototype.then = function (onResolved, onRejected) {
    // 未决议时 先存起来 等待决议
    if (this.state == 'pending') {
        this.onResolved = onResolved;
        this.onRejected = onRejected;
        return;
    }
    
    let finalCallback = this.state == 'resolved' ? onResolved : onRejected;

    if (typeof finalCallback == 'function') {
        console.log('triggered by then');
        finalCallback.call(this, this.value)
    }
};
```

reject函数与resolve的实现方式类似，状态流转成rejected后，调用`onRejected`函数。需要注意的是每个promise的状态只能流转一次，因此`resolve`和`reject`中需要判断其状态，否则先后调用`resolve`和`reject`函数（见上面的执行用例）会出现把promise的状态由resolved流转为rejected的诡异情况。
``` javascript
this.reject = function (err) {
	// 每个promise的状态只能流转一次
	if (this.state != 'pending') {
		return;
	}
    this.state = 'rejected'; // 流转状态
    this.rejectedReason = err; // 存储reject的值

    if (typeof this.onRejected == 'function') {
        console.log('triggered by reject');
        this.onRejected(err);
    }
}
```

同时在resolve中加入try catch捕获非预期错误后调用reject流转状态。
``` javascript
this.resolve = function (val) {
    try {
	    if (this.state != 'pending') {
			return;
		}
        this.state = 'resolved';
        this.resolvedVal = val;
        if (typeof this.onResolved == 'function') {
            console.log('triggered by resolve');
            this.onResolved(val);
        }
    } catch(err) {
        this.reject(err);
    }
}
```

## 四、简易Promise完整版(v4)
在完整版中，将加入以下的特性
1. 支持`then`链式调用，每次调用`then`均返回一个新的promise
2. 决议值为promise（非简单数值）以及 `then`返回promise时，需要反解出结果
3. 当`then`未传入任何回调，此时应该透传上一个promise的结果

### 1. 实现源码
``` javascript
function MyPromise(fn) {
    this.state = 'pending'; // 状态
    this.resolvedVal = ''; // 决议值
    this.rejectedReason = null; // 拒绝值
    this.cached = null; 
	
	this.resolve = function (val) {
		try {
            if (this.state != 'pending') {
                return this.state == 'resolved' ? this.resolvedVal : this.rejectedReason;
            }

            // 如果决议值是一个promise或thenable对象（已决议或已拒绝），非数字字符串等，那么把结果反解出来
            if (typeof val == 'object' && typeof val.then == 'function') {
                val.then(this.resolve.bind(this), this.reject.bind(this));
                return;
            }

			this.state = 'resolved';
            this.resolvedVal = val;
            
			if (this.cached) {
				this.handler(this.cached);
			}
		} catch(err) {
			this.reject(err);
		}
	}
	
	this.reject = function (err) {
        if (this.state != 'pending') {
            return this.state == 'resolved' ? this.resolvedVal : this.rejectedReason;
        }

		this.state = 'rejected';
		this.rejectedReason = err;
        
        if (this.cached) {
            this.handler(this.cached);
        }
    }
    
    typeof fn == 'function' && fn(this.resolve.bind(this), this.reject.bind(this));
}

MyPromise.prototype.handler = function (opt = {}) {
    // 未决议时，缓存onResolved, onRejected, resolve, reject
    if (this.state == 'pending') {
        this.cached = opt;
        return;
    }

    // 存在then未传入任何回调的情况，这时应该透传上一个promise的结果 
    // new Promise(fn).then().then((val) => {console.log(val)})
    if (typeof opt.onResolved != 'function' && typeof opt.onRejected != 'function') {
        if (this.state == 'resolved') {
            opt.resolve(this.resolvedVal);
        } else {
            opt.reject(this.rejectedReason);
        }
        return;
    }

    try {
        let result;
        if (this.state == 'resolved') {
            // 执行resolved状态的回调
            result = opt.onResolved(this.resolvedVal);
            // 将回调的返回值作为新promise的决议值
            opt.resolve(result);
        } else {
            result = opt.onRejected(this.rejectedReason);
            opt.reject(result);
        }
    } catch (err) {
        opt.reject(err);
    }


};

MyPromise.prototype.then = function (onResolved, onRejected) {
    return new MyPromise((resolve, reject) => {
        this.handler({onResolved, onRejected, resolve, reject});
    });
};
```

### 2. 执行用例
``` javascript
new MyPromise((resolve, reject) => {
    resolve(100);
    // reject('error');
}).then((val) => {
    console.log(val);
    return val + 5;
}, (err) => {
    console.log(err);
}).then((val) => {
    console.log(val);

    return new MyPromise((resolve) => {
        resolve(200);
    });
}).then((val) => {
    val++;
    console.log(val);
}, (err) => {
    console.log(err);
});
```

### 3. 实现分析
#### （1）每次调用`then`均返回一个新的Promise
这一点除了用于支持链式调用以外，还很好地解决了一个Promise的状态只能流转一次的规定，因为调用`resolve`或`reject`之后，这个Promise的生命周期就结束了。也就是说链式调用`then`，每次处理的都是一个全新的Promise。
``` javascript
Promise.prototype.then = function (onResolve, onReject) {
    return new Promise((resolve, reject) => {
	    // 通过handler函数统一处理resolve和reject逻辑
        this.handler({onResolve, onReject, resolve, reject});
    });
};
```
#### （2）反解内部的promise
上面几个版本的用例中，`resolve`接受的值以及`then`的返回值都是一个简单的字符或数字，如果类似下面，是一个promise的话，还需要p2和p3的值200和300解出来之后再作为决议值传给`then`。
``` javascript
// example
new MyPromise((resolve, reject) => { // 父promise
	const p2 = new MyPromise((resolve, reject) => { // 子promise
		resolve(200);
	});
    resolve(p2);
}).then((val1) => { // then1
	console.log(val1 == 200) // true
	const p3 = new MyPromise((resolve, reject) => {
		resolve(300);
	});
	return p3;
}).then((val2) => { // then2
	console.log(val2 == 300) // true
});
```
实现这个特性，可以在`resolve`内增加以下逻辑：当接收的参数是一个promise（thenable对象）时，执行该promise的`then`，将父promise的`resolve`和`reject`通过bind的方式绑定this后传入。目的是为了后面能够流转父promise的状态，若不流转状态的话（尝试将bind去掉），then1是不会被执行的。
``` javascript
this.resolve = function (val) {
	...
	if (typeof val == 'object' && typeof val.then == 'function') {
	   val.then(this.resolve.bind(this), this.reject.bind(this));
	   return;
	}
	
	this.state = 'resolved';
    this.resolvedVal = val;
    ...
}
```
#### （3） then未传入任何回调，透传上一promise决议值
``` javascript
new MyPromise(resovle => { // promise1
	resolve(123);
})
.then() // promise2
.then((val) => {
	console.log(val) // 123 
});
```
每次调用`then`都会返回一个新的promise，如果要第二个then被调用，则需要将第一个then返回的promise2的状态流转成resolved。因此在`handler`中增加相关的条件判断。
``` javascript
MyPromise.prototype.then = function (onResolved, onRejected) {
    return new MyPromise((resolve, reject) => {
	    // this是外层promise
	    // 调用resolve流转promise2的状态
        this.handler({onResolved, onRejected, resolve, reject});
    });
};

MyPromise.prototype.handler = function (opt = {}) {
	...
	
    if (typeof opt.onResolved != 'function' && typeof opt.onRejected != 'function') {
        if (this.state == 'resolved') {
	        // 流转promise2的状态，this.resolvedVal == 123
            opt.resolve(this.resolvedVal);
        } else {
            opt.reject(this.rejectedReason);
        }
        return;
    }
    
    ...
};
```
#### （4） 关于错误吞噬
下面的`onRejected`不会执行，这一点现在就比较好理解了。每个then都会返回新的promise，错误是发生在p2里面的，而`onRejected`捕获的是p1的错误。
``` javascript
new MyPromise(resolve => { // p1
	resolve(666);
}).then(throwError, onRejected); // p2
```

## 五、思考题答案
> 假设 doSomething() 和 doSomethingElse() 返回一个 promise 对象，这些 promise 对象都代表了一个异步操作。那么：finalHandler分别会接受到什么值？

``` javascript
① doSomething().then(function () {
    return doSomethingElse();
}).then(finalHandler);

doSomething
|-----------------|
                  doSomethingElse(undefined)
                  |------------------|
                                     finalHandler(resultOfDoSomethingElse)
                                     |------------------|


② doSomething().then(function () {
    doSomethingElse();
}).then(finalHandler);
doSomething
|-----------------|
                  doSomethingElse(undefined)
                  |------------------|
                  finalHandler(undefined)
                  |------------------|
           
                  
③ doSomething().then(doSomethingElse()).then(finalHandler);
doSomething
|-----------------|
doSomethingElse(undefined)
|---------------------------------|
                  finalHandler(resultOfDoSomething)
                  |------------------|


④ doSomething().then(doSomethingElse).then(finalHandler);
doSomething
|-----------------|
                  doSomethingElse(resultOfDoSomething)
                  |------------------|
                                     finalHandler(resultOfDoSomethingElse)
                                     |------------------|
```
上面四道题的重点其实大致可以归到第四章分析里的·这三点
1. 每次调用`then`均返回一个新的Promise
2. 反解内部的promise 
3. then未传入任何回调，透传上一promise决议值


### 第一题
为什么finalHandler的执行顺序在doSomethingElse之后？
finalHandler是then2的onResolve回调，等待的是then1生成的promise。而then1生成的promise的决议值是`doSomethingElse()`的返回值。

### 第二题
为什么doSomethingElse和finalHandler几乎同时执行？
`doSomethingElse()`并没有作为then1的返回值，因此then1生成的promise会立即流转状态为resolved，决议值为undefined，决议之后finalHandler作为then2的onResolve回调，会立即执行。

### 第三题
doSomethingElse()返回值是一个promise，不能作为then1的onResolve回调，因此这种情况相当于then未传入任何回调，这时会将doSomething的决议值透传到then2的finalHandler中去。

### 第四题
这是正常的用法，doSomethingElse作为then1的onResolve回调，接收doSomething()的决议值，执行后返回另一个promise，then1会将这个promise解开并将其决议值作为他的promise的决议值，因此等待then1生成的promise的finalHandler就可以获取到resultOfDoSomethingElse。

## 参考文章
> http://www.mattgreer.org/articles/promises-in-wicked-detail/
> https://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html
> http://liubin.org/promises-book/
