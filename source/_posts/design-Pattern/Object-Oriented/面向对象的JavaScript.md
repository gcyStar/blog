---
title: 面向对象的JavaScript
date: 2017-02-11
tags: [面向对象]
author: gcyStar
---

#概述
本来打算写设计模式的，在谈论js的设计模式之前先说一下js的面向对象,因为很多设计模式的实现都掺杂着面向对象的思想,所以先做一下前期铺垫。

js我们都知道是一种动态类型脚本型语言,变量类型无法保证,所以我们可以尝试调用任意对象的任意方法,不用考虑它原本是否被设计为拥有该方法。

什么是面向对象本文不在论述,接下来说一下面向对象的三大特征在js当中的实现。
##封装
    这个特性在设计一个框架时需要认真考虑的。封装的目的是将信息隐藏,其主要可分为以下几类

 - 数据封装

  在一些静态类型的语言如java中,本身语法就提供了这些功能。js当中只能依靠变量的作用域来实现封装的特性,并且只能模拟出public和private两种特性。
     ```
/**
 -  利用函数创建的作用域达到数据封装的目的。
 - @type {{getName, setName}}
  */

  var obj=(function () {
     var _name="gcy";
     return {
         getName:function () {
             return _name;
         },
         setName:function (val) {
             _name=val;
         }

     };
 })();
 obj._name;  //undefined
 obj.getName(); //gcy
```


 - 封装实现

    封装实现就是是对象内部的变化对外界是透明的,不可见。这种做法使对象之间低耦合,便于维护升级,团队协作开发。
    $(selector).each(function(index,element))。就比如这个each函数,我们不用关心内部实现,只要提供的功能正确就行。我们关注的只是接口调用形式。

##继承
继承在静态语言中,例如java有关键字,虽然在es6中也有extend以及class,但其本质仍实现仍是基于原型机制。
```
/**
 * 简单的es5原型继承
 * @constructor
 */
var A=function () {
}
A.prototype={name:"gcy"};

var B=function () {
};
B.prototype=new A();

var b=new B();
console.log(b.name);
/**
 * e6继承实现demo
 */
class People{
    constructor(name){
        this.name=name;
    }
    getName(){
        return this.name;
    }
}

class Black extends People{
    constructor(name){
        super(name);
    }
    speak(){
        return " i am black";
    }
}
var peo=new Black("gcy");

console.log(peo.getName()+' says '+peo.speak());
```
  其实有原型继承方式写法很多,我认为还是理解原型链机制比较重要,关键就是理解prototype和__proto__.
##多态
多态其实就是把做的内容和谁去做分开。在java中我们可以通过向上转型,也就是面向接口编程。因为js是动态语言,多态性本身就有。
下面这个例子就说明了,一个动物能否实现叫声,只取决于makeSound,不针对某种类型的对象。

```
 /**
     * 多态的实现案例
     * @param animal
     */
    var makeSound=function (animal) {
        animal.sound();
    }
    var Duck=function () {
    }
    var Dog=function () {
    }
    Duck.prototype.sound=function () {
        console.log("嘎嘎嘎")
    }
    Dog.prototype.sound=function () {
        console.log("旺旺旺")
    }
    makeSound(new Duck());
    makeSound(new Dog());

```


