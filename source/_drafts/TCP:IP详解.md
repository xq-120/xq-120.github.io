### IP

IP协议位于四层模型中的网络层。网络模型常见的有7，5，4层模型。

#### IP报文

![](https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20170301092349308.png)

**版本**：IP协议的版本，目前的IP协议版本号为4，下一代IP协议版本号为6。

**首部长度**：IP报头的长度。固定部分的长度（20字节）和可变部分的长度之和。共占4位。最大为1111，即10进制的15，以32bits4字节为单位，代表IP报头的最大长度可以为15个32bits（4字节），也就是最长可为15*4=60字节，除去固定部分的长度20字节，可变部分的长度最大为40字节。

**服务类型**：Type Of Service。

**总长度**：IP报文的总长度。报头的长度和数据部分的长度之和。用16位二进制数表示，单位为8bits1字节，因此，IP报文的最大长度为65535字节。但由于MTU的限制，超过的话需要分片传输。所以就必须要有分片相关信息的字段，后面三个就是分片相关的字段。

**标识**：唯一的标识主机发送的每一份数据报。通常每发送一个报文，它的值加一。当IP报文长度超过传输网络的MTU（最大传输单元）时必须分片，这个标识字段的值被复制到所有数据分片的标识字段中，使得这些分片在达到最终目的地时可以依照标识字段的内容重新组成原先的数据。

**标志**：共3位。R、DF、MF三位。目前只有后两位有效，DF位：为1表示不分片，为0表示分片。MF：为1表示“更多的片”，为0表示这是最后一片。

**片偏移**：本分片在原先数据报文中相对首位的偏移量，字段长度为13位，以8bit1字节为单位，主要作用是为了使分片数据的接受者能够**按照正确的顺序**重组报文。如果没有这个片偏移值那么接收方收到分片后就不知道把这个分片的数据拼接在哪个位置了。

**生存时间**：IP报文所允许通过的路由器的最大数量。每经过一个路由器，TTL（Time To Live）减1，当为0时，路由器将该数据报丢弃。TTL 字段是由发送端初始设置一个 8 bit字段.推荐的初始值由分配数字 RFC 指定，当前值为 64。发送 ICMP 回显应答时经常把 TTL 设为最大值 255。

**协议**：指出IP报文携带的数据使用的是那种协议，以便目的主机的IP层能知道要将数据报上交到哪个进程（不同的协议有专门不同的进程处理）。和端口号类似，此处采用协议号，TCP的协议号为6，UDP的协议号为17。ICMP的协议号为1，IGMP的协议号为2.

**首部校验和**：计算IP头部的校验和，检查IP报头的完整性。

**源IP地址**：标识IP数据报的源端设备。

**目的IP地址**：标识IP数据报的目的地址。

有兴趣的可以研究下IP分片与分片重组。

#### 参考

