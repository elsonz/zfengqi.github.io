---
layout:     post
title:      webpack4实用配置指南-上手篇
subtitle:
date:       2018-06-18
author:     elson
header-img:
header-bg-color: 337ab7
catalog: true
tags:
    - webpack
---

## 零、前言
算起来已经有3到4个项目的webpack构建打包经历。然而每次搭建起来还是有新手既视感，比较捉急。工具用法的东西，好记性不如烂笔头，有必要系统梳理下webpack的配置常用配置以及构建优化。

分为上手篇和优化篇，本篇为上手篇，先介绍常用配置。
篇幅较长，可完整阅读，也可在遇到问题时即查即用。

此次采用webpack4，也顺便尝尝鲜。

``` shell
# webpack4 把命令行工具抽离成了独立包 webpack-cli
npm install webpack webpack-cli -D
```

## 一、了解下webpack4的零配置
项目下没有`webpack.config.js`情况下，命令行直接运行`webpack`，webpack4不再像webpack3一样，提示未找到配置文件：
![](https://ws3.sinaimg.cn/large/006tKfTcgy1fselj0horwj30wm03q3yv.jpg)

而是提示：
![](https://ws4.sinaimg.cn/large/006tKfTcgy1fseliv5mkij31kw0ak403.jpg)

修改后可以发现零配置下系统的默认配置为：
1. 入口路径为：`/src/index.js`，打包输出路径：`/dist/main.js`
2. 未传`--mode`参数时，默认是`-mode production`，会进行压缩混淆。传入`--mode development`指定为开发环境打包。

![](https://ws1.sinaimg.cn/large/006tKfTcgy1fselin7spkj30qq092aao.jpg)

```
├── dist
│   └── main.js
├── node_modules
├── package-lock.json
├── package.json
└── src
    ├── index.js
    └── utils.js
```

体验了一把，所谓的零配置的灵活度极低，目测没啥实际用处，了解一下罢了。所以现阶段还是老老实实地学怎么配置吧。

## 二、webpack cli执行
如果命令行直接`webpack`会运行全局安装的webpack，想运行当前目录下的webpack，可以采取以下方法，不嫌麻烦当然也可以每次都`./node_modules/.bin/webpack`：

### npx webpack
npx是npm 5.2.0及以上内置的包执行器，`npx webpack --mode development`会直接找项目的/node_modules/.bin/里面的命令执行，方便快捷。

### npm run build
使用npm脚本，配置好之后直接`npm run xxx`

``` json
// package.json
"scripts": {
    "build": "webpack --mode development"
},
```

## 三、配置结构

``` javascript
// webpack.config.js
module.exports = {
    mode: 'development', // development|production
	entry: '', // 入口配置
	output: {}, // 输出配置
	module: {}, // 放置loader加载器，webpack本身只能打包commonjs规范的js文件，用于处理其他文件或语法
	plugins: [], // 插件，扩展功能
	// 以下内容进阶篇再涉及
	resolve: {}, // 为引入的模块起别名
	devServer: {} // webpack-dev-server
};
```

各个配置项的用法和细节将结合具体的功能实现来讲。

> [官方配置文档](https://doc.webpack-china.org/configuration/#%E9%80%89%E9%A1%B9)

## 四、基本的项目脚手架功能

###  1. 多入口配置
一个入口文件对应输出一个出口文件，因为太简单，不再赘述。这里讲下多对一、多对多。

这里涉及到webpack配置中的灵魂成员：`entry` 及 `output`

#### (1) 多进一出
`entry`传入数组相当于将数组内所有文件都打包到bundle.js中。

``` javascript
const path = require('path');

module.exports = {
	entry: ['./src/index.js', './src/index2.js'], // 入口文件
	output: {
		filename: 'bundle.js', // 打包输出文件名
		path: path.join(__dirname, './dist') // 打包输出路径（必须绝对路径，否则报错）
	}
};
```

#### (2) 多进多出
1. `entry`传入对象，key称之为chunk，将不同入口文件分别打包到不同的js。
2. `output.filename`改为用中括号占位来命名，从而生成多个文件，name是entry中各个chunk，具体可参考[官方文档](https://doc.webpack-china.org/configuration/output#output-filename)

``` javascript
const path = require('path');

module.exports = {
    entry: { // 入口文件，传入对象，定义不同的chunk（如app, utils）
        app: './src/index.js',
        utils: './src/utils.js'
    },
    output: {
        // filename: 'bundle.js', // 此时因为有多个chunk，因此不能只定义一个输出文件，否则报错
        filename: '[name].[hash].js',
        path: path.join(__dirname, './dist')
    }
};
```
PS. 在output中，还有一个叫`publicPath`非常重要，设置不正确会导致生成错的引用路径，从而找不到资源。这里先不展开，后面结合图片处理再细述。

### 2. 清空某目录或子目录及文件
这里先插入一个实用功能，因为在每次打包后，dist目录都有无用文件残留，最好每次打包前都清空dist目录。

``` shell
npm install -D clean-webpack-plugin
```

``` javascript
// webpack.config.js
const CleanWebpackPlugin = require('clean-webpack-plugin');
const cdnDir = path.resolve(__dirname, '../../../', 'imgcache.gtimg.cn/vip/')
module.exports = {
	...
	plugins: [
		new CleanWebpackPlugin(['dist']), // 清空项目根目录下dist
		new CleanWebpackPlugin(['dist/images', 'dist/style']),
		new CleanWebpackPlugin(['dist'], {
			root: cdnDir // 指定根目录 清空cdn目录下的dist
		}),
	]
};
```

### 3. html自动构建
回到正题，通过上面的配置，js已实现正确的进出关系，那该怎么引用呢，难道需要手动引入吗？下面看下怎样配置实现将html文件进行自动构建。这里需要借助插件。

``` shell
npm install html-webpack-plugin -D
```

``` javascript
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');

module.exports = {
	...
	plugins: [
		new HtmlWebpackPlugin({
			filename: path.join(__dirname, 'entry.html'), // 生成的html(绝对路径：可用于生成到根目录)
            filename: 'html/entry.html', // 生成的html文件名（相对路径：将生成到output.path指定的dist目录下）
            template: './src/index.html' // 以哪个文件作为模板，不指定的话用默认的空模板
        })
	]
};
```

在上面1的配置基础上加上plugins，就可以将打包文件自动注入到`entry.html`中，而且资源的引用路径都是正确的。

![](https://ws2.sinaimg.cn/large/006tKfTcgy1fseli8z04fj31kw07wq3o.jpg)

#### 多页面场景
需要构建多个页面，每个页面分别引用不同入口，怎么破？

``` javascript
module.exports = {
	...
	plugins: [
		new HtmlWebpackPlugin({
            filename: 'html/page1.html',
            template: './src/index.html',
            chunks: ['utils', 'app']
        }),
        new HtmlWebpackPlugin({
            filename: 'html/page2.html',
            template: './src/index2.html',
            chunks: ['app'],
            minify: {
                removeComments: true // 删除注释
            },
            hash: true // 加hash
        })
	]
};
```

要几个页面就new几个，通过`chunks`传入需要引用的入口。

![](https://ws2.sinaimg.cn/large/006tKfTcgy1fseli0qchfj31kw097aas.jpg)

#### 其他功能
从上面的配置可以看到，除了自动引用之外，html-webpack-plugin还提供了压缩、url后加hash等实用功能。具体参考[配置文档](https://github.com/jantimon/html-webpack-plugin#options)。


### 4. CSS处理——内联
有了JS和HTML，我们看看CSS该怎样自动内联进页面。

因为webpack原生具有了模块打包的能力，因此我们可以直接用commonjs规范，无需其他插件。而如果我们在js中直接require或者import了一个css文件，此时肯定是需要额外步骤告诉webpack该怎样处理。这里涉及到webpack另一个配置项：`module`及相关的`loader`。

下面以处理css以及less为例：
- less：先编译成css，再把css内联进页面。
- css：内联进页面

#### loader
处理less和css等非js资源，需要安装相对应的loader

``` shell
npm install -D css-loader # 负责处理其中的@import和url()
npm install -D style-loader # 负责内联

npm install -D less less-loader # less编译，处理less文件
```

#### module配置
我觉得`module`配置是webpack里面最繁琐的一块，光是配置loader就有三种不同的写法。下面只列出`loader`配置项，具体其他的module配置项可参见[官方文档](https://doc.webpack-china.org/configuration#%E9%80%89%E9%A1%B9)。

记住module的配置的其中一种套路：
- `module.rules[i].test`：命中规则
- `module.rules[i].use`：传入**数组**（loader名或对象数组），从右到左执行
- 或`module.rules[i].loader` ：传入字符串，这是`module.rules[i].use: [ { loader } ] `的简写。

``` javascript
// index.js
import './style/index.css';
import './style/test.less';

// webpack.config.js
module.exports = {
	...
	module: {
        rules: [
            {
                test: /\.css$/,
                // 从右到左，loader安装后无需引入可直接使用
                use: ['style-loader', 'css-loader']
            },
            {
                test: /\.less$/,
                use: [
                    {loader: 'style-loader'},
                    {loader: 'css-loader'},
                    {loader: 'less-loader'}
                ]
            }
        ]
    }
};
```

最终以style的形式内联进页面
![](https://ws1.sinaimg.cn/large/006tKfTcgy1fselhlmtphj30ns0d0dgs.jpg)

### 5. CSS处理——合并抽离
样式少可以内联，多了还是得抽离。而抽离文件已超过了loader的范围，需要借助plugins来完成：`extract-text-webpack-plugin`。

> BTW: 有了之前的html自动构建配置，抽离后的CSS也会自动引入

``` shell
# @next为webpack4使用版本
npm install -D extract-text-webpack-plugin@next
```

#### 抽离套路：
1. 实例化ExtractTextPlugin
	- **每个实例抽离成文件时是以entry为单位，所以一个入口文件（entry）只能抽出一个文件**，多entry时在设置filename需要注意[写法](https://github.com/webpack-contrib/extract-text-webpack-plugin#options)。
	- 可多次实例化，分别抽离CSS、LESS，下同
2. 将实例放入到`plugins`
3. 在css对应的`module.rules.use`调用`extract`方法。

``` javascript
const ExtractTextPlugin = require('extract-text-webpack-plugin');

// 实例化1：用于CSS
const extractCSS = new ExtractTextPlugin({
	disable: process.env.NODE_ENV == 'development' ? true : false, // 开发环境下直接内联，不抽离
    filename: 'style/extractFromCss.css', // 单个entry时，可写死
    filename: 'style/[name].css', // 多entry时
});
// 实例化2：用于LESS
const extractLESS = new ExtractTextPlugin({省略...});

module.exports = {
	module: {
		rules: [
			{
				test: /\.css$/,
				use: extractCSS.extract({
                    fallback: 'style-loader',
                    use: 'css-loader'
                })
			},
			{
                test: /\.less$/,
                use: extractLESS.extract({
                    fallback: 'style-loader',
                    use: ['css-loader', 'less-loader']
                })
            }
		]
	},
	plugins: [extractCSS, extractLESS]
};
```

![](https://ws2.sinaimg.cn/large/006tKfTcgy1fsell28av9j31g00c0myq.jpg)

#### 关于extract方法
这里将use的值改成extract之后，感觉怎么和上面说的套路又不一样了。

莫慌，其实它只是个语法糖，从返回值就知道，还是返回loader对象数组。

``` javascript
console.log(extractCSS.extract({
	fallback: 'style-loader',
    use: ['css-loader']
}));
```

![](https://ws4.sinaimg.cn/large/006tKfTcgy1fselpr8y03j318q03ut92.jpg)

此外，它可以作为实例方法也可以作为静态方法调用，见[源码](https://github.com/webpack-contrib/extract-text-webpack-plugin/blob/b254e96978dfbeb37b770640e2d264028f24c90d/src/index.js#L236)。

``` javascript
// 等价
extractCSS.extract()
ExtractTextPlugin.extract()
```

#### 关于options.fallback
顾名思义就是不被抽离时的降级处理。什么时候降级呢？
1. 实例时传入了`disable: true`
2. code splitting异步打包的文件内如果有引用样式，默认情况下这些样式不会被抽离，此时被降级。[参考博文](https://github.com/zhengweikeng/blog/issues/9)

如果异步文件也想抽离样式怎么办？用`allChunks`

``` javascript
const extractCSS = new ExtractTextPlugin({
	disable: false,
    filename: 'style/[name].css',
    allChunks: true //设置为true
});
```

### 6. 图片(字体/svg)处理
好了轮到图片、字体这些资源了。我们希望做到：
1. 图片能正确被webpack打包，小于一定大小的图片直接base64内联。
2. 打包之后各个入口（css/js/html）还能正常访问到图片，图片引用路径不乱。

字体和svg等资源同理，以下以图片为例。

#### (1) 安装依赖

``` shell
npm install -D url-loader file-loader
```

`url-loader`: 小于limit值时，直接base64内联，大于limit就干脆不管了，直接扔给`file-loader`处理，不装直接报错，之前还以为会自动调用，所以这两者都得装上。

#### (2) 不同入口（css/js/html）引用图片

**html**
html文件是通过html-wepback-plugin生成的，如果希望webpack能够正确处理打包之后图片的引用路径，需要在模板文件中这样引用图片。

``` html
<!-- 正确：会交给url-loader 或 file-loader -->
<!-- require让图片和html产生依赖引用关系 -->
<img src="<%= require('./images/sett1.png') %>" alt="">

<!-- 错误：原样输出，不做任何处理 -->
<img src="./images/sett1.png" alt="">
```

**css/js**

``` css
/* 图片作为背景图 */
#main {
    background: url("../images/cjl.jpg") #999;
    color: #fff
}
```

``` javascript
// app.js
import sett1 from './images/sett1.png';
const img = document.createElement('img');
img.src = sett1;
document.body.appendChild(img);
```

#### (3) 配置

``` javascript
// webpack.config.js
module.exports = {
	...
	modules: {
		rules: [
			{
				test: /\.(png|jpe?g|gif)$/,
				use: {
					loader: 'url-loader',
					options: {
						limit: 1024 * 8, // 8k以下的base64内联，不产生图片文件
						fallback: 'file-loader', // 8k以上，用file-loader抽离（非必须，默认就是file-loader）
						name: '[name].[ext]?[hash]', // 文件名规则，默认是[hash].[ext]
						outputPath: 'images/', // 输出路径
						publicPath: ''  // 可访问到图片的引用路径(相对/绝对)
					}
				}
			}
		]
	}
};
```

上述配置除了`limit`和`fallback`是url-loader([文档](https://www.npmjs.com/package/url-loader))的参数以外，其他配置项如`name`, `outputPath`都会透传给file-loader([文档](https://github.com/webpack-contrib/file-loader))。

关于`name`, `outputPath`, `publicPath`
1. 图片最终的输出路径：`path.join(outputPath, name)`
2. 图片的引用路径：
	- 指定了publicPath：`path.join(publicPath, name)`，这里会忽略掉outputPath
	- 否则用默认的output.publicPath：`path.join(__webpack_public_path__, outputPath, name)`

#### (4) 打包结果

**html**
![](https://ws2.sinaimg.cn/large/006tKfTcgy1fselu4ep1yj31j20dataw.jpg)

**css背景图**
![](https://ws1.sinaimg.cn/large/006tKfTcgy1fselu9yw6vj31ge0dcq3o.jpg)

**js动态插入**
![](https://ws2.sinaimg.cn/large/006tKfTcgy1fselulob84j30ww04kjrq.jpg)

咦，有没发现背景图引用路径不对？
根据上面的引用路径生成规则`path.join('', 'images/','[name].[ext]?[hash]')`，也就是各引用入口（html/css）必须和images目录同级才能访问到图片。而css放在了style目录下，与images不同级。

#### 引用路径不对？用`publicPath`修正
要解决上面的问题，可以在抽离css时设定publicPath。
``` javascript
extractCSS.extract({
	fallback: 'style-loader',
	use: 'css-loader',
	publicPath: '../' // 默认取output.publicPath
})
```

![](https://ws4.sinaimg.cn/large/006tKfTcgy1fsem38nqo3j31hc0ayjrx.jpg)

#### 【重点】来看下到底output.publicPath是什么？

publicPath的值会作为前缀附加在loaders生成的所有URL前面。
比如上面的`images/cjl.jpg`，如果设置了`output.publicPath:"../"`，那最终打包之后就会变成`../images/cjl.jpg`。

> The value of the option is prefixed to every URL created by the runtime or loaders. Because of this the value of this option ends with / in most cases.

它指定了output目录的访问路径，也就是浏览器怎样找到output目录。

> This option specifies the public URL of the output directory when referenced in a browser.

比如设置了`output.publicPath:"../"`，就说明output目录在html所在目录的上一级。
那这样设置了的话，css和html的目录层级关系并不符合要求，所以单独在extractCSS.extract中设置publicPath起到了覆盖output.publicPath的作用。

### 7. ES6转义

``` shell
npm install -D babel-core babel-loader babel-preset-env babel-preset-stage-0
```
- babel-core 核心包
- babel-loader
- babel-preset-env 定案内语法编译 （babel-preset-es2015已废弃）
- babel-preset-stage-0 预案内语法编译

``` javascript
// webpack.config.js
module: {
	rules: [
		{
	        test: /\.js$/,
	        exclude: /node_modules/, // npm包不做处理
	        include: /src/, // 只处理src里面的
	        use: {
	            loader: 'babel-loader',
	            options: {
	                presets: ['env', 'stage-0'] // 【重要】顺序右到左，先处理高级或特殊语法
                }
            }
        }
	]
}
```

options的内容也可以单独写在`.babelrc`

``` json
{
	"presets": ["env", "stage-0"]
}
```

### 8. webpack-dev-server

开发调试怎么少的了本地服务器。`npm install -D webpack-dev-server`。基础配置比较简单，参见[官方文档](https://webpack.docschina.org/configuration/dev-server/)，配置好之后直接`webpack-dev-server`即可。
下面提到的是使用时会遇到的一些问题。

#### contentBase和publicPath
contentBase和publicPath两个参数比较重要，设置错了的话会导致文件404
``` javascript
devServer: {
	contentBase: './public',
    publicPath: '/',
	host: 'localhost',
	port: 3000
}
```

``` shell
ℹ ｢wds｣: webpack output is served from /
ℹ ｢wds｣: Content not from webpack is served from ./public
```

##### (1) contentBase

> Content not from webpack is served from

也就是指定静态服务器的根目录，可以访问到**不通过webpack处理的文件**。

``` javascript
devServer: {
    index: 'main.html', // 为了可以展示目录
    contentBase: __dirname, // 默认值就是项目根目录
    host: 'localhost',
    port: 3000
}
```

启动服务器之后，访问`localhost:3000`，可以看到根目录的所有文件
![](https://ws1.sinaimg.cn/large/006tKfTcgy1fs67u4auqej31de0e0glp.jpg)

需要说明下的是，为了演示，如果不设置`index: 'main.html'`，`localhost:3000`会直接访问项目入口文件`index.html`。

举个具体例子，如果我们在入口文件直接引入`<link rel="stylesheet" href="public/test.css">`，这个资源不会经过webpack处理，因此最终访问路径是`localhost:3000/public/test.css`。

同理，当`contentBase: path.join(__dirname, 'public')`时，访问`localhost:3000`只会显示public下的文件。因此在出现文件404时，检查下引用的资源url是否和contentBase里的文件一一对应。

如果所有的静态资源文件都会经过webpack打包，其实可以直接设置为`false`，此时可以再试试访问`localhost:3000` (Cannot GET /)。

##### (2) publicPath
> webpack output is served from

对于webpack打包的文件：虽然我们指定了打包输出目录dist，但是实际上并不会生成dist，而是打包后直接传给devserver，然后放到内存中。不过可以通过：`http://localhost:3000/webpack-dev-server`查看打包目录下的文件。
![](https://ws2.sinaimg.cn/large/006tKfTcgy1fs66gcb45cj30rw0e674l.jpg)

publicPath是告诉浏览器通过什么路径去访问上面的webpack打包目录。默认值是`/`。也就是说我们可以通过：`http://localhost:3000/index.html`,`http://localhost:3000/style/vutify.css`来访问打包文件。这点要和contentBase的静态文件服务器区分开。

另一个容易导致文件404的是：把publicPath设置为打包目录`/dist`。这样的话，就需要多加一层：
`http://localhost:3000/dist/index.html`,
`http://localhost:3000/dist/style/vutify.css`才能访问。

#### 热更新 HMR
webpack-dev-server在试图重新加载整个页面（LiveReload）之前，会尝试使用热更新（HMR）来更新。

``` javascript
devServer: {
	host: 'localhost',
	port: 3000,
	hot: true // 还需要在plugin中配置new webpack.hotModuleReplacementPlugin()
}
```

可以发现在更新css以及html文件时，页面是不会刷新的(css-loader/html-webpack-plugin已具备热更新功能)，但更新js时会。因此可以在入口文件添加以下代码，实现热更新。

``` javascript
// index.js
import './dependency.js';

if (module.hot) {
	module.hot.accept();
}
```

#### 域名与代理

##### 场景1：
真机上访问devServer，进行开发、调试、体验。

解决方法：
host指定为无线网卡的ip，如`192.168.0.104`，PC与其他移动设备处于同一wifi环境下时即可访问。

``` javascript
devServer: {
    host: '192.168.0.104', // 默认值是localhost
    port: 3000,
    hot: true
}
```

##### 场景2：
后台数据接口为`http://c.y.qq.com/xxx`，如果用`localhost:3000`访问的话，会遇到跨域的问题，需要使用`y.qq.com`域名访问。

解决方法：
将本地服务器放在80端口上（Mac下需要sudo起服务），配置host：`y.qq.com 127.0.0.1`，此时使用`http://y.qq.com/`即可访问本地服务器。

``` javascript
devServer: {
    host: '127.0.0.1',
    port: 80,
    hot: true,
    allowedHosts: [ // 允许哪些域名访问devServer
	    'y.qq.com'
    ]
}
```

##### 场景3：
后台服务搭在本地8360端口，页面在3000端口。

``` javascript
// bad: 入侵代码
const url = process.env.NODE_ENV == 'development'
	? 'http://127.0.0.1:8360/common/getmultiple'
	: '/common/getmultiple'
```

解决方法：

``` javascript
devServer: {
    host: '127.0.0.1',
    port: 3000,
    hot: true,
    proxy: {
	    "/common/getmultiple": "http://localhost:3000"
    }
}
```

proxy配置项非常强大，可以pathRewrite重写请求路径，bypass绕过代理等，这里抛砖引玉，其他功能参考[devserver-proxy官方文档](https://webpack.docschina.org/configuration/dev-server/#devserver-proxy)。

## 五、未完待续
经过上面的配置，实现了基本的常用的脚手架功能，对webpack的entry、output、publicPath、loader等灵魂级配置项以及dev-server有了一定的理解，弄清它们的关系之后，常见的问题自然迎刃而解。在优化篇会分享一下code spliting、dll预编译等优化及提升开发体验的功能，敬请期待。
