# 可靠性交付的实现

TCP 是一种提供可靠性交付的协议。    
也就是说，通过 TCP 连接传输的数据，无差错、不丢失、不重复、并且按序到达。    
但是在网络中相连两端之间的介质，是复杂的，并不确保数据的可靠性交付，那么 TCP 是怎么样解决问题的？    
这就需要了解 TCP 的几种技术：    

1. 滑动窗口    
2. 超时重传    
3. 流量控制    
4. 拥塞控制    

下面来分别讲一下这几种技术的实现原理。    

# 超时重传
## 重传时机
TCP 报文段在传输的过程中，下面的情况都是有可能发生的：

1. 数据包中途丢失；
2. 数据包顺利到达，但对方发送的 ACK 报文中途丢失；
3. 数据包顺利到达，但对方异常未响应 ACK 或被对方丢弃；

当出现这些异常情况时，TCP 就会超时重传。    
TCP 每发送一个报文段，就对这个报文段设置一次计时器。只要计时器设置的重传时间到了，但还没有收到确认，就重传这一报文段，这个就叫做「超时重传」。    

## 重传算法

### 先认识两个概念

#### RTO ( Retransmission Time-Out ) 重传超时时间
指发送端发送数据后、重传数据前等待接收方收到该数据 ACK 报文的时间。    
大白话就是，需要等待多长时间还没收到确认，就重新传一次。    

RTO 的设置对于重传非常重要：    

1. 设长了，重发就慢，没有效率，性能差；
2. 设短了，重发得就快，会增加网络拥塞，导致更多的超时，更多的超时导致更多的重发。

#### RTT ( Round Trip Time ) 连接往返时间
指发送端从发送 TCP 包开始到接收它的 ACK 报文之间所耗费的时间。    
而在实际的网络传输中，RTT 的值每次都是随机的，无法事先预预知。    
TCP 通过测量来获得连接当前 RTT 的一个估计值，并以该 RTT 估计值为基准来设置当前的 RTO。    
这就引入了一类算法的称呼：自适应重传算法（Adaptive Restransmission Algorithm）    
这类算法的关键就在于对当前 RTT 的准确估计，以便适时调整 RTO。    

