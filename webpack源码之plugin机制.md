---
title: webpack源码之plugin机制
date: 2018-03-28 
tags: [webpack,plugin]
---
### 引言
> 在上一篇文章Tapable中介绍了其概念和一些原理用法,和这次讨论分析webpack plugin的关联很大。下面从实现一个插件入手。

### demo插件
```
function FileListPlugin(options) {}

FileListPlugin.prototype.apply = function(compiler) {
  compiler.plugin('emit', function(compilation, callback) {
    var filelist = 'In this build:\n\n';
    for (var filename in compilation.assets) {
      filelist += ('- '+ filename +'\n');
    }
    compilation.assets['filelist.md'] = {
      source: function() {
        return filelist;
      },
      size: function() {
        return filelist.length;
      }
    };

    callback();
  });
};

module.exports = FileListPlugin;
```

**说明** demo例子参考了webpack的[官方文档](https://webpack.js.org/contribute/writing-a-plugin/),使用这个简短的demo作为我们分析的入口,一步一步来分析。
首先我们写插件都是这种结构形式,只有这样webpack才能解析。而上面这个简短的插件的作用是将build后asset目录下的所有的文件遍历后取出文件名,然后生成一个filelist.md文件。
原型上为什么要有apply方法呢?因为在安装插件时，apply方法会被 webpack compiler 调用一次。调用的目的是为了注册你的逻辑,指定一个绑定到 webpack 自身的事件钩子。

webpack的事件钩子有很多如下所示,列举几个比较重要常用的的,加深下印象
>* compile 编译器开始编译
>* compilation 编译器开始一个新的编译过程
>* emit  在生成资源并输出到目录之前
>* done  完成编译

查看更多事件[钩子](https://doc.webpack-china.org/api/compiler/#event-hooks)
在上[一篇文章](https://segmentfault.com/a/1190000014031536)中分析谈到过compiler是继承自tapable,正是因为它mix了Tapable 类，才具备注册和调用插件功能,而执行plugin方法其实就相当hook.tap(tapOpt, options.fn)进行存储, 然后webpack在启动运行期间,到达某个阶段,就会触发调用相应的事件。额外传入一个 callback 回调函数,只有在插件中操作是异步的时候才需要,同步操作不需要传入和执行这个callback。
还有一点需要注意的是compiler和compilation区别?
> compiler 对象代表了完整的 webpack 环境配置。这个对象在启动 webpack 时被一次性建立，并配置好所有可操作的设置，包括 options，loader 和 plugin。当在 webpack 环境中应用一个插件时，插件将收到此 compiler 对象的引用。可以使用它来访问 webpack 的主环境。
  
> compilation 对象代表了一次资源版本构建。当运行 webpack 开发环境中间件时，每当检测到一个文件变化，就会创建一个新的 compilation，从而生成一组新的编译资源。一个 compilation 对象表现了当前的模块资源、编译生成资源、变化的文件、以及被跟踪依赖的状态信息。compilation 对象也提供了很多关键时机的回调，以供插件做自定义处理时选择使用。

## 安装使用
```
const FileListPlugin = require('fileList');
module.exports = {
    entry: './src/main.js',
    output:{
        path: path.join(__dirname,'dist'), 
        filename: '[name].js'
    },
    plugins: [
        new FileListPlugin({options: true})
    ]
}
```
输出结果
![webpack-plugin-jsdt](https://img.wuage.com/15222238440703webpack.png)

[demo完整链接](https://github.com/gcyStar/daily-practice/tree/master/webpack/webpack4/case1)

参考链接

https://doc.webpack-china.org/contribute/writing-a-plugin
https://doc.webpack-china.org/api/compiler/#event-hooks
https://webpack.js.org/contribute/writing-a-plugin/






























