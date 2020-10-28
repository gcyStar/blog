---
title: webpack源码之loader机制.md
date: 2018-04-04
tags: [webpack,loader]
author: gcyStar
---
## loader概念
loader是用来加载处理各种形式的资源,本质上是一个函数, 接受文件作为参数,返回转化后的结构。

>loader 用于对模块的源代码进行转换。loader 可以使你在 import 或"加载"模块时预处理文件。因此，loader 类似于其他构建工具中“任务(task)”，并提供了处理前端构建步骤的强大方法。loader 可以将文件从不同的语言（如 TypeScript）转换为 JavaScript，或将内联图像转换为 data URL。loader 甚至允许你直接在 JavaScript 模块中 import CSS文件！


## loader和plugin区别
之前一篇文章中介绍了plugin机制,和今天研究的对象loader,他们两者在一起极大的拓展了webpack的功能。它们的区别就是loader是用来对模块的源代码进行转换,而插件目的在于解决 loader 无法实现的其他事。为什么这么多说呢?因为plugin可以在任何阶段调用,能够跨Loader进一步加工Loader的输出,在构建运行期间,触发事件,执行预先注册的回调,使用compilation对象做一些更底层的事情。

## loader用法
### 配置
```
module: {
    rules: [
      {
        test: /\.css$/,
        use: [
          { loader: 'style-loader' },
          {
            loader: 'css-loader'
          }
        ]
      }
    ]
  }
```
### 内联
```
import Styles from 'style-loader!css-loader?modules!./styles.css';
```
### CLI

```
webpack --module-bind 'css=style-loader!css-loader'
```
**说明**  上面三种使用方法作用都是将一组链式的 loader, 按照从右往左的顺序执行,loader 链中的第一个 loader 返回值给下一个 loader。先使用css-loader解析 @import 和 url()路径中指定的css内容,然后用style-loader 会把原来的 CSS 代码插入页面中的一个 style 标签中。

## loader实现
```
//将css插入到head标签内部
module.exports = function (source) {
    let script = (`
      let style = document.createElement("style");
      style.innerText = ${JSON.stringify(source)};
      document.head.appendChild(style);
   `);
    return script;
}
//使用方式1
resolveLoader: {
   modules: [path.resolve('node_modules'), path.resolve(__dirname, 'src', 'loaders')]
},
{
    test: /\.css$/,
    use: ['style-loader']
},
//使用方式2
将自己写的loaders发布到npm仓库,然后添加到依赖,按照方式1中的配置方式使用即可
```
**说明** 上面是一个简单的loader实现,同步的方式执行,相当于实现了style-loader的功能。
## loader原理

 ```/Users/gcyAccount/Desktop/learnDemo/workspace/zf/20181/32.webpack-debug/node_modules/.4.4.1@webpack/lib/SingleEntryPlugin.js
 function iteratePitchingLoaders(options, loaderContext, callback) {
 	var currentLoaderObject = loaderContext.loaders[loaderContext.loaderIndex];
 	// load loader module
 	loadLoader(currentLoaderObject, function(err) {
 		var fn = currentLoaderObject.pitch;
 		runSyncOrAsync(
 			fn,
 			loaderContext, [loaderContext.remainingRequest, loaderContext.previousRequest, currentLoaderObject.data = {}],
 			function(err) {
 				if(err) return callback(err);
 				var args = Array.prototype.slice.call(arguments, 1);
 				if(args.length > 0) {
 					loaderContext.loaderIndex--;
 					iterateNormalLoaders(options, loaderContext, args, callback);
 				} else {
 					iteratePitchingLoaders(options, loaderContext, callback);
 				}
 			}
 		);
 	});
 }
 ```
 **说明**  上面是webpack源码中loader执行关键步骤,递归的方式执行loader,执行机流程似于express中间件机制
 
 参考源码
 webpack: "4.4.1"
 webpack-cli: "2.0.13"
 
 参考文档
 https://webpack.js.org/concepts/loaders/
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
