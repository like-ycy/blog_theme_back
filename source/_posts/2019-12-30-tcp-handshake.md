---
title:      " TCP 协议，握手挥手不是你想的那么简单 "
date:       2019-12-30
banner_img: "https://image.my-blog.wang/header/header.jpg"
tags: [ TCP ]


---

## TCP 报文段结构

一谈到 TCP 协议，大家最先想到的词就是「**面向连接**」和「**可靠**」。没错，TCP 协议的设计就是为了能够在客户端和服务器之间建立起一个可靠连接。

在讲连接过程之前，我们先来看看 TCP 的报文段结构，通过这个结构，我们可以知道 TCP 能够提供什么信息：

![TCP报文结构](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-30/tcp-message.png)

这里有几点是需要注意的：

- TCP 协议需要一个**四元组**（源IP，源端口，目的IP，目的端口）来确定连接，这要和 UDP 协议区分开。多说一句，IP 地址位于 IP 报文段，TCP 报文段是不含 IP 地址信息的。
- **基本 TCP 头部**的长度是 20 字节，但是由于「**选项**」的长度是不确定的，所以需要「**首部长度**」字段明确给出头部长度。这里要注意的是，首部长度字段的单位是 32bit，也就是 4 字节，所以该字段的最小值是 5。
- 标橙色的字段（**确认序号，接收窗口大小，ECE，ACK**）用于「回复」对方，举个例子，服务器收到对方的数据包后，不单独发一个数据包来回应，而是稍微等一下，把确认信息附在**下一个**发往**客户端**的数据帧上，也就是**捎带**技术。
- 窗口大小是一个 16 位无符号数，也就是说窗口被限制在了 65535 字节，也就限制了 TCP 的吞吐量性能，这对一些高速以及高延迟的网络不太友好（可以想想为什么）。所幸 TCP 额外提供了**窗口缩放**（Window Scale）选项，允许对这个值进行缩放。

下面是 8 个标志位的含义，有的协议比较旧，可能没有前两个标志位：

![8个标志位的含义](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-30/8flag.png)

标志位虽然很多，但是如果放到具体场景里来看的话，就很容易理解他们的作用了。

## 三次握手

三次握手就是为了在客户端和服务器间建立连接，这个过程并不复杂，但里面有很多细节需要注意。

![三次握手](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-30/three-handshake.png)

这张图就是握手的过程，可以看到客户端与服务器之间一共传递了三次消息，这三次握手其实就是两台机器之间互相确认状态，我们来一点一点看。

**第一次握手**

首先是**客户端发起连接**，第一个数据包将 SYN 置位（也就是 SYN = 1），表明这个数据包是 SYN 报文段（也被称为**段 1**）。这一次发送的目的是告诉服务器，自己的**初始序列号**是 `client_isn` ，还有一个隐含的信息在图里没有表现出来，那就是告知服务端自己想连接的**端口号**。除了这些，客户端还会发送一些**选项**，不过这跟三次握手没多大关系，暂且按下不表。

段 1 里最需要注意的就是这个`client_isn` ，也就是初始序列号。「RFC0793^1」指出:

> When new connections are created, an initial sequence number (ISN) generator is employed which selects a new 32 bit ISN. The generator is bound to a (possibly fictitious) 32 bit clock whose low order bit is incremented roughly every 4 microseconds.  Thus, the ISN cycles approximately every 4.55 hours.

翻译过来就是，初始序列号是一个 32 位的（虚拟）计数器，而且这个计数器每 4 微秒加 1，也就是说，ISN 的值**每 4.55 小时循环一次**。这个举措是为了**防止序列号重叠**。

但即使这样还是会有安全隐患——因为初始 ISN 仍然是可预测的，恶意程序可能会分析 ISN ，然后根据先前使用的 ISN **预测**后续 TCP 连接的 ISN，然后进行攻击，一个著名的例子就是「The Mitnick attack^2」 。这里摘一段原文：

> Mitnick sent SYN request to X-Terminal and received SYN/ACK response.  Then he sent RESET response to keep the X-Terminal from being filled up. He repeated this for twenty times. He found there is a pattern between  two successive TCP sequence numbers. It turned out that the numbers were not random at all. The latter number was greater than the previous one  by 128000.

