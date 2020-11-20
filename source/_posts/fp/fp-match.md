---
title: match
date: 2017-06-17
tags: [javascript,fp]
author: gcyStar
---

# 引言
> js的模式匹配不强大,但是也有一些应用,最近在利用业余周末时间断续系统性的学scala,我学习scala目的就是理解和掌握它的函数式编程， 整体还没学完。在这篇文章中会js结合scala的方式一起对比分析总结下,不说明默认是js写法。

# Destructure

## Object

```
let name;
let {people:{age=Infinity,name}} = {people:{name:"JSDT"}}
// name "JSDT"
```

## Array

```
let [a=1,b=2,c]=['a',]
// a 'a' 、 b 2 、c 'undefined'
```

## Function

```
function JSDT([first,...rest]) {
    console.log(first,rest)
}
JSDT([1,2,3,4]);
//1,[2,3,4]
```

# ADT(Algebraic Data Type)

上述本质来说基于位置match,基础且常用,scala中也有,比较基础,我不想重复枚举了。在js中不支持自定义类型匹配,但是在模式匹配中这是重要一环,由于最近在学scala,所以借鉴一下里面的思想和实现方式,因为其原生提供ADT方式的match。

## 原生类型(Scala)

```
  def acceptAny(x:Any):String={
    x  match {
      case s:String =>"a string"
      case i:Int if(i<20) => s"an int less than 20: $i"
      case  _ => "don`t know"
    }
  }
   def main(args: Array[String]): Unit = {
      println(acceptAny(10));  //an int less than 20: 10
   }
```
## 自定义数据类型(scala)

```
case object Nil extends List[Nothing]

case class Cons[+A](head: A, tail: List[A]) extends List[A]
<!------------------------------->
def sum(ints: List[Int]): Int = ints match {
  case Nil => 0 // 空list 返回0
  case Cons(x,xs) => x + sum(xs) // recursive 求和
}

  val x = List(1,2,3,4,5) match {
    case Cons(x, Cons(2, Cons(4, _))) => x
    case Nil => 42
    case Cons(x, Cons(y, Cons(3, Cons(4, _)))) => x + y  
    case Cons(h, t) => h + sum(t)
    case _ => 101
  }

  def main(args: Array[String]): Unit = {
   println( sum(List(1,2,3)))  //6
   println( x)  //3

  }
```
 **说明** 为了搞明白和运行这个例子研究了有好一会儿,书上(scala函数式编程Page:25)写的没问题,但是写的不完整,我做了部分补充。
sum示例中模式匹配空构造类型Nil和非空构造类型Cons(由head和tail{tail由List构成}构成), 求和是通过递归的方式;
x match虽然case3、4、5都匹配,但是第一次匹配上的才会生效。
总的来说ADT这种方式优点是灵活,借助这种强大的自定义类型匹配系统,可以简化代码结构,使代码更易读和维护。


## 自定义数据类型

```
const ListOf = T => {
    var List = Type({
        Nil:[],
        Cons:[T,List]
    });
    return List;
}
const LoN = ListOf(Number);
let list= LoN.Cons(1,LoN.Cons(2,LoN.Cons(3,LoN.Nil)));
// let list= LoN.Cons(1);

const sum = LoN.case({
    Cons:(head,tail) => head + sum(tail),
    Nil:(x) => 0
});
console.log(sum(list));  //6
```

**说明**  js本身不支持adt的方式匹配,借助第三方工具[union-type](https://github.com/paldepind/union-type)可以实现,上述我写了一个和scala sum模式匹配功能相同的match例子,由于是非原生,所以写法上冗余,但其思想上和scala一样,只是形式上不一样。













