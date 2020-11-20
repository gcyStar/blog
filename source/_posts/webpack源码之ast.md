---
title: webpack源码之ast简介
date: 2018-04-02 20:25:31
tags: [webpack,AST]
author: gcyStar
---
## 什么是AST
树是一种重要的数据结构,由根结点和若干颗子树构成的。 根据结构的不同又可以划分为二叉树,trie树,红黑树等等。
今天研究的对象是AST,抽象语法树,它以树状的形式表现编程语言的语法结构，树上的每个节点都表示源代码中的一种结构。
通过操作这棵树,可以精准的定位到声明、赋值、运算语句,从而实现对代码的分析、优化、变更等操作。

## AST应用场景
![ast-babel](https://img.wuage.com/152274423775587astbabel.png)


>* 代码风格,语法的检查,IDE中的错误提示,格式化,自动补全等等
 
>* 优化变更代码，代码压缩等等

>* es6转es5,以及TypeScript、JSX等转化为原生Javascript等等


## AST处理步骤

js中借助于一些库可以把js源码解析为语法树，比如 Babylon, esprima、acorn、UglifyJS、AST explorer等等,如下所示是一个简单的示例。
```
var a = 42;
var b = 5;
ar c = a + b;
```
![ast](https://img.wuage.com/152266353792190ast.png)
**说明**  一个简单的ast树示例,对应的json格式如下所示
```
{
  "type": "Program",
  "start": 0,
  "end": 37,
  "body": [
    {
      "type": "VariableDeclaration",
      "start": 0,
      "end": 11,
      "declarations": [
        {
          "type": "VariableDeclarator",
          "start": 4,
          "end": 10,
          "id": {
            "type": "Identifier",
            "start": 4,
            "end": 5,
            "name": "a"
          },
          "init": {
            "type": "Literal",
            "start": 8,
            "end": 10,
            "value": 42,
            "raw": "42"
          }
        }
      ],
      "kind": "var"
    },
    {
      "type": "VariableDeclaration",
      "start": 12,
      "end": 22,
      "declarations": [
        {
          "type": "VariableDeclarator",
          "start": 16,
          "end": 21,
          "id": {
            "type": "Identifier",
            "start": 16,
            "end": 17,
            "name": "b"
          },
          "init": {
            "type": "Literal",
            "start": 20,
            "end": 21,
            "value": 5,
            "raw": "5"
          }
        }
      ],
      "kind": "var"
    },
    {
      "type": "VariableDeclaration",
      "start": 23,
      "end": 37,
      "declarations": [
        {
          "type": "VariableDeclarator",
          "start": 27,
          "end": 36,
          "id": {
            "type": "Identifier",
            "start": 27,
            "end": 28,
            "name": "c"
          },
          "init": {
            "type": "BinaryExpression",
            "start": 31,
            "end": 36,
            "left": {
              "type": "Identifier",
              "start": 31,
              "end": 32,
              "name": "a"
            },
            "operator": "+",
            "right": {
              "type": "Identifier",
              "start": 35,
              "end": 36,
              "name": "b"
            }
          }
        }
      ],
      "kind": "var"
    }
  ],
}
```
通过操纵解析出来的ast,可以实现我们AST应用场景中列出的一些应用。
下面针对上面列出的ast树做一些简单说明:
任何一颗ast树根节点的类型都是Program,start和end记录了字符的位置,body表示程序体,其内部是三个简单的变量声明,每个变量声明中记录了标示符以及字面量的值。最后一个变量c中init是一个BinaryExpression(二元运算表达),记录的不是字面值,而是引用到的标示符和操作符。想要实现应用场景中举的示例,大致就是遍历,修改,删除,移动这棵树上的节点,最后遍历处理后ast生成最终代码。



## webpack和ast

```
//compile.js
this.hooks.make.callAsync(compilation, err => {});
----
//NormalModules.js
runLoaders(
    {
        resource: this.resource,
        loaders: this.loaders,
        context: loaderContext,
        readResource: fs.readFile.bind(fs)
    },
    (err, result) => {
        this._source = this.createSource(
            this.binary ? asBuffer(source) : asString(source),
            resourceBuffer,
            sourceMap
        );
       
        return callback();
    }
);
----------------------------------
//Parse.js
const acorn = require("acorn-dynamic-import").default;
ast = acorn.parse(code, parserOptions);
if (this.hooks.program.call(ast, comments) === undefined) {
    this.detectStrictMode(ast.body);
    this.prewalkStatements(ast.body);
    this.walkStatements(ast.body);
}
```
**说明**  上面是webpack源码中摘取的和ast处理有关的上下文关键片段
在webpack执行流程中,make是一个重要的阶段,在一个新的 Compilation 创建完毕后，即将从 Entry 开始读取文件，根据文件类型和配置的 Loader 对文件进行编译，将loader处理后的文件通过acorn抽象成抽象语法树AST,然后遍历AST，递归分析构建该模块的所有依赖。



## 总结
发现ast水很深,平时接触的也比较少, 今天算是个入门了解下,作为理解webpack源码前的铺垫。
 参考源码
webpack: "4.4.1"
webpack-cli: "2.0.13"
参考文档
https://github.com/acornjs/acorn
https://zh.wikipedia.org/wiki/%E6%8A%BD%E8%B1%A1%E8%AA%9E%E6%B3%95%E6%A8%B9
https://www.sitepoint.com/understanding-asts-building-babel-plugin/



