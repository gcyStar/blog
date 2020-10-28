---
title: qcon总结
date: 2017-05-06
tags: [gc,node]
author: gcyStar
---


# front  to  back

* 业务上    : front 面向展示  交互 ;                      back   功能 服务  数据一致性等等
* 环境     : front  browser webview 单机;              back  集群  高并发  
* 思想差异  : front  快速开发  快速渲染  视觉效果  等等  ; back  服务稳定,性能,内存泄漏等等


#  V8 内存的简介

首先通过memoryUsage可以查看node进程的使用情况,具体介绍如下

```

process.memoryUsage();   查看node进程内存使用情况  单位是字节
{ rss: 20725760,  (resident set size)进程的常驻内存   
  heapTotal: 7376896,   已申请到的堆内存
  heapUsed: 3881280,  当前使用的堆内存
  external: 8772  ,  c++的内存使用,受v8管理
  }
  
```

针对上面api使用写了个相关例子

```
let  showMem=function () {
    var mem=process.memoryUsage();
    var format=function (bytes) {
        return (bytes/1024/1024).toFixed(2)+'MB';
    }
    console.log('Process :heapTotal '+format(mem.heapTotal)+' heapUsed '+ format(mem.heapUsed)+' rss '+format(mem.rss));
    console.log("------------------------------------")
}
var useMem=function () {
    var size=200*1024*1024;    //每次构建200MB对象
    var buffer=new Buffer(size);
    for(var i=0;i<size;i++){
        buffer[i]=0;
    }
    return buffer;
}
var totoal=[];
for(var j=0;j<15;j++){
    showMem();
    totoal.push(useMem());
}
showMem();

```

heapTotal和heapUsed变化极小,唯一变化rss值,且该值远超V8内存分配限制(说明:V8的内存限制：64位系统约为1.4GB、32位系统约为0.7GB  (这个规定是node源码 中限制的),具体测试可以将上述实例中buffer改成array即可)

**说明**：node 内存由通过v8进行分配的部分(新生到和老生代)(只有这部分才会受v8垃圾回收的限制)和node进行自行分配的部分(例如buffer)

**额外补充**



* --max-old-space-size 命令就是设置老生代内存空间的最大值

*  --max-new-space-size 命令则可以设置新生代内存空间的大小  

这两个参数只能在node启动时进行设置
 
 
下面从gc层面谈论下v8内存,这个可以说很多,我的云笔记中关于java和js的内存使用,分配,gc等整理了好几个系列,下面我用自己的话简单总结下。

关于内存分区有以下两个大类:
* new  space(特征:对象存活时间短)  又分为from和to两个区域,采用复制算法,空间换时间。如果存活多次,转移到老生代中。

*  老生代中(对象存活时间长),采用mark-sweep(标记清除)和mark-compact(标记整理),缺点容易形成内存碎片,导致无法分配大对象。v8主要使用mark-sweep,在内存空间不够时,才使用标记整理。老生代又可以细分Old Space(新生代中gc活过2次晋升到这个空间)、Large Object Space(大对象,一般指超过1MB,初始时直接分配到此),Map Space(“隐藏类”的指针,便于快速访问对象成员)、Code Space(机器码,存储在可执行内存中)



关于gc,总的来说在js中不同对象存活时间不同,没有一种算法适应所有场景,内存垃圾进行分代,针对不同分代使用相应高效算法,有些gc算法和java是一样的,复制(空间换时间,new Space),标记清除和标记整理(old space)等等。

**补充存活标记依据**
* 全局变量或者有全局变量出发,可以访问到的对象
* 正在执行的函数中的局部变量,包括这些局部对象可以访问到的对象。
* 闭包中的变量

关于内存,偏底层语言(例如c)和js以及java不一样

