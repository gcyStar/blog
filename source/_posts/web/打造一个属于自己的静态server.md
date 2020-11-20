---
title: 打造一个属于自己的server
date: 2018-03-01
tags: [es6,javaScript]
author: gcyStar
---
## 什么是server
在开始打造自己的服务器之前,我们首先明确一下服务器的定义:一个管理资源并为用户提供服务的计算机软件。

## 根据功能服务器划分为以下两类

>* static web server,例如常见的nginx apache等等
>* dynamic web server,例如常见的tomcat,jboss,resin等等

## 动静态服务器区别

对于静态服务器来说一般就是读取资源然后返回给browser;动态服务器意味着返回给browser的文件是经过逻辑处理动态产生的。

# 服务器具有的功能特性
>* nginx,tomcat这个两个之前用过,也研究过,所以拿这两个举一下示例,不过现在很少用了,现在基本上都是使用node相关的,所以最后构建的serve会基于node。

## nginx

### nginx特点

>* 配置简单,灵活(只有一个主配置文件nginx.conf)
>* 支持高并发(静态小文件)
>* 占用资源相对较少(2w并发,开启10个线程,内存消耗只有几百M)
>* 功能种类多(例如proxy,cache,Log,Gzip等等)

### nginx应用场景
#### 静态服务器(图片,js,css等等)
```
server {
        listen       8080;
        server_name  localhost;

        location / {
            root   html;
            index  index.html index.htm;
        }
}
```

**说明** 上面是nginx配置,指定访问根目录和默认主页,以及监听端口

```
➜  ~ clear
➜  ~ curl -i  http://127.0.0.1:8080
HTTP/1.1 200 OK
Server: nginx/1.12.2 //服务器类型和版本
Date: Fri, 02 Mar 2018 08:49:44 GMT
Content-Type: text/html
Content-Length: 11
Last-Modified: Fri, 02 Mar 2018 08:46:27 GMT //支持Last-Modified缓存机制
Connection: keep-alive //支持持久连接
ETag: "5a990f63-b"  //支持ETag缓存机制
Accept-Ranges: bytes // 支持断点续传

hello  jsdt% //响应体
```

**说明**  上面是本地测试请求,从响应头中可以看到支持很多功能

#### 反向代理,负载均衡

![jsdt-server-demo](https://img.wuage.com/151998515776555server-jsdt-demo.png)
**说明**  上面是试验效果

```
 upstream  jsdt.com {  
          server    127.0.0.1:8083  max_fails=3 fail_timeout=30s weight=1;
          server    47.97.xxx.xxx:8084  max_fails=3 fail_timeout=30s  weight=2;  //为了安全 隐藏真实ip地址
      } 
    server {
        listen       8080;
        server_name  localhost; 

        location / {
            root   html;
           # index  index.html index.htm;
            proxy_pass http://jsdt.com;  
            proxy_redirect default;
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }}
        
```
**说明**  上面我摘取了关键的部分配置,采用了轮训+weight算法,其它还有ip_hash、url_hash等算法。真实的应用情况,还需要考虑很多问题,例如集群的session同步,记得大学实习期间,当时公司用的是cookie+memcache集群的方案。

## tomcat

### tomcat特点
> tomcat运行在jvm上,跨平台,是一个Servlet容器(可以运行Servlet,编译jsp),实现了在http请求响应处理中所需要的http接口相关实现类。除此之外也支持虚拟主机,session共享,静态文件处理等等,只不过没那么专业而已。

### tomcat应用场景
![tomcat-jsdt](https://img.wuage.com/152005445853060tomcat-jsdt.png)
**说明** 如上所示,我们可以在页面中添加动态的处理逻辑,返回的数据根据用户可定制化(相比静态服务器优点),最终.jsp被tomcat编译为.java,然后被javac编译为通用字节码文件,最终运行在jvm上。


## 如何实现一个自己服务器
> 在实现自己的服务器之前,首先我们明确一下server的本质,server属于应用层的协议,基于tcp的封装, 而tcp的应用实现是基于socket(无论是node,还是java都有socket)的封装。
socket监听某个端口,获取面向流的数据data,我们的server所要做的就是对data进行解析封装,以使其符合http的规范。

## 接下来实现自己的静态server
因为有http模块,所以node当中实现一个基础server很简单。但是如果附加额外的功能,例如压缩,缓存,断点续传,反向代理什么的就需要自己添加了。
接下来首先看一下项目结构,bin目录主要是放启动脚本相关的,主逻辑在app.js中,然后根据功能将代码拆分成不同的模块。templatet目录放置编译的原始模板。
```
|____bin
| |____.DS_Store
| |____deamon.js
| |____start
| |____yargsConfig.js
|____node_modules
|____package-lock.json
|____package.json
|____readme.md
|____src
| |____.DS_Store
| |____app.js
| |____asset
| |____cacheSupport.js
| |____config.js
| |____picGuard.js
| |____template
| |____util.js
```
在server运行前,首先我们通过yargs模块获取解析好的命令行参数。如下所示
```
if(argv.D){
    let sp = cp.spawn(process.execPath, ['deamon.js'],{
        cwd: __dirname,
        stdio: ['ignore','ignore','ignore'],
        env: argv,
        detached: true  //http://nodejs.cn/api/child_process.html#child_process_child_process_spawn_command_args_options
    } )
    sp.unref()
} else {
    let config = Object.assign({}, defautConfig, argv)
    let server = new Server(config);
    server.start();
    console.log('server already started')
}
```
**说明**  如果开启deamon模式,则通过子进程的方式让服务在后台运行,反之则直接启动server实例

在启动server之后,开始接受并处理请求,下面以断点续传功能模块作为示例
```
function byteRangeStream(req, res, filepath, statObj) {
    let start = 0
    let end = statObj.size-1
    let range = req.headers['range']
    if (range){
        res.setHeader('Accept-Range','bytes')
        res.statusCode = 206 //a part of content
        let result = range.match(/bytes=(\d*)-(\d*)/);
        if (result) {
            start = isNaN(result[1]) ? start : parseInt(result[1]);
            end = isNaN(result[2]) ? end : parseInt(result[2]) - 1;
        }
    }
    return fs.createReadStream(filepath,{
        start,
        end
    })
}
module.exports ={
    byteRangeStream
}
```
**说明** 在主模块app.js中,导入上述模块,如代码中所示首先判断客户端是否支持断点续传,依据range请求头,如果有请求范围,直接返回请求范围内的数据,否则全部读取返回,靠的是browser和server的协商机制,需要双方都支持才能完成整个过程。
更多功能模块可以参考我的[github](https://github.com/gcyStar/server-cli), 欢迎star。

## 总结
写这篇文章,总结了下server的相关知识,参考了之前大学时做的笔记,看到之前做的记录,回忆当时在学校学习和公司实习的经历,感慨万千。时光易逝,做好当下的自己。


























