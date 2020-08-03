TCP

传输控制协议（Transmission Control Protocol，TCP）是一种面向连接的、可靠的、基于字节流的传输层通信协议。

### TCP报文格式

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



### 参考

[面试官，不要再问我三次握手和四次挥手](https://yuanrengu.com/2020/77eef79f.html)

[【115期】TCP协议面试10连问，总会用得到，建议收藏~](https://mp.weixin.qq.com/s/cfa1f8NR6hs42I5FYCkl6A?)

[理解TCP序列号（Sequence Number）和确认号（Acknowledgment Number）](https://blog.csdn.net/a19881029/article/details/38091243?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-3.nonecase&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-3.nonecase)   译文,通过抓包来解释。

[Understanding TCP Sequence and Acknowledgment Numbers](https://packetlife.net/blog/2010/jun/7/understanding-tcp-sequence-acknowledgment-numbers/)  原文

[TCP中的序号和确认号](https://andrewpqc.github.io/2018/07/17/sequence-number-and-ack-number-in-tcp/)



[Http系列(二) Http2中的多路复用](https://juejin.im/post/6844903935648497678)





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



