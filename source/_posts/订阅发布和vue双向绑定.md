---
title: 订阅发布和vue
date: 2017-03-26
tags: [设计模式,vue双向绑定]
author: gcyStar
---

## 引言
> 最近在看vue的源码,有些感触,下面阐述一些个人理解。
之前写过[一篇文章][1],是讲述关于观察者模式的,与本文主旨有关,需要的朋友可以看一下。

vue的核心是mvvm,vue2又增加了虚拟dom。我的研究方向也是以这两个为主。本文主要讲述mvvm,至于vdom(主要研究方向是整个思想和diff算法)下次再论述。

## mvvm概念理解
m(model)v(view)vm(view-model),重要特性双向绑定,m和v之间无耦合,通过操作m,利用vm提供的机制,自动实现v的更新。用过vue应该会有较深体会。

和其相对比比较容易误解是mvc,m(model)v(view)c(controller),和mvvm的显著区别是m和v之间存在耦合,业务逻辑集中在c当中。 用过springmvc应该会有体会。

## mvvm实现

**说明**  解析不包括模板,指令,以及Mustache等,因为模板编译、解析等和本文核心主题无关,这些仅仅是基于双向绑定的应用场景。本文主题想讨论是结合观察者模式简单实现双向绑定的核心。

双向绑定的核心是实现Dep、observer、Watcher。

 * Dep 存放收集Watcher,当订阅的消息发生时(即数据发生改变),notify所有watch,让订阅者执行实现自己定义的update逻辑
 
 * Watcher  转化表达式,收集依赖,当被观察的值发生变化时,触发callback。重要update方法,当model数据发生变化时,修改依赖指令或组件中的值,触发callback函数

 * observer  是一个类,附属于每一个被观察的对象(即model数据),defineReactive=>observe方式给对象以及子属性(实质调用defineProperty方式)建立get和set属性方法
 
 结合下图,以便于大致理解。
 
![observe-jsdt-demo](http://img.wuage.com/149227174416340observer.png)

关于其执行流程具体如下,
图中画的其实也只是大致流程,实现过程中有很多细节,列举几个我认为比较重要的。

 * this.get()执行的时候,会把Dep的全局tatget对象置为当前对象,而data的get方法以此作为judge调用者依据,从而完成依赖收集,收集完cleanupDeps,将其置空,以免影响下次收集。

 * 当data持续变化时,queueWatcher中has[id]的judge防止watcher重复添加,nextTick的使用保证view更新时使用的最新值,异步的应用有三种实现方式(兼容性考虑),基于优先级顺序依次是Promise、MutationObserver、setTimeout。
 
 * data变化,view更新的实现是通过notify->run-> expOrFn.call(vm,vm)->vm._update(vnode, hydrating)->vm.__patch__(prevVnode, vnode)一系列调用过程实现。
 

![observe-jsdt-demo2](http://img.wuage.com/149285289588613vue2.png)

## 观察者体现
> 至于观察者模式是如何在其中体现呢? 

observer和Watcher是一对多的关系,连接机制通过dep(其中注册了一些观察者), 消息触发根源是user,
其行为导致model改变,被观察者(data)进行notify, 具体订阅者watcher执行自己预先定义好的update逻辑。
整个过程很好的体现了注册-通知的机制。

## 总结
这篇文章分析基于Vue(Version 2.2.6),示例中图画的丑,画图真心不易。个人观点,如有不同理解,欢迎comment。


[1]: https://segmentfault.com/a/1190000008743294