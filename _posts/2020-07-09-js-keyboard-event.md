---
layout: post
title: "POS的快捷键管理模块"
subtitle: '发布订阅模式'
author: "lee"
header-style: text
tags:
  - keyboard event of javascript
---


## POS的快捷键管理模块

首先POS的快捷键管理模块（以下统称keyboard）是订阅发布模型的，但是又多了一点特殊之处，原因在于POS本身是一个单页面应用，同一时刻下只有一个页面处于显示状态，但是所有页面的`ViewModel`都向keyboard订阅了键位事件且都存在于内存中未被销毁，所以keyboard多了一个逻辑就是__只执行特定的一个订阅者（即处于显示状态的页面）的回调函数__，否则会出现当前A页面在显示，但是却执行了非显示状态下的B页面的回调函数，这样就乱套了。

keyboard内部会维护三个数组作为优先级队列，按优先级倒序分别是topQueue、normalQueue、bottomQueue。keyboard只会执行优先级最高且处于队头的已注册页面（以下统称订阅者）的回调函数，在后面会说到的publish函数会体现出来，至于为什么要设计这样的优先级队列，其实我还不太明白，可能有其他的地方用到了吧。

那keyboard是怎么知道该通知哪个订阅者页面呢？或者说是什么把当前显示中页面移到队头的呢？答案在`switchPage.js`中可以找到，就是页面切换的时候会执行页面VM的keyboard函数，keyboard函数中需要自己把会把该页面移到队列队头的操作。

keyboard我在原代码上稍微改了一下，函数内容都没变，只改了一下定义方式和变量名，或许会更好理解一点，代码如下：

### keyboard模块

keyboard模块由两个部分组成，一个是订阅者，一个是消息中心。

#### 订阅者

```javascript
function Listener(opts) {
	this._opts = opts;
	this._callback = {};
}

Listener.prototype = {
      /**
       * 订阅键盘键位事件
       * @param {} key 要订阅的键位 
       * @param {*} callback 回调函数 
       */
      on: function (key, callback) {
        // 根据键位进行回调函数的分类，若无指定键位，则任何键位的按下都会触发该回调函数
        if (arguments.length == 1) {
          callback = key;
          key = "All";
        }
        this._callback[key] = this._callback[key] || [];
        this._callback[key].push(callback);
      },
    
      /**
       * 收到通知时的的执行函数，消息中心监听到键位事件时会调用订阅者的该方法。
       * @param {*} key 
       * @param {*} keyCode 
       * @param {*} event 
       */
      onKey: function (key, keyCode, event) {
        var callback = this._callback[key] || [];
        callback.forEach(function (c) {
          c && c(key, keyCode, event);
        });
        callback = this._callback["All"] || [];
        callback.forEach(function (c) {
          c && c(key, keyCode, event);
        });
      }
    
      /**
       * 页面切换时，就需要调用该方法把自己移到所属队列的队头
       */
      toTop: function () {
        var queue = null;
        if (this._opts.isBottom) {
          queue = bottomQueue;
        } else if (this._opts.isTop) {
          queue = topQueue;
        } else {
          queue = normalQueue;
        }
        var index = queue.indexOf(this);
        if (index == -1) {
          queue.unshift(this);
        } else {
          queue.splice(index, 1);
          queue.unshift(this);
        }
        return this;
      },
          
      /**
       * 页面隐藏时，如果自己处于队头且是优先级最高的队列，最好调用该方法把自己从所属队列中移除，否则会          * 出现当前显示中的页面因为优先级不够高，即使处于队头也无法收到消息中心的通知。
       */
      remove: function () {
        var queue = null;
        if (this._opts.isBottom) {
          queue = bottomQueue;
        } else if (this._opts.isTop) {
          queue = topQueue;
        } else {
          queue = normalQueue;
        }
        var index = queue.indexOf(this);
        if (index != -1) {
          queue.splice(index, 1);
        }
        return this;
      },
          
      /**
       * 挂起自己，作用是即使当前this指向的订阅者处于队头，也不会触发其回调函数。
       * 作用？原因在于DOM的事件冒泡，冒泡顺序：keydown -> keypress -> input。当用户在输入框输  		* 入时，需要调用这个方法把自己挂起，否则会触发快捷键事件。
       */
      suspend: function () {
        this._opts.suspend = true;
      },

      /**
       * 唤醒自己，允许执行回调函数
       */
      resume: function () {
        this._opts.suspend = false;
      },
}

```

#### 消息中心

