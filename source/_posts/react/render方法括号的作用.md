---
title: render方法括号的作用
date: 2017-12-07
tags: [react,render,括号表达式]
author: gcyStar
---
在学习过程中,发现一个小问题,因为小很少人关注讨论过这个问题, react构建component的render方法中return后面为什么要加括号?
下面尝试给出些解释。
首先('<p></p>'),(1,2)这些都是原生支持的写法,括号中多个合法的表达式都会执行,返回最后一个表达式的值。如下demo示例所示
```
function a() {
console.log('a')
}
function b(){
console.log('b')
}
var ans = (a(),b(),'c')
console.log(ans)
//输出 a b c
```
但是()在return后的作用又不一样了,起到分隔的作用
```
function a(){
return 1
2
}
a() //输出1
上面的等价于下面的写法,按照行自动添加分号,分号表示一句执行表达式结束。并且只会执行表达式1
function a(){
return 1;
2;
}

function c(){
return( 
1,
2)
}
c() //输出 2
与上面的表达式不同的是,表达式1和2都会执行,这就是括号的作用。
```

而react的render方法return括号中不是可执行表达式,而是一些html标签,执行会报错
```
function test () {
	return (
                <div>
                    <p>test</p>
                </div>
            )
}
上面写法等价于下面的写法
function test () {
 return <div>
         <p>test</p>
  </div>
       }
最终经过babel-jsx转义才能被浏览器执行,转义结果如下。
function test() {
  return React.createElement(
    "p",
    null,
    React.createElement(
      "span",
      null,
      "test"
    )
  );
}
```
所以可以得出结论,render方法中return结果添加括号的目的,是为了更符号原生编码习惯的的思维,并且在一些IDE,例如webstrome中编写时html标签自动对齐方式更好。


















