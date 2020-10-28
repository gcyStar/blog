---
title: webpack&loader
date: 2017-05-12
tags: [webpack,loader]
author: gcyStar
---


#引言
在回答一个问题时,引发一些疑问,分析总结下,作为备忘


# webpack
> webpack对于我来说,应用场景主要是,编译打包我通过模块化组织书写的文件,用其提供的各种loader可以让我在js中模块化的加载、管理各种格式resource,以及其附属生态圈各种plugin进行功能拓展(例如常用的CommonsChunkPlugin、UglifyjsWebpackPlugin等等), webpack-dev-server做热加载等等。
具体一些使用配置介绍,可以参考我之前一篇[相关文章][1]
## webpack 加载方式
首先webpack build后的文件中代码,通过灵活的配置可以被打包成各种标准格式的代码(AMD,CMD等等),
在js中表面上可以通过形如import x form xx、require('x')、require.ensure()等方式引入各种资源,本质上是先通过resolve解析,然后loader加载,类似管道的效果进行各种预先设置好加工处理。


# loader
> A webpack loader is a Node module that tells webpack how to take some input content and transform it into output JavaScript.
loader本质上是一个可以模块化使用的加载处理resource的函数。本文着重讨论,以raw loader作为样本分析。

##  raw loader

> Loads raw content of a file (as utf-8).

**配置**
```
 entry:{
        "case7":__dirname+'/index.js',
    },
    output:{
        path: __dirname+'/dist',
        filename:'[name].js',
    },
    module: {
        rules: [
            {
                test: /\.css$/,
                use: 'raw-loader'
            }
        ]
    }
```

**index.js**
```
import str from './file.css';
console.log(str);
```
执行webpack后,运行case7.js,输出如下
```
body{
    background: black;
}
```
更改配置
```
原配置中添加
{
    test: /\.css$/,
    use: 'raw-loader'
}
--------------------------------  equivalent way
原配置中替换
{
    test: /\.css$/,
        use: ['raw-loader','raw-loader']
}
```
更改配置后输出
```
module.exports = "body{\n    background: black;\n}"
```
为什么加载两次后输出会不一样,module.exports从何而来?

```
module.exports = function(content) {
        <!-- other  logic …… -->
	return "module.exports = " + JSON.stringify(content);
}
```
上面是raw源码中截取的部分内容,content是原样读取的文件,本身可能带有\n,\t等,直接启动触发入口
```
return fn.apply(context, args);  //loaderRunner.js line:119
```
一开始输出没有\n等,因为这些特殊字符没有转义,解释执行时保持其含义输出,module.exports是raw-loader中自带,webpack以此作为标识解析loader中输出的模块化资源,否则webpack打包时无法处理资源,默认为空对象{}。

```
modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);   执行module函数,有导出借助modle参数,没导出仅做执行分割后文件逻辑使用  //line:20

--------
             (function(module, exports) {     //line:78

            module.exports = "body{\n    background: black;\n}"

             }),
 编译后文件中自执行函数中的参数,每一个参数都代表一个模块化的资源,模块资源导出的资源被存储到module上。
 ----------------
return module.exports;   // Return the exports of the module,在其它模块中被引入时会被使用   //line:33
 
```
重复执行两次,第二次的输入源是第一次的返回值
![gcy-loader-example](https://img.wuage.com/149457866124712loader-gcy.png)
![gcy-loader-example](https://img.wuage.com/149459140595735loader-test.png)

上面两幅图具体的说明了输出结果的起因,输入源经过stringify后特殊字符被转义,拼接 "module.exports =" 后作为第二次执行的输入源,webpack在模块处理时,同一个资源最后一次输出为准。

下面一幅图分别是单loader和双loader编译结果图。
![gcy-loader-example](https://img.wuage.com/149457928258826loader-ans-gcy.png)

# 总结
通过简单的例子,复习了webpack编译后执行流程,探索了编译流程,研究意义还是有的。
相关问题
https://segmentfault.com/q/1010000009368556?_ea=1907400


参考链接: [loader1][2]
参考链接: [loader2][3]


[1]:https://segmentfault.com/a/1190000008683201
[2]:https://webpack.js.org/concepts/loaders/
[3]:https://bocoup.com/blog/webpack-a-simple-loader

































