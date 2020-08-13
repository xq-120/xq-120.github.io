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

![](http://c.biancheng.net/uploads/allimg/191108/6-19110Q62344I5.gif)

1.序号的计算

**SYN**：同步序号，用于 **建立连接** 过程。
当 **SYN=1，ACK=0** 时表示：这是一个连接请求报文段。
若同意连接，则在响应报文段中 **SYN=1，ACK=1**。
因此，SYN=1表示这是一个 **建立连接请求**，或 **同意建立连接** 报文。SYN这个标志位只有在TCP建产连接时才会被置1，握手完成后SYN标志位被置0。
**注意：**

- SYN=1的报文 **不能携带数据**，但要 **消耗掉一个序号**
- SYN=0、ACK=1的报文 **可以携带数据**，如果不携带数据则不消耗序号，此时下一个报文的序号与当前报文的序号相同。？？？



2.TCP传输的每一个字节是否都有序列号？TCP报文中的序号字段的值代表的是哪个字节的序号？



3.四次挥手

TCP一般是通过4次挥手后关闭连接，但在满足一些情况下3次挥手也可以关闭连接，这是一个优化。

From Wikipedia:

> It is also possible to terminate the connection by a 3-way handshake, when host A sends a FIN and host B replies with a FIN & ACK (merely combines 2 steps into one) and host A replies with an ACK.

原话出自computer networks，作者Andrew S. Tanenbaum。

用wireshark抓包也经常抓到三次挥手断开连接的：

![](https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20200805213029.png)

我的理解是：比如客户端发送FIN，然后服务器也没有要发送的数据，也没有其他准备工作于是就可以发送FIN+ACK（两步并一步），最后客户端发送FIN，这样三次挥手后连接关闭。

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



