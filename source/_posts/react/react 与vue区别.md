---
title: vue和react的差异
date: 2017-11-25
tags: [vue,react]
author: gcyStar
---
#引言
平时开发单页项目应用基于vue,目前另外两个比较热的库还有angular和react,angular 1系列用过,进入公司后由于基于vue技术栈就没在关注了。
一直在关注react,目的不是学习用法,只是为了拓展自己的视野和思维,通过了解一些使用上的差异性,来进一步的思考其底层设计的思想。

# 环境搭建
在具体业务逻辑开发前,我们首先要做的是搭建项目骨架,vue的话可以使用vue-cli,通过脚手架产生的配置完全暴露出来,我们可以灵活的修改配置来定制化需求。
我常用几个配置如下

```
    build时assetsPublicPath会修改成相对引入,或者配置成公共前缀,方便测试
    dev时proxyTable代理一些接口,联调用,自己测试话就直接用mock数据服务
    关闭devtool,去除contenthash,chunkhash,html的minify,添加externals,定制化一些eslint……等等
    其它也有一些不通用改动,上面是每个项目都通用的配置
```

至于react可以使用create-react-app架手架,然后就直接可以使用create-react-app创建项目了,默认隐藏配置,定制化配置可以直接npm run eject。需要注意点就是使用﻿create-react-app创建比较慢,需要做如下设置
```
    ﻿npm config set registry https://registry.npm.taobao.org 
 ```
其实自定义配置直接在配置依赖包里也可以改动,不过这样不太好。其次脚手架创建项目最好不要在已有git目录下载创建,不然使用npm run eject会报错,此时用空目录下创建即可。
无论是哪个脚手架常用的配置,都是基于webpack核心生态圈构建,所以核心重点就是webpack用的熟练的话,无论使用哪个自定义化配置都不成问题。

# 使用
vue稍微复杂些单页会基于vue+vuex+vue-router,react栈是react+redux+react-router。
不同点是vue组件是(html+css+js),对开发友好,上手容易。react一切都是js,特别的灵活,通过在render中利用纯js逻辑控制渲染输出模板。下面是基于react的一个示例。
```
render() {
    let name = this.state.flag ? 'true':'false'
    return (
        <ul>
            {this.props.items.map(item => (
                <li key={item.id}>{item.text}</li>
            ))}
            {name}
        </ul>
    );
}
```
vue的语法糖可以让我们在表单上轻松的实现双向绑定,而react是纯粹的基于UI=render(data)的理念。vue利用es5的set,get机制收集依赖,能详细的定位修改元素,react每次setState对组件构造函数中私有属性进行修改时,组件都会更新,除非你在shouldComponentUpdate加入一些逻辑处理。另外vue提供的api多,有很多好用又方便的指令,但是具有两面性,而react核心概念少,js用的溜,上手挺容易的。

# 数据流
无论是vue还是react都支持组件私有属性;组件之间prop,组件之间简单关数据系修改的话,可以使用事件的方式,vue此时又提供了个语法糖sync,吼吼,其它的话没啥差异。

比较麻烦的一点就是复杂网状组件之间数据流动时处理,此时就需要合理组织数据了,不然维护,调试就是一个大坑,事件方式就不适合了,此时就要说到vuex和redux了,vuex的getter获取state中数据,映射到组件data属性上,Mutation同步commit修改数据,Action中dispatch可同步or异步修改数据,核心是单一状态树,通过设计层层方法最后达到修改数据的目的就是为了更好的管理,检测数据的流动。redux因为平时用的少,所以此刻我要描述的详细些,以做备忘。
```
import {createStore, applyMiddleware, compose} from 'redux'
import thunk from 'redux-thunk'
import {Provider} from 'react-redux'
const store = createStore(counter,
    compose(
        applyMiddleware(thunk),
        window.devToolsExtension?window.devToolsExtension(): f => f
    )) 
ReactDOM.render(
    <Provider store={store}>
        <JSDT/>
    </Provider>,
    document.getElementById('root')
);
```
了解背后原理，可以让我们更灵活的控制代码，下面开始详细的分析。
执行createStore,提供一个状态树的初始化环境,返回一个对象,其中包含一些闭包函数,引用createStore初始化环境中中的各种函数和变量,例如dispatch(派发action),getState(获取只读状态树上的值),subscribe(订阅数据改变)等等,其中
```
dispatch({ type: ActionTypes.INIT })(源码:248)
```
 此代码的目的是创建store的时候,给reduce一个默认值,初始化currentState值,方便初次getState调用。

applyMiddleware中间件机制,可以在处理store前后加一些通用处理,其它例如express,koa,springMVC这些框架中都有这种思想,实现方式不同,目的都一样,解耦,可插拔效果,便于维护。
而rudux中applyMiddleware实现原理是利用高阶函数compose,通过reduce将多个函数组合成一个可执行执行函数,关键步骤代码如下所示。

```
//applyMiddleware.js
chain = middlewares.map(middleware => middleware(middlewareAPI))
dispatch = compose(...chain)(store.dispatch) 
```
```
//compose.js
funcs.reduce((a, b) => (...args) => a(b(...args)))
```

thunk 至于这个就是用来方便做异步处理的,是一个高阶函数中间件,如下所示,一般action返回的都是一个行为描述对象,但是这个在你对store进行处理前加了一层逻辑判断,以便我们在组件上统一的方式写dispatch相关的代码。
```
 if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }
```

devToolsExtension //开启调试工具,没有vue方便,不能自动检测

Provider提供一个上下文环境,让一个树上的所有组件都能访问同一个对象。必要前提条件是添加加childContextTypes和getChildContext。


# 优化
为了更好的体验,一般我们会采取一些措施,下面总结一下针对vue和react的优化措施。
重单页应用,在路由中我们可以异步加载组件,虽然两者都支持,原理类似,但是vue使用极其方便,如下所示
```
const My = () => import('../components/My.vue')
```
有一点需要注意的就是需要配置下assetsPublicPath,这样打包后的url中是全路径,否则按照相对路径处理容易出问题;相对而言react还要用模板代码包装下组件,这一点不好,不过未来react未来会开启异步渲染组件的支持,这一点很赞。


vue和react都是基于dom diff更新差异元素。vue中,我们操作的data数据是vue封装处理过,修改数据时,vue会将数据初始化时收集的相关依赖元素进行更新,而react每次setState更新数据针对的是组件,
为了优化,组件设计的时候尽量细粒度,尤其是react当中的展示类组件。

因为两者使用的是针对web页面情况优化过的dom diff算法,以使复杂度降低为O(n),所以有一些默认前提,理解并按照默认diff规则才能使代码达到最优,比如说,保持根节点一致;一个dom父节点下有多个子节点
并列时,给子节点添加key,防止对父节点使用insertBefore插入子节点这种情况等等。


# 总结
陆陆续续的学习过程中,将自己的想法自己总结下来,踏实走好每一步。

说明,文中的总结基于下面这些版本,
vue ^2.4.2
react 16.0.0-rc.2
redux 3.7.2
react-router 4.2



















