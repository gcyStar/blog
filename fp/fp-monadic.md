---
title: monadic
date: 2017-06-11
tags: [javascript,fp]
author: gcyStar
---

## functor
> functor是可以被map over的容器类型。

关于这句话比较难理解的是,什么是map over?

> (a -> b) -> f a -> f b 意思就是说一个a到b的映射,在接受a的functor之后,返回一个b的functor，下面是map over应用示例，可以结合着理解这句话。

```
const f= x => x.split("").reverse().join("");
const rs='olleh';
const b=f(rs);  
console.log(b); //hello


function JSDT(val) {
    this._val=val;
}
JSDT.prototype.map=function (f) {
    return new JSDT(f(this._val));
};
JSDT.of=function (val) {
    return new JSDT(val);
}
const fa=JSDT.of(rs);
const fb=fa.map(f);  
console.log(fb);   //JSDT {_val: "hello"}

```

**说明**  实现了map的container是Functor的实例(JSDT),map是将函数应用到container内部的方法。

functor出自范畴论(Category Theory),数学的一个分支,满足一些定律。
* fa.map(x => x) === fa;
* fa.map(x => f(g(x))) === fa.map(g).map(f);

**说明**  关于这两个定律下面有一个很好的理解例子

```
var a=[1,2,3].map(x => x);
console.log(a);  //[ 1, 2, 3 ]

const f= x => x+2;
const g= x => x*2;
var b1=[1,2,3].map(x => f(g(x)));
var b2=[1,2,3].map(g).map(f);
console.log(b1.toString()===b2.toString()); //true
```

## Applicative Functor

> Applicative本质是Functor的一种,可以将一个含有函数的容器应用到另一个容器中的值上。

```
JSDT.prototype.ap=function (container) {
    return container.map(this._val);
}
const f=JSDT.of(x => x+2);
let fy=f.ap(JSDT.of(3));
console.log(fy);  //JSDT { _val: 5 }
```

**说明**  Applicative的特性就是多了一层ap,示例中JSDT实例f中,this._val是抽象函数单元,可以应用到另一个匿名实例上。

applicative满足的定律
* a.of(x => x).ap(val) === val;
* a.of(f).op(a.of(x)) === a.of(f(x));
* u.ap(a.of(y)) === a.of(f => f(y)).ap(u);

## Monad

>Monad是一种特殊的Functor,可以Flat(铺平)map的结果。

```
function Nothing() {
}
Nothing.prototype.map=function () {
    return this;
}
const nothing=new Nothing();
JSDT.prototype.flat=function () {
    return this._val;
}
JSDT.prototype.flatMap=function (f) {
    return this.map(f).flat();
}
let fm=JSDT.of({val:0}).
    flatMap(x => {
    if(x) return JSDT.of(x.val);
    else return nothing;
})
    .map(x => x+1)
    .map( x => 2/x);
console.log(fm);  //JSDT { _val: 2 }
```

**说明**  如果用普通map,第一层异常时,会连续执行,通过monad的方式可以在异常发生时,无论怎么map最后还是它自己,从而可以在异常发生时避免不必要的错误执行。

## 总结
monadic编程重要且普遍,典型的一种应用就是promise、还可以配合其它reactive编程框架使用,理解原理能帮助我们更好的使用。在深入过程中发现各种稀奇古怪的名词,但耐心去分析和思考,也不难理解。



















