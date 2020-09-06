---
layout: post
title: "NodeJS-EventEmitter 观察者模式"
subtitle: '结合实际案例运用观察者模式'
author: "lee"
header-style: text
tags:
  - Typescript
  - NodeJS
  - 设计模式
  - 观察者模式
---
### 观察者模式
观察者模式是大家耳熟能详的一种设计模式，是什么在此就不叙述了。每个人有每个人的理解。  
> [百科](hhttps://baike.baidu.com/item/%E8%A7%82%E5%AF%9F%E8%80%85%E6%A8%A1%E5%BC%8F/5881786?fr=aladdin)

观察者模式有三个关键主体
- 事件，一般对应执行者或程序执行流程的一些关键状态。
- 执行者
- 观察者

有两个关键动作
- 发布事件
- 订阅事件，也即为事件添加回调函数。  

其实观察者模式主要用于实现代码间的解耦，不希望功能上可以互相独立的代码互相嵌套或者希望一个对象在执行流程中的某些关键状态可以被外界所感知。  
### NodeJS中的events模块
NodeJS中有有一个原生模块`events`，其实也就只有一个`EventEmitter`类，可以快速的帮助我们实现观察者模式。具体提供了什么API我也不多叙述了，官方文档就有。
> [NodeJS-Events](http://nodejs.cn/api/events.html#events_events)

下面是一个我在日常项目开发中的实际案例，希望对大家有所帮助。
### 观察者模式的实际案例
现在有一个WEB APP需要实现自动升级的功能，因为是WEB APP，编译过后运行的就是HTML、CSS、JS，所以自动升级只需要替换响应的文件即可。
自动升级分三步走：
- 下载升级包
- 解压升级包
- 替换相应文件  

并且，在进行上诉流程时，希望能够通过UI界面向用户反馈升级进度。这里，升级进度展示和实际的升级流程就是两个功能上可以独立的两个对象，如果不使用观察者模式，就很可能会实际的升级流程代码写一点，然后穿插一点UI操作的代码，这样写出来的代码耦合严重不易于维护，利用观察者模式就可以很好的实现两者的解耦。  
Talk is cheap, show you the code.
#### 定义事件
```Typescript
enum AppAutoUpdateEvents {
  downloading = '正在下载升级包',
  compressing = '正在解压升级包',
  replacing = '正在替换文件',
}
```
#### 定义上下文
上下文`Context`，很多地方都能看到这个概念，例如CPU的执行上下文，JS的函数执行上下文，到底什么是上下文呢，个人觉得是对某端代码、某个功能、某段业务、某个对象从开始到结束的状态或者说数据的封装。这里我也定义一个上下文，为什么需要定义一个上下文呢？当我们发布一个事件时，是可以传递参数给观察者的，每个事件对应的参数很可能都是不同的，当有100个事件，每个事件有5、6、7、8个希望传递给观察者的参数时，想想这是多么痛苦的一件事，当定义一个上下文，直接把上下文传递给观察者时就会很方便。Koa中的Context对request和response的持有也是类似的想法。
```
class AppAutoUpdateContext {

  /**
   * 升级包的大小，单位字节
   */
  packageSize: number = 0

  /**
   * 当前正在解压的文件数组下标
   */
  compressingIndex: number = 0

  /**
   * 当前正在替换的文件数组下标
   */
  replacingIndex: number = 0

  /**
   * 升级包中的文件列表
   */
  fileList: Array<{
    /**
     * 解压后的文件路径
     */
    path: string
  }> = []

}
```
#### 声明执行者
```Typescript
interface AppAutoUpdateEventListener {
  (ctx: AppAutoUpdateContext): void
}

interface AppAutoUpdaterInterface extends EventEmitter{
  once(event: string, listener: AppAutoUpdateEventListener): this
  on(event: string, listener: AppAutoUpdateEventListener): this
}
```
上面提到了，我们引入了Context这个概念，现在希望每个观察者接收的都是Context这个参数，所以这里声明了一个观察者的`interface`。
#### 实现执行者
```
实际的升级流程的执行者，继承了NodeJS提供的EventEmitter。
​```Typescript
class AppAutoUpdater extends EventEmitter implements AppAutoUpdaterInterface {

  static events = AppAutoUpdateEvents

  private context: AppAutoUpdateContext

  constructor() {
    super({
      captureRejections: true
    })
    this.context = new AppAutoUpdateContext()
  }

  private download() {
    // 模拟升级包下载过程
    this.context.packageSize = 100000
    this.emit(AppAutoUpdateEvents.downloading, this.context)
  }

  private compress() {
    // 模拟解压升级包
    for (let index = 0; index < 10; index++) {
      this.context.compressingIndex = index
      this.context.fileList.push({
        path: `/app/dist/file-${index}.js`
      })
      this.emit(AppAutoUpdateEvents.compressing, this.context)
    }
  }

  private replace() {
    // 模拟替换文件
    this.context.fileList.forEach((file, index) => {
      this.context.replacingIndex = index
      this.emit(AppAutoUpdateEvents.replacing, this.context)
    })
  }

  public update() {
    this.download()
    this.compress()
    this.replace()
  }

  public once(event: string, listener: AppAutoUpdateEventListener): this {
    super.once(event, listener)
    return this
  }

  public on(event: string, listener: AppAutoUpdateEventListener): this {
    super.on(event, listener)
    return this
  }
}
```
#### 观察者
在这个案例中，观察者即我们的UI控制模块，这里使用`console`模拟，真实情况可能是vue实例等。
```Typescript
const autoUpdater = new AppAutoUpdater()
autoUpdater.once(AppAutoUpdateEvents.downloading, (ctx) => {
  console.log(`${AppAutoUpdateEvents.downloading}，升级包大小为${ctx.packageSize}Byte`)
}).on(AppAutoUpdateEvents.compressing, (ctx) => {
  console.log(`${AppAutoUpdateEvents.compressing}，文件为${ctx.fileList[ctx.compressingIndex].path}`)
}).on(AppAutoUpdateEvents.replacing, (ctx) => {
  console.log(`${AppAutoUpdateEvents.replacing}，文件为${ctx.fileList[ctx.replacingIndex].path}`)
}).update()
```

![](https://lee-img-bed.oss-cn-shenzhen.aliyuncs.com/Dcodejs-fundamental-practicenode-best-practiceevent-emitterevent-emitter-node.png)

`Typescript`真香~

### 写到最后

通过使用观察者的模式，UI代码和实际升级的实现代码很好的很分离开来了，可读性、可维护性、扩展性无疑是大大的提高了的，在后续的功能扩展中，只需要新增events和新增对应的观察者即可，无论是对UI代码的修改还是实际升级的实现代码的修改，只要events不变，就不会影响到原有代码。  
但是这里其实有个问题，观察者能感知到执行者的存在的，也是存在耦合的。如果使得观察者和执行者之间不会感知到互相的存在呢？其实这就是观察者模式的升级版，发布订阅模式。发布订阅模式在执行者和观察者之间加入了一个事件调度者，执行者通过事件调度者发布事件，观察者通过事件调度者去订阅事件。