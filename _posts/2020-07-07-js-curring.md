---
layout: post
title: "JS的函数柯里化"
subtitle: '函数式编程'
author: "lee"
header-style: text
tags:
  - Javascript
  - curring function
---
# `Vue` && 函数柯里化

函数柯里化是一种利用闭包特性实现使一个原本接受多个参数的函数变为一系列接受部分参数的函数并返回最终结果的技术。

所有函数柯里化具有两个作用

- 延迟执行
- 参数缓存

## 一阶函数柯里化

一阶函数柯里化是我取的，也不知道对不对。

```javascript
/**
 * 基础柯里化函数
 * @param {Function} fn 执行函数 
 * @param  {...any} rest 部分参数
 */
function basicCurrying(fn, ...rest) {
    return function(...rest2) {
        return fn.apply(null, rest.concat(rest2))
    }
}

module.exports = {
    basicCurrying
}
```

使用柯里化包装原函数，第一次先接受部分的参数值，再返回一个接受剩余参数的函数。这里其实就是利用了闭包来保存第一次接受的部分的参数值，因为返回的匿名函数中访问了`basicCurrying`函数的执行上下文中的`rest`变量，所以在`basicCurrying`的执行上下文出栈后，它的变量对象不会被垃圾回收掉，且因为在返回的匿名函数的执行上下文的创建阶段，它的作用域链中会存有`basicCurrying`的变量对象，所以返回的匿名函数可以访问到`basicCurrying`执行上下文的变量。

```javascript
const {basicCurrying} = require('./lib/currying.basic')

function infiniteAdd(...param) {
    return param.reduce((p, c) => {
        return p + c
    }, 0)
}

const curringInfiniteAdd = basicCurrying(infiniteAdd, 1, 2, 3)
console.log(curringInfiniteAdd(4, 5, 6)) 
```

应用就像上面的例子一样，可以使得`infiniteAdd`这个函数延迟执行且缓存了部分参数。

> [Array.prototype.reduce](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce)

普通的柯里化有个缺点就是延迟执行的次数是固定的两次，要是想延迟执行两次，就必须把柯里化返回的匿名函数再进行一次柯里化，要想延迟执行10次，就得10次柯里化...

比如，现在有一个打印日志的函数如下

```javascript
function logMethod(methodName, message, spendTime) {
    console.log(`method: ${methodName}\t message: ${message}\t spendTime: ${spendTime}`)
}

function test(tag) {
    // 现在知道函数名了，先使用一次函数柯里化缓存函数名
    let logger = basicCurrying(logMethod, test.name)
    const startTime = Date.now()
    if (tag) {
        // 现在知道自定义消息了，再使用一次函数柯里化缓存函数名
        logger = basicCurrying(logger, 'tag is true')
    }
    else {
        logger = basicCurrying(logger, 'tag is false')
    }
    let count = 0
    do{
        count++
    }
    while(count <= 2e6) 
    // 最后，日志记录所需的三个参数都收集完成了，可以执行日志打印了
    logger(Date.now() - startTime)
}

test(true)
test(false)
```

可以看到，我们想延迟执行2次，就得手动进行2次函数柯里化嵌套，要实现任意次数得延迟执行，可以使用一种很骚很hack的操作，高阶函数柯里化。

一阶函数柯里化其实就是通过创建一个函数作用域temp，通过temp的变量对象来储存实际要执行的函数target的部分形参，但是temp执行完毕出栈后，其变量对象tempVo会被gc回收，所以我们需要在temp内部定义一个函数temp2，temp2内部通过作用域链访问了tempVo，所以tempVo不会被回收，至此tempVo就成为了一个闭包，也即我们通过tempVo这个闭包缓存了target的形参，这就是一阶函数柯里化的核心原理。

> 变量对象、作用域链

## 高阶函数柯里化

高阶函数柯里化其实也是一阶函数柯里化的嵌套，只不过不需要我们自己来嵌套，而是高级函数柯里化函数内部通过递归的方式实现，问题规模随着已缓存的形参数量增多而减小，递归出口就是已缓存的形参数量等于实际要执行的函数的函数签名的形参数量。

```javascript
const {basicCurrying} = require('./currying.basic')

function higherCurrying(fn, len) {
    let length = len || fn.length
    return function(...rest) {
        return rest.length >= length ? 
            fn(...rest):
            /**
             * basicCurrying(fn, ...rest)：
             * 这一步是闭包的嵌套，fn的作用域中有上一次basicCurrying的执行上下文的变量对象，
             * 所有会存有上一次执行basicCurrying时输入的参数。
             */
            higherCurrying(basicCurrying(fn, ...rest), length - rest.length)        
    }
}

module.exports = {
    higherCurrying
}
```

## 应用场景

### 表单校验

> 延迟执行及参数缓存

```javascript
const {higherCurrying} = require('./lib/currying.higher')
const {basicCurrying} = require('./lib/currying.basic')


function formValidator(name, phone, gender, email) {
    if(typeof name !== 'string' || typeof phone !== 'string' || !['male', 'female'].includes(gender) || typeof email !== 'string') {
        console.log('表单输入不合法')
    }else {
        console.log('表单输入合法')
    }
}

/**
 * 使用基础柯里化的嵌套写法
 */
const basicFormValidator1 = basicCurrying(formValidator, 'lee')
const basicFormValidator2 = basicCurrying(basicFormValidator1, 'phone')
const basicFormValidator3 = basicCurrying(basicFormValidator2, 'malee')
const basicFormValidator4 = basicCurrying(basicFormValidator3, 'email')
basicFormValidator4()

/**
 * 使用高阶柯里化的写法
 */
let curryingFormValidator = higherCurrying(formValidator)
for (const val of ['lee', '173', 'male', '163@163.com']) {
  curryingFormValidator = curryingFormValidator(val)  
}
```



### `Vue`中使用到的函数柯里化

> 函数缓存

`vue`中在解析HTML模板时需要判断当前标签是原生的HTML标签还是自定义组件，那`vue`是预先硬编码了所有HTML标签的字符串，通常我们会写一个这样的函数

```javascript
const strs = "div,p,li,span,ul,img,section"

/**
 * 普通写法，循环判断HTML标签名数组中是否有该标签
 */
function isHtmlTag(target) {
    return strs.split(",").includes(target)
}

isHtmlTag('div') // console true
```

把HTML标签字符串分割成数组后循环遍历该数组进行查找，那时间复杂度是O(n)。

`Vue`中利用了函数柯里化进行优化

```javascript
const strs = "div,p,li,span,ul,img,section"
function createFn(htmlTagsStr) {
    const map = new Map()
    htmlTagsStr.split(",").forEach(tag => {
        map.set(tag, true)
    })
    return function(target) {
        return !!map.get(target)
    }
}
const isHtmlTagPro = createFn(strs)
isHtmlTagPro('div') // console true
```