```
#c代码
 #include <stdio.h>
 void init()
{
int arr[5]={};
for(int i=0;i<5;i++){
	arr[i]=i;
}
}
void test(){
	int arr[5];
	for(int i =0;i<5;i++){
		printf("%d\n",arr[i]);
	}
}
int main(int argc,char const *argv[]){
	init();
	test();
	return 0;
}
//  0 1 2 3 4
# java代码
public class Test {

    public static  void  main(String[] args){
            init();
            test();
    }
    public static void init(){
        int[] arr=new int[]{0,1,2,3,4};
//        for(int i=0;i<5;i++){
//            System.out.println(arr[i]);
//        }
    }
    public static  void test(){
        int[] arr=new int[5];
        for(int i=0;i<5;i++){
            System.out.println(arr[i]);
        }
    }

}
//  0 0 0  0 0
```

上述例子说明c语言堆栈执行过程中,函数执行完堆栈释放,程序员如果不关注释放过程,出现一些问题 ,
而js和java就不会,因为内存自动分配,垃圾自动回收,不用考虑脏数据擦除。


## 内存泄漏

首先什么是内存泄漏?
>  对象不再被应用程序使用，但是垃圾回收器却不能移除它们，因为它们正在被引用。
node进程运行中garbage没被回收肯定属于内存泄漏,关于这个定义引起讨论是在node进程退出或崩溃后的情况,
有部分内存比如共享内存(用于进程间通信),如果没被释放,这依然属于内存泄漏,内存泄漏不仅仅只进程层面。


ppt有的,我不在列举,我在分享过程中有同学提出个疑问,关于exports使用中为什么会导致泄漏,印象比较深刻,可能当时讲的不清楚,下面写个具体例子详细阐述下。

```
//A.js
var leakAry=[];
exports.leak=function(){
    leakAry.push('leak '+" gcy "+new Date());
}
//B.js
var obj=require('./a');
for(var i=0;i<10;i++){
    obj.leak();
}
```

在node当中,为了加速module访问,所有module都会被编译缓存,上述代码导致泄漏的原因就是模块上,被缓存局部变量重复访问,因为没初始化,导致内存占用不断增大。

其次平时如何定位内存泄漏具体问题。

```
var http=require('http');
 var heapdum=require('heapdump');
var leakArray=[];
var leak=function () {
    for(var i=0;i<100000;i++){
        leakArray.push("leak "+Math.random());
    }
   
};
var i=0;
http.createServer(function (req,res) {
    leak(); //泄漏触发位置
    i++;
    res.writeHead(200,{'Content-Type':'text/plain'});
    res.end("hello world gcy"+i);
}).listen(1337);
console.log(process.pid);
console.log("server start  gcy");

-------------------------------------------------------------
for ((i=1;i<=10000;i++));
 do   curl -v --header "Connection: keep-alive" "http://127.0.0.1:1337/"
  done
  批量100和10000个请求,记录heap dump镜像,通过对比视图查看变化比较大地方,通过分析定位具体泄漏位置,展示效果如下图
```