[IP报文格式详解](https://blog.csdn.net/mary19920410/article/details/59035804)

[IP报文格式](http://www.023wg.com/message/message/cd_feature_ip_message_format.html)



### TCP

传输控制协议（Transmission Control Protocol，TCP）是一种面向连接的、可靠的、基于字节流的传输层通信协议。

TCP协议位于四层模型中的传输层

#### TCP报文

![](https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/032325597971641.jpg)

**源端口**：16bit，共65536个端口，端口0有特殊含义，不能使用，这样可用端口最多只有65535。

**目的端口**：16bit，接收方的端口号。

**序号**：32bit，可靠传输服务的关键字段。序号（sequence number， seq）是报文段首字节的字节流编号，TCP把数据看成是有序的字节流，TCP会隐式地对数据流的每个字节进行编号。第一个包的编号由本地随机产生，到达2^32-1后会再回到0。客户端、服务器端有各自的seq序号。

**确认号**：32bit，可靠传输服务的关键字段。确认号（acknowledgment number，ack）是期望的对方的下一个序号。

**首部长度**：4bit，以32bits4字节为单位，这样首部最大长度为15 * 4 = 60字节。除去固定部分的长度20字节，可变部分的长度最大为40字节。首部长度也称数据偏移，由于报头的长度可变 ，首部长度指示了数据区在报文段中的起始偏移值 。

**保留**：6bits。

**标志**：6bits。URG、ACK、PSH、RST、SYN、FIN共6个，每一个控制位表示一个控制功能。

* URG：紧急指针标志，为1时表示紧急指针有效，为0则忽略紧急指针。

* ACK：确认号标志，为1时确认号有效 ，为0时确认号无效。

* PSH：推送标志，为1表示是带有push标志的数据，指示接收方在接收到该报文段以后，应尽快将这个报文段交给应用程序，而不是在缓冲区排队。

* RST：重置连接标志，用于重置由于主机崩溃或其他原因而出现错误的连接。或者用于拒绝非法的报文段和拒绝连接请求。

* SYN：同步序号标志，用于建立连接，在连接建立时同步序号，SYN=1时表示这是一个连接请求或连接接受报文

  > 当SYN=1且ACK=0时，表明这是一个连接请求报文段
  >
  > 当SYN=1且ACK=1时，表明对方同意建立连接

  SYN=1的报文**不能携带数据**，但要**消耗掉一个序号**。SYN=0、ACK=1的报文**可以携带数据**，如果不携带数据则不消耗序号，此时下一个报文的序号与当前报文的序号相同。

* FIN：终止标志，用于释放连接，值为1表示要发送的数据报已经发送完毕，需要释放传输连接。

**窗口**：16bit，单位字节。滑动窗口大小，用于接收端告知发送端，本接受端的**缓存大小**，以此控制发送端发送数据的速率，从而达到流量控制。窗口最大为2^16-1=65535字节。这也限制了TCP的吞吐量，可以通过窗口缩放选项对这个值进行缩放。

**校验和**：奇偶校验，此校验和是对整个的 TCP 报文段，包括 TCP 头部和 TCP 数据，以 16 位字节进行计算所得。由发送端计算和存储，并由接收端进行验证。

**紧急指针**：只有在URG字段打开时才有效。暂时不清楚有啥用

**选项**：长度可变，最长40字节，当没有选项时报头长度是20字节。常见的有MSS，window scale，时间戳，SACK。

**填充**：选项长度不一定是32位的整数倍，所以要加填充位，即在这个字段中加入额外的零，以保证TCP头是32的整数倍。

**数据部分**： TCP 报文段中的数据部分是可选的。
在一个连接建立和终止时，双方交换的报文段仅有 TCP 报头。如果一方没有数据要发送，也使用没有任何数据的报头来确认收到的数据。在处理超时的许多情况中，也会发送不带任何数据的报头。





1.TCP报文的每一个字节是否都有编号？TCP报文中的序号字段的值代表的是哪个字节的编号？

是的，TCP会隐式地对数据流的每个字节进行编号。而序号是报文段首字节的字节流编号。假如当前报文的序号为1，长度为200，则下一个TCP报文的序号是201。



2.三次握手



2.1为什么需要三次握手，两次不行吗？



2.2客服端和服务器同时发起SYN握手会怎样？



3.四次挥手

TCP一般是通过4次挥手后关闭连接，但在满足一些情况下3次挥手也可以关闭连接，这是一个优化。

From Wikipedia:

> It is also possible to terminate the connection by a 3-way handshake, when host A sends a FIN and host B replies with a FIN & ACK (merely combines 2 steps into one) and host A replies with an ACK.

原话出自computer networks，作者Andrew S. Tanenbaum。

用wireshark抓包也经常抓到三次挥手断开连接的：

![](https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20200805213029.png)

我的理解是：比如客户端发送FIN，然后服务器也没有要发送的数据，也没有其他准备工作于是就可以发送FIN+ACK（两步并一步），最后客户端发送ACK，这样三次挥手后连接关闭。

标准的四次挥手：

![](https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20200814160414.png)

### 参考

[不要再问我三次握手和四次挥手](https://yuanrengu.com/2020/77eef79f.html)

[【115期】TCP协议10连问，总会用得到，建议收藏~](https://mp.weixin.qq.com/s/cfa1f8NR6hs42I5FYCkl6A?)

[理解TCP序列号（Sequence Number）和确认号（Acknowledgment Number）](https://blog.csdn.net/a19881029/article/details/38091243?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-3.nonecase&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-3.nonecase)   译文,通过抓包来解释。

[Understanding TCP Sequence and Acknowledgment Numbers](https://packetlife.net/blog/2010/jun/7/understanding-tcp-sequence-acknowledgment-numbers/)  原文

[TCP中的序号和确认号](https://andrewpqc.github.io/2018/07/17/sequence-number-and-ack-number-in-tcp/)

[就是要你懂 TCP](http://jm.taobao.org/2017/06/08/20170608/)   阿里中间件团队博客 



[Http系列(二) Http2中的多路复用](https://juejin.im/post/6844903935648497678)



[TCP快速打开]([https://zh.wikipedia.org/wiki/TCP%E5%BF%AB%E9%80%9F%E6%89%93%E5%BC%80](https://zh.wikipedia.org/wiki/TCP快速打开))  wiki



什么是RFC

比如TCP的：

<img src="https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20200810174003.png" style="zoom:50%;" />

[RFC](https://zh.wikipedia.org/wiki/RFC)  wiki

[[译] 如何阅读 RFC 文档](https://juejin.im/post/6844903716051484679)  如何阅读RFC









[干眼症如何预防和治疗？](https://www.zhihu.com/question/20100498)

**多眨眼、勤热敷；少看屏幕，多远眺。**

**第一，有意识的多眨眼。**可以放松睫状肌，滋润眼球。

当然，你也别走极端式眨眼法，一秒十下，毕竟太快了谁都受不了；

**第二，热敷。**眼睛除了能分泌泪水，**还能分泌少量的油脂，帮助保湿。**

但有时候，**油脂腺**会被堵塞，用热毛巾敷眼睛，**能疏通堵塞的腺体，有效缓解眼睛干涩。**

**第三，少看屏幕多远眺。**

这个就不多说了，你要问我时间，可以记住 20 原则：**就是用眼 20 分钟后，要盯着至少 20 米以外的地方，休息至少 20 秒。**

人眼**往上看时，睑裂将会增大**，泪膜需要覆盖的区域就得更广，而且往上看时**更容易「瞪眼睛」**，也即出现长时间不眨眼的现象。

用电脑时，眼睛最好距离屏幕**半米**，电脑的中心在眼睛**下方 15 cm 左右**。



