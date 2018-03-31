---
layout:     post
title:      js继承
subtitle:   在《javascript模式》一书中，把继承分为了类式继承和原型继承，以下是对这部分的一个总结。
date:       2015-07-12
author:     elson
header-img: 
header-bg-color: 337ab7
catalog: true
tags:
    - javascript
---

## 类式继承
### 1.最常用的继承组合模式 —— 借用构造函数 & 设置原型
```javascript
    // 父类
    function Parent(name) {
        this.name = name;
    }

    Parent.prototype.show = function () {
        return this.name;
    };

    // 子类 继承Parent
    function Child(name) {
        // 借用构造函数，只能继承父类this的属性
        Parent.apply(this, arguments);
    }

    // 设置原型 继承父类this属性以及父类的原型
    Child.prototype = new Parent();
```
缺点：父构造函数被调用了两次，从而导致同一个属性会被继承两次（this.name）

### 2. 共享原型
在上面的模式当中，`Child.prototype = new Parent();` 其实是有缺陷的。

1. 继承了父类自身的属性
2. 继承了父类的原型属性(方法)

基于上面的问题，如果 `Child.prototype` 不指向 Parent的实例 `new Parent()`，而是指向 `Parent.prototype` 的话，那么问题就可以解决了。

但是，如果直接共享原型
```javascript
    Child.prototype = Parent.prototype;
```
这样很明显是有问题的，因为对Child原型的修改，会影响到所有对象和祖先对象！

### 3. 临时（代理）构造函数
要想实现 2.共享原型 的理念，而又不出现上面的问题，则需要**切断父对象和子对象原型的直接链接关系**。

所以**实现方式**是：声明一个空白的函数，用这个空白函数充当子对象和父对象之间的代理。

**类式继承的圣杯模式**
```javascript
    function inherit(Child, Parent) {
        // 空白的代理函数
        function F() {}

        // 代理函数与父原型建立直接链接关系
        F.prototype = Parent.prototype;

        // 因为F函数是空白的，所以也不存在模式1当中的问题
        Child.prototype = new F();

        // 修正constructor指向（此修复会使得constructor变成可枚举）
        Child.prototype.constructor = Child;
    }
```
### 4. 总结
根据上述1~3，可以总结出以下的继承模式
```javascript
    // 父类
    function Parent(name) {
        this.name = name;
    }

    Parent.prototype.show = function () {
        return this.name;
    };

    // 子类 继承Parent
    function Child(name) {
        // 借用构造函数，只能继承父类this的属性
        Parent.apply(this, arguments);
    }

    // 用 3.代理构造函数
    inherit(Child, Parent);

    // 与 3.代理构造函数 等价 【Object.create 实现见下面】
    Child.prototype = Object.create(Parent.prototype);
```

## 原型继承

原型继承并不涉及到类，这里的对象都是继承自其他对象。

其实指的就是ES5里面的 `Object.create()` 方法。
```javascript
    var parent = {
        name: "dad"
    };

    var child = Object.create(parent);

    console.log(child.name); // "dad"
```
以下是 `Object.create()` 方法的polyfill
```javascript
    if (typeof Object.create === 'undefined') {
        Object.create = function (o) {
            function F() {}
            F.prototype = o;
            return new F();
        };
    }
```
###通过复制属性实现继承 —— 浅复制 & 深复制

继承的目的是为了实现代码复用，所以一个对象要从另一个对象中获取功能，把目标对象的属性和方法复制过来也是一种方法。
```javascript
    $.extend(child, parent);
    // 这时，child对象就具有parent对象的属性和方法，即被扩展（extend）了
```
####浅复制
```javascript
    function clone(parent, child) {
        child = child || {};

        for (var key in parent) {
            if (parent.hasOwnProperty(key)) {
                child[key] = parent[key];
            }
        }
        return child;
    }
```
####深复制
```javascript
    function cloneDeep(parent, child) {
        child = child || {};

        for (var key in parent) {
            if (parent.hasOwnProperty(key)) {
                if (typeof parent[key] === 'object') {
                    child[key] Object.prototype.toString.call(parent[key]) === '[object Array]' ? [] : {};
                    cloneDeep(parent[key], child[key]);
                }
                else {
                    child[key] = parent[key];
                }
            }
        }
        return child;
    }
```