关于自适应重传算法，经历过多次的迭代和修正。    
从 1981 年的 [RFC793](https://tools.ietf.org/html/rfc793) 提及的经典算法，到 1987 年 Karn 提出的 Karn/Partridge 算法，再到后来的 1988 年的 Jacobson / Karels 算法。    
最后的这个算法在被用在今天的 TCP 协议中（Linux的源代码在：[`tcp_rtt_estimator`](http://lxr.free-electrons.com/source/net/ipv4/tcp_input.c?v=2.6.32#L609)）。    

自适应重传算法的发展读者有兴趣可以参考其他资料，在这里我拎一个现在在用的算法出来讲讲，随意感受一下。    

### Jacobson / Karels 算法
1988年，有人推出来了一个新的算法，这个算法叫 Jacobson / Karels Algorithm（参看[RFC6298](https://tools.ietf.org/html/rfc2988)）。    
其计算公式：

> SRTT = SRTT + α ( RTT – SRTT )  —— 计算平滑 RTT

> DevRTT = ( 1-β ) * DevRTT + β * ( | RTT - SRTT | ) ——计算平滑 RTT 和真实的差距（加权移动平均）

> RTO= µ * SRTT + ∂ * DevRTT 

其中：
* `α`、`β`、`μ`、`∂` 是可以调整的参数，在 RFC6298 中给出了对应的参考值，而在Linux下，α = 0.125，β = 0.25， μ = 1，∂ = 4；

* SRTT 是 Smoothed RTT 的意思，是 RTT 的平滑计算值，即根据每次测量的 RTT 和旧的 RTT 进行运算，得出新的 RTT。SRTT 的值，会在每一次测量到 RTT 之后进行更新；

* DevRTT 是 Deviation RTT 的意思，根据每次测量的 RTT 和旧的 SRTT 值进行运算，得出新的 DevRTT；

由算法可以知道 RTO 的值会根据每次测量的 RTT 值变化而变化，基本要点是 TCP 监视每个连接的性能，由每一个 TCP 的连接情况推算出合适的 RTO 值，根据不同的网络情况，自动修改 RTO 值，以适应负责的网络变化。

# 拥塞控制
## 慢启动（Slow Start） 与 拥塞避免（Congestion Avoidance）
- [ ] TODO

## 快速重传（Fast Retransmit） 与 快速恢复（Fast Recovery）
- [ ] TODO

# 滑动窗口 Sliding Window
滑动窗口协议比较复杂，也是 TCP 协议的精髓所在。    

TCP 头里有一个字段叫 Window，叫 Advertised-Window，这个字段是接收端告诉发送端自己还有多少缓冲区可以接收数据。于是发送端就可以根据这个接收端的处理能力来发送数据，而不会导致接收端处理不过来。    

滑动窗口分为「接收窗口」和「发送窗口」    
因为 TCP 协议是全双工的，会话的双方都可以同时接收和发送，那么就需要各自维护一个「发送窗口」和「接收窗口」。    

## 发送窗口
大小取决于对端通告的接受窗口。    
只有收到对端对于本端发送窗口内字节的 ACK 确认，才会移动发送窗口的左边界。    

下图是发送窗口的示意图：

![tcps-send-wwindows.png](http://om6ayrafu.bkt.clouddn.com/post/understand-tcp-udp/FCA43D210DF50C93E428DFD04FBBBF32.png)

对于发送窗口，在缓存内的数据有四种状态：

- \#1 已发送，并得到接收方 ACK 确认；
- \#2 已发送，但还未收到接收方 ACK；
- \#3 未发送，但接收方允许发送，接收方还有空间
- \#4 未发送，且接收方不允许发送，接收方没有空间

如果下一刻，收到了接收方对于 32-36 字节序的数据包的 ACK 确认，那么发送方的窗口就会发生「滑动」。    
并且发送下一个 46-51 字节序的数据包。    

![tcps-send-wslide.png](http://om6ayrafu.bkt.clouddn.com/post/understand-tcp-udp/4C22A2B58DB2F0B885A0DC50057D2768.png)

滑动窗口的概念，描述了 TCP 的数据是怎么发送，以及怎么接收的。    
TCP 的滑动窗口是动态的，我们可以想象成小学常见的一个数学题，一个水池，体积 V，每小时进水量 V1, 出水量 V2。    
当水池满了就不允许再注入了，如果有个液压系统控制水池大小，那么就可以控制水的注入速率和量了。    
应用程序可以根据自身的处理能力变化，通过 API 来控制本端 TCP 接收窗口的大小，来进行流量控制。    

## 接收窗口
大小取决于应用、系统、硬件的限制。    

下图是接收窗口的示意图（找不到图，唯有自己画了）：    

![tcps-receive-wwindows.png](http://om6ayrafu.bkt.clouddn.com/post/understand-tcp-udp/F4B7AEDE41EE179676E79DEF2601D4A4.png)

相对于发送窗口，接受窗口在缓存内的数据只有三种状态：

* 已接收已确认；
* 未接收，准备接收；
* 未接收，并未准备接收；

下一刻接收到来自发送端的 32-36 数据包，然后回送 ACK 确认报，并且移动接收窗口。    

![tcps-receive-wslide.png](http://om6ayrafu.bkt.clouddn.com/post/understand-tcp-udp/95A36446FAD21CC3DD086FA683942FFA.png)

另外接收端相对于发送端还有不同的一点，只有前面所有的段都确认的情况下才会移动左边界，    
在前面还有字节未接收但收到后面字节的情况下，窗口不会移动，并不对后续字节确认，以此确保对端会对这些数据重传。    
假如 32-36 字节不是一个报文段的，而是每个字节一个报文段的话，那么就会分成了 5 个报文段。    
在实际的网络环境中，不能确保是按序收到的，其中会有一些早达到，一些迟到达。    

![tcps-receive-disorder.png](http://om6ayrafu.bkt.clouddn.com/post/understand-tcp-udp/686E3FC14C2DEF657C61ECBC16C9C954.png)

如图中的 34、35 字节序，先收到了，接收窗口也不会移动。    
因为有可能 32、33 字节序会出现丢包或者超时，这时就需要发送端重发报文段了。    

# 参考
[The TCP/IP Guide](http://www.tcpipguide.com/free/t_TCPSlidingWindowAcknowledgmentSystemForDataTranspo.htm)    
[TCP 的那些事儿（下）](http://coolshell.cn/articles/11609.html)    
[《后台开发 核心技术与应用实践》](https://book.douban.com/subject/26850616/)    
[《计算机网络》](https://book.douban.com/subject/2970300/)    
