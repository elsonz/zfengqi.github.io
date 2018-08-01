---
layout:     post
title:      从webpack4打包文件说起
subtitle:   一堆的webpack配置教程看腻了？这里有webpack4的打包及加载机制，要不了解一下？
date:       2018-08-01
author:     elson
header-img:
header-bg-color: 337ab7
catalog: true
tags:
    - webpack
---

# 从webpack4打包文件说起

一堆的webpack配置教程看腻了？这里有webpack4的打包及加载机制，要不了解一下？而这一切就得从打包文件说起。

相信大家都和我一样，用webpack打完包之后，很少或者极度反感打开bundle.js来看的，里面一坨坨的编译后代码和没完没了的`/****/`注释，完全不知所云。看起来虽然恶心，但还挺有营养。下面通过打包文件来深入了解下webpack4的模块化处理以及代码拆分加载机制。

使用的webpack配置如下，通过调整entry的内容来观察对比打包文件的异同。

``` javascript
// webpack.config.js
module.exports = {
	mode: 'development', // 不压缩
	entry: {
		chunk1: './src/index.js'
	},
	output: {
		path: './dist',
		filename: '[name]-[chunkhash:8].js' // 为了后面的多入口
	},
	devtool: '' // 去掉sourcemap，模块不会被eval包裹，更直观
};
```

## 一、webpack4的模块化处理机制

``` javascript
// index.js
const name = require('./name.js');
console.log(name);

// name.js
module.exports = 'elson';
```

执行`npx webpack`后得到`chunk1-cfdec98e.js`（精简改造后）。

``` javascript
// chunk1-cfdec98e.js

// 模块数据映射表
var modulesData = {
	"./src/commonjs/index.js": function (module, exports, __webpack_require__) {
        const name = __webpack_require__("./src/commonjs/name.js");
        console.log(name);
    },
	"./src/commonjs/name.js": xxx
};

(function (modules) {
	function __webpack_require__() {...}
	return __webpack_require__("./src/commonjs/index.js");
})(modulesData);
```

首先可以看到，我们的**各个模块(文件)都被一个匿名函数包裹着**传入`module`, `exports`, `__webpack_require__webpack`三个参数（这就不难理解为什么可以直接在js里使用这几个变量了）。

通过一个自执行函数，将每个模块的路径及“包裹函数”以对象键值对`modulesData`的方式传给`modules`，函数体内，webpack自己实现了一个`__webpack_require__`，以入口文件`index.js`作为起点开始执行`__webpack_require__("./src/commonjs/index.js")`，并返回执行结果（剧透一下，返回的正是module.exports）。

下面看下`__webpack_require__`的实现。

``` javascript
// 每个模块的缓存
var installedModules = {};

function __webpack_require__(moduleId) {

    // 查看是否已缓存，有则直接返回exports对象
    if (installedModules[moduleId]) {
        return installedModules[moduleId].exports;
    }
    // 无缓存，则新建一个module对象
    var module = installedModules[moduleId] = {
        i: moduleId,
        l: false,
        exports: {}
    };

    // 重点：执行模块文件代码，也就是上面modulesData的数据
    modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);

    // 标识为已加载
    module.l = true;

    // 返回传说中的module.exports对象
    return module.exports;
}
```

这里的`moduleId`就是模块路径，如`./src/commonjs/index.js`。

> webpack4中只有optimization.namedModules为true，此时moduleId才会为模块路径，否则是数字id。为了方便开发者调试，在development模式下optimization.namedModules参数默认为true。

其中，`module`对象中的`exports`非常关键，对应的模块函数（上面所说的被匿名函数包裹着的模块）`modules[moduleId]`的`this`会绑定到`module.exports`后执行，传入`module`，`module.exports`以及`__webpack_require__`（也就是我们未编译前的require）。

``` javascript
function (module, exports, __webpack_require__) {
   const name = __webpack_require__("./src/commonjs/name.js");
   console.log(name);
}
```

最后返回`module.exports`。

### 1. commonjs：this/exports/module.exports

在commonjs中，exports和module.exports总容易弄混。

从上面可以看出，在模块函数里面，`this`和`exports`都指向于`module.exports`，因此也可以通过`this.name = 'elson'`，`exports.name = 'elson'`对外输出。但不能`this = 'elson'`，`exports = 'elson'`，原因很简单，这样`this`和`exports`不再指向最终返回的`module.exports`。

因此如果想对外输出一个基本数据类型或者函数的话，则只能赋值给`module.exports`。