```javascript
 define(function (require, exports, module) {
    // 当前已注册的订阅者（页面）
    var instances = {};
	// 三个优先级队列
    var topQueue = normalQueue = bottomQueue = []
    // 页面注册的时候的默认配置
    var defaultOpts = {
      isTop: false,
      isBottom: false,
      suspend: false,
      win: "main"
    };
     
    // 监听键位事件，keypress：非修饰键键位按下，keydown：键位按下
    $(document).on("keypress", function (event) {
      var key = String.fromCharCode(event.charCode);
      // keypress监听会有其他情况，key存在
      if (event.keyCode == 13) key = "Enter";
      publish(key, event);
    });

    $(document).on("keydown", function (event) {
      var key = codeKeyTable[event.keyCode];
      publish(key, event);
    });
     
    /**
     * 监听到键位事件后，通知优先级最高的那个订阅者
     * @param {*} key 用户按下的键位
     * @param {*} event 键位事件
     */
    function publish(key, event) {
      if (!key) return;
      // 获取窗口
      var win = null;
      // 这里可以看到，永远只有处于优先级队列队头的那一个订阅者才可以收到通知，所以不明白设置三个队列的作		  用...
      if (topQueue.length > 0) {
        win = topQueue[0];
      } else if (normalQueue.length > 0) {
        win = normalQueue[0];
      } else {
        win = bottomQueue[0];
      }
      win && !win._opts.suspend && win.onKey(key, event.keyCode, event);
    }

     /**
     * 供页面注册成为订阅者，keyboard模块会把所有的订阅者收集到instances对象中。
     * @param {*} opts 
     */
    function WinSingletonRegistry(opts) {
      if (typeof opts == "string") {
        opts = {
          win: opts
        }
      }
      opts = $.extend({}, defaultOpts, opts);
        // 全局单例，一个页面只能成为一个订阅者
        if (!instances[opts.win]) {
        instances[opts.win] = new Listener(opts);
      }
      return instances[opts.win];
    }
    
    // 暴露一个该方法供页面在VM中注册成为订阅者
    module.exports = WinSingletonRegistry;

 }
```

keyboard模块的实现就是上面的样子，对照着原代码应该能更好的理解。

### 页面路由

上面说到，所有页面在初始化时都会在消息中心中注册成为订阅者，而消息中心在监听到键位事件时，只会通知优先级最高的那__一个__订阅者，下面就是优先级最高的那一个订阅者的诞生过程。

switchPage.js中的`switch()`有这样的代码，`switch()`在页面切换时会执行。

```javascript
// switchPage.js
pageTVM.keyboard && pageTVM.keyboard();
```

pageTVM即page.js，代码如下：

```javascript
// page.js
keyboard() {
    let self = this;
    self.allPageModule[this.VM.activePage]['initKeyboard'] 
    && 				      
    self.allPageModule[this.VM.activePage]['initKeyboard']();
},
```

每个页面的VM中必须手写一个`initKeyboard`函数并在里面调用订阅者实例的`toTop()`把自己移到队列的队头，哈哈哈...

### 举个例子

假设现在有个页面testPage，下面是testPage的ViewModel要做的事情。

```javascript
define(function (require, exports, module) {
    // 1. 引入keyboard模块
    const WinSingletonRegistry = require("UU/keyboard")
    // 2. 注册成为订阅者
    const listener = WinSingletonRegistry({
        win: "testPage",
        isTop: true
    });
    // 3. 订阅 
    // 订阅所有键位
    // or
    // 订阅特定的键位，比如订阅键位‘m’: listener.on('m', callback)
    listener.on((key) => {
        console.action('you click char m');
        testPage._VM.msg = 'you click ' + key;
        keyboard.toTop()
    })
    // 页面的ViewModel
    const testPage = {
        _VM: undefined,
        init() {
            this._VM = new Vue({
                el: '#testPageEl',
                template: view.template('page/testPage/testPage'),
                data: {
                    msg: 'i am test page'
                }
            })
        },
        // 4. 页面切换时，switchPage.js中会调用这个函数
        initKeyboard() {
            listener.toTop()
        },
        // 5. 页面隐藏即显示其他页面时，switchPage.js会调用这个函数
        afterHide() {
            listener.remove()
            // WinSingletonRegistry.remove('testPage');
        }
    }
    module.exports = testPage
}
```

其实`listener.toTop()`和`listener.remove()`这两个操作可以由消息中心完成，不用每个页面自己维护，页面注册成为订阅者之后只需要订阅即可，这样的设计或还许有别的细节。

___

> 延伸阅读

## `KeyboardEvent`

`KeyboardEvent`对象描述了用户与键盘的交互。每个事件都描述了用户与__一个按键或一个按键和修饰键的组合__。

> 关于修饰键：
>
> Windows下，修饰键常用的就是ctrl、shift、alt、CapsLock、NumLock等等键位，具体参考[这里](https://developer.mozilla.org/zh-CN/docs/Web/API/KeyboardEvent/getModifierState)。

### 事件类型

#### keydown

一个键位被按下时触发一个此事件，按住不放会持续触发。

#### keypress

一个非修饰键键位被按下时触发此事件。

> 先触发keydown之后才会再触发keypress，所以在keydown事件的回调函数中调用[Event.prototype.preventDefault()][https://developer.mozilla.org/en-US/docs/Web/API/Event/preventDefault]的话会阻止keypress的触发。

#### keyup

一个键位被释放。

### 常用属性

#### key

返回按下的键位的可打印字符串。

- 如果是字符键位，则返回一个非空的Unicode字符串。
- 如果是修饰键位，则返回一个硬编码预定义的值，参考[这里][https://developer.mozilla.org/zh-CN/docs/Web/API/KeyboardEvent/key/Key_Values]。

- 如果按键无法识别，则返回 `"Unidentified"`。

#### code

该属性的值与键位在键盘上的物理位置有关，因此键盘的布局不会影响这个值。