所以为了让初始序列号**更难预测**，现代系统常常使用**半随机**的方法选择初始序列号，详细的方法就不在这里展开了。

**第二次握手**

当服务器接收到客户端的连接请求后，就会向客户端发送 **ACK** 表示自己收到了连接请求，而且，服务器还得**把自己的初始序列号告诉客户端**，这其实是两个步骤，但是发送**一个数据包**就可以完成，用的就是前面说的**捎带**技术。图里的 `ACK = client_isn + 1` 是指**确认号字段**的值，要注意和 **ACK 标志位**区分开。

ACK 字段其实也有不少需要注意的点，不过这个跟滑动窗口一块讲比较直观，这里就先不提了。

这里重点强调一下，当一个 SYN 报文段到达的时候，**服务器会检查处于 SYN_RCVD 状态的连接数目是否超过了 `tcp_max_syn_backlog` 这个参数，如果超过了，服务器就会拒绝连接**。当然，这个也会被黑客所利用，「SYN Flood」就是个很好的例子。因为服务器在回复 SYN-ACK 后，会等待客户端的 ACK ，如果一定时间内没有收到，认为是丢包了，就重发 SYN-ACK，重复几次后才会断开这个连接，linux 可能要一分钟才会断开，所以攻击者如果制造一大批 SYN 请求而不回复，服务器的 SYN 队列很快就被耗尽，这一段时间里，正常的连接也会得不到响应。

服务器的这种状态称为**静默**（muted）。为了抵御 SYN Flood 攻击，服务器可以采用「SYN cookies」，这种思想是，当 SYN 到达时，**并不直接为其分配内存**，而是把这条连接的信息编码并保存在 SYN-ACK 报文段的**序列号**字段，如果客户端回复了，服务器再**从 ACK 字段里解算出 SYN 报文的重要信息**（有点黑魔法的感觉了），验证成功后才为该连接分配内存。这样，服务器不会响应攻击者的请求，正常连接则不会受到影响。

但 SYN cookies 本身有一些限制，并不适合作为默认选项，有兴趣可以自行 Google。

**第三次握手**

这是建立 TCP 连接的最后一步，经过前两次握手，客户端（服务器）已经知道对方的**滑动窗口大小**，**初始序列号**等信息了，这不就完了吗？为什么还要第三次握手？

这是因为服务器虽然把数据包发出去了，但他**还不知道客户端是否收到了这个包**，所以服务器需要等待客户端返回一个 ACK，表明客户端收到了数据，至此，连接完成。

## 四次挥手

有了三次握手的基础，四次挥手就比较容易理解了：

![四次挥手](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-30/four-breakup.png)

四次挥手的过程其实很简单，就是服务器和客户端互相发送 FIN 和 ACK 报文段，告知对方要断开连接。

四次挥手里值得关注的一点就是 **TIME_WAIT** 状态，也就是说主动关闭连接的一方，即使收到了对方的 FIN 报文，也还要等待 2**MSL** 的时间才会彻底关闭这条连接。（这里面的 MSL 指的是**最大段生成期**，指的是报文段**在网络中**被允许存在的最长时间。）可**为什么不直接关闭连接呢**？

一个原因是，**第四次挥手的 ACK 报文段不一定到达了服务器**，为了不让服务器一直处于 LAST_ACK 状态（服务器会重发 FIN，**直到收到 ACK**），客户端还得等一会儿，看看是否需要重发。假如真的丢包了，服务器发送 FIN ，这个 FIN 报文到达客户端时不会超过 2MSL（一来一回最多 2MSL），这时候客户端这边的 TCP 还没关掉，还能重发 ACK。

另一个原因是，**经过 2MSL 之后，网络中与该连接相关的包都已经消失**了，不会干扰新连接。我们来看一个例子：假如客户端向服务器建立了**新的连接**，**旧连接中某些延迟的数据坚持到了新连接建立完毕，而且序列号刚好还在滑动窗口内，服务器就误把它当成新连接的数据包接收**，如下图所示：

![四次挥手](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-30/example.png)

2MSL 机制就避免了这种情况。