![gcy-mem-leak-example](https://img.wuage.com/149406063689981memleak.png)

可以看到有三处对象明显增长的地方，string、concatenated string以及 array 对象增长。点击查看一下对象的引用情况：
其实这三处对象增长都是一个问题导致的。leak执行,leakArray没有初始化，导致其里面字符串没有被清除，从而导致内存泄漏。

## 关于异步处理
问题:不适合处理复杂的状态机

解决方案:是使用队列结构进行可控的异步并发。

关于这一点理解分享中上有人提出可异议,
我的理解是比如逻辑中有大量的promise待处理,一旦此逻辑比较多,我们没法掌控,没法知晓具体执行情况,但是通过队列,在高并发请求下,大量的状态机promise通过队列
的管理,我们可以做到可控,哪些被消费了,状态机所处某个过程的比例都可以统计,一旦有这些统计信息,我们就可以进行相应的处理。求证。

## 弱计算
什么是 IO 密集型?  控制器busy

什么是 CPU 密集型?  运算器busy

关于弱计算的分析可以用node生成profile文件,然后通过chrome进行分析或者webstrome自带的v8 profifing进行分析,可以得到一系列函数执行时间统计和调用堆栈过程时间消耗统计,依据这些信息,我们在做相应的优化。

![gcy-cpu1-example](https://img.wuage.com/149407706457463profile1-by-gcy.png)

![gcy-cpu2-example](https://img.wuage.com/149407706457463profile2-by-gcy.png)



# 部署

## child_process

```
//普通情况,只是作为用法示例
var fork=require('child_process').fork;
var cpus=require('os').cpus();
for(var i=0;i<cpus.length;i++){
    fork('./work.js')
}
-----------------------
//work.js
var http=require('http');
http.createServer(function (req,res) {
   res.writeHead(200,{'Content-Type':'text/plain'});
    res.end('hello world gcy');
}).listen(Math.round((1+Math.random())*1000),'127.0.0.1');
---------------------------------------------------
---------------------------------------------------
升级版(相比上面外界访问有点外界访问只有一个端口,实际场景应用)
var server= require('net').createServer();
server.listen(1136,function () {
    child1.send('server',server);
    child2.send('server',server);
    // server.close();
});
```

### 总结
 **child_process 模块给node提供了一下几个方法创建子进程** 
  1:  spwan(); 启动一个子进程执行命令
  2: exec()  启动一个子进程执行命令,与spwan不同的是,他有一个回调函数获知子进程状况
  3: fork()  与spwan类似  不同地方在于 在创建子进程的时候只需指定 需要执行的JavaScript文件即可

上面方法很少用了,node当中有cluster模块,简单的几个方法就可以创建集群,其本质基于上面的封装,底层实现原理基于句柄共享。
```
var index=require('./app');
if (cluster.isMaster)
    for (var i = 0, n = os.cpus().length; i < n; i += 1)
        cluster.fork();
else
    index.app();
-------------------------------------------
//app.js
function app() {
    var server = http.createServer(function(req, res) {
        res.writeHead(200);
        res.end('hello world gcy\n');
        console.log(cluster.worker.id);
    });
    server.listen(8088);
}
exports.app=app;

```

cluster使用中有三个问题
1、round-robin是当前cluster的默认负载均衡处理模式（除了windows平台），自行设定负载算法
 可以在cluster加载之后未调用其它cluster函数之前执行：cluster.schedulingPolicy = cluster.SCHED_NONE。


 2、进程监控问题
 master进程不会自动管理worker进程的生死，如果worker被外界杀掉了，不会自动重启，
 只会给master进程发送‘exit’消息，开发者需要自己做好管理。


 3、数据共享问题
 各个worker进程之间是独立的，为了让多个worker进程共享数据（譬如用户session），
 一般的做法是在Node.js之外使用memcached或者redis。


cluster适用于在单台机器上，如果应用的流量巨大，多机器是必然的。这时，反向代理就派上用场了，我们可以用node来写反向代理的服务（比如用 http-proxy ）
，好处是可以保持工程师技术栈的统一，不过生产环境，我们用的更多的还是nginx，部分重要配置如下。

nginx做集群

```
upstream gcy.com{
        server 127.0.0.1:3000 weight=1;
        server 127.0.0.1:3001 weight=2;
        server 127.0.0.1:3002 weight=6;

    }
```


#  维护

维护主要做好以下三点
* 日志
* 异常处理
*  第三方依赖管理。

异常处理解释下,有人提出疑问:异常没有被捕获一路冒泡 ,会触发uncaughtException 事件,
如果异常出现之后,没有进行正确的恢复操作可能导致内存泄漏,清理已使用的资源 
(文件描述符(清除文件的占用)、句柄(Master传给work的标识)等) 然后 process.exit。



# 总结
qcon现场听一次,自己分享一次,总结一次,感觉收货蛮大的,有些地方自己加深了理解,分享中front  to  back开头部分,原先8页太多,本应一笔带过,讲的时候废话太多,原本我只想阐述其中几个关键名词。
分享本身是一个学习,开拓眼界的的过程,切身体会,还是需要自己实践,写具体case。
上述基本上是我讲的过程记录,做个总结,以便回顾。
演示部分链接, [demo](https://github.com/gcyStar/daily-practice/tree/master/node/normal)

