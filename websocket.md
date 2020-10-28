---
title: websocket
date: 2018-05-27
tags: [websocket,js]
author: gcyStar
---
## 什么是websocket
Websocket是一个持久化的协议，改协议定义了一个 API 用以在browser和server建立一个 socket 连接。WebSocket是建立在http的基础上，复用了HTTP的握手环节，握手成功后经过协商在走Websocket协议格式的数据。

![Websocket-spec](https://img.wuage.com/152741006812351websocket-spec.jpeg)

## websocket应用场景
websocket可以用来实现弹幕，协同编辑，在线社交聊天等功能。因为websocket的兼容性问题,如下所示，只支持ie11+
![websocket-jian](https://img.wuage.com/15274104996345websocket-jian.png)
所以还有其它几种方案作为兼容性问题的补充，常见的比如说轮训方案（浏览器定时重复向服务器发送http请求，服务器相应，客户端得到数据后，将其显示出来，不断的重复这一过程 ）、长轮询（客户端打开了一个到服务端的 HTTP 连接直到返回响应数据）。

## websocket使用

```
//server
let WSServer = require('ws').Server;
let wsServer = new WSServer({ port: 8888 });
wsServer.on('connection', function (socket) {
    console.log('客户端已经连接');
    //监听客户端发过来的ith
    socket.on('message', function (message) {
        console.log(message);
        socket.send(' from  server:' + 'hello');
    });
});
//client
 let socket = new WebSocket('ws://localhost:8888');
    socket.onopen = function () {
        socket.send('from client hello');
    }
    socket.onmessage = function (event) {
        console.log(event.data);
    }
```
输出结果
![websocket-data](https://img.wuage.com/152752097917975websocketdata.png)
**说明**  上面是一个简单的示例，建立连接并发送数据。
在真实项目应用中，会借助于第三方成熟的库，例如Socket.IO（包括了brower和server（node）的实现代码），支持跨平台，根据浏览器兼容自动从WebSocket、AJAX长轮询、Iframe流等等各种方式中选择最佳的方式来实现网络实时应用，简单使用举例如下所示。
```
//server
var express = require('express');
var path = require('path');
var app = express();

app.get('/', function (req, res) {
    res.sendFile(path.resolve('index.html'));
});

var server = require('http').createServer(app);
var io = require('socket.io')(server);

io.on('connection', function (socket) {
    console.log('客户端已经连接');
    socket.on('message', function (msg) {
        console.log(msg);
        socket.send('sever:' + msg);
    });
});
server.listen(80);
//client
<script src="/socket.io/socket.io.js"></script>
var socket = io.connect('/');
socket.on('connect',function(){
   //客户端连接成功后发送消息'welcome'
   socket.send('welcome');
});
//客户端收到服务器发过来的消息后触发
socket.on('message',function(message){
   console.log(message);
});
```
**说明**  除了上面演示中的功能外，Socket.IO还支持命名空间，房间的划分，也支持指定范围的广播。

## websocket实现





















