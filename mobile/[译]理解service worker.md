---
title: 理解service worker
date: 2017-08-21
tags: [js,pwa,译文,service-worker]
author: gcyStar
---
在网络早期,很难想象在用户离线的时候一个网页可以访问。你总需要一直在线。
![Connected!](http://p0.qhimg.com/t01a8e49479d8e9864f.jpg)

**链接网络吧,伙伴们都在这,不要离开了。**

但随着移动互联网的出现,以及这个世界上越来越多的地区连接上网络,参差不齐的网络连接速度在现代web用户中越来越普遍。


因此,对于与网站来说,研究如何使用户不受到网络链接限制能力,使他们在离线的时候也能访问变得越来越有价值。

AppCache最初被引入作为html5的标准一部分,目的是作为web离线应用的解决方案之一。它包括HTML和javascript,由一个可以写声明性语语言的,缓存manifest配置文件进行管理。

Service workers提供了一个不会过时的,程序命令式的,针对离线web问题的解决方案,取代了声明式的AppCache。

Service workers以持久的方式执行代码,后台进程的方式运行在浏览器中(意思是说不会影响js线程的执行),代码是事件驱动的,意味着在Service workers中事件被触发是因为外在的行为。

文章的剩余部分主要讲述了Service workers当中的事件,作为分析开始,你首先需要知道,在你的web app中需要注册service worker。

## 注册
下面的代码介绍了在客户端浏览器中怎样注册service worker。下面register的执行方式放在你的web app中的某处即可。

```
if (navigator.serviceWorker) {  
  navigator.serviceWorker.register('/sw.js')
    .then(registration => {
      console.log('congrats. scope is: ', registration.scope);
    })
    .catch(error => {
      console.log('sorry', error);
    });
}
```

上面代码作用是告诉浏览器哪里去找Service Worke的实现。浏览器找到sw.js文件,将其保存为一个 Service Worker放在可以访问到的域中。在sw.js中包含了怎么处理Service Worker生命周期中的一些事件。

![service worker!](http://p0.qhimg.com/t014714b593d5e5563b.png)

**上面图示当中显示一个已经注册的service worker 在Chrome DevTools上。**

它设置了你的ServiceWorker的作用范围。文件名/sw.js意味着SW的范围是项目的根路径的URL(或http://localhost:3000/)。这意味着在根路径下的任何请求将是可见的,对于SW被触发的事件来说。
而一个文件名/js/sw.js表明将只捕获在 http://localhost:3000/js下的请求。

作为一种选择,你也可以通过传递参数的方式给register方法, 来显式的设置SW的作用域范围

```
navigator.serviceWorker.register('/sw.js', { scope: '/js' }).
```

## Event Handlers

既然你的Service Worker已经注册,是时候去实现事件处理在Service Worker的各个生命周期阶段。

### Install Event

install事件在第一次成功注册Service Worker时触发,或者在Service Worke文件(/sw.js)更新(浏览器会自动检测到更新)的时候。

install事件是有用的逻辑,在你想执行的Service Worker初始化期间,它是一次性的操作。一个常见的用例是在安装步骤时加载缓存的文件。

这里是一个例子,一个安装事件处理程序,它将数据添加到缓存中

```
const CACHE_NAME = 'cache-v1';  
const urlsToCache = [  
  '/',
  '/js/main.js',
  '/css/style.css',
  '/img/bob-ross.jpg',
];

self.addEventListener('install', event => {  
  caches.open(CACHE_NAME)
    .then(cache => {
      return cache.addAll(urlsToCache);
    });
});
```

urlsToCache包含了一系列我们想添加到缓存中的url。

caches是一个全局的CacheStorage对象,通过这个对象可以管理你的缓存在浏览器中。
我们调用open方法以获得确切的我们想使用的缓存对象。

cache.addAll接受URLs集合,然后遍历发送一系列的request,然后存储response在缓存中。它使用request body 作为每个缓存的key。[addAll](https://developer.mozilla.org/en-US/docs/Web/API/Cache/addAll)的参考文档。

![add All!](http://p0.qhimg.com/t01ecefe54d9dd7b768.png)


**Cached data 在Chrome DevTools中**

### Fetch event
Fetch事件被触发在每次web页面发出请求的时候。当它被触发时,您的service worker有能力“拦截”请求,并决定如何返回——不管是缓存数据,抑或是一个实际的网络请求响应。

下面的例子说明了cache-first策略:任何缓存的数据,在相匹配的请求中将被首先发送,而没有一个网络请求。只有在没有缓存数据时才会有网络请求。

```
self.addEventListener('fetch', event => {  
  const { request } = event;
  const findResponsePromise = caches.open(CACHE_NAME)
    .then(cache => cache.match(request))
    .then(response => {
      if (response) {
        return response;
      }

      return fetch(request);
    });

  event.respondWith(findResponsePromise);
});
```
request包含请求主体,其被包含在FetchEvent对象中。它是用来在缓存中查找匹配响应。

cache.match将试图找到一个缓存来响应相匹配的指定请求。如果匹配不到,promise将解析为undefined。我们检查这种情况,调用执行fetch方法在这种情况下,这会导致产生一个网络请求并且返回一个promise。

FetchEvent对象上有一个特殊的方法event.respondWith,对于浏览器的请求我们可以使用这个方法返回一个相应。这个函数接受一个promise参数,返回一个response。

### 缓存策略

fetch事件非常重要,因为只有通过这个方法你才能定义缓存策略。也就是说,你可以决定何时使用缓存数据,何时使用网络数据。

Service Workers的美妙之处在于他是一个拦截requests的初级api,允许你返回自己定制的response。这给了我们是使用缓存还是网络数据的自由。这里有几个基本的缓存策略,你可以使用它们让你的web app变得更好。

mdn是一个非常方便的文档资源,他上面介绍了计重不同的缓存策略。这里也有一个Jake Archibald写的 [The Offline Cookbook](https://developers.google.com/web/fundamentals/instant-and-offline/offline-cookbook),
,里面介绍了一些缓存策略,与mdn上相比有相同的,也有不同的。

在上面的示例中,我们演示了一个基本的cache-first策略。下面是一个我找到的适用于自己的项目的例子:缓存和更新策略。该方法将首先缓存响应,但随后在后台网络请求。来自于这个后台请求的response被用于更新缓存中的值,以便下次更新响应提供访问。

```
self.addEventListener('fetch', event => {  
  const { request } = event;

  event.respondWith(caches.open(CACHE_NAME)
    .then(cache => cache.match(request))
    .then(matching => matching || fetch(request)));

  event.waitUntil(caches.open(CACHE_NAME)
    .then(cache => fetch(request)
      .then(response => cache.put(request, response))));
});
```
event.respondWith 被用来提供响应针对请求。通过它,我们开启一个缓存,并从缓存中寻找匹配的response,如果缓存中匹配不到,就通过网络来相应。

随后,我们调用event.waitUntil,解析Promise在Service Worker结束之前。在这里我们发送一个网络请求,然后缓存响应。一旦这个异步操作完成后,waitUntil操作将终止。

### Activate Event

activate是一个记录事件,在你更新service worker文件,执行清理或者维护先前版本的service worker时,这个事件非常重要。
当你更新你的service worker文件(/sw.js)时,浏览器能探测到这种改变并显示在Chrome DevTools:

![Activate!](http://p0.qhimg.com/t016b73616670d23995.png)
**你的service worker 正等在被激活**

当web页面关闭,再次重启打开,浏览器将用新的service worker取代旧的service worker,并且依次触发activate,install事件。如果你需要清理缓存或执行维护service worker的旧版本,激活事件是实现需求的最佳时机。

### Sync event
sync事件可以让你延迟网络任务,直到用户连接。它实现的功能通常被称为后台同步。这是有用的,可以确保任何网络任务,在用户开始在离线模式,然后又恢复网络链接模式时顺利执行。

这里有一个例子演示后台任务,你只需要在你的js代码中注册一个同步事件,在其中做一些相应的处理在你的service worker中:

```
// app.js
navigator.serviceWorker.ready  
  .then(registration => {
    document.getElementById('submit').addEventListener('click', () => {
      registration.sync.register('submit').then(() => {
        console.log('sync registered!');
      });
    });
  });

```
在这里我们声明了一个按钮上 的click事件通过调用sync.register在ServiceWorkerRegistration 对象上。

基本上,你想要确保在任何时候网络连接上时实现一些操作,需要注册一个类似上面的同步事件。

你想要实现的操作可能类似发送一个评论,获取用户数据等等都将被定义在Service Worker的事件处理器中。

```
// sw.js
self.addEventListener('sync', event => {  
  if (event.tag === 'submit') {
    console.log('sync!');
  }
});
```

在这里我们监听一个sync事件,检查SyncEvent对象上的tag,看是否匹配我们submit。

如果在sync已经注册的情况下,声明sync多次,此时sync事件只会执行一次。

对于这个例子,如果用户离线,并点击按钮七次,当恢复网络链接,所有同步注册执行优先级将会提高,并执行一次。

在那种情况下,你可能想针对每种同步点击事件做不同的标识处理。

### 同步事件什么时候被触发?

如果用户在线,同步事件无论是什么任务,都会立即执行,没有任何延迟。

如果用户离线,同步事件会尽可能快的被触发在恢复网络连接的时候。

你可以试试,验证一下上面的结论在你的chrome中,再试之前确保断开网路连接,检查Chrome DevTools中的network选项。

想要了解更多信息,参考[这篇文档](https://github.com/WICG/BackgroundSync/blob/master/explainer.md),抑或是
[background syncs](https://developers.google.com/web/updates/2015/12/background-sync). 同步事件很多浏览器不支持(仅仅chrome中有提到),一定会发生变化,所以持续关注。

### 通知推送

通知推送是service worker中一个能够使用的特性,与此先关的是浏览器上的一个[push api](https://developer.mozilla.org/en-US/docs/Web/API/Push_API)

当谈到网络推送通知,在工作中涉及到两种技术:通知和推送消息。

### 通知
通知在service worker中是一个非常简单容易实现的特性

```
// app.js
// ask for permission
Notification.requestPermission(permission => {  
  console.log('permission:', permission);
});

// display notification
function displayNotification() {  
  if (Notification.permission == 'granted') {
    navigator.serviceWorker.getRegistration()
      .then(registration => {
        registration.showNotification('this is a notification!');
      });
  }
}
```

```
// sw.js
self.addEventListener('notificationclick', event => {  
  // notification click event
});

self.addEventListener('notificationclose', event => {  
  // notification closed event
});
```
你首先需要向用户请求获取权限在web页面中。从那以后,你可以切换通知,处理某些事件,比如当用户关闭一个通知。


### 消息推送

消息推送牵涉到利用浏览器提供的push api,以及后端的实现。如何实现push api可以写成一篇独立的文章。但是基本的逻辑如下:

![ push](http://s3.qhres.com/static/9aaae42e4bb9a833.svg)

它是一个复杂和轻微难懂的流程,具体阐述不在本文范围内,如果你想了解更多,推荐这篇文章 [introduction to push notifications](https://developers.google.com/web/ilt/pwa/introduction-to-push-notifications)

## 题外话

后面还有一小节阐述Ember和service worker的结合应用,这个我并不感兴趣,就略过了,如果Ember换成vue或react就好啦。接下我打算写一篇实战例子。


原文链接

http://blog.88mph.io/2017/07/28/understanding-service-workers/


















