---
title: node异常总结
date: 2017-05-28
tags: [node,js]
author: gcyStar
---
# 引言
>  对错误进行分类,理解错误是如何产生的,以及错误发生后怎么定位解决,这在构建一个稳定运行的程序过程中会经常遇到。总结一下,以便有更清晰的认知。

# 异常分类

## 显式异常

```
function validateName(name) {
    if(!name){
        throw new Error('name is required');
    }
}
try {
    validateName();
}catch (err){
    console.log(err.message,err.stack);
}
```
**说明** 有以下几点需要注意

* 一般用于同步方法,或者异步执行前,否则无法捕获异常,process直接exit。

* 抛出的错误一般都是继承于Error。使用简单的字符串例如(throw 'error happen')无法获得调试堆栈信息。

* 不要在内置的node方法中抛出异常,没有意义,因为在node中回调函数第一个参数就是错误对象,可以直接处理。

## 隐藏的异常

```
function getName() {
    return name;
}
getName();

```

**说明**  隐藏的异常是不是有throw触发,是在运行时发生的,例如上面常见的ReferenceError,这种异常可以使用eslint等工具检查出来。
```
no-undef:error
```

## 错误事件

```
var EventEmitter=require('events').EventEmitter;
var ee=new EventEmitter();
ee.on('error',function (err) {
    console.error(err.message,err.stack);
    console.log("end");
});
ee.emit('error',new Error('no handled event error'));

```

**说明**  当EventEmitter实例发生错误时,会触发error事件,如果没有error事件的监听器,默认会打印堆栈,然后退出程序。如果实例比较多,想要统一管理,可以使用domain模块管理。例如下面的示例,集中处理connection时发生的异常。
```
var  domain=require('domain');
var http=require('http');
var d=domain.create();
d.run(function () {
    var server=http.createServer(function (req,res) {
        d.on('error',function () {
            res.statusCode=500;
            res.end('internal server error');
            server.close();
            setTimeout(process.exit,5000,1);
        })
        response.end('hello world');
    }).listen(3000);

});
```
使用域的好处是,可以把各式异常放到一个域的异常处理函数中,且不影响其他的域。在非阻塞式api中,一般都是用domain集中处理异常。想要全局处理,可以按照下面的方式来个统一处理，不推荐。
```
var http=require('http');
var server=http.createServer(function (req,res) {
    response.end("hello world");
}).listen('3000');
process.on('uncaughtException',function (err) {
    console.log(err);
})
```
uncaughtException是最后一道防线,比较好的做法是记录错误信息,然后重启。

## 错误参数

```
    fs.readFile('./config.json',function (err,buf) {
        // if(err) throw err;
        if(err)  throw new Error('read file filed');;
        var config=JSON.parse(buf.toString());
        console.log(config);
    })
```
**说明**  上述示例中,如果忽略readFile返回的错误,有可能无法获取buf而抛出异常,所以错误参数还是需要处理一下。

# 问题debug
比较low的方法就是打日志,这个在开发中不用,一般是用来记录查看线上问题;开发中一般都是断点调试,重点分析下。

## 调试方式一
Node.js 6.3以下，使用老的方式话，node-inspect作为模块需要单独安装，其作用通过websocket的方式充当一个连接通道，负责dev tool和node中运行程序产生的debug信息进行通信。

node-inspector  &  ;node-debug app.js
出现下图所示就代表成功了。
![node-debug1-gcy](https://img.wuage.com/149602217013137debug1.png)

## 调试方式二

node6.3+中, node --inspect app.js  ,使用前开启chrome的Node debugging选项,出现下面效果标识就成功了。
![node-debug2-gcy](https://img.wuage.com/14960233432134debug2.png)

**说明**  上述两种调试方式,我很少用,一般我都是用webstrome来调试,通过调整Node Interpreter的版本来使用相应的调试方式,原理和上面是一样的,只是调试形式上变化了下。

7以上我通常都是用 --inspect的方式,否则会提示你'node --debug is deprecated',但是用老方式也没什么问题。区别其实就是获取debug信息的通信协议方式不同,新的方式使用的是Chrome Debugging Protocol,而老的是 V8 Debugging Protoco。

# 总结
上面我只是总结了通用的异常情况,其实还有一些特殊的使用方式,例如throw在generator当中的使用,以及promise中可以直接抛出异常,都不是我上面所说的一般情况,不在列举,
因为阮老师关于这两个[api](http://es6.ruanyifeng.com/#docs/generator)的论述很详细,此处不在赘述。














