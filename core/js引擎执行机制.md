---
title: js的执行机制
date: 2018-01-17
tags: [EventLoop,异步,microtask,microtask]
author: gcyStar
---

## js在哪执行
js的执行引擎基于v8(c++编写),在chrome和node中都有应用,执行时有以下两部分构成

> * 内存堆(内存分配)
> * 调用栈(代码执行)

上述两部分的联系就是代码在调用栈中执行,执行过程中会存取一些对象在内存堆上。

我们写的js代码经过js引擎(解释器)转化为高效的机器码,现在的v8引擎由TurboFan和Ignition两部分构成,其中Ignition是解释器,而TurboFan主要对代码做些优化,以提高执行性能。

基于执行引擎的执行原理在代码层面我们可以做些优化,可以参考我之前的[一篇文章](https://segmentfault.com/a/1190000011531171)


## js如何执行
### js同步执行
js按照代码顺序执行,在栈上分配执行空间,按照调用顺序,会有出栈入栈等各种情况,比较好分析,唯一值的说的地方就是js只有一个主线程,栈空间有限,如果递归执行过深会发生溢出,所以在编写代码层面需要注意这种情况。

### js异步执行
### 为什么要有异步?
  同步单线程代码处理起来方便,代码表达也容易,更符合我们的思维方式,为什么还会出现异步呢? 
  因为同步会发生阻塞,在现在这个高并发时代,不能很好的处理海量请求,同时也不能充分利用硬件资源(想想cpu和io之间处理速度差异你就深有体会)。
  但是为什么不多线程呢,例如java,主要是单个线程上运行代码相对来多线程来说说容易写，不必考虑在多线程环境中出现的复杂场景，例如死锁等等。    
  
### 异步执行机制?
异步执行相对来说复杂些,所以详细描述下,关键是在各种使用情况下执行顺序问题,在此就需要引入一个概念-->Event Loop。结合下面这幅图进行大致说明下:
   ![event-loop-gcy](https://img.wuage.com/151644432366617eventLooop.png)
        Event Loop的概念
 > * 所有任务在主线程上执行，形成一个执行栈（execution context stack),上图stack区域所示。
 > * 执行过程中可能会调用异步api,其中Background Threads负责具体异步任务执行,结束后将宏任务回调逻辑放入task queue中,微任务回调逻辑放入micro task队列中。
 > * 主线程执行完毕,检查microtask队列是否为空,会执行到队列空为止
 > * 从宏任务队列中取出一个在执行,执行完后,检查并取出执行microtask队列的任务,然后不断重复这个步骤,对于这整个循环过程,一个对应的描述名词就叫做event loop。
              
### node中异步
异步任务分类  
 > * macrotask类型包括 script整体代码,setTimeout,setInterval,setImmediate,I/O……
 > * microtask类型包括  Promise process.nextTick Object.observe  MutaionObserver……
 
node中event loop各个阶段的操作如下图所示    
                      
![node-event-loop](https://img.wuage.com/public/151644816076861375609_900_572.png)
说明,上图中每个盒子表示了event loop的一个阶段,每个阶段执行完毕后,或者执行的回调数量达到上限后，event loop会进入下个阶段。
    timers: 在达到这个下限时间后执行setTimeout()和setInterval()这些定时器设定的回调。
    I/O callbacks: 执行除了close回调，timer的回调，和setImmediate()的回调,例如操作系统回调tcp错误。
    idle, prepare: 仅内部使用。
    poll: 获取新的I/O事件,例如socket的读写事件；node会在适当条件下阻塞在这里,如果poll阶段空闲,才会进入下一阶段。
    check: 执行setImmediate()设定的回调。
    close callbacks: 执行比如socket.on('close', ...)的回调。
下面结合一些具体例子进行说明

```

require('fs').readFile('./case1.js', () => {
    setTimeout(() => {
        console.log('setTimeout in poll phase');
    });
    setImmediate(() => {
        console.log('setImmediate in poll phase');
    });
});

``` 
```
输出结果是:
setImmediate in poll phase
setTimeout in poll phase
Process finished with exit code 0
```
**说明** setImmediate的回调永远先执行,因为readFile的回调执行是在 poll 阶段，所以接下来的 check 阶段会先执行 setImmediate 的回调。

```
setTimeout(() => console.log('setTimeout1'), 1000);
setTimeout(() => {
    console.log('setTimeout2');
    process.nextTick(() => console.log('nextTick1'));
}, 0);
setTimeout(() => console.log('setTimeout3'), 0);

process.nextTick(() => console.log('nextTick2'));
process.nextTick(() => {
    process.nextTick(console.log.bind(console, 'nextTick3'));
});
 Promise.resolve('xxx').then(() => {
    console.log('promise');
    testPromise();
});
process.nextTick(() => console.log('nextTick4'));
``` 


输出结果是:

````
    nextTick2
    nextTick4
    nextTick3
    promise
    setTimeout2
    setTimeout3
    nextTick1
    setTimeout1
``` 
   
在描述什么是event loop中,大概描述了microtask机制,但具体到nextTick比较特别,有一个Tick-Task-Queue专门用于存放process.nextTick的任务,且有调用深度限制,上限是1000。js引擎执行 Macro Task 任务结束后,会先遍历执行Tick-Task-Queue的所有任务,紧接着再遍历 Micro Task Queue 的所有任务。具体执行逻辑可以下面代码表示。

```
for (macroTask of macroTaskQueue) {

    // 1. Handle current MACRO-TASK
    handleMacroTask();

    // 2. Handle all NEXT-TICK
    for (nextTick of nextTickQueue) {
        handleNextTick(nextTick);
    }

    // 3. Handle all MICRO-TASK
    for (microTask of microTaskQueue) {
        handleMicroTask(microTask);
    }
}
```

所以才会先输出process.nextTick然后才会是promise,其它的输出顺序不在赘述,前面讲event-loop机制时已经说明了。根据上面代码表述的执行逻辑,很显然可以得到下面的个结论,当递归调用时会发生死循环,而宏任务就不会。

```
testPromise();
function testPromise() {
    promise = Promise.resolve('xxx').then(() => {
        console.log('promise');
        testPromise();
    });
}
//将之前步骤的promise任务换成这个,setTimeout2以及之后的输出永远没机会出来,类比到nextTick也是这种效果
```


看了一些书,参考了很多资料,将自己学习的东西,理解后在输出,希望大家辩证的看待,有空的话接下来研究一下源码,毕竟通过demo验证结论的说服力没有源码来的那么直接。

参考链接
https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/
https://blog.sessionstack.com/how-does-javascript-actually-work-part-1-b0bacc073cf
https://cnodejs.org/topic/592e377e855efbac2cf7a4dd
https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/EventLoop#%E4%BA%8B%E4%BB%B6%E5%BE%AA%E7%8E%AF
https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/