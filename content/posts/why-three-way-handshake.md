---
title: "Why Three Way Handshake"
date: 2021-07-17T00:49:58+08:00
draft: false
---
# 为什么TCP握手要3次
从两个面试题谈起

* ping 目标主机100ms, 不考虑网络联通问题，理想情况(不考虑丢包）下http多少ms，https多少ms(不考虑非对称加密解密耗时)
* TCP三次握手为啥是3次，为什么不是2或者4次，TCP握手到底在干嘛？

第一个是朋友谈起的一个面试题，第二个是我问候选人的面试题。

先来说第二个问题，候选人给了我意料中的回复：

> 打个比方比如打电话：
>
> 1. 你听得到吗？
> 2. 我能听到，你听得到？
> 3. 我也能听到
>
> 所以需要三次

这回答让人感概颇深, 国内互联网充斥着以讹传讹的博文，却没多少人想着去翻翻RFC。
就连教材往往都是语焉不详。

### 连接是什么

所以为什么建立连接需要三次？所以TCP握手究竟干了什么？连接是什么？：

> The reliability and flow control mechanisms described above require that TCPs initialize and maintain certain status information for each data stream. The combination of this information, including sockets, sequence numbers, and window sizes, is called a connection.

[rfc793](https://datatracker.ietf.org/doc/html/rfc793) 明确定义了连接，这里看最后一句The combination of this information, including sockets, sequence numbers, is called a connection。也就是说连接是一种包括sockets, 序号, 窗口大小这些信息的机制。

连接清楚了，那么接下来探究为什么建立连接需要三次，握手期间是怎么做到初始化信息，注意这里不会讨论其他，主要讨论为啥是三次。

### 为什么需要三次

#### 旧的重复连接与序号的获取

> The principle reason for the three-way handshake is to prevent old duplicate connection initiations from causing confusion.

同样来自[rfc793](https://datatracker.ietf.org/doc/html/rfc793) ，为了阻止历史的重复连接初始化 。这里不妨发挥下脑补能力。现实生活并不美好，真实的网络是不可靠的，大家都知道互联网传输的可靠性在传输层来保证，他是怎么保证的呢。先来看握手图：

![three-handshake](/img/why-three/handshake-rfc.png)

其实关于第一个面试题中的http的耗时，从这个图里就可以得到结论了。（100/2）* （3 + 1）=200ms，传输层3次握手加一次应用层协议确立。https以此类推即可。

然后发挥想象：

A发起一个连接，但是过了很久都没收到B的响应（超时了，丢包了等），A不知道是什么原因，于是重发，重发再重发直到放弃。

B收到了A的SEQ是100的请求，只是A不知道还在继续发，B收到了后就知道有关A想搞事儿，如果B不想理就丢弃了，A最终会放弃这个没问题。但是如果B想逗一下A于是就理会了，发了个应答包给A。

当然B也不确定A收到没，而且这里还有可能SEQ是99的包绕路比100还慢, 所以这里得分情况讨论：

* SEQ 100先到了，那这个时候再回应就就是重复了，接受方B不知道哪个更早哪个更迟，所以B的响应会将SEQ+1写到ACK字段里也就是ACK=101给A， 同理B得带上自己的序号SEQ=300 。

* SEQ 99到了。接受方B不知道哪个更早哪个更迟，所以B的响应会将SEQ+1写到ACK字段里也就是ACK=100给A， 同理B得带上自己的序号SEQ=301。

B发送的应答（ACK=101）的可能会发送很多次，但是只要一次到了A，A就认为B同意建立连接搞事儿了，因为对A来说，有去有回。这时候A会给B发送应答应答的应答，B在等这个，应答，也就是A-》B， SEQ=101，ACK=301的应答。

B发送的应答（ACK=100）的可能会发送很多次，但是只要一次到了A，A就发现我要的ack是101， 那么直接发CTL =RST SEQ=100中止了B的SEQ=99的那次连接。

**所以需要第三次来保证旧的连接会被发送方关了，且通信双方都有来有回双方都能互通。**

到此，我们即握手了三次，又能阻止网络丢包导致的旧握手请求问题，还同步了双方各自发的包的序号，关于序号的获取rfc有如下描述 ：

> A three way handshake is necessary because sequence numbers are not tied to a global clock in the network, and TCPs may have different mechanisms for picking the ISN’s. The receiver of the first SYN has no way of knowing whether the segment was an old delayed one or not, unless it remembers the last sequence number used on the connection (which is not always possible), and so it must ask the sender to verify this SYN. The three way handshake and the advantages of a clock-driven scheme are discussed in [3].

那么还剩下最后一个问题，**次数**。

#### 次数

上面简单说了3次能保证双方都是有来有回，但是其实只是取了TCP Header优化后的，本质上其实更多次按照这逻辑来做也是行的，比如B对A的响应ACK和SEQ完全可以分两次发，只是头部的设计能让通信更优化一点，B的ACK和SEQ可以一次发送。所以使用最少次数即：**3次**。

![tcp-header-format](/img/why-three/tcp-header-format.png)

这里第二个面试题也回答了。