``` javascript
exports.name = 'elson'; // require('xx').name === 'elson'
module.exports = 'elson'; // require('xx') === 'elson'
module.exports = function() {}; // require('xx')()
```

整个函数的实现要比想象中简单很多，与node里面的commonjs实现也基本一致。

### 2. ES6 module： export/export default

对于ES6的模块语法，webpack也同样支持（也就是说如果只用到es6的模块语法，是不需要babel）。

``` javascript
// ./src/esmodules/name.es.js
export let obj =  {a: 1, b: 2};
export function getName() {return 'elson';};
let name;
export default name = 'elson';

// ./src/esmodules/index.es.js
import * as all from './name.es.js';
console.log(all);
window.all = all;
```

在name.es.js中使用export/export default进行导出，在index.es.js使用import进行导入，看下webpack会打包成什么样子。

忽略掉webpack的runtime代码，我们写的模块会被打包成以下模样：

``` javascript
{
	"./src/esmodules/index.es.js":(
	    function(module, __webpack_exports__, __webpack_require__) {

	        "use strict";
	        __webpack_require__.r(__webpack_exports__);
	        /* harmony import */
	        var _name_es_js__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__("./src/esmodules/name.es.js");
	        console.log(_name_es_js__WEBPACK_IMPORTED_MODULE_0__);
	        window.all = _name_es_js__WEBPACK_IMPORTED_MODULE_0__;
	    }
	),

	"./src/esmodules/name.es.js":(
	    function(module, __webpack_exports__, __webpack_require__) {

	        "use strict";
	        __webpack_require__.r(__webpack_exports__);
	        /* harmony export (binding) */
	        __webpack_require__.d(__webpack_exports__, "obj", function() { return obj; });
	        /* harmony export (binding) */
	        __webpack_require__.d(__webpack_exports__, "getName", function() { return getName; });

	        let obj =  {a: 1,b: 2};
	        function getName() {return 'elson';};
	        let name;
	        /* harmony default export */
	        __webpack_exports__["default"] = (name = 'elson');
	    }
	)
}
```

这里用到了挂载在`__webpack_require__`上的两个函数`d`和`r`：
- `d`在exports对象上为某一属性设置getter函数。
- `r`在exports对象上设置属性`__esModule: true`。

``` javascript
// define getter function for harmony exports
__webpack_require__.d = function(exports, name, getter) {
    if(!__webpack_require__.o(exports, name)) {
        Object.defineProperty(exports, name, { enumerable: true, get: getter });
    }
};

// define __esModule on exports
__webpack_require__.r = function(exports) {
    if(typeof Symbol !== 'undefined' && Symbol.toStringTag) {
        Object.defineProperty(exports, Symbol.toStringTag, { value: 'Module' });
    }
    Object.defineProperty(exports, '__esModule', { value: true });
};
```

简单来说，对于ES6模块，webpack会先在`module.exports`对象上标记这是ES6模块，import进来的模块通过`__webpack_require__`加载，而export输出的值则通过`Object.defineProperty`设置响应的getter。

看到这里，可能会产生几个疑问：
1. 为什么要设置getter？
2. export default输出的值怎么没有设置getter？

设置getter是为了实现ES6模块的动态绑定，即export的值修改之后能够动态更新到import。但如果export default一个非函数或class，则不会动态绑定。如下：

``` javascript
// name.es.js
export let obj =  {a: 1,b: 2};
export let liveName = 'elson';
export function getName() {return 'elson';};
let deadName;
export default deadName = 'elson'; // default导出

// 3秒后修改导出值
setTimeout(() => {
    liveName = 'peter';  // 会更新
    deadName = 'peter'; // 不会更新
    obj.a = 222; // 会更新
    console.log('changed!!');
}, 3000);

// index.es.js
import * as all from './name.es.js';
console.log(all);
window.all = all;
```

此外由于ES6模块编译后使用了`Object.defineProperty`，因此无法兼容IE8-。如果有低版本IE兼容的需要，建议还是用回commonjs进行模块化，否则最后还得为此引入各种polyfill，得不偿失。

> **export var uses getter via Object.defineProperty that breaks <=IE8**
> https://github.com/webpack/webpack/issues/2729

### 3. 混合使用commonjs和ES module
webpack支持上述两者混合使用，我们可以`export default`导出`require()`引入，或者`module.exports`导出`import`引入。

从上面的打包代码分析可以知道，`export default`导出的值是挂载在default属性上，这也是为什么在一些混用场景下，需要通过`require().default`才能取到值。

而对于import引入`module.exports`导出的模块时，webpack做了如下处理：

编译前：

``` javascript
// a.js
module.exports = {elson: 'elson'};

// index.js
import elson from 'a.js';
console.log(elson);
```

