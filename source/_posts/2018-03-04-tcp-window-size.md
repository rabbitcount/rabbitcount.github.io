---
title: TCP 协议中的 Window Size与吞吐量
date: 2018-03-04 17:50:39
tags:
- tcp
- tcp窗口
---

# 摘要

「rwnd」：接收窗口
「cwnd」：拥塞窗口
「ssthresh」：慢启动阈值

---

[引自：TCP 协议中的 Window Size与吞吐量](http://blog.sina.com.cn/s/blog_c5c2d6690102wpxl.html)

TCP协议中影响实际业务流量的参数很多，这里主要分析一下窗口的影响。

# ​TCP窗口目的

为了获得最优的连接速率，使用TCP窗口来控制流速率（flow control），滑动窗口就是一种主要的机制。
这个窗口允许源端在给定连接传送数据分段而不用等待目标端返回ACK，一句话描述：

> 窗口的大小决定在不需要对端响应（acknowledgement）情况下传送数据的数量。

​官方定义：“<font color=#0099ff>The amount of octets that can be transmitted without receiving an acknowledgement from the other side</font>”。

## TCP窗口机制 - 接收窗口「rwnd」

​​`TCP header`中有一个`Window Size`字段，它其实是指接收端的窗口，即接收窗口(<font color=#0099ff>「rwnd」</font>)，用来告知发送端自己所能接收的数据量，从而达到一部分流控的目的。其实TCP在整个发送过程中，也在度量当前的网络状态，目的是为了维持一个健康稳定的发送过程，比如拥塞控制。因此，数据是在某些机制的控制下进行传输的，就是窗口机制。发送端的发送窗口是基于接收端的接收窗口来计算的，也就是我们常说的TCP是有连接的发送，数据传输需要对端确认，发送的数据分为如下四类来看

![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/wiki/tcp-window-size/%E7%AA%97%E5%8F%A3%E6%BB%91%E5%8A%A8%E5%8F%91%E9%80%81%E6%95%B0%E6%8D%AE.jpg)窗口滑动发送数据

- 已经发送并且对端确认（**Sent/ACKed**）：发送窗外缓冲区外
- 已经发送但未收到确认数据（**Sent/UnACKed**）： 发送窗内缓冲区内​
- 允许发送但尚未防的数据​（**Unsent/Inside**）： 发送窗内缓冲区内​
- 未发送暂不允许（**Unsent/Outside**）： 发送窗外缓冲区内​

TCP窗口就是这样逐渐滑动，发送新的数据，滑动的依据就是发送数据已经收到ACK，确认对端收到，才能继续窗口滑动发送新的数据。可以看到窗口大小对于吞吐量有着重要影响，同时ACK响应与系统延时又密切相关。

需要说明的是：
- 发送端的窗口过大：会引起接收端关闭窗口，处理不过来；
- 发送端的窗口设置较小：结果就是不能充分利用带宽；

所以仔细调节窗口对于适应不同延迟和带宽要求的系统很重要。

## TCP接收窗口「rwnd」大小

最早`TCP`协议涉及用来大范围网络传输时候，其实是没有超过`56Kb/s`的​连接速度的。因此，`TCP`包头中只保留了`16bit`用来标识窗口大小，允许的最大缓存大小不超过`64KB`。为了打破这一限制，`RFC1323`规定了TCP窗口尺寸选择，是在TCP连接开始的时候三步握手的时候协商的（**SYN, SYN-ACK, ACK**），会协商一个  `Window size scaling factor`，之后交互数据中的是`Window size value`，所以最终的窗口大小是二者的乘积.

** <font color=#0099ff>Window size value</font>: 64 or 0000 0000 0100 0000 (16 bits) **
** <font color=#0099ff>Window size scaling factor</font>: 256 or 2 ^ 8 (as advertised by the 1st packet) **

> The actual window size is 16,384 (64 window size value \* 256 window size scaling factor)

这里的窗口大小就意味着，直到发送16384个字节，才会停止等待对方的ACK.随着双方回话继续，窗口的大小可以修改`window size value` 参数完成变窄或变宽，但是注意：**Window size scaling factor 乘积因子**必须保持不变。在RFC1323中规定的偏移（`shift count`）是14，也就是说最大的窗口可以达到**Gbit**,很大。

![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/wiki/tcp-window-size/wireshark%E6%8A%93%E5%8C%85%E5%AE%9E%E4%BE%8B.jpeg) Wireshark抓包实例

这一机制并不总是默认开启的和系统有关，貌似Linux默认开启，Windows默认关闭。

# ​TCP窗口参数设置

TCP窗口起着控制流量的作用，实际使用时这是一个双端协调的过程，还涉及到TCP的`慢启动`​（`Rapid Increase/Multiplicative Decrease`），拥塞避免，拥塞窗口和拥塞控制。可以记住，**发送速率是由`min(拥塞窗口，接收窗口)`**，接收窗口在下文有讲。

## TCP接收窗口「rwnd」优化设置​

### TCP接收窗口「rwnd」设置方法

1. 一个简单的原则是2倍的`BDP`.这里的`BDP`的意思是**bandwidth-delay
product**，也就是带宽和时延的乘积，带宽对于网络取最差连接的带宽。
> buffer size = 2 \* bandwidth \* delay​

2. 还有一种简单的方式，使用ping来计算网络的环回时延（RTT），然后表达为：
> buffer size = bandwidth \* RTT​

#### 接收窗口「rwnd」计算方法中为什么是2倍？

因为可以这么想，如果滑动窗口是 `bandwidth * delay`，当发送一次数据最后一个字节刚到时，对端要回ACK才能继续发送，就需要等待一次单向时延的时间，所以当是2倍时，刚好就能在等ACK的时间继续发送数据，等收到ACK时数据刚好发送完成，这样就提高了效率。

举个例子

> 带宽是`20Mbps`,通过`ping`我们计算*单向时延*(RTT)是`20ms`;
可以计算：**20000000bps \* 8 \* 0.02 = 52,428bytes​**
因此我们最优窗口用 **104,856bytes = 2 x 52,428**
所以说当发送者发送 **104,856 bytes** 数据后才需要等待一个ACK响应，当发送了一半的时候，对端已经收到并且返回ACK（理想情况），等到ACK回来，又把剩下的一半发送出去了，所以发送端就无需等待ACK返回。

发现了么？这里的窗口已经明显大于`64KB`了，所以机制改善了，上一级。

## TCP窗口流量控制​

​现在我们看看到底如何控制流量。TCP在传输数据时和 `windows size` 关系密切，本身窗口用来控制流量，在传输数据时，发送方数据超过接收方就会丢包，流量控制，流量控制要求数据传输双方在每次交互时声明各自的接收窗口**「rwnd」**大小，用来表示自己最大能保存多少数据，这主要是针对接收方而言的，通俗点儿说就是让发送方知道接收方能吃几碗饭，如果窗口衰减到零，也就是发送方不能再发了，那么就说明吃饱了，必须消化消化，如果硬撑胀漏了，那就是丢包了。

![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/wiki/tcp-window-size/%E6%B5%81%E9%87%8F%E6%8E%A7%E5%88%B6.jpeg) 流量控制

### 慢启动 （ 拥塞窗口「cwnd」）

虽然流量控制可以避免发送方过载接收方，但是却无法避免过载网络，这是因为接收窗口**「rwnd」**只反映了服务器个体的情况，却无法反映网络整体的情况。

为了避免网络过载，慢启动引入了拥塞窗口**「cwnd」**的概念，用来表示发送方在得到接收方确认前，最大允许传输的未经确认的数据。**「cwnd」**同**「rwnd」**相比不同的是：它只是发送方的一个内部参数，无需通知给接收方，其初始值往往比较小，然后随着数据包被接收方确认，窗口成倍扩大，有点类似于拳击比赛，开始时不了解敌情，往往是次拳试探，慢慢心里有底了，开始逐渐加大重拳进攻的力度。

![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/wiki/tcp-window-size/%E6%8B%A5%E5%A1%9E%E7%AA%97%E5%8F%A3%E6%89%A9%E5%A4%A7.jpeg) 拥塞窗口扩大

在慢启动的过程中，随着**「cwnd」**的增加，可能会出现网络过载，其外在表现就是丢包，一旦出现此类问题，**「cwnd」**的大小会迅速衰减，以便网络能够缓过来。

![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/wiki/tcp-window-size/%E6%8B%A5%E5%A1%9E%E7%AA%97%E5%8F%A3%E4%B8%8E%E4%B8%A2%E5%8C%85.jpeg) 拥塞窗口与丢包

说明： 
> 网络中实际传输的未经确认的数据大小取决于 `min (「rwnd」和「cwnd」)`

### 拥塞避免​ （ 慢启动阈值「ssthresh」）

从慢启动的介绍中，我们能看到，发送方通过对**「cwnd」**大小的控制，能够避免网络过载，在此过程中，丢包与其说是一个网络问题，倒不如说是一种反馈机制，通过它我们可以感知到发生了网络拥塞，进而调整数据传输策略；
实际上，这里还有一个`慢启动阈值` **「ssthresh」**的概念，
- **「cwnd」** < **「ssthresh」**，那么表示在慢启动阶段；
- **「cwnd」** > **「ssthresh」**，那么表示在拥塞避免阶段，此时**「cwnd」**不再像慢启动阶段那样呈*指数级增长*，而是趋向于*线性增长*，以期避免网络拥塞，此阶段有多种算法实现，通常保持缺省即可，这里就不一一说明了。

### 如何调整 - 接收窗口「rwnd」

很多时候TCP的传输速率异常偏低，很有可能是接收窗口**「rwnd」**过小导致，尤其对于时延较大的网络，实际上接收窗口**「rwnd」**的合理值取决于`BDP`的大小，也就是带宽和延迟的乘积。假设带宽是 `100Mbps`，延迟是 `100ms`，那么计算过程如下：

> BDP = 100Mbps * 100ms = (100 / 8) * (100 / 1000) = 1.25MB​

​此问题下如果想最大限度提升吞度量，接收窗口**「rwnd」**的大小不应小于 1.25MB。

### 如何调整 - 拥塞窗口「cwnd」
一般来说**「cwnd」**的初始值取决于MSS的大小，计算方法如下：

> min(4 * MSS, max(2 * MSS, 4380))

以太网标准的MSS大小通常是1460，所以**「cwnd」**的初始值是`3MSS`。当我们浏览视频或者下载软件的时候，**「cwnd」**初始值的影响并不明显，这是因为传输的数据量比较大，时间比较长，相比之下，即便慢启动阶段**「cwnd」**初始值比较小，也会在相对很短的时间内加速到满窗口，基本上可以忽略不计。下图使用IxChariot完成一次设置​

![](http://anocelot-wiki.oss-cn-hangzhou.aliyuncs.com/wiki/tcp-window-size/%E8%AE%BE%E7%BD%AEcwnd.jpeg) 设置cwnd

不过当我们浏览网页的时候，情况就不一样了，这是因为传输的数据量比较小，时间比较短，相比之下，如果慢启动阶段**「cwnd」**初始值比较小，那么很可能还没来得及加速到满窗口，通讯就结束了。这就好比博尔特参加百米比赛，如果起跑慢的话，即便他的加速很快，也可能拿不到好成绩，因为还没等他完全跑起来，终点线已经到了

