---
title:      " TCP 超时与重传 "
date:       2019-12-31
banner_img: "https://image.my-blog.wang/header/header.jpg"
tags: [ Nginx ]


---

我们都知道 TCP 协议具有重传机制，也就是说，如果发送方认为发生了丢包现象，就重发这些数据包。很显然，我们需要一个方法来「**猜测**」是否发生了丢包。最简单的想法就是，接收方每收到一个包，就向发送方返回一个 **ACK**，表示自己已经收到了这段数据，反过来，如果发送方一段时间内没有收到 ACK，就知道**很可能**是数据包丢失了，紧接着就重发该数据包，直到收到 ACK 为止。

你可能注意到我用的是「猜测」，因为即使是超时了，这个数据包也可能并没有丢，它只是绕了一条远路，来的很晚而已。毕竟 TCP 协议是位于**传输层**的协议，不可能明确知道数据链路层和物理层发生了什么。但这并不妨碍我们的超时重传机制，因为接收方会自动忽略重复的包。

超时和重传的概念其实就是这么简单，但内部的细节却是很多，我们最先想到的一个问题就是，**到底多长时间才能算超时呢**？

## 超时是怎么确定的？

一刀切的办法就是，我**直接把超时时间设成一个固定值**，比如说 200ms，但这样肯定是有问题的，我们的电脑和很多服务器都有交互，这些服务器位于天南海北，国内国外，延迟差异巨大，打个比方：

- 我的个人博客搭在国内，延迟大概 30ms，也就是说正常情况下的数据包，60ms 左右就已经能收到 ACK 了，但是按照我们的方法，200ms 才能确定丢包（正常可能是 90 到 120 ms），这**效率实在是有点低**。
- 假设你访问某国外网站，延迟有 130 ms，这就麻烦了，**正常的数据包都可能被认为是超时，导致大量数据包被重发，可以想象，重发的数据包也很容易被误判为超时。。。雪崩效应的感觉**

所以设置固定值是很不可靠的，**我们要根据网络延迟，动态调整超时时间**，延迟越大，超时时间越长。

在这里先引入两个概念：

- RTT（Round Trip Time）：往返时延，也就是**数据包从发出去到收到对应 ACK 的时间。**RTT 是针对连接的，每一个连接都有各自独立的 RTT。
- RTO（Retransmission Time Out）：重传超时，也就是前面说的超时时间。

比较标准的 RTT 定义：

> Measure the elapsed time between sending a data octet with a particular sequence number and **receiving an acknowledgment that covers that sequence number** (segments sent do not have to match segments received). This measured elapsed time is the Round Trip Time (RTT).

### 经典方法

最初的规范「RFC0793」采用了下面的公式来得到平滑的 RTT 估计值（称作 SRTT）：

**SRTT  <-  α·SRTT +（1 - α）·RTT**

RTT 是指最新的样本值，这种估算方法叫做「指数加权移动平均」，名字听起来比较高大上，但整个公式比较好理解，就是利用现存的 SRTT 值和最新测量到的 RTT 值取一个加权平均。

有了 SRTT，就该设置对应的 RTO 的值了，「RFC0793」是这么算的：

**RTO = min(ubound, max(lbound, (SRTT)·β))**

这里面的 **ubound** 是 RTO 的**上边界**，**lbound** 为 RTO 的**下边界**，β 称为**时延离散因子**，推荐值为 1.3 ~ 2.0。这个计算公式就是将  (SRTT)·β 的值作为 RTO，只不过另外**限制了 RTO 的上下限**。

这个计算方法，初看是没有什么问题（至少我是这么感觉的），但是实际应用起来，有两个缺陷：

> There were two known problems with the RTO calculations specified in RFC-793. First, the accurate measurement of RTTs is difficult **when there are retransmissions**. Second, the algorithm to compute the smoothed round-trip time is inadequate [TCP:7], **because it incorrectly assumed that the variance in RTT values would be small and constant**. These problems were solved by **Karn's and Jacobson's algorithm**, respectively.

这段话摘自「RFC1122」，我来解释一下：

- 当**出现数据包重传的情况**下，RTT 的计算就会很“麻烦”，我画了张图来说明这些情况：

![](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-31/rtt.png)

- 图上列了两种情况，这两种情况下计算 RTT 的方法是不一样的（这就是所谓的重传二义性）：

  但是对于客户端来说，它不知道发生了哪种情况，选错情况的结果就是 RTT 偏大/偏小，影响到 RTO 的计算。（最简单粗暴的解决方法就是**忽略有重传的数据包，只计算那些没重传过的**，但这样会导致其他问题。。详见 Karn's algorithm）

- - 情况一：RTT = t2 - t0
  - 情况二：RTT = t2 - t1

- 另一个问题是，**这个算法假设 RTT 波动比较小**，因为这个加权平均的算法又叫**低通滤波器**，对突然的网络波动不敏感。如果网络时延突然增大导致实际 RTT 值远大于估计值，会导致不必要的重传，增大网络负担。（ RTT 增大已经表明网络出现了过载，这些不必要的重传会进一步加重网络负担）。

### 标准方法

说实话这个标准方法比较，，，麻烦，我就直接贴公式了：

**SRTT  <-  (1 - α)·SRTT  + α·RTT**  //跟基本方法一样，**求 SRTT 的加权平均**

**rttvar  <- (1 - h)·rttvar + h·(|RTT - SRTT |)**  //计算 **SRTT 与真实值的差距**（称之为绝对误差|Err|），同样用到**加权平均**

