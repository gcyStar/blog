---
title: javaScript代码优化
date: 2017-02-18
tags: [node,javaScript]
author: gcyStar
---
# v8层面
### 对象和属性
```
var person = {
    multiply:function (a,b) {
        return a*b;
    },
    name:'gcy'
}

for(let  i=0;i<1000;i++){
person.multiply(i,i);
}
```
**说明** 定义对象的时候,最好一开始就初始化好对象的属性,不要动态的添加,并始终保持以相同的顺序实例化对象属性,这样便可以共享隐藏类。在java或c++静态语言中,类的信息是确定的,所以每个对象包含哪些成员和成员在
对象中偏移量在编译阶段确定,基地址和偏移地址就可以快速的访问对象内部信息。js中对象属性可以动态添加或删除,为了实现按照索引的方式快速访问成员,v8内部产生了一种数据
结构即隐藏类(基于运行中的对象分类)。动态添加属性会造成隐藏类派生,同时无法使用v8优化(针对热点代码,v8会使用优化编译器,目前默认是 CrankShaft,比如上述示例for循环中会进行参数预测,标记为整形)后的代码。另外推荐在编写代码的时候进行不要让程序进行类型推导,方案有flow和typeScript,flow我用过,侵入性低、容易上手,推荐使用,这样做的目的一方面在大项目协作过程中可以使代码具有良好的维护性,其次还可以提高v8执行效率,避免优化回退(重新执行函数->ast->机器码过程)。
### 标记值

V8用32位表示对象和数字,其中一位用来判断是对象(flag=1)还是整数(flag=0)。所以如果一个数值大于31位,v8会将数字装箱,转化为double,并创建一个新对象将该数字放在里面。所以要尽可能使用31位有符号数字，从而避免昂贵的转换为JS对象的装箱操作。

### 方法和数组

由于v8内联缓存,重复执行相同方法的代码将比只执行一次的代码运行的快。其次使用数组时最好不要用稀疏数组,且尽量避免预分配大数组。元素不全的稀疏数组是一个哈希表,访问数组中的元素比较耗费时间。


# c++层面

```
 void Method(const FunctionCallbackInfo<Value>& args) {
        Isolate* isolate = args.GetIsolate();
        //isolate  V8 runtime 
        args.GetReturnValue().Set(String::NewFromUtf8(isolate, "hw gcy"));
    }

    void init(Local<Object> exports) {
        NODE_SET_METHOD(exports, "hw", Method);  //向外导出方法
    }

    NODE_MODULE(addon, init)
   
    <!--使用-->
    const binding = require('./build/Release/binding');
     console.log(binding.hw()); 
```
c++拓展node 模块(针对底层拓展和效率要求极高场景)

为什么使用c++编写node?
http://www.tuicool.com/articles/fea63m3

#  tools层面

[Prepack][2] (js执行效率优化)

webpack2 (tree-shaking) [体积优化][1]

#  内存层面

```
var vm=new  WeakMap();
var b=new Object();
vm.set(b,{'name':'gcy'});
b=null;
vm.get(b);
```
使用weakMap,在key对应的对象,可能会消失的情况下,会减少内存泄漏发生的概率,合理的利用和使用内存的方式之一。
参考链接 [weakmap][3]

#  dom层面

 **最小化dom访问次数,注意重绘和重排,尽可能在JavaScript端处理,'离线操作DOM tree',使用缓存,减少访问布局信息次数**

### 案例一
不要频繁获取布局信息,因为导致渲染队列刷新,例如常用的offsetTop属性,offsetWidth属性等等。

###  案例二
批量修改dom时,使元素脱离文档流(!important),应用多重改变逻辑,然后再把元素放回文档中

脱离文档流的三种方式
* 隐藏元素,应用修改,重新显示
* fragment方式额外创建DOM tree
* 原始element copy,然后修改后,在 replace
* 动画元素使用绝对定位的方式
* virtual dom方式,操作vnode,diff后在实际元素上应用最终逻辑   
 
 **使用事件委托减少事件处理器数量(本质上是利用冒泡机制,比较简单,不在举例)**

# js层面
## 案例一
```

function ProcessArray(items,process,callback) {
    let todo=items.concat();
    setTimeout(function () {
        process(todo.shift());
        //若谷还有待处理元素,创建另外一个定时器
        if(todo.length>0){
            setTimeout(arguments.callee,20);
        }else {
            callback(items);
        }
    },20)
}
```
**说明**  上面优化的目的是为了避免browser的锁定,还可以使用web work的方式
```
var worker=new Worker('process.js');
worker.onmessage=function (event) {
    //logic
}
worker.postMessage(items)
// process.js
self.onmessage=function (event) {
    var ans=process(event.data);
    self.postMessage(ans);

}

```
## 案例二
```
var num1=5;
var num2=6;
eval("num1+num2");
new Function("arg1","arg2","return arg1+arg2");
setTimeout("sum1=num1+num2",100)
```
**说明**  避免双重求值

## 案例三

```
function fact1(n) {
    if(n==0)return 1;
    return n*fib(n-1);
}
-----------------------------------
fact2(n,acc=1){
    if(n==0)return acc;
    return fib2(n-1,acc*n);
}

```
**说明**  尾递归的优化的目的是为了防止栈溢出

关于js方面的技巧蛮多的,比如延迟加载,位操作,throttle等等,不在一一列举。

# 总结

js优化这篇文章小部分是工作的总结,大部分是看书(高性能js和NODE进阶之路等)后的理解和总结,其实点蛮多的,以后再补充吧。

参考链接

https://blog.sessionstack.com/how-javascript-works-inside-the-v8-engine-5-tips-on-how-to-write-optimized-code-ac089e62b12e

[1]: https://segmentfault.com/a/1190000008683201
[2]: https://prepack.io/
[3]: http://es6.ruanyifeng.com/#docs/set-map

