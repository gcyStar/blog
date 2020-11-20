---
title: react路由浅析
date: 2018-04-12
tags: [路由,router,react,JavaScript]
author: gcyStar
---


## 引言
在使用react做复杂的spa开发中,开发中必不可少的就是react-router,它使用[Lerna](https://juejin.im/post/5a989fb451882555731b88c2)管理多个仓库, 在browser端常使用的几个如下所示
* react-router  提供了路由的通用核心功能,容易搞混的就是他和react-router-dom的区别,区别就是react-router-dom中多了Link BrowserRouter 这样的 DOM 类组件,至于router和route都是一样的。
* react-router-dom 浏览器端使用的router
* react-router-redux  和redux集成使用时会用到

## react-router中路由分类

### BrowserRouter
    基于html5提供的,history API(pushState, replaceState 和 popstate 事件) 来保持 UI 和 URL 的同步。大致流程就是使用history.pushState塞历史记录到浏览器中,监听window.onpopstate事件,url变化的时候render对应组件
### HashRouter

    HashRouter 是一种特定的 <Router>， HashRouter 使用 URL 的 hash (例如：window.location.hash) 来保持 UI 和 URL 的同步。大致流程是直接给window.location.hash赋值,监听hashchange事件,hash变化时,render对应的组件
    
### MemoryRouter

    保存'url'历史记录在内存中,在demo测试或者 React Native等非browser环境下使用
###  StaticRouter

        服务端渲染中会用到
        
        
## 路由使用
就拿 HashRouter  使用来作为示例, 其它使用形式上类似,多说一句话Hash除了今天说的路由用途之外,还可以做锚点定位。
```
import React, { Component } from 'react';
import {BrowserRouter as Router, Route, Link,Switch,Redirect} from 'react-router-dom'

class App extends Component {
  render() {
    return (
        <Router>
              <div>
                      <div>
                          <li><Link to='/'>home </Link></li>   //link 本质实现其实就是一个a链接
                          <li><Link to='/about'>about</Link> </li>
                          <li><Link to='/list'>list</Link> </li>
                      </div>
                      <hr/>
                  <Switch>// 匹配到第一个匹配的路由就停止
                         <Route exact path="/" component={Home} />  //exact  表示路径要精确完全匹配
                         <Route path="/about" component={About} /> //写法1
                         <Route path="/list"  render={() => <div>list</div>} /> //写法2
                      <Redirect to="/"/>  // 当都不匹配的时候,执行这个
                  </Switch>
              </div>
        </Router>
    );
  }
}

const Home = () => {
  return   <div>home page</div>
}
const About = () => (
    <div>
        <h2>About</h2>
    </div>
);
export default App;

-------------------------
//关于Router导入还有一种等价的使用方式,如下所示
import createHistory from 'history/createHashHistory'
  <Router history={history}>
        里面的内容同上
   </Router>  
```
**说明**   在react-router-dom内部包含很多组件,例如route,link,switch等等,更多组件请参考 [这里](https://reacttraining.com/react-router/)。上面只是一个简单的实例,实现简单的路由跳转,关于说明都放在注释里了。 

## 路由剖析
在上面的示例中,Router是转发的枢纽,在这个中转站有很多线路(Route),通过开关(Link)可以启动列车的运行,乘坐列车就可以发现新大陆(compontent) 。深入进去可以发现Router只是提供了一个上下文环境, 具体的路由功能的实现依靠传入的history属性, 这个属性的功能由history模块提供,history模块里面封装了createBrowserHistory,createHashHistory,createMemoryHistory等等。因为所有模块提供的功能接口一样 所以我们以其中的createHashHistory模块作为示例分析下, 首先其提供的接口如下

```
 const history = {
    length: globalHistory.length,  //历史记录数量 window.history.length
    action: "POP",   //操作表示 可以为REPLACE  PUSH
    location: initialLocation, //内部封装的简版window.location
    createHref, //创建一个hash路由
    push,  
    replace,
    go, // window.history.go(n);
    goBack,  // go(-1);
    goForward, //go(1);
    block, // 地址变化,离开当前页时设置提示信息
    listen
  };

  return history;
```
下面选出几个比较重要的详细说明下
### push 
```
// const pushHashPath = path => (window.location.hash = path);
```
**说明** 摘出的主要逻辑添加浏览器hash地址
### replace  更换浏览器hash地址
```
const hashIndex = window.location.href.indexOf("#");

  window.location.replace(
    window.location.href.slice(0, hashIndex >= 0 ? hashIndex : 0) + "#" + path
  );
```
**说明**  摘出的主要逻辑更换浏览器hash地址
###  listen

```
 const listen = listener => {
    const unlisten = transitionManager.appendListener(listener); 
    checkDOMListeners(1);  

    return () => {
      checkDOMListeners(-1);
      unlisten();
    };
  };
  
```
**说明**  在transitionManager实例中维护了一个listener数组,appendListener添加一个Listener, checkDOMListeners是绑定事件

```
 window.addEventListener(HashChangeEvent, handleHashChange);
```
每当location地址改变,HashChangeEvent触发的时候, 会取出listeners然后执行,如下所示

```
 transitionManager.notifyListeners(history.location, history.action);
```
## block
```
  const unblock = transitionManager.setPrompt(prompt); //注册提示信息
  unblock() //执行后解除地址变化时提示信息
  
```

## 总结
 断断续续的终于把这篇文章写好了,在此期间看了history源码,写了一些示例,尽可能将自己理解的东西以简洁直白的方式输出出来,希望大家看后能产生共鸣。

参考源码
history  4.7.2
react-router 4.3.0-rc.2

参考链接
http://reacttraining.cn/web/api/matchPath
https://juejin.im/post/5995a2506fb9a0249975a1a4