编译后：

``` javascript
// index.js（部分代码）
var _a_js__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__( "./src/mixed/a.js");

var _a_js__WEBPACK_IMPORTED_MODULE_0___default = __webpack_require__.n(_a_js__WEBPACK_IMPORTED_MODULE_0__);

console.log(_a_js__WEBPACK_IMPORTED_MODULE_0___default.a);
```

通过`__webpack_require__`获取到`{elson: 'elson'}`后，传入`__webpack_require__.n`：生成一个getter函数，getter上定义a属性并指向getter，最后返回getter。而模块中**最终通过这个a属性来访问module.exports的值**。至于为什么要放到一个a属性上，这点还是不太理解，求各路大神指教。

``` javascript
__webpack_require__.n = function(module) {
    var getter = module && module.__esModule ?
        function getDefault() { return module['default']; } :
        function getModuleExports() { return module; };
    __webpack_require__.d(getter, 'a', getter);
    return getter;
};
```

## 二、webpack4的代码拆分加载机制
在项目中，如果一股脑把所有东西都打包到一个js中，那就只能唱首凉凉给你了。因此对第三方库、公共代码、按需加载的代码、甚至webpack的runtime代码进行拆分是常见的优化手段。下面了解一下如何准确配置拆分点以及运行时webpack是怎样加载被拆分了的代码。

### 1. 配置拆分点
webpack4使用`optimization.splitChunks`来配置拆分点，与webpack3的commonChunkPlugin相比，更加易操作、易理解。

