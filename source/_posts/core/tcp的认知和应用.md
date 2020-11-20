---
title: tcp的认知和应用
date: 2018-02-06
tags: [node,tcp]
author: gcyStar
---
# 什么是tcp
## tcp定义
tcp是网络七层模型当中传输层的协议,由IETF的RFC 793定义,是面向连接的、可靠的、基于字节流的通信协议。而传输层位于七层模型的中间,下面是网络层,上面的话有应用层,承上启下,地位还是很重要的。在传输层中还有一种协议udp(无连接、不保证可靠性)。相比来说tcp有以下特点
> * 传输可靠,数据丢失有重传机制
> * 数据分段打包传输,对每个数据编号,控制顺序
> * 流量控制,避免拥塞,因为TCP连接双方有固定大小缓冲空间。TCP的接收端只允许另一端发送接收端缓冲区所能接纳的数据。


## tcp数据包格式
![tcp-data-jsdt](https://img.wuage.com/15179837313711tcp-data.png)
针对上面图中的结构划分,下面进行下解释

**source port 和 destination port** 都是16位,计算机端口识别访问那个服务, 其中source port 是随机的,而destination port决定接受方由那个程序来接受并且因为是16位,所以说程序的最大端口号65535

**sequence Number ** 是发送数据包中的第一个字节的序列号,假设当前的序列号为 s，发送数据长度为 l，则下次发送数据时的序列号为 s + l。在建立连接时通常由计算机生成一个随机数作为序列号的初始值

**Acknowledgment Number **它等于下一次应该接收到的数据的序列号。可以认为这个位置以前所有的数据都已被正常接收。
**header length** TCP 首部的长度，单位为 4 字节。如果没有可选字段，那么这里的值就是 5。表示 TCP 首部的长度为 20 字节(除掉data和option,刚好20字节)。
**reserved**
> * URG表示Urgent Pointer字段有意义
> * ACK表示Acknowledgment Number字段有意义
> * PSH表示有 DATA数据传输
> * RST表示复位TCP连接
> * SYN表示SYN报文（在建立TCP连接的时候使用）
> * FIN表示没有数据需要发送了（在关闭TCP连接的时候使用）


**Windows** 表示接收缓冲区的空闲空间，16位，用来告诉TCP连接另一端自己能够接收的最大数据长度,流量控制的机制就基于此。
**Checksum**  差错控制，TCP校验和的计算包括TCP首部、数据和其它填充字节。在发送TCP数据段时，由发送端计算校验和，当到达目的地时又进行一次检验和计算。如果两次校验和一致说明数据是正确的，否则将认为数据被破坏，接收端将丢弃该数据。
 **Urgent** 是紧急指针，16位，只有URG标志位被设置时该字段才有意义，表示紧急数据相对序列号（Sequence Number字段的值）的偏移。
 
## tcp三次握手
![tcp](https://img.wuage.com/151798868021999curl-jsdt.png)
**说明** 上面是一个我写的测试demo,是一个普通的http请求,下面我们抓包分析一下tcp三次握手的细节

![tcp1](https://img.wuage.com/151798857977684tcp-syn1.png)
**说明** 第一次握手client通过发送一个SYN标识位和Sequence Numbers(同步序列号)的数据段给server请求连接，通过该数据段告诉server希望建立连接
![tcp2](https://img.wuage.com/151799782736420tcp2-new.png)
**说明** 第二次握手是server用一个确认应答ACK标志位和Acknowledgment Number告诉client收到了数据段，同时自己也发送一个syn包(syn标志位和Sequence Numbers)。
![tcp3](https://img.wuage.com/151798857977684tcp-syn3.png)
**说明** 第三次客户端收到服务器的SYN+ACK包，向服务器发送确认包ACK，此包发送完毕，客户端和服务器进入ESTABLISHED（TCP连接成功）状态，完成三次握手。
**说明** 三次握手完毕,链接通道建立成功,下面开始正式数据传输
![tcp-data](https://img.wuage.com/151798857977684tcp-data2.png)
针对上面整个过程下面再来一幅图详细总结说明下
![tcp-three](https://img.wuage.com/151807044806724tcp-threepng.png)
>* 客户端通过向服务器端发送一个SYN来表明希望建立连接，作为三次握手的一部分。客户端把这段连接的序号设定为随机数c。
>* 服务器端应当为一个合法的SYN回送一个SYN/ACK。ACK的确认码应为c+1，SYN/ACK包本身又有一个随机产生的序号s。
>* 最后，客户端再发送一个ACK。当服务端收到这个ACK的时候，就完成了三次握手，并进入了连接创建状态。此时包的序号被设定为收到的确认号c+1，而响应号则为s+1。

## tcp四次断开
![tcp-close-jsdt](https://img.wuage.com/151799115180471tcp-close.png)
下面描述下整个过程,过程和三次握手机制一样,也是用一些标志位来实现断开的过程,详细步骤过程不在图形化,因为需要四幅步骤图,太占篇幅了,下面根据上面的流程图大致解释说明一下。
>* client发送FIN标志位,表示出要断开连接的请求的含义
>* server进行响应，发送ack标识,表示确认收到断开连接请求
>* server 发送FIN标志位,提出反方向的关闭要求
>* client 发送ack标志位,标识确认收到的主机B的关闭连接请求

# tcp应用场景
适合对传输数据要求比较可靠的地方,例如常见的ftp,http,https,smtp等等这些应用层的协议都是基于tcp的封装;还有实战中常见套接字编程,可以用来开发浏览器通信模块,以及p2p的下载工具等等,这些底层都是都是基于tcp协议。

# 如何使用tcp
协议只是一个规范,实现形式有多种,具体而言在node当中使用的话如下形式
```
let net = require('net');
let path = require('path');
let ws = require('fs').createWriteStream(path.join(__dirname, 'msg.txt'));
let server = net.createServer(function (socket) {
    socket.pipe(ws, { end: false });

});
server.listen(8080);
创建一个tcp服务器监听8080端口,并将接受的数据存储到msg.txt文件中,上面的socket是一个双工流,既可写又可以读,下面是测试的输出结果
```
![tcp-inaction](https://img.wuage.com/151799300663248tcp-inaction.png)
**说明** 上面只是一个简单的实例,更多用法,可以参考node当中net模块
# tcp优化
## 内核参数优化
tcp内核参数很多,网上也有很多介绍,下面我拿其中一个作为实例,分析下。
结合ab命令来压测机器优化网络


首先必须明确的是优化一定要看场景,有一定实际需求场景的情况下在进行优化,优化前充分了解相关命令参数的含义,并且最好有相关的实验数据作支撑,不能为了优化而优化,这种内核级别的优化操作如果优化不当,后果很严重,我上面只是为了学习而做的一些测试demo。
## 提升http性能
# http服务器实现


参考链接
https://zh.wikipedia.org/wiki/%E4%BC%A0%E8%BE%93%E6%8E%A7%E5%88%B6%E5%8D%8F%E8%AE%AE
http://nodejs.cn/api/
http://www.freebuf.com/column/133862.html
