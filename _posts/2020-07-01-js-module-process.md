---
layout: post
title: "JS的模块化历程"
subtitle: 'CMD下的sea.js, ES6的标准'
author: "Hux"
header-style: text
tags:
  - module
  - AMD
  - CMD
  - ES6
  - sea.js
  - require.js
---

## 一. 伪模块

### 1. 文件拆分

只是简单把每一个JS文件当作一个模块，每一个模块内部只是定义的各种函数，形如：

```javascript
// module math.js
function add(num1, num2) {
  // 实现代码
}

function multiply(num1, num2) {
  // 实现代码
}
```

这种方式缺点很多：

- 通过`HTML`文件中`<script>`引入。
- 模块依赖文件，假设A模块依赖B模块，而A模块的`<script>`标签在B模块的`<script>`标签前，那A模块就完了。
- 全局变量污染，每个模块定义的变量都在全局对象中，后续引入的模块会覆盖同名变量。
- 模块内部的变量无法做到私有性，一旦模块外部无意中修改了模块内部的变量，那么该模块很可能无法正常工作了。

### 2. 自执行 + 闭包

通过自执行函数以及闭包，可以解决全局变量污染和模块内部私有性的问题，形如：

```javascript
// module counter.js
(function counter() {
    var click = 0;
    function init(click) {
        this.click = click
    };
    function start(callback) {
        var worker = setInterval(() => {
            console.log(this.click)
            if (this.click <= 0) {
                clearInterval(worker)
                callback()
                return
            }
            else this.click--
        }, 1000)
    };
    window.counter = {
        init,
        start
    }
})()
```

成员内部变量`click`只有其模块允许改变，模块外部是无法访问的，这种方式已经有点模块的味道了。



## 二  模块化

### 1 Require.JS - AMD规范

和`Sea.js`大体相似，原`definiton`写法是：

```javascript
//Calling define with module ID, dependency array, and factory function
define('myModule', ['dep1', 'dep2'], function (dep1, dep2) {

    //Define the module value by returning a value.
    return function () {};
});
```

提前写好模块依赖关系，并且在`factory`的参数表中进行依赖模块的注入，后来也提供了`require`方式。

```javascript
define(function (require) {
    var dependency1 = require('dependency1'),
        dependency2 = require('dependency2');

    return function () {};
});
```

但无论是何种依赖注入方式，都是**提前加载和执行的**。

### 2 Sea.JS - CMD规范

CMD全称`Common Module Definition`通用模块定义标准，定义了模块的基本书写格式和交互规则。`sea.js`[官方文档](https://seajs.github.io/seajs/docs/#docs)。

#### 2.1 基本书写格式

##### 2.1.1  `define`

`define`是一个全局函数，用于进行一个模块的定义，具体参数表如下：

```javascript
define(id?:String, dependencies?:Array, factory:Any|)
```

**`id`指定该模块的全局唯一标识**，**`dependencies`指定当前定义中的模块所依赖的其他模块**，这两个参数是**可选的**，开发中不写这两个参数可以方便开发，但是可能会有潜在的坑点（详见[为什么要有约定和构建工具](https://github.com/seajs/seajs/issues/426)）：

- 不手写`id`参数，假设使用构建工具诸如`webpack`、`gulp`等等把多个JS模块文件合并了之后，由于一个JS文件下有多个`define`，会导致引入模块时无法区分引入哪个`define`。
- 代码压缩后，没有 `id`指定参数的话，在 IE6-9 下也有可能会出现问题。
- 提供`dependencies`参数的话，`sea.js`需要通过正则表达式提取模块定义体内的依赖模块路径，会带来一点性能上的提升。

##### 2.2.2 `factory`

 `factory`可以是对象、字符串、数字等任意的数据类型，一般来说`factory`会是一个函数，是该模块的构造函数，参数表如下：

```javascript
function(require:Function, exports:Object, module:Object)
```

`require`用于引入其他的模块。`exports`是一个对象，通过为其定义一个属性来为该模块暴露供外部使用的接口。`module`是一个对象，代表当前定义中的模块。注意，直接把`exports`重新赋值为一个新的对象是不行的，因为`exports`参数其实是`module`的`exports`属性的一个引用。

#### 2.2 交互规则 

##### 2.2.1  `require`

`require`也是一个函数，但最好把它当作一个**关键字**，它**用于在模块定义体中引入其他的模块**，`require`函数**只接受一个字符字面量**，不能是表达式，因为`require`存在类似`var变量提升`的现象，此时表达式还不能求值，所以只能用字符字面量。

```javascript
var moduleA = require('./js/modules/math')
```

`require`可以接受相对路径和绝对路径，绝对路径是相对于`seajs.config`中`base`字段指定的基础路径的，具体参考[模块标识](https://github.com/seajs/seajs/issues/258)。

##### 2.2.2 `seajs.use`

该函数用于定义一个总入口，和`require`类似也是用于引入模块，但是`seajs.use`是用于在HTML页面中加载模块，详见[模块的加载启动](https://github.com/seajs/seajs/issues/260)。

```javascript
<script src="path/to/sea.js"></script>
<script>
  seajs.use('./main');
  //or
  seajs.use(['./m1', './m2'])
  //or
  seajs.use('./main', function() {
      // 加载完main模块后的回调函数
  })
</script>
```

### 3 ES6标准

> waiting...

### 4 规范间的区别

- CMD规范中的模块是**延迟加载**，只有在**第一次*`require`时才加载

- AMD规范中的模块是**提前加载**的，会提前确立好模块间的依赖关系，无论是否真的用到了都一并加载。

  ```javascript
  define(["a", "b", "c", "d", "e", "f"], function(a, b, c, d, e, f) {
      // 等于在最前面申明并初始化了要用到的所有模块
     if (false) {
         // 即便压根儿没用到某个模块 b，但 b 还是提前执行了
         b.foo()
     }
  })
  ```

  



###   