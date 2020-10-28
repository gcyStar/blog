---
title: javaScript中的函数应用
date: 2017-02-18
tags: [函数,javaScript]
author: gcyStar
---
##概要
js当中的函数站在不同的角度有不同的分类和应用,本文站在高阶函数的角度来讨论js当中函数的应用场景。

首先明确高阶函数定义:

 1. 函数可以作为参数被传递
 2. 函数可以作为返回值输出

##应用场景
###函数作为参数被传递
这种情况其实是比较常见的,日常用的挺多的,例如ajax请求成功之后的callback函数、各种插件,框架当中的钩子函数等等。下面举个我们常用的例子:

```
var tmpAry=[2,1,3];

tmpAry.sort(function(a,b){
    return a-b;
});
console.log(tmpAry);   //[1,2,3]
```


目的:把函数中易变或公用逻辑抽离,封装成函数参数,隔离变与不变部分,达到更好的封装和复用。
###函数作为返回值输出
这个其实我在日常写代码中用的比较少,但并不能说明这个不常见,所以下面我会多举几个例子。
####
解决老版本浏览器bind函数兼容性问题

```
Function.prototype.bind=function(context){
    var self=this;
    return function(){
        return self.apply(context,args);
    }
}
```


###判断数据类型

```
var isType=function(type){
    return function(obj){
        return Object.prototype.toString.call(obj)==='[object '+ type + ']';
    }
}
var isString=isType('String');
console.log(isString('cgy'));  //true
```


###高阶函数实现aop
AOP也就是面向切面编程,在java中这个概念基础且重要。具体就是通过reflect或动态代理(jdk动态代理和cglib动态代理)动态的在业务函数之前或之后添加一些
可复用的代码逻辑,例如典型的日志统计、权限控制等等。用aop的目的就是希望业务逻辑模块高内聚,同时达到非业务公共模块可复用,易维护。

```
Function.prototype.before=function (beforefn) {
    var _self=this;//保存原函数的引用
    return function () {  //代理函数
        beforefn.apply(this,arguments);
        _self.apply(this,arguments);
    }
};

Function.prototype.after=function (afterfn) {
    var _self=this;
    return function () {
        var ret=_self.apply(this,arguments);
        afterfn.apply(this,arguments);
        return ret;  //执行原函数,并返回原函数的执行结果
    }
}
var  func=function () {
    console.log('ing');
}
func=func.before(function () {
    console.log("before")
}).after(function () {
    console.log("after");
})
func();
```


这个其实有很多引用场景,比如统计函数执行时间,动态改变函数参数等等。
###function currying
首先currying的概念是部分求值,具体含义就是指动态的接受一些参数,不立即求值,在需要计算求值时访问之前闭包中保存的参数,一次性进行求值。
下面是一个部分求值通用的currying函数。

```
var currying=function (fn) {
    var args=[];
    return function () {
        if(arguments.length===0){
            return fn.apply(this,args);
        }else{
            [].push.apply(args,arguments);
            return arguments.callee;  //指向当前匿名函数,以便于参数连续暂存
        }
    }
}
var cost=(function () {
    var money=0;
    return function () {
        for(var i=0,len=arguments.length;i<len;i++){
            money+=arguments[i];
        }
        return money;
    }
})();

var cost=currying(cost);
var step1=cost(1);
var step2=step1(2);
var step3=step2(3)
console.log(step3());

```


##总结
其实高阶函数应用场景很多,我只是总结列举了一些自己学习中碰到的典型例子,以后遇到一些适当的,会继续完善。
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 