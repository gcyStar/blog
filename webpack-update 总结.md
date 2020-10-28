---
title: webpack2-update 总结
date: 2017-03-05
tags: [webpack,javaScript]
author: gcyStar
---
##引言
这里是webpack official提供的一到二的[升级指南][1],二兼容一的写法,给我的感觉是二的写法更规范,使用起来也比较简洁。
##重要特性枚举
###特性1
webpack2增加了对es6特性的支持。支持import和export写法。之前需要通过babel来弄这个。

重要特性tree-shaking(另一个就是rollup),基于es6静态模块分析(仅支持标准写法)。大致原理就是通过分析js的AST,依赖检查等步骤,建立一个'对象依赖树',从而将被使用和被引用的的
对象抽出,合成最小可用程序集。
```
export function A() {

}
export function B() {

}
export default {
    'A':A,
    'B':B,
}
```
上面是反模式写法,相当于export default是把后面变量先赋值给default在输出,输出的是对象,没法利用ES6模块系统的多输入多输出特性。

正规写法,分别输出。这样之后整体import只能通过import * as XXX的方式。
```
// export {
//     'A':A,
//     'B':B,
// }
```
做了个[demo][3],实验结果如下,效果还是比较明显的。

![result](http://img.wuage.com/1488618752445612017-03-04_17-06-45.png)

说一下应用场景,由于有些第三方库,是编译好的commonjs格式,虽然可以模块化引入加载,但是没法tree-shaking,除非你有未编译的es6 source。
另外在使用过程中,babel 默认是将import和export转为require,所以需要关闭预设功能。
```
["env", {"modules": false}]
```
另外我想说的在做这个demo工程中,发现开启tree-shaking后会标记无用代码,但是不会删除,要做到真正的[DCE][2],还需要进行Uglify处理。

第二个应用场景就是有利于组件化开发,下面这个截图中是平时使用到的vue component,每个component集成了html、js、css。可以作为单独组件存在,
这些组件既具有重用性,同时也可以利用es6的模块化机制结合webpack2,实现组件之间的依赖。这样做的目的就是为了项目工程化,实现大项目的分工协作,提高开发效率。

![components](http://img.wuage.com/148871726265882components.png)

###特性2

loaders改名为rules,且写loader时不能缺少后缀, 针对loader增加option参数。提倡这样做。
```
options: {
                cacheDirectory: true,
                presets: [
                    ['es2015', {modules: false}] 
                ]
            }
```
旧版loader连接器写法
```
loader: "style-loader!css-loader!less-loader"。
```

下面是我项目中的写法,之前还遇到个坑,最后那一项拆开了就error了,这种写法没条理性,不易阅读,不推荐。

```
 { test: /\.styl$/, loader: ExtractTextPlugin.extract("style-loader", "css-loader!stylus-loader") }
```
新webpack2连接器写法,非常好,简介明了。

use: [
"style-loader",
"css-loader",
"less-loader"
]
###特性3
resolve.extensions配置项将不再要求强制转入一个空字符串,下面是我使用的配置。

```
resolve: {
        extensions: ['.js', '.css','.sass', '.scss', '.less',  '.vue'],
    },
```
原先的话第一个为空,现在被移动到resolve.enforceExtension。
##特性4
json-loader在2中已经内置,读取json文件不用加loader。
取消module.preLoaders,具体用法如下所示
```
 preLoaders: [{
         test: /\.js$/,
         loader: "eslint-loader",//webpack1写法
         exclude: /node_modules/
         }]
-------------------------------------------------         
        rules: [{
            test: /\.js$/,
            loader: "eslint-loader",
            exclude: /node_modules/,
            enforce: 'pre' //webpack2写法
        }      
```
##特性5
1系列的[ExtractTextPlugin][5]不兼容wp2,需单独安装[ExtractTextPlugin][5]的v2版本
```
module: {
  rules: [
    {
      test: /.css$/,
-      loader: ExtractTextPlugin.extract("style-loader", "css-loader", { publicPath: "/dist" })
+      use: ExtractTextPlugin.extract({
+        fallback: "style-loader",
+        use: "css-loader",
+        publicPath: "/dist"
+      })
    }
  ]
}
--------------------------------------------------------------------------------------------------
plugins: [
-  new ExtractTextPlugin("bundle.css", { allChunks: true, disable: false })
+  new ExtractTextPlugin({
+    filename: "bundle.css",
+    disable: false,
+    allChunks: true
+  })
]
```
##tips
在写这个的过程中写了个[脚手架][4]基于webpack2,自己以后项目开发打算就基于这个,还需要完善。

在写这个过程中遇到几个问题:

1. 遇到的坑就是每次package install的时候都要卡一下，一定要显式的excluded(webstrome)或者Terminal install

2. 还有就是http://yeoman.io/generators/    显示不出来，说是one hour  synchronize可是超过了一个小时还是没看到。
   在此的建议就是直接install  yo一下试试看能不能用，不要干等着。


3. npm run start而不是直接webpack-dev-server，前者基于当前package.json中的版本，后者是基于系统版本。

[1]: https://webpack.js.org/guides/migrating/
[2]:https://en.wikipedia.org/wiki/Dead_code_elimination
[3]:https://github.com/gcyStar/daily-practice/tree/master/webpack
[4]:https://github.com/gcyStar/generator-webpack-2-es-6
[5]:https://github.com/webpack-contrib/extract-text-webpack-plugin