**RTO = SRTT  + 4·rttvar** //估算出来的新的 RTO，rttvar 的系数 4 是调参调出来的

这个算法的整体思想就是结合**平均值**（就是基本方法）和**平均偏差**来进行估算，一波玄学调参得到不错的效果。如果想更深入了解这个算法，参考「RFC6298」。



## 重传——TCP的重要事件

### 基于计时器的重传

这种机制下，**每个数据包都有相应的计时器**，一旦超过 RTO 而没有收到 ACK，就重发该数据包。没收到 ACK 的数据包都会存在重传缓冲区里，等到 ACK 后，就从缓冲区里删除。

首先明确一点，对 TCP 来说，超时重传是**相当重要**的事件（RTO 往往大于两倍的 RTT，超时往往意味着拥塞），一旦发生这种情况，**TCP 不仅会重传对应数据段，还会降低当前的数据发送速率**，因为TCP 会认为当前网络发生了拥塞。

简单的超时重传机制往往比较低效，如下面这种情况：

![](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-31/transfer.png)

假设数据包5丢失，数据包 6,7,8,9 都已经到达接收方，这个时候客户端就只能等服务器发送 ACK，注意对于包 6,7,8,9，服务器都不能发送 ACK，这是滑动窗口机制决定的，因此对于客户端来说，他完全不知道丢了几个包，可能就悲观的认为，5 后面的数据包也都丢了，就重传这 5 个数据包，这就比较浪费了。

### 快速重传

快速重传机制「RFC5681」基于接收端的反馈信息来引发重传，而非重传计时器超时。

刚刚提到过，基于计时器的重传往往要等待很长时间，而快速重传使用了很巧妙的方法来解决这个问题：**服务器如果收到乱序的包，也给客户端回复 ACK**，只不过是重复的 ACK。就拿刚刚的例子来说，收到乱序的包 6,7,8,9 时，服务器全都发 ACK = 5。这样，客户端就知道 5 发生了空缺。一般来说，如果客户端连续三次收到重复的 ACK，就会重传对应包，而不需要等到计时器超时。

![](https://gitee.com/like-ycy/images/raw/master/blog/2019-12-31/fast-transfer.png)

但快速重传仍然没有解决第二个问题：到底该重传多少个包？

### 带选择确认的重传

改进的方法就是 SACK（Selective Acknowledgment），简单来讲就是在快速重传的基础上，**返回最近收到的报文段的序列号范围**，这样客户端就知道，哪些数据包已经到达服务器了。

来几个简单的示例：

- case 1：第一个包丢失，剩下的 7 个包都被收到了。

  当收到 7 个包的**任何一个**的时候，接收方会返回一个带 SACK 选项的 ACK，告知发送方自己收到了哪些乱序包。注：**Left Edge，Right Edge 就是这些乱序包的左右边界**。

```bash
             Triggering    ACK      Left Edge   Right Edge
             Segment

             5000         (lost)
             5500         5000     5500       6000
             6000         5000     5500       6500
             6500         5000     5500       7000
             7000         5000     5500       7500
             7500         5000     5500       8000
             8000         5000     5500       8500
             8500         5000     5500       9000
```

- case 2：第 2, 4, 6, 8 个数据包丢失。

- - 收到第一个包时，没有乱序的情况，正常回复 ACK。
  - 收到第 3, 5, 7 个包时，由于出现了乱序包，回复带 SACK 的 ACK。
  - 因为这种情况下有很多碎片段，所以相应的 Block 段也有很多组，当然，因为选项字段大小限制， Block 也有上限。

```bash
          Triggering  ACK    First Block   2nd Block     3rd Block
          Segment            Left   Right  Left   Right  Left   Right
                             Edge   Edge   Edge   Edge   Edge   Edge

          5000       5500
          5500       (lost)
          6000       5500    6000   6500
          6500       (lost)
          7000       5500    7000   7500   6000   6500
          7500       (lost)
          8000       5500    8000   8500   7000   7500   6000   6500
          8500       (lost)
```

不过 SACK 的规范「RFC2018」有点坑爹，接收方可能会在提供一个 SACK 告诉发送方这些信息后，又「食言」，也就是说，接收方可能把这些（乱序的）数据包删除掉，然后再通知发送方。以下摘自「RFC2018」：

> Note that the data receiver is permitted to discard data in its queue that has not been acknowledged to the data sender, even if the data has already been reported in a SACK option. **Such discarding of SACKed packets is discouraged, but may be used if the receiver runs out of buffer space.**

最后一句是说，**当接收方缓冲区快被耗尽时**，可以采取这种措施，当然并不建议这种行为。。。

由于这个操作，发送方在收到 SACK 以后，也不能直接清空重传缓冲区里的数据，一直到接收方发送普通的，ACK 号大于其最大序列号的值的时候才能清除。另外，重传计时器也收到影响，重传计时器应该忽略 SACK 的影响，毕竟接收方把数据删了跟丢包没啥区别。

### DSACK 扩展

DSACK，即重复 SACK，这个机制是在 SACK 的基础上，额外携带信息，**告知发送方有哪些数据包自己重复接收了**。DSACK 的目的是帮助发送方判断，是否发生了包失序、ACK 丢失、包重复或伪重传。让 TCP 可以更好的做网络流控。

关于 DSACK，「RFC2883」里举了很多例子，有兴趣的读者可以去阅读一下，我这里就不讲那么细了。