---
layout:     post
title:      来，vue弹窗插件走一个
subtitle:   vue弹窗插件开发、插件中使用slot
date:       2018-05-12
author:     elson
header-img:
header-bg-color: 337ab7
catalog: true
tags:
    - vuejs
---

# 来，vue弹窗插件走一个

## 零、前言
记得有一次组内分享，以弹窗为例讲了如何创建可复用的vue组件，后面发现这个例子并不恰当（bei tiao zhan），使用组件需要先import，再注册，然后再按照`props in events out`原则使用，无论从流程或者使用方式来说都相当麻烦。

每个页面在使用弹窗时如果都按照这个流程走一遍的话，我们的脸基本上就黑了。

弹窗应该是插件，注册一次永久使用，如`this.$alert('QQ音乐')`。下面我们就一起撸一个试试。

> 以下例子在[vuetify.js](https://vuetifyjs.com/zh-Hans/components/dialogs)的弹窗`v-dialog`组件基础上进行，[这里查看完整demo源码](https://github.com/zfengqi/vue-dialogs-plugin-demo)。



## 一、如何安装插件

``` javascript
// 引入插件
import dialogs from './plugins/dialogs';

// 安装
Vue.use(dialogs, {title: 'QQ音乐'});

new Vue({
    el: '#app',
    render: h => h(App)
})
```

是不是很眼熟，和vue-router用法一样，只要调用`Vue.use()`，传入插件和初始化参数即可。重点就是传入的`dialogs`到底是什么。

## 二、dialogs插件开发
插件开发步骤在[官方文档](https://cn.vuejs.org/v2/guide/plugins.html)已经说得很清楚，可以看下。下面我们具体到dialogs这个插件上，来看看怎么实现。

``` javascript
// dialogs.js
import Dialog from '../components/Dialogs.vue';
const dialogs = {
	install(Vue, options) {
		Vue.prototype.$alert = (opt = {}) => {};
		console.log('installed!');
	}
};

export default dialogs;
```

要求很低，只要export的对象里有`install`方法，其他的怎么折腾都可以。

调用`Vue.use()`实际上就是调用`install`方法，它会传入Vue对象和在use时传入的初始化参数`{title: 'QQ音乐'}`。

可在install中添加全局/实例方法。

### 1. 弹窗调用方式
支持传入字符串，配置对象，支持指定回调函数，支持连续调用（用于二次确认）。

``` javascript
this.$alert('你好');

this.$confirm({
    hideOverlay: false,
    title: '我是弹窗',
    content: '你好',
    btnTxt: ['取消', '呵呵']
});

// 连环调用
this.$confirm({
    content: '二次确认',
    btnTxt: ['取消', '不要拦我'],
    cb: (btnType) => {
        if (btnType == 1) {
            this.$confirm({
                content: '三次确认',
                btnTxt: ['好吧我放弃', '去意已决'],
                cb: (btnType) => {
                    btnType == 1 && this.$alert('成功rm -rf /*');
                }
            });
        }
    }
});
```

### 2. $alert核心实现

``` javascript
Vue.prototype.$alert = (opt = {}) => {
	...
	// 创建包含组件的Vue子类
	let Dialogs = Vue.extend(Dialog);
	// 实例化，将组件放置在根DOM元素
    let vm = new Dialogs({el: document.createElement('div')});
    // 将上面实例使用的根DOM元素放到body中
    document.body.appendChild(vm.$el);

	// 保存当前弹窗实例
	this.vm = vm;

	...
	// 以下代码与Dialogs.vue实现有关
	// 显示弹窗组件
	vm.show = true;
	vm.$on('close', () => {
		// 收到弹窗关闭事件时，移除根元素，并销毁实例
		document.body.removeChild(vm.$el);
        vm.$destroy();
        this.vm = null;
	});
};
```

### 3. components/Dialogs.vue实现
从上面可以看到，$alert其实就是换了种方式调用组件，以下是Dialogs.vue的实现（对vuetify.js中的`v-dialog`的进一步封装）。

1. `show`和`dialogShow`：组件显示隐藏
2. `type: 'alert' || 'confirm'`：弹窗类型（按钮个数）
3.  `title`或`slot name="title"`：标题
4.  `content`或`slot name="content"`：正文
5.  `btnTxt`：按钮个数及文案
6.  `closeDialog()`：按钮点击处理
7.  `this.$emit('close', btnNo, this.type);`：触发弹窗关闭事件，并告知按钮编号

组件的实现细节说明这里不过多展开。

``` htmlbars
<!-- Dialogs.vue -->
<template>
    <v-dialog v-model="dialogShow" persistent :width="width" :hide-overlay="hideOverlay">
        <v-card style="background:#fff;">
            <v-card-title>
                <div class="headline">
                    <template v-if="title">
                        {{title}}
                    </template>
                    <slot v-else name="title"></slot>
                </div>
            </v-card-title>
            <v-card-text v-if="content" v-html="content"></v-card-text>
            <slot v-else name="content"></slot>
            <v-card-actions>
                <v-spacer></v-spacer>
                <v-btn
                    v-for="(item, idx) in btnTxt"
                    v-if="type == 'confirm' || (type == 'alert' && idx == 0)"
                    :key="idx"
                    class="green--text darken-1"
                    flat="flat"
                    @click.native="closeDialog(idx)"
                >
                    {{item}}
                </v-btn>
            </v-card-actions>
        </v-card>
    </v-dialog>
</template>

<script>
export default {
    props: {
        show: {
            type: Boolean,
            default: false
        },
        title: {
            type: String,
            default: '提示'
        },
        content: {
            type: String,
            default: ''
        },
        type: {
            type: String,
            default: 'alert'
        },
        btnTxt: {
            type: Array,
            default: function () {
                return ['我知道了'];
            }
        },
        width: {
            type: Number,
            default: 300
        },
        hideOverlay: {
            type: Boolean,
            default: false
        }
    },
    data() {
        return {
            dialogShow: this.show
        }
    },
    watch: {
        show(showUp) {
            this.dialogShow = showUp;
        }
    },
    methods: {
        closeDialog(btnNo) {
            this.dialogShow = false;
            this.$emit('close', btnNo);
        }
    }
}
</script>
```

### 4. $alert传参
下面看下从调用`this.$alert(opt)`开始，怎样与默认参数结合，最终传递到Dialog.vue中去的。

``` javascript
Vue.prototype.$alert = (opt = {}) => {
	...
	// 默认参数
    let defaultOpt = {
	    type: 'type',
	    title: 'QQ音乐',
	    content: '',
	    btnTxt: ['好的'],
	    width: 300,
	    cb: null
    };

	// 传入字符串时指定为content
    if (typeof opt == 'string') {
	    defaultOpt.content = opt;
    }

	// 覆盖关系：调用参数 -> 插件安装时初始化参数 -> 默认参数
    opt = {...defaultOpt, ...installOptions, ...opt};

	let Dialogs = Vue.extend(Dialog);
    let vm = new Dialogs({el: document.createElement('div')});
    document.body.appendChild(vm.$el);
    this.vm = vm;

    // 最终传参给组件实例
    Object.assign(vm, opt);

	...
};
```

### 5. 其他细节处理

#### `$alert`和`$confirm`逻辑复用
这两个弹窗其实就是type值不一样，因此将公共逻辑进行抽离复用。

``` javascript
// dialog.js

const dialog = {
	vm: null,
	create(componentType = 'alert', Vue, installOptions, opt) {
		// 之前$alert的逻辑抽离到这里
	},
	install(Vue, options) {
		Vue.prototype.$alert = (opt = {}) => {
            this.create('alert', Vue, options, opt);
        };

		Vue.prototype.$confirm = (opt = {}) => {
            this.create('confirm', Vue, options, opt);
        };
	}
};
```

#### 多次点击时，防止页面同时出现多个弹窗
之前的处理是：多次点击按钮时，~~销毁之前的弹窗~~。
这样就会造成其他弹窗干扰当前弹窗，当前弹窗会直接消失。

其实应该实现弹窗队列：同时多处调用弹窗方法，此时应该放进队列里，待当前弹窗消失后，再调取队列执行。

``` javascript
const dialogs = {
    vm: null, // 保存当前实例
    queue: [],
    create(componentType = 'alert', Vue, installOptions, opt) {
        // OUTDATE: 多次点击按钮时，销毁之前的弹窗
        // UPDATE: 改为：当前弹窗未关闭再次调用时，保存到栈
        if (this.vm) {
            this.queue.push({type: componentType == 'confirm' ? '$confirm' : '$alert', opt});
            return;
        }

        ...

        vm.$on('close', (btnType) => {
            setTimeout(() => {
                document.body.removeChild(vm.$el);
                vm.$destroy();
                typeof opt.cb == 'function' && opt.cb((componentType == 'confirm' && btnType == 1) ? 1 : 0);
                this.vm = null;
                // 查看栈中有无未执行的弹窗
                if (this.queue.length > 0) {
                    let cur = this.queue.shift();
                    Vue.prototype[cur.type](cur.opt);
                }
            }, 400); // 缓出动画为300ms，因此延迟400ms后再销毁实例
        });
    }
}
```

#### 待缓出动画结束后再销毁实例

``` javascript
vm.$on('close', (btnType) => {
    setTimeout(() => {
        document.body.removeChild(vm.$el);
        vm.$destroy();
        this.vm = null;

        typeof opt.cb == 'function' && opt.cb((componentType == 'confirm' && btnType == 1) ? 1 : 0);
    }, 400); // 缓出动画为300ms，因此延迟400ms后再销毁实例
});
```

## 三、如何在插件中使用slot
实际上弹窗不应该只局限于在标题和正文中显示文字和html结构，如果想传入其他vue组件，实现一个上传文件的弹窗，像下面这样是不行的。

``` javascript
this.$confirm({
    content: `
    <v-flex xs10 offset-xs1 class="mr10">
        <v-text-field
            prepend-icon="attachment"
            single-line
            v-model="fileName"
            :label="label"
            required
            readonly
            ref="fileTextField"
            @click.native="onFileInputFocus"
        ></v-text-field>
        <input type="file" :style="{position:'absolute', left: '-9999px'}" :multiple="true" ref="fileInput" @change="onFileChange">
    </v-flex>`
});
```

结果是会原封不动将未编译的vue组件标签直接塞入dom。
![](https://ws4.sinaimg.cn/large/006tKfTcgy1fr8y8k3if5j31kw0ikdht.jpg)


**这个时候需要借助slot。**

在上面的Dialogs.vue中，title和content是支持传入`slot`。那么在插件中怎样传入slot？我们尝试在`$alert` `$confirm`基础上新增一个`$uploadFile`方法。

``` javascript
Vue.prototype.$uploadFile = (opt = {}) => {
    ...
    // 模板
    const slotTemplate = `传入上例中content的内容`;
	// 编译模板，返回渲染函数
    const renderer = Vue.compile(slotTemplate);

    const slotContent = {
        data() {
            return {uploadShow: false, fileName: '', label: '', formData: null}
        },
        methods: {
            onFileChange($event) {
                ...
            },
            onFileInputFocus() {
               ...
            },
            getFormData(files) {
                ...
            }
        },
        render: renderer.render,
        staticRenderFns: renderer.staticRenderFns
    };
    // 将相关内容赋给名为content的slot
    vm.$slots.content = [vm.$createElement(slotContent)];
}
```

> 然而在官方文档中看到 `vm.$slots`是只读，这有点费解。

## 后记
以上是对开发vue弹窗插件的梳理总结，vue的插件机制很强大，弹窗涉及的范围比较有限，有机会再对其他复杂插件开发以及vue源码进行研究。

> 我的博客即将搬运同步至腾讯云+社区，邀请大家一同入驻：https://cloud.tencent.com/developer/support-plan?invite_code=6mq9jujn5bb2
