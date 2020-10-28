---
title: redux浅析
date: 2018-04-22
tags: [redux,JavaScript]
author: gcyStar
---

## redux概念
redux是一个状态管理容器,使用redux可以更好的管理和监测组件之间需要通信的数据。

## redux基本原则
### 单一数据源
在redux中,整个应用保持一个数据源,数据源是一个树形的结构
### 状态只读

状态只读意思是不能直接修改,需要通过dispatch action方式才可以,返回的是一个新的状态对象

### 纯函数操作改变数据
改变数据的纯函数是指reducer,如下形式

```
reducer(state, action)
```
函数内部通常是switch case这些代码,函数的结果完全由两个参决定,同样的输入条件,会产生同样的输出结果

## redux使用
```
//reducer.js
export default (state = 0, action) => {
  switch (action.type) {
    case 'INCREMENT':
      return state + 1
    case 'DECREMENT':
      return state - 1
    default:
      return state
  }
}
//main.js
import { createStore } from 'redux'
const store = createStore(counter)
store.getState()  //获取状态
store.dispatch({ type: 'INCREMENT' })  //改变状态
```
**说明**  上面是纯redux的一个简单示例, 目的是改变store内部计数器的状态
结合react使用例子,可以参考我之前的[分析](https://segmentfault.com/a/1190000012166510)

## redux分析
redux的核心非常简洁, 提供了中间件机制可以拓展功能。核心模块主要有以下几部分

```
  createStore,
  combineReducers,
  bindActionCreators,
  applyMiddleware,
  compose
```
### 下面依次来分析,首先是createStore模块,主要实现逻辑如下
```
export default function (reducer, preloadedState) {
    let currentState = preloadedState;
    let listeners = [];
    function getState() {
        return currentState
    }
    //派发action
    function dispatch(action) {
        //通过旧状态和新action计算出新状态
        currentState = reducer(currentState, action);
        //执行监听函数
        listeners.forEach(listener => listener());
    }
    //派发了一个动作进行初始化
    dispatch({ type: '@@redux/INIT' });
    //供外界订阅本仓库中状态的变化 
    function subscribe(listener) {
        listeners.push(listener);
        //返回一个取消订阅函数
        return function () {
            listeners = listeners.filter(item => item != listener)
        }
    }
    return {
        getState, dispatch, subscribe
    }
}
```
**说明** 这个模块主要通过闭包的方式创建一个store,封装了处理逻辑,对外只提供getState(获取状态),dispatch(派发action),subscribe(订阅改变)三个接口。
### combineReducers主要实现逻辑如下

```
export default function combineReducers(reducers) {
    const finalReducerKeys = Object.keys(reducers)
    //返回的是合并后的reducer
    return function combination(state = {}, action) {
        let hasChanged = false
        const nextState = {}
        for (let i = 0; i < finalReducerKeys.length; i++) {
          const key = finalReducerKeys[i]
          const reducer = finalReducers[key]
          const previousStateForKey = state[key]
          //计算子store中,state的值
          const nextStateForKey = reducer(previousStateForKey, action)
        
          nextState[key] = nextStateForKey
          //如果没改变的话,还是用之前的state
          hasChanged = hasChanged || nextStateForKey !== previousStateForKey
        }
        return hasChanged ? nextState : state
      }
    }
    

```
**说明**  源码有快200行了,考虑了各种边界情况,核心逻辑就上面一点, 和并多个reducer成一个对象

### bindActionCreators主要实现逻辑如下
```
export default function bindActionCreators(actionCreators, dispatch) {
  const keys = Object.keys(actionCreators)
  const boundActionCreators = {}
  for (let i = 0; i < keys.length; i++) {
    const key = keys[i]
    const actionCreator = actionCreators[key]
    //把一个 value 为不同 action creator 的对象，转成拥有同名 key 的对象
    if (typeof actionCreator === 'function') {
    //使用 dispatch 包装 action creator 以便子组件可以直接调用。
      boundActionCreators[key] = function() { return dispatch(actionCreator.apply(this, arguments)) }
    }
  }
  return boundActionCreators
}

```
**说明**  惟一会使用到 bindActionCreators 的场景是把 action creator 往下传到一个组件上，却不想让这个组件觉察到 Redux 的存在，而且不希望把 dispatch 或 Redux store 传给它。

### applyMiddleware 

```
export default function applyMiddleware(...middlewares) {
    //下面这句话在create.js中 enhancer(createStore)(reducer, preloadedState)时会执行,目的是创建一个增强版的store
  return (createStore) => (...args) => {
    //还是调用原生createStore逻辑进行初始化创建store
    const store = createStore(...args)
    let dispatch = store.dispatch
    let chain = []
    // 传递给中间件使用
    const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => dispatch(...args)
    }
    //加工中间件,注入middlewareAPI
    chain = middlewares.map(middleware => middleware(middlewareAPI))
    //形成洋葱中间件形式包裹dispatch结构的形式
    dispatch = compose(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
}
```

**说明** applyMiddleware中间件机制,可以在处理store前后加一些通用处理,实现这个机制功能最重要,也是最难理解的一步操作就是compose,见下面分析。

### compose

```
export default function compose(...funcs) {
  
  return funcs.reduce((a, b) => (...args) => a(b(...args)))
}
```
**说明**  将多个函数合并成一个函数，嵌套执行,举个例子,例如compose(f, g, h)后就变成f(g(h())),而 函数的执行顺序从右到左，h() g() f()
## 总结
为了方便理解,我主要将源码中关键主流程逻辑做了摘要简化输出,加以注释说明,额外想说的一点感想就是,在redux生态体系中用到了大量函数式编程中的一些东西,例如高阶函数,以及随时间不断增长的action动作列表(可以等价思考认为redux对这个列表进行reduce操作),而处理action的就是纯函数理念的reducer,还有处理异步的中间件thunk函数等等。

参考源码
"redux": "3.7.2",
参考链接
https://redux.js.org/