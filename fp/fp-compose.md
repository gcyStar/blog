---
title: compose
date: 2017-06-04
tags: [javascript,fp]
author: gcyStar
---
## 引言
> 在fp编程中,compose是一个重要应用场景,下面谈论一些个人理解。

```
const R=require('ramda');
const data=[{
    name:'gcy',
    sex:'male'
},{
    name:'ycg',
    sex:'female'
}]
const getName=R.compose(
    R.map(d => d.name),
    R.filter(d => d.sex=='male')
);

console.log(getName(data));
```

**说明**   数据和操作解耦,抽象化的功能函数组件更易复用和维护。

上面filter做到了只接受谓词函数作为唯一约束,自由变量是如何被过滤,这涉及到curry,如下所示。

```
var _ = require('lodash');

const Rfilter=function (predicate) {
    return function (data) {
        return _.filter(data,predicate);
    }
}
console.log(Rfilter( d => d.name=='gcy')(data));

```
**说明**  curry一言以蔽之就是为每个参数返回一个函数。好处是可以依赖透明,无观察副作用,哈哈,其实所有fp纯函数都有这个特性,其次简化函数使用难度(相对来说,为curry而curry就得不偿失了)和方便compose。

```
const {comp,pipeline,partial,inc,filter,sort}=require('mori');
const todos=[{
    name:'g1',
    age:'20'
},{
    name:'g2',
    age:'21'
},{
    name:'g1',
    age:'22'
}];
const sortByname=partial(sort,(x,y) => x.name<y.name);
const filterByAge= partial(filter,x => x.age>=21);
console.log(pipeline(todos,filterByAge,sortByname))
```

**说明**  无论普通的compose,partial,还是curry可读性都不好,上面既具有组合性,同时管道式的阅读方式看起来顺畅,而不是自内向外,自右向左的函数式组合读法。

# 总结
通过已有的函数进行组合,最大程度的复用已有函数,需要提前构思好抽象单元函数。最近挺忙的,发现一后端写的代码A(){B()|C()},B(){C()},C(){D()}然后依赖一些全局变量,有极大的观察副作用,同时没有模块化,零件无组织的散落个各个文件中,典型的反例,感触颇深（＞﹏＜)。





















