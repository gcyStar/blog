---
title: Genrator理解-浅析
date: 2017-03-12
tags: [es6,javaScript]
author: gcyStar
---
> 生成器的主要功能:通过一段程序,持续迭代或枚举出符合某个公式或者算法的有序数列中的元素.

在js中的具体表现形式就是function*。通过generator可以做到按需获取。
怎么理解,比如我们想获取一定数量的fibonacci,可以通过下面这种方式;
```
function * fibo() {
    let a=0;
    let b=1;
    yield a;
    yield b;

    while (true){
        let next = a+b;
        a=b;
        b=next;
        yield  next;
    }
}
let generator=fibo();
for(let i=0;i<10;i++){
    process.stdout.write(generator.next().value+" ");
}
console.log(".")   //0 1 1 2 3 5 8 13 21 34 .
```
上面比较特殊的一点就是yield,作用与return类似,但并非退出函数体,而是切出函数运行时。
切出的过程类似将一个值带出协程(coroutine);同时主线程可以在一系列sync或者async操作之后,通过generator的next方法与其通信,恢复协程执行。

### 随意创建一个generator,通过其原型链查看到generator主要有一下几个方法

 * generator.next(value) 获取下一次生成器切出状态(第一次执行时为第一个切出状态)
 * generator.throw(error) 向当前生成器返回执行对象抛出错误,终止生成器运行,除非被异常被捕获
 * generator[Symbol.iterator] 为生成器提供实现可迭代的方法(通过for-of语句可用)
 * generator.return 返回给定的值，终结遍历Generator函数
 
其中用的最多用的是第一个方法。
 
###生成器主要应用场景(现阶段我感觉只有下面两个用处)
 
 * 同步化的方式表达异步操作,解决金字塔回调问题。
 
before like this
```
        //normal version
        function echo(content,cb) {
            cb(null,content);
        }
        function run() {
            echo('hello', (...rest0) => {
                 console.log(rest0[1]+" world");
            });
        }
        run();  //hello world
```

generator equivalent writing
```
//thunk version
function echo(content) {
    return cb => {
        cb(null,content);  //generator作为logic主体,主线程执行异步方法
    }
}
function run(genFn) {
    const gen=genFn();
    const next= value =>{
        const ret=gen.next(value);
        if(ret.done)return;
        ret.value((err,val) => {  //async tasks
            if(err) return console.error(err);
            next(val);  //recursion
        });

    }
    next();
}
run(function* () {
    const msg0= yield  echo('hello');
    const msg1= yield  echo(`${msg0} world`);
    console.log(msg1); //hello world
});
```

 * 对象上部署Iterator接口,使其具有可遍历性
```
function* setIterator(obj) {
     let keys = Object.keys(obj);
     for (let i=0,len=keys.length; i < len; i++) {
       let key = keys[i];
       yield [key, obj[key]];
     }
   }

   let obj = { name: 'ycg', age: 'infinity' };

   for (let [key, value] of setIterator(obj)) {
     console.log(key, value);
   }

// name ycg
// age infinity
```
 
 
 
 
 
 
 
 
 
 
 
 