#### （1）默认配置
[默认配置](https://webpack.js.org/plugins/split-chunks-plugin/#optimization-splitchunks)中，`optimization.splitChunks`只拆分通过`import()`引入的异步加载代码，官方文档的[案例](https://webpack.js.org/plugins/split-chunks-plugin/#examples)可更直观了解在默认配置下的打包结果。

``` javascript
// 默认的optimization.splitChunks配置
// webpack.config.js
module.exports = {
	...
	optimization: {
		splitChunks: {
			chunks: 'async', // 抽离类型：async、initial、all，默认是async
		    minSize: 30000, // 抽离包大小下限，默认超过30kb才会抽离
		    maxSize: 0, // 抽离包大小上限，抽离后大小若超过上限，且包含多个可再拆分的模块时，会再次拆分，保证单个文件不会过大
		    minChunks: 1, // 至少要有1个及以上的chunk共用同一模块才会抽离
		    maxAsyncRequests: 5,
		    maxInitialRequests: 3,
		    automaticNameDelimiter: '~',
		    name: true,
            cacheGroups: { // 可自行配置缓存组，组中的配置会覆盖掉父级的配置
		        vendors: { // 异步chunk中的第三方模块会单独抽离
		          test: /[\\/]node_modules[\\/]/,
		          priority: -10
		        },
		        default: {
		          minChunks: 2,
		          priority: -20,
		          reuseExistingChunk: true
		        }
		    }
        }
	}
};
```

#### （2）自定义拆分案例

``` javascript
// chunk1.js
import name from './name.js';
import zepto from 'zepto';
console.log(name);

// chunk2.js
import name from './name.js';
import(/* webpackChunkName: "math" */'./math.js').then(() => {console.log('math loaded!');}); // 通过注释指定异步chunk的名字
console.log(name);

// name.js 与 math.js各自无其他依赖
```

希望做到：
1. 抽离webpack的runtime代码
2. 抽离公共代码name.js
3. 抽离第三方库zepto为vendor.js

``` javascript
// webpack.config.js
module.exports = {
	entry: {
		chunk1: './src/esmodules/chunk1.js',
		chunk2: './src/esmodules/chunk2.js',
	},
	optimization: {
		runtimeChunk: 'single', // 抽离webpack的runtime代码
		splitChunks: {
			chunks: 'all', // 异步、非异步均纳入抽离范围
			minSize: 0, // 抽离包大小下限不做限制，30k以下的也抽离
			cacheGroups: {
				vendor: {
                    test: /[\\/]node_modules[\\/]/,
                    name: "vendor" // 抽离后的文件名称
                }
			}
		}
	}
};
```
打包结果：
- index.html
- 入口1：chunk1-559865ad.js
- 入口2：chunk2-3d2f5b4a.js
- 入口1~2公共代码(name.js)：chunk1~chunk2-6fa130db.js
- 异步加载的math.js：math-7512974a.js
- webpack runtime：runtime-dc502348.js
- 第三方库zepto：vendor-af69430f.js

值得一提的是如果引入了多个第三方库造成vendor.js太大的话，可以配置maxSize，当vendor超过max值时会拆成多个小包。结合http2，效果更佳。

### 2. 加载拆分代码机制分析
[html-webpack-plugin](https://webpack.docschina.org/plugins/html-webpack-plugin/) 会将上面的非异步脚本按照依赖顺序注入页面，下面我们看下具体webpack是怎样执行的。

``` html
<script type="text/javascript" src="runtime-dc502348.js"></script>
<script type="text/javascript" src="chunk1~chunk2-6fa130db.js"></script>
<script type="text/javascript" src="chunk1-559865ad.js"></script>
<script type="text/javascript" src="vendor-af69430f.js"></script>
<script type="text/javascript" src="chunk2-3d2f5b4a.js"></script>
```

我们已经把runtime代码单独抽离出来，那么除了runtime.js以外，其他脚本都长得非常类似：

``` javascript
// chunk1~chunk2.js
(window["webpackJsonp"] = window["webpackJsonp"] || []).push([
    ["chunk1~chunk2"],
    {"./src/esmodules/name.js": (function() {})}
]);

// chunk1.js
(window["webpackJsonp"] = window["webpackJsonp"] || []).push([
    ["chunk1"],
    {"./src/esmodules/chunk1.js": (function() {})},
    [["./src/esmodules/chunk1.js","runtime","chunk1~chunk2"]]
]);

// chunk2.js
(window["webpackJsonp"] = window["webpackJsonp"] || []).push([
    ["chunk2"],
    {"./src/esmodules/chunk2.js": (function () {})},
    [["./src/esmodules/chunk2.js","runtime","vendor","chunk1~chunk2"]]
]);
```

可以看出无论是入口chunk还是非入口chunk，都是将一个数组push进`window["webpackJsonp"]`。这个数组包括2-3个元素：
1. 各自的chunk名：如`["chunk1~chunk2"]`
2. chunk所包含的模块：如`{"./src/esmodules/name.js": (function() {})}`
3. 对于入口chunk来说，还有说明entry文件所依赖哪些chunk的数组：如`[["./src/esmodules/chunk2.js","runtime","vendor","chunk1~chunk2"]]`。

那么，这真的仅仅只是一个数组的push操作吗？下面看下核心——runtime.js还做了些什么。

``` javascript
// runtime.js（部分）
var jsonpArray = window["webpackJsonp"] = window["webpackJsonp"] || [];
var oldJsonpFunction = jsonpArray.push.bind(jsonpArray); // 将原生的push方法存起来
jsonpArray.push = webpackJsonpCallback; // 暂时劫持原生push方法
// jsonpArray === window.webpackJsonp true
jsonpArray = jsonpArray.slice(); // 复制一份webpackJsonp数组
// jsonpArray === window.webpackJsonp false
for(var i = 0; i < jsonpArray.length; i++) webpackJsonpCallback(jsonpArray[i]);
var parentJsonpFunction = oldJsonpFunction;
```

从`jsonpArray.push = webpackJsonpCallback;`可以看出webpack将数组的原生push方法劫持成了`webpackJsonpCallback`，因此在所有拆分的代码中执行的`window["webpackJsonp"].push()`实际上是执行`webpackJsonpCallback`，而真正的push操作则放在了`webpackJsonpCallback`里面进行。

``` javascript
function webpackJsonpCallback(data) {
    var chunkIds = data[0]; // ["chunk1"]
    var moreModules = data[1]; // {"./src/esmodules/chunk1.js": (function() {})}
    var executeModules = data[2]; // [["./src/esmodules/chunk1.js","runtime","chunk1~chunk2"]]

    var moduleId, chunkId, i = 0, resolves = [];
    for(;i < chunkIds.length; i++) {
        chunkId = chunkIds[i];

        // 此分支用于加载异步脚本，如math.js
        // true代表其值为Promise，意思是加载中
        if(installedChunks[chunkId]) {
            // 将promise的resolve函数推入数组，稍后批量执行
            resolves.push(installedChunks[chunkId][0]);
        }
        // 当前chunk置为已加载完成
        installedChunks[chunkId] = 0;
    }
    // 将当前chunk的所有模块都放入modules对象中
    for(moduleId in moreModules) {
        if(Object.prototype.hasOwnProperty.call(moreModules, moduleId)) {
            modules[moduleId] = moreModules[moduleId];
        }
    }
    // 在这里才真正执行push进数组的操作！window.webpackJsonp.push(data)
    if(parentJsonpFunction) parentJsonpFunction(data);

    // 执行异步脚本（如果有）的resolve，resolve后便会触发then回调
    while(resolves.length) {
        resolves.shift()();
    }

    // executeModules是入口chunk才会传入的参数
    // 将入口文件以及依赖推入deferred队列
    deferredModules.push.apply(deferredModules, executeModules || []);

    // 处理deferred队列
    return checkDeferredModules();
}
```

在`webpackJsonpCallback`中会处理两种脚本：已注入页面的非异步chunk以及按需加载的异步chunk（如math.js）。

#### （1）非异步chunk的加载

对于非异步chunk来说，经过`webpackJsonpCallback`的处理，已经将chunk中的所有模块都存进了`modules`对象中。这对于非入口chunk（如chunk1~chunk2.js）已经没什么需要处理的了，而对于入口chunk（如chunk1.js）则还需要执行entry模块如`./src/esmodules/chunk1.js`。

而这部分工作在`webpackJsonpCallback`末尾交给了`return checkDeferredModules()`处理。

``` javascript
function checkDeferredModules() {
    var result;
    // 以[["./src/esmodules/chunk1.js","runtime","chunk1~chunk2"]]为例
    for(var i = 0; i < deferredModules.length; i++) {
        var deferredModule = deferredModules[i];
        var fulfilled = true;
        // 第0项是入口文件，第1项开始是所依赖的chunk名称："runtime","chunk1~chunk2"
        for(var j = 1; j < deferredModule.length; j++) {
            var depId = deferredModule[j];
            // 检查所依赖的chunk是否已加载完毕
            if(installedChunks[depId] !== 0) fulfilled = false;
        }
        // 只有加载完毕才会执行入口文件
        if(fulfilled) {
            deferredModules.splice(i--, 1);
            // 执行入口文件./src/esmodules/chunk1.js
            result = __webpack_require__(__webpack_require__.s = deferredModule[0]);
        }
    }
    return result;
}
```
`checkDeferredModules`先检查所依赖的chunk是否都加载完毕，是的话才会执行入口文件。

#### （2）异步chunk的加载

最后来看下异步按需加载的chunk是如何加载的。

``` javascript
// chunk1.js
__webpack_require__.e("math")
	.then(__webpack_require__.bind(null, "./src/esmodules/math.js"))
	.then(() => {console.log('math loaded!')});
```

以math.js为例，我们在源码中通过`import('math.js')`标识其为需要按需加载的chunk。而webpack则是交由`__webpack_require__.e`函数，通过动态插入script来实现异步加载。

``` javascript
// 存储chunk加载状态
// undefined = chunk not loaded,
// null = chunk preloaded/prefetched
// Promise = chunk loading
// 0 = chunk loaded
var installedChunks = {
 	"runtime": 0
};

__webpack_require__.e = function requireEnsure(chunkId) {
    var promises = [];
    var installedChunkData = installedChunks[chunkId];

    /*部分省略*/

    var promise = new Promise(function(resolve, reject) {
        // 存储promise函数 即标识为加载中
        installedChunkData = installedChunks[chunkId] = [resolve, reject];
    });
    promises.push(installedChunkData[2] = promise);

    // 动态插入script实现异步加载
    var head = document.getElementsByTagName('head')[0];
    var script = document.createElement('script');
    var onScriptComplete;

    /*部分省略*/
    script.src = jsonpScriptSrc(chunkId);
    onScriptComplete = function (event) {/*超时及错误处理*/};
    script.onerror = script.onload = onScriptComplete;
    head.appendChild(script);

    return Promise.all(promises);
};
```
> `installedChunks[chunkid]`的值只有为true的时候才表示正在加载中。

整个math.js的异步加载过程需要结合`webpackJsonpCallback`进行理解。

1. 首先`__webpack_require__.e("math")`执行过程中会生成一个promise，将相应的resolve和reject函数，闭包存储在`installedChunks['math']`，此时值为true，表示加载中（第4步会用来做判断条件）；
2. 动态插入script，加载math.js，并返回promise；
3. 加载完毕后执行math.js：`window["webpackJsonp"].push()`，也就是执行`webpackJsonpCallback()`;
4. 判断`installedChunks['math']`是否为true(promise)，取出之前存储起来的`resolve`执行。
5. promise在resolved后自动执行then方法：`console.log('math loaded!')`


## 参考
- https://stackoverflow.com/questions/39276608/is-there-a-difference-between-export-default-x-and-export-x-as-default
- https://webpack.docschina.org/api/module-methods/
- https://webpack.docschina.org/plugins/split-chunks-plugin/#optimization-splitchunks
- https://github.com/happylindz/blog/issues/6
