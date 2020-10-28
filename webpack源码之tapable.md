---
title: webpack源码之tapable
date: 2018-03-28 
tags: [webpack,tapable]
---

## 引言
> 去年3月的时候当时写了一篇[webpack2-update](https://segmentfault.com/a/1190000008683201)之路,到今天webpack已经到了4.2,更新挺快的,功能也在不断的完善,webpack4特性之一就是零配置, webpack生命力真的很顽强,积极跟上环境的变化,响应社区的需求,不断的迭代,因为parcel在其之前就有这个特性了。直接运行webpack命令,默认production模式,但是会有WARNING。

如下所示在package.json中启动脚本配置
```
"scripts": {
    "build": "webpack --mode production",  //代码做了压缩/作用域提升(就是将依赖模块内容直接放到当前模块内)/去掉了开发模式下存在的代码/更容易使用输出的资源文件(assets做了优化处理)
    "dev": "webpack-dev-server --open --mode development", //支持注释/提示/source maps
  },
```

这种约定大于配置的开发方式,在很多框架中都存在, 默认的配置覆盖了大部分用户使用的场景,提高了大部分人的生产力。当然除了默认配置外还有其它一些特性,例如支持导入更多的模块类型,可以解析.json.wasm类型的文件等等。

虽然使用形式上变化的这么快, 但是其核心思想没多大变化。 其中webpack内部有一个事件流机制,基于tapable,也是本文研究的对象,它的作用是将各个插件串联起来,还有webpack中负责编译的Compile也是tapable的实例,所以这个蛮重要的,下面详细说一下。

## tapable概念
tapable类似于node中的EventEmitter,专注于自定义事件的触发和处理,自身可以被继承或混入到其它模块中。例如compile的实现就用到了tapable,如下所示。
```
var Tapable = require("tapable");

function Compiler() {
    Tapable.call(this);
}

Compiler.prototype = Object.create(Tapable.prototype);
```


#### Hook分析
```
class Car {
	constructor() {
		this.hooks = {
			accelerate: new SyncHook(["newSpeed"]),
			break: new SyncHook(),
			calculateRoutes: new AsyncParallelHook(["source", "target", "routesList"])
		};
	}

}
const myCar = new Car();
//使用tap方法添加了一个消费者,其中tap的第一个参数,一般是用来确认插件的名称
myCar.hooks.accelerate.tap("LoggerPlugin", newSpeed => console.log(`Accelerating to ${newSpeed}`));
myCar.hooks.accelerate.call('100')
//输出   Accelerating to 100
```
**说明** 其中SyncHook继承了Hook方法,主要作用是重写了compile方法。而Hook内部维护了	this.taps = [],每次执行tap时,都会进行insert,call的时候通过
```
this[name](...args)
```
进行执行,在call执行的时候其内部涉及到工厂模式,因为call的调用,需要先执行当前SyncHook的compile方法,用工厂模式的目的就是根据传入的不同option返回不同的通过new Function拼接出的
处理逻辑函数,因为Hook有好几种实现,在实现类的实例中添加的消费者可以是sync,promise,async等等,都需要对应不同compile来进行处理。

## tapable使用分析

```
const {Tapable,SyncHook} = require("tapable");
const myCar = new Tapable();
myCar.hooks = {
    myHook: new SyncHook()
};
let speed = 0;
myCar.plugin("my-hook", () => speed+=2);
myCar.hooks.myHook.call();
myCar.plugin("my-hook", () => speed += 10);
myCar.hooks.myHook.call();
console.log(speed);
//输出14
```
**说明**
plugin(name:string, handler:function)：允许将一个自定义插件注册到 Tapable 实例 的事件中。它的行为和 EventEmitter 的 on() 方法相似，用来注册一个处理函数/监听器，来在信号/事件发生时做一些事情,他最终还是调用hook.tap(tapOpt, options.fn)进行存储。而call就全部取出来执行。

## 总结
上面这些知识是理解插件和webpack运行原理的前置条件,更多内容待下次分解



**参考源码版本说明**
tapable: "1.0.0",
webpack: "^4.2.0",
 
**参考链接**
https://medium.com/webpack/webpack-4-released-today-6cdb994702d4
https://medium.com/webpack/webpack-4-mode-and-optimization-5423a6bc597a
https://github.com/dwqs/blog/issues/60
https://doc.webpack-china.org/api/tapable/
https://github.com/webpack/tapable
