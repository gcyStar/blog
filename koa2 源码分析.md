---
title: javaScript中的函数应用
date: 2017-03-26
tags: [koa,node]
author: gcyStar
---
## 引言
> 最近在写koa2相关例子,顺便看了下koa2的源码,下面是一些个人理解。

koa1核心基于generator,但是严重依赖co的包装。koa2完全不需要,基于async(其实质是generator的语法糖调用包装),在node v7 下可直接运行。
关于async和generator的语法,本文不做赘述。下面先创建一个koa实例,然后基于入口一步步分析。

```
//koa code
const Koa=require('koa');
const app=new Koa();

app.use(async function (ctx, next) {
    console.log('>> one');
    await next();
    console.log('<< one');
});
app.use(ctx => {
    ctx.body='hello world gcy';
});
app.listen('3000',function () {
    console.log('listening on port 3000');
});
```
**说明**  上面这段代码似乎有些神秘,其实质是下面http module的封装调用。
```

// native code
let http=require('http');
let server=http.createServer(function (req,res) {
    res.writeHead(200,{'Content-type':'text/plain'});
    res.write('hello world gcy');
    res.end();
});
//start service listen
server.listen(8000,function () {
    console.log('listening on port 8000');
});

```

下面基于koa构造函数入口做进一步分析。
```
constructor() {
    super();

    this.proxy = false;
    //创建一个空数组存放放middleware，洋葱流程的真相,下面会分析法到
    this.middleware = [];
    //决定忽略的子域名数量，默认为2
    this.subdomainOffset = 2;
    //处理环境变量
    this.env = process.env.NODE_ENV || 'development';
    //实例上挂载context,request,response
    this.context = Object.create(context);
    this.request = Object.create(request);
    this.response = Object.create(response);
  }

```
**说明**  上面是构造函数入口,启动入口如下
```
 //借用原生http.createServer，添加app.callback。
  listen() {
    debug('listen');
    const server = http.createServer(this.callback());
    return server.listen.apply(server, arguments);
  }
```
**说明**  通过上面两个步骤一个完整的web服务器建立起来。对于监听接受到的请求解析处理,是通过callback函数,调用一系列中间件来完成。
下面分析中间件执行流程,我认为koa的主要内涵也就在这,所以做一下重点来论述。
首先引入经典中间件洋葱图,以便理解。

![koa-onion](http://img.wuage.com/149050753701669koa-onion.png)

结合这幅图再看下面的代码
```
////核心代码application 126 行  const fn = compose(this.middleware);
 return function (context, next) {
    let index = -1
    return dispatch(0)
    function dispatch (i) {
      if (i <= index) return Promise.reject(new Error('next() called multiple times'))
      index = i
      let fn = middleware[i]
      if (i === middleware.length) fn = next
      //立即返回处于resolve状态promise实例,以便后续逻辑继续执行
      if (!fn) return Promise.resolve()
      try {
     
        //   await next();  //当fn里面执行这句话时,就会执行dispatch(i+1),导致洋葱执行过程
        //   整个过程类似堆栈执行释放过程中的的递归调用,虽然有差异,可借用类比思考其执行顺序流程
        return Promise.resolve(fn(context, function next () {
          return dispatch(i + 1)
        }))
      } catch (err) {
        return Promise.reject(err)
      }
    }
    // 核心代码application 136 行 return fn(ctx)[是一个立即状态的promise].then(handleResponse).catch(onerror);
  }
}

```

如果结合注释看上述代码过程中存在疑惑,可进一步参考下面的进行思考,反之忽略即可。

```
const Koa=require('koa');
const app=new Koa();


app.use(async function (ctx, next) {
    console.log('>> one');
    await next();
    console.log('<< one');
});
app.use(async function (ctx, next) {
    console.log('>> two');
    ctx.body = 'two';
    await next();
    console.log('<< two');
});
app.use(async function (ctx, next) {
    console.log('>> three');
    await next();
    console.log('<< three');
});
//如果放到首部,不方便理解洋葱执行流程,因为没有调用next函数
app.use(ctx => {
    ctx.body='hello world gcy';
});
app.listen('3000',function () {
    console.log('listening on port 3000');
});
```
**说明**  koa基于中间价架构,核心简洁,除此之外还有一些其它相对重要的方法。

### application.js
-  use(fn) //组装use的传参
- createContext(req, res)  创建初始化的上下文，将req和res挂载在context上
-  onerror(err) 错误处理,当设定this.slient为true，不会输出信息,在emit触发时执行
- respond(ctx) http response简单封装,信息返回
### context.js
- delegate(proto, 'request')     //Request相关方法委托,从而让context作为调用入口
- onerror(err)  //中间件执行过程中异常处理逻辑

除此之外还有request和response的参数解析文件,因为逻辑简单,不做叙述。
虽然核心文件不多,但是其也require了不少包,下面列举几个比较重的,以作为示例。

### require

- events application继承自Emitter,从而可以实现事件的订阅发布。
- koa-compose 中间件的封装,核心逻辑之一,上面已分析。
- debug 错误信息格式封装处理
- statuses http状态码和和相应信息对应处理
- koa-convert  把generator转为promise
-   …… and so on

### 总结
写这篇文章起因,是在写case的过程中,同一解决方案下中间件选择的纠结症,尤其是在选择render template过程中,为找其本质间差异,探寻到此。
源码分析基于koa(version 2.2.0),通读源码之后,可以较清晰的开发或者使用别人的中间件,
如果你有不同的理解,欢迎留言交流。
















