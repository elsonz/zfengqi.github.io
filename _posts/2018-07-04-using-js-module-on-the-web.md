---
layout:     post
title:      【译】浏览器的ES Modules
subtitle:
date:       2018-07-04
author:     elson
header-img:
header-bg-color: 337ab7
catalog: true
tags:
    - 模块化
---

> 原文：《Using JavaScript modules on the web》
> https://developers.google.com/web/fundamentals/primers/modules

标题是意译的，原文说的JS modules，实际上就是ES6的模块化特性，通过`<script type="modules">`可以实现不经过打包直接在浏览器中import/export，此玩法确实让人眼前一亮。

先看看`<script type="modules">`的[兼容性](https://caniuse.com/#feat=es6-module)。目前只有较新版本的chrome/firefox/safari/edge支持此特性，看来要普及使用还任重道远。下面跟着这篇文章深入了解一下涨涨姿势。
![](https://ws4.sinaimg.cn/large/006tKfTcgy1fslletg3xaj31kw0ppabf.jpg)

本文将介绍JS模块化；怎样在不经过打包的情况下直接在浏览器中使用模块化；以及Chrome团队在JS模块化的优化和普及上正在做的一些事情。

## JS模块化

你可能用过命名空间、CommonJS或者AMD规范进行JS模块化，但所有的这些模块系统万变不离其宗：既可以将其他模块引入(import)，也可以作为一个模块输出(export)。目前ES6正式将模块化的语法进行统一。在一个模块中，你可以使用`export`关键字输出任何东西：`const`、`function`等。

``` javascript
// lib.mjs
export const repeat = (string) => `${string} ${string}`;
export function shout(string) {
  return `${string.toUpperCase()}!`;
}
```

然后你可以用`import`关键字从另一个模块中引进来。下面代码将lib模块中的`repeat`和`shout`函数引到了我们的主模块main中。

``` javascript
// main.mjs
import {repeat, shout} from './lib.mjs';
repeat('hello');
// → 'hello hello'
shout('Modules in action');
// → 'MODULES IN ACTION!'
```

你也可以通过`default`关键字，输出一个默认值。

``` javascript
// lib.mjs
export default function(string) {
  return `${string.toUpperCase()}!`;
}
```

而通过上面的`default`输出的模块，在引入时可以用其他任何变量名。

``` javascript
// main.mjs
import shout from './lib.mjs';
//     ^^^^^
```

模块脚本与常规脚本有所区别：

- 模块脚本默认开启了严格模式
- 不支持HTML风格的注释`<!-- comment -->`
- 模块具有词法顶级作用域。也就是说在模块中`var foo = 42;`并不会像传统脚本一样，创建一个全局变量`foo`，可以通过`window.foo`访问。
- 新的`import`和`export`语法仅限于在模块脚本中使用，不能用在常规脚本中。

正因为这些差异，模块脚本和传统脚本显然需要各自不同的解析方式。因此JS解析器需要标识出哪些脚本属于是模块类型的。

## 浏览器如何识别模块脚本

你可以通过设置` <script> `元素的`type`属性为`module`，以此告诉浏览器这段script需要以模块进行处理。

``` html
<script type="module" src="index.mjs"></script> <!--下文称作模块脚本-->
<script nomodule src="fallback.js"></script> <!--下文称作传统脚本-->
```

那些支持`type=module`的浏览器会忽略掉`nomodule`的脚本，而不兼容也会优雅降级，执行fallback.js。

> 译者注：亲测在IE7+到edge，oppo系统浏览器都能够降级而执行fallback.js。不过加载fallback的同时，也会把index.mjs一并加载，而支持module的浏览器则不会加载fallback。
> ![IE](https://ws1.sinaimg.cn/large/006tKfTcly1fsq7licceij30cw073dfp.jpg)
>IE系列均会执行fallback.js
>![IE Network](https://ws4.sinaimg.cn/large/006tKfTcly1fsq7ks6impj30po058aaf.jpg)
>加载fallback的同时，也会把index.mjs一并加载
>![](https://ws1.sinaimg.cn/large/006tKfTcly1fsq7luj29vj30u504xjrj.jpg)
>而支持module的浏览器则只会加载模块

有没想过另外一个好处：既然浏览器能够识别module，那它必然也能够支持ES67的其他特性，如箭头函数、async-await。你不需要为这些特性进行babel编译，现代浏览器跑着更小和最大部分未编译的模块化代码，而不兼容的则使用nomodule的降级代码。

### 浏览器加载方面的异同：模块脚本vs传统脚本

上面介绍了模块脚本和传统脚本在语言层面的异同，除此之外，在浏览器加载过程中也有所不同。

#### 同样的模块脚本只会执行一次，而传统脚本会声明多次。

``` html
<script src="classic.js"></script>
<script src="classic.js"></script>
<!-- classic.js executes multiple times. -->

<script type="module" src="module.mjs"></script>
<script type="module" src="module.mjs"></script>
<script type="module">import './module.mjs';</script>
<!-- module.mjs executes only once. -->
```

#### 模块脚本跨域需要加跨域头
模块脚本及其依赖是通过CORS来获取的，也就是说模块脚本一旦跨域就需要加上适当的返回头，比如`Access-Control-Allow-Origin: *`。而众所周知，传统脚本则不需要（译者注：还记得传说中的JSONP吗）。

#### async属性对内联脚本有效

``` html
<script async>var test = 1;</script>
<!-- async无效 -->

<script async type="module">import {a} from './a.mjs'</script>
<!-- async有效 -->
```
加了async属性会使得脚本在下载过程中不阻塞DOM渲染，而下载完成后立即执行，两个async脚本之间的执行时序不确定，执行时机也不确定，有可能在domContentLoaded之前或者之后。但这一属性对传统的内联脚本是无效的，而对模块的内联脚本却是有效的。

### 关于`.mjs`文件后缀

你可能会对前面的`.mjs`后缀感到好奇，但是在互联网的世界里，文件后缀并不重要，只要服务器下发的MIME类型(`Content-Type: text/javascript`)正确就可以。浏览器是通过script标签上的type属性来识别模块脚本的，而不是后缀名。

所以无论使用`.js`还是`.mjs`都是可以的。但是我们还是建议使用`.mjs`，原因有两个：
1. 在开发的时候，可以不需要看代码，通过后缀名非常直观地看出哪些是模块脚本。
2. nodejs中，[ES6的模块化特性仍在实验性阶段](https://nodejs.org/api/esm.html)，而该特性只支持`.mjs`后缀的脚本。

### 模块资源标识符 - module specifier

在import一个模块时，后面的相对或绝对路径字符串称为module specifier或import specifier，也就是模块资源路径。

``` javascript
import {shout} from './lib.mjs';
//                  ^^^^^^^^^^^
```

浏览器对于模块资源路径做了一些限制。不支持类似下面这种只有模块名或部分文件名的资源路径（称之为bare module specifiers）。这样的限制是为了以后浏览器在支持自定义模块加载器之后，加载器能够自行决定bare module specifiers的解析方式。

``` javascript
// Not supported (yet):
import {shout} from 'jquery';
import {shout} from 'lib.mjs';
import {shout} from 'modules/lib.mjs';
```

目前，模块资源路径必须是完整的URL，或者以`/`,`./`,`../`开头的相对URL

``` javascript
// Supported:
import {shout} from './lib.mjs';
import {shout} from '../lib.mjs';
import {shout} from '/modules/lib.mjs';
import {shout} from 'https://simple.example/modules/lib.mjs';
```

### 模块script默认是defer
传统脚本的加载和解析会阻塞html的解析，可以通过添加[`defer`属性](https://html.spec.whatwg.org/multipage/scripting.html#attr-script-defer)解决（让脚本加载和html解析并行）

![](https://ws4.sinaimg.cn/large/006tKfTcgy1fsrdj7ntvyj31kw0d30tm.jpg)

但这里想告诉你的是，模块脚本默认具备defer的并行功能，因此无需画蛇添足加上defer属性。还有不仅仅只有主模块与html解析并行，其他子模块也一样。

## ES模块化的其他特性

### 动态引入： `import()`

我们之前仅仅用到了静态的`import`，它需要在首屏就把全部模块资源都下载下来。但有时候按需加载或异步加载会更为合理，这有助于提高首次加载时间，而[`import()`](https://developers.google.com/web/updates/2017/11/dynamic-import)可以用来解决这个问题。

``` html
<script type="module">
  (async () => {
    const moduleSpecifier = './lib.mjs';
    const {repeat, shout} = await import(moduleSpecifier); // lib会在主模块及其依赖都加载并执行完毕之后才会import
    repeat('hello');
    // → 'hello hello'
    shout('Dynamic import in action');
    // → 'DYNAMIC IMPORT IN ACTION!'
  })();
</script>
```

不像静态`import`只能用在`<script type="module>"`一样，动态`import()`也可以用在普通的script。具体可以看下我们[关于动态import的文章](https://developers.google.com/web/updates/2017/11/dynamic-import)。

> NOTE: [Webapck自己实现了一套`import()`方案](https://developers.google.com/web/fundamentals/performance/webpack/use-long-term-caching)，可以动态将import()进去的模块抽离出来，生成单独的文件。

### import.meta

另一个和ES模块相关的新特性是`import.meta`，它能提供关于当前模块的meta信息。准确的meta信息并不是ECMAScript规范指定的部分，它取决于宿主环境。在浏览器拿到的meta信息和在nodejs里面拿到的是有区别的。

下面的例子中，图片的相对路径默认是基于HTML所在位置来解析的，但通过`import.meta.url`可以实现基于当前模块来解析。

``` javascript
function loadThumbnail(relativePath) {
  const url = new URL(relativePath, import.meta.url);
  const image = new Image();
  image.src = url;
  return image;
}

const thumbnail = loadThumbnail('../img/thumbnail.png');
container.append(thumbnail);
```

## 性能优化建议

### 继续使用打包工具

通过模块脚本，开发时我们可以无需再用webpack、Rollup、Parcel等打包工具就可以享受原生的模块化福利，在以下场景建议可以直接使用原生的模块脚本：
1. 开发环境下
2. 不超过100个模块且相对较浅的依赖层级关系（小于5）的小型web应用

However, as we learned during our bottleneck analysis of Chrome’s loading pipeline when loading a modularized library composed of ~300 modules, the loading performance of bundled applications is better than unbundled ones.

然而，我们[性能瓶颈分析](https://docs.google.com/document/d/1ovo4PurT_1K4WFwN2MYmmgbLcr7v6DRQN67ESVA-wq0/pub)中发现在加载一个模块化库（大约300个模块），经过打包的性能数据要比未经过打包直接使用原生模块脚本的好。

![](https://ws2.sinaimg.cn/large/006tKfTcgy1fstpb081k8j30xc0ciq3d.jpg)

其中一个原因是`import`/`export`语法是可以静态分析的，因此可以帮助打包工具分析并移除那些并未使用过的模块。从这可以看出，静态的`import`/`export`不仅仅只是语法特性，还具备关键的工具属性！

> 我们的总体建议是继续使用打包工具进行上线前的模块打包处理。毕竟从某种程度上，打包可以帮助你尽可能减少代码体积，用户不必要加载无用的脚本，更有利于页面性能。

[开发者工具的代码覆盖率检查](https://developers.google.com/web/updates/2017/04/devtools-release-notes#coverage)能帮助你检测源码中是否存在无用代码。我们同时也建议通过代码分割对模块进行合理拆分，以及延迟加载非首屏关键路径的脚本。

**打包与使用模块脚本的权衡取舍**

通常在web开发领域，所有方案都有利弊，需要权衡取舍。与加载一个未经过代码拆分的打包脚本相比，使用模块脚本也许会降低首次加载性能（cold cache），但是可以提升用户再次加载（warm cache）的速度。比如对于总大小200KB的代码，在修改一个细颗粒化的模块之后，那么用户只需要更新有变更的代码，这总比重新加载所有代码（打包脚本）要强。

如果相对于首次访问体验来说，你更关注用户再次访问体验，并且你的应用不超过数百个细颗粒化模块的话，你不妨尝试下使用模块脚本，通过性能数据对比之后再做出最后的选择。

浏览器工程师们正努力提升模块脚本的性能，我们希望模块脚本以后能够适用于更多的应用场景。

### 使用细颗粒化的模块

尽可能让你的代码以细颗粒化的模块进行组织。当在开发时，每个模块最好不要输出过多的内容。

下面的`./util.mjs`模块，输出了`drop` `pluck`和`zip`三个函数。

``` javascript
export function drop() { /* … */ }
export function pluck() { /* … */ }
export function zip() { /* … */ }
```

如果你的代码仅仅只需要`pluck`，你也许会这样引入：

``` javascript
import { pluck } from './util.mjs';
```

在这种情况下，如果没有构建打包编译，浏览器会还是会下载、解析和编译整个`./util.js`模块，即使只仅仅需要其中一个export。

如果`pluck`不与`drop`和`zip`有引用或依赖关系的话，最好还是将它独立成一个模块`./pluck.mjs`。以达到无需加载其他无用函数的目的。

``` javascript
export function pluck() { /* … */ }
```

这不仅能够让你的源码简洁，还能够减少对打包工具（移除冗余代码）的依赖。如果在你的应用中其中一个模块从未被`import`过，那么浏览器就不会去下载。而那些真正有用的模块则会被浏览器[缓存](https://v8project.blogspot.com/2018/04/improved-code-caching.html)起来。

此外，使用细颗粒化的模块也有助于对接[未来的浏览器原生打包功能](https://developers.google.com/web/fundamentals/primers/modules#web-packaging)。

### 预加载模块

通过`<link rel="modulepreload">`你可以进一步优化模块加载。浏览器会预加载甚至预解析和编译这些模块及其依赖。

``` html
<link rel="modulepreload" href="lib.mjs">
<link rel="modulepreload" href="main.mjs">
<script type="module" src="main.mjs"></script>
<script nomodule src="fallback.js"></script>
```

这对于有复杂依赖关系模块的应用尤为重要。没有`rel="modulepreload"`,浏览器需要发出多个HTTP请求来计算出整个依赖关系。而如果你把所有依赖模块通过`rel="modulepreload"`提前告诉浏览器，那么浏览器则无需再渐进式地去计算。

### 采用HTTP/2协议

HTTP/2支持[多路复用](https://developers.google.com/web/fundamentals/performance/http2/#request_and_response_multiplexing)，多个请求及响应信息可以同时进行传输，这有助于提高模块树的加载效率。

Chrome团队还预研了[服务器推送](https://developers.google.com/web/fundamentals/performance/http2/#server_push)——另一个HTTP/2特性，是否能够作为部署高度模块化应用的一个可行方案。但结局令人失望，[HTTP/2的服务器推送比想象中要难以应用](https://jakearchibald.com/2017/h2-push-tougher-than-i-thought/)，并且web服务器及浏览器的对其实现目前并没有针对高度模块化web应用进行优化。另一方面，服务器很难只推送未被缓存的资源。如果通过告知服务器完整的用户缓存状态来解决这个问题的话，又存在隐私泄露风险。

无论如何，采用HTTP/2协议吧！只要记住目前HTTP/2的服务器推送还不能作为一个好的解决方案。

## 目前的使用情况

ES modules正在缓慢地被接纳使用。我们的[使用统计](https://www.chromestatus.com/metrics/feature/timeline/popularity/2062)显示只有0.08%（不包括动态`import()`或者[worklets](https://drafts.css-houdini.org/worklets/)）的页面目前使用了`<script type="module">`。

## ES Modules未来的发展

Chrome团队正在通过不同的方式，致力于提高基于ES modules的开发体验。下面列举其中的几种。

### 更高效、确定性更高的模块解析算法

我们提交了一版对于目前模块解析算法的优化。新算法目前已经被同时列入了[HTML规范](https://github.com/whatwg/html/pull/2991)和[ECMASciprt规范](https://github.com/tc39/ecma262/pull/1006)，并且已在[Chrome 63](http://crbug.com/763597)版本中实现。希望这项优化能够在更多的浏览器中落地。

新算法更快更高效，旧算法在计算依赖图谱（dependency graph）大小的时间复杂度为O(n²)，旧时Chrome也是一样。但新算法则提升至O(n)。

此外，新算法在报解析错误时更加准确。如果一个依赖图谱（graph）中有多个错误，那么基于旧算法，每次执行都会报不同的解析错误。这给开发调试带来不必要的困难。新算法则保证每次执行都会报相同的解析错误。

### Worklets 和 web workers

Chrome实现了[worklets](https://drafts.css-houdini.org/worklets/)，允许web开发者自定义那些在浏览器底层的硬编码逻辑。目前开发者可以将一个JS模块引入到渲染管道（pipeline）或者音频处理管道。

Chrome65版本支持了[`PaintWorklet`](https://developers.google.com/web/updates/2018/01/paintapi)，也称为CSS绘制API（the CSS Paint API），用于控制如何绘制一个DOM元素。

``` javascript
const result = await css.paintWorklet.addModule('paint-worklet.mjs');
```

Chrome66版本支持了[`AudioWorklet`](https://developers.google.com/web/updates/2017/12/audio-worklet)，允许开发者注入自定义的音频处理代码。同时这个版本开始了[`AnimationWorklet`的公测](https://groups.google.com/a/chromium.org/d/msg/blink-dev/AZ-PYPMS7EA/DEqbe2u5BQAJ)，开发者可以创造视差滚动效果(scroll-linked)以及其他高性能程序动画（procedural animations）。

最后，[`LayoutWorklet`](https://drafts.css-houdini.org/css-layout-api/)，又称为CSS布局API（the CSS Layout API）已在Chrome67版本中实现。

我们正在对Chrome中的[web workers支持传入模块脚本](https://bugs.chromium.org/p/chromium/issues/detail?id=680046)。你可以通过输入`chrome://flags/#enable-experimental-web-platform-features`开启这个特性。

``` javascript
const worker = new Worker('worker.mjs', { type: 'module' });
```

在shared workers和service workers传入模块脚本也即将支持。

``` javascript
const worker = new SharedWorker('worker.mjs', { type: 'module' });
const registration = await navigator.serviceWorker.register('worker.mjs', { type: 'module' });
```

### 包名映射表 - Package name maps

在nodejs/npm中，我们经常会通过它们的包名引入模块，比如：

``` javascript
import moment from 'moment';
import { pluck } from 'lodash-es';
```

[根据现行的HTML规范](https://html.spec.whatwg.org/multipage/webappapis.html#resolve-a-module-specifier)，类似上述的包名写法（bare import specifiers）会抛出异常。我们提交的“包名映射表”提案将会支持上述写法（包括在生产环境）。该映射表（JSON格式）将帮助浏览器将包名转换为完整资源路径（full URLs）。

包名映射表目前仍处于提案阶段（proposal stage）。

### Web packaging：浏览器原生打包

Chrome loading团队正在探索[一种原生的web打包格式](https://github.com/WICG/webpackage)（下称为web packaging），作为一种新模式来分发web应用。web packaging的主要特性如下：
1. [Signed HTTP Exchanges](https://wicg.github.io/webpackage/draft-yasskin-http-origin-signed-responses.html)：可以让浏览器信任某个HTTP请求对（request/response）确实是来自于所声明的源服务器。
2. [Bundled HTTP Exchanges](https://wicg.github.io/webpackage/draft-yasskin-wpack-bundled-exchanges.html)：是多个请求对的集合，不要求当中的每个请求都进行签名（signed），只要携带某些元数据（metadata）用于描述如何将请求束作为一个整体来解析。

两者结合起来，这种web打包格式就能够将多个同源资源安全地整合到一个HTTP GET相应中。

市面上的打包工具如webpack、Rollup、Parcel，都会将多个模块最终打包成一个或少数几个bundle，这会导致源码中进行的模块拆分在上线后就丧失了它的意义。那么通过原生打包，浏览器可以将bundle反解成原样。

简单来说，你可以把一个HTTP请求对包（Bundled HTTP Exchange）理解为一个资源文件包，它可以通过目录表（manifest）随意访问，并且里面的资源能够被高效地缓存以及根据相对优先级的高低来标记。有了这个机制，原生模块能够提升开发调试的体验。当你在Chrome开发者工具查看资源时，浏览器会精准定位到原生的模块代码中，而不需要复杂的source-map。

Chrome已经实现了一部分提案（SignedExchanges），但是打包格式（bundling format）以及在高度模块化app中的应用仍在探索阶段。


### Layered APIs
移植新的功能和API到浏览器中无可避免会带来持续性的维护以及运行成本。每一个新特性都会污染浏览器的命名空间，增加启动开销，并且也增大引入bug的可能性。Layered APIs的目的是以一种更具扩展性的方式通过浏览器来实现或移植一些高级API。而模块脚本是实现Layered APIs的一项关键技术。
- 由于模块是显式引入的，所以通过模块来引入layered APIs可实现按需使用（不会默认内置）。
- 模块的加载源可自定义，因此layered APIs实现了一套自动加载polyfill（当不支持时）的机制。

The details of how modules and layered APIs work together are still being worked out, but the current proposal looks something like this:

模块脚本和layered APIs如何协同运作，具体细节[仍在制定中](https://github.com/drufball/layered-apis/issues)，但目前的协议如下：


``` html
<!-- src中竖杠后面是指定polyfill的路径，浏览器不支持时可自动加载，不错的降级方式 -->
<script
  type="module"
  src="std:virtual-scroller|https://example.com/virtual-scroller.mjs"
></script>

<virtual-scroller>
  <!-- Content goes here. -->
</virtual-scroller>
```

这个模块脚本引入了`virtual-scroller`API，如果浏览器支持则会直接读取内置layered APIs集合（std:virtual-scroller），反之则网络加载对应的polyfill。

> 译者：对于Layered APIs更多的中文介绍 https://zhuanlan.zhihu.com/p/37008246


## 译者后记

算是第一篇完整翻译的技术文章，英语水平有限，英语水平好的同学可以直接看原文。翻译讲求信达雅，不少地方在自己的理解下进行了意译，理解有误的地方请各路大神赐教。咦已经过了12点了，给今年的生日留做纪念。
