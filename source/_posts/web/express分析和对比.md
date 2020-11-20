---
title: express分析和对比
date: 2018-03-08
tags: [node,javaScript,web,koa,express]
author: gcyStar
---

## 前言
目前express最新版本是4.16.2,所以本文分析也基于这个版本。目前从npm仓库上来看express使用量挺高的,express月下载量约为koa的40倍。所以目前研究下express还是有一定意义的。

## 源码分析
直接切入主题,由于目前express是一个独立的路由和中间件web框架。所以分析的方向也以这两个为主。源码的研究只注重关键步骤和流程思想, 具体的hack,异常,边界处理不做过多精力关注。
### 关键步骤
#### 中间件的执行
```
/**
 * //负责具体请求的逻辑处理,当一维layer中所对应的stack执行完后执行done(目的是为了一维路由列表中进行下一个路由的匹配执行)。
 * @param done(路由中的next)
 */
Route.prototype.dispatch = function dispatch(req, res, done) {
    var idx = 0;
    var stack = this.stack;
    next();
    //递归方式执行stack中的layer,通过next控制流程的执行
    function next(err) {
        // 出错直接退出当前stack和路由列表中回调的后续执行
        if (err && err === 'router') {
            return done(err)
        }
        //出错直接退出当前stack列表后续执行,进行下一个路由匹配
        if (err && err === 'route') {
            return done();
        }
        var layer = stack[idx++];
        //执行结束
        if (!layer) {
            return done(err);
        }
        if (layer.method && layer.method !== method) {
            return next(err);
        }
            //调用具体注册好的逻辑
        if (err) {
            layer.handle_error(err, req, res, next);
        } else {
            layer.handle_request(req, res, next);
        }
    }
};
```
**说明** 上面是源码的部分摘要,去除了无关的信息,对关键步骤加了注解。
#### 路由匹配
```
/**
 * 对具体请求进行路由分发处理
 * out 是最后一个处理器,默认是请求的回调,不传的话是内部提供的error handlers
 */
proto.handle = function handle(req, res, out) {
    var self = this;
    var idx = 0;
    var paramcalled = {};
    // middleware 和 roues
    var stack = self.stack;
    var done = restore(out, req, 'baseUrl', 'next', 'params');

    next();
    //递归方式遍历注册的一维路由
    function next(err) {
        var layerError = err === 'route'
            ? null
            : err;
        var layer,match,route;
        //取出注册好的路由,进行请求匹配
        while (match !== true && idx < stack.length) {
            layer = stack[idx++];
            //req的path匹配,如果有注册参数路由会解析参数到layer.params上
            match = matchLayer(layer, path);
            route = layer.route;
        }

        if (match !== true) {
            return done(layerError);
        }
        //将解析好的路径参数放到请求对象上,以便后续参数回调逻辑的使用
        req.params = layer.params;

         //路径参数回调回调  ,例如请求 /user/1 ,参数回调 app.param('user',cb1) app.get('/user/:user',cb2)   会先执行注册的param cb1,然后才会是router中注册的cb2
        self.process_params(layer, paramcalled, req, res, function (err) {
            if (err) {
                return next(layerError || err);
            }
            if (route) {
                return layer.handle_request(req, res, next);
            }

        });
    }
};

```
**说明**  上面摘取了请求进行路由分发处理的关键步骤,并做了相应的注解。
### 执行流程
当一个请求过来的时候交给handler方法,进行路由的匹配,以递归的方式遍历(路由匹配一节中介绍过),当匹配到某一个路由的时候在dispatch执行,web应用启动初始化前注册好的回调逻辑,执行的方式也是以递归的方式(中间件执行一节中介绍过)。
请求匹配执行逻辑已经介绍过了,下面结合着初始化的逻辑,进行分析,具体如下图所示。
![express-jsdt](https://img.wuage.com/152083811955039express.jpeg)

**说明** 
#### 初始化
针对上图做一下说明,启动服务应用的时候先进行初始化,注册'/jsdt'的get请求,并给路由绑定相应的回调。其中我们的路由放在一维的layer中,每一层路由,例如'/jsdt'又对应一个处理列表,这个二维列表里又存储着一系列layer(具体的回调处理逻辑),因为一维layer中已经记录了路径'/jsdt',所以二维的layer中就不用记录路径了,给个默认值'/',以保持一维layer和二维layer中结构的统一。
#### 请求
当一个请求过来的时候先去一维layer所存储的路由中进行路径匹配,以递归的方式。匹配到了,通过dispatch方法,在执行路由所对应的二维layer中的回调逻辑,也是以递归的方式执行,在递归的过程中,如果发生异常,通过路由中传递过来的out,直接进行下一个路由的匹配。
```
/**
 *递归执行stack中的handle
 * @param out  路由中的next
 */
Route.prototype.dispatch = function (req, res, out) {xxx}
```
当执行遇到res.end()的时候,数据响应完后,整个请求响应过程就结束了。


## koa vs express 
> [kao源码](https://segmentfault.com/a/1190000008836418)去年的时候有分析过,现在对比分析思考下。
koa比较迷你,微内核,拓展性强,所以一些web框架例如阿里的eggjs就是基于koa。而express集成了路由和static中间件所以显得重一些。

### express中间件和koa中间件区别,以及与redux中间件区别 ? 
网上很多文章都说一个是线性的,一个是洋葱模型。因为两个我都研究过,我觉的这种说法不对,其实执行的时候都是洋葱形。主要的区别是koa内核底层原生支持async语法,koa中的middlewares经过compose以后,每次执行到await next返回的都是一个promise,所以我们可以在顶层加一个try……catch进行异常捕获,这算是koa比较方便的一点。还有一点很多人提到过说koa可以记录处理时间,那是因为每次res.body赋值的时候并没有res.end,所以在第一个中间件很容易记录处理的时间,如下所示,在中间件执行完后,在handleResponse中才会将请求处理结果返回给客户端
```
fn(ctx)[是一个立即状态的promise].then(handleResponse).catch(onerror);
```
而express每次res.send的时候数据已经发给客户端了,当然也可以实现这种需求,只不过没有koa方便。多说几句,其实java中也有类似的实现,例如java中的aop,过滤器,将通用的逻辑,例如日志,权限等模块通过配置的方式进行灵活的插入,配置在xml中。比较灵活,每个模块之间互相解耦,根据需求实现可插拔效果。
在说一下与react中redux中间件机制区别,redux,可以在处理store前后加一些通用处理,利用高阶函数compose,通过reduce将多个函数组合成一个可执行执行函数
```
//applyMiddleware.js
chain = middlewares.map(middleware => middleware(middlewareAPI))
dispatch = compose(...chain)(store.dispatch) 
```
```
//compose.js
export default function compose(...funcs) {
  return funcs.reduce((a, b) => (...args) => a(b(...args)))
}
```
组合好最后执行的时候是洋葱模型效果,举个例子 compose(a, b, c)变成 a(b(c())),而a,b,c的结构类似下面这种形式
```
function fnA(next) {
  return function() {
    console.log('fnA start')
    next()
    console.log('fnA end')
  }
}
```
所以a(b(c()))()执行的时候,就会通过next来控制调用下一个中间件,整体的执行等价于express递归调用方式
### express路由和  vue路由共同点
vue中也有路由,官方的vue-router,他解决的是url和模板组件匹配渲染的问题, 而express中解决的url和handler匹配执行的问题,而koa内核里面没有集成路由。 从vue-router和express路由中可以看出路由的共性,是为了解决路径和相应处理逻辑的匹配问题在web开发中。


## toy版本express
根据上述分析的逻辑,实现了一个简化版的[express](https://github.com/gcyStar/toy-express) ,融入了express的核心思想,有详尽的步骤注释,有需要的可以参考下。



**参考源码版本说明**
express 4.16.2
koa 2.2
redux 3.7.2
**参考链接**
https://github.com/koajs/koa/blob/master/docs/koa-vs-express.md
https://www.reddit.com/r/node/comments/692w23/express_vs_koa_with_asyncawait/
https://blog.jscrambler.com/migrate-your-express-app-to-koa-2/