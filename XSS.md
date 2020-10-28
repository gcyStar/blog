---
title: XSS
date: 2017-03-26
tags: [XSS,安全]
author: gcyStar
---
# 引言
> 平时工作中常接触到XSS、CSRF、SQL注入等等这些安全领域的知识。接下来准备重温整理一些概念,以便加深自己的理解,通过结合具体的实例(基于node)。

# XSS概念
> 译为跨站脚本攻击,具体是指攻击者在Web页面里插入恶意Script脚本，当用户浏览该网页时，Script代码会被执行，从而进行恶意攻击。

# XSS分类
## 反射型
 反射型又被称作非持久型XSS, 是指把用户输入的数据 '反射' 给浏览器,通过诱使用户点击恶意链接的方式。
 > href : xss?un='\<script>alert('hw gcy')<\/script>'
 ```
 <%- un %>
 ```
 ![xss1-jsdt](https://img.wuage.com/149224166545260xss1.png)
 
## 存储型
 存储型XSS又被称为持久型(存储型)XSS,相对上一种较危险。例如下面示例在文章评论中输入非法内容
 ```
 <input  type="text" name="content" value="<script>alert('hw gcy')</script>"/>
 ```
## DOM型
 输入链接同反射型一样,效果看起来也一样,之所以单独划分作为一个分类,是因为其形成原因特别,且是通过修改页面DOM节点形成的XSS。
 ```

 <script>
   let href=document.URL;
   let index=document.URL.indexOf('un=')+4;
   document.write(decode(href.substring(index)));
 </script>
 ```
 
# XSS的危害
 * cookie劫持,冒充登录
 * 改变网页内容
 * URL跳转,恶意导航(遇到多次)
 * XSS Worm
 * and so on  
 
针对以上危害,除第四种外,其它见名知意。在次特此剖析下XSS Worm.

## XSS Worm

译为XSS蠕虫,一般发生在用户交互行为的页面中,比如留言,站内信等等。影响力和破坏力巨大,因为其传染力极强。   
一般其本质基于存储型,且攻击者需要熟悉攻击目标网站的业务,功能接口才能实现。

历史上的XSS Worm情景,根据提供的关键词search , 结合定义看一遍就能理解

* 2005年 MySpace.com(社区网站)被攻击 
* 2007年 百度空间

# XSS预防
* 关键cookie字段设置httpOnly
* 输入检查,特殊字符 < > / &等,对其进行转义后存储