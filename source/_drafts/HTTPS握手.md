HTTPS是使用TLS/SSL加密的HTTP协议。HTTP协议通过明文进行信息传输，存在信息窃听、信息篡改和身份冒充的风险，而协议TLS/SSL具有**信息加密**、**完整性校验**和**身份验证**的功能，可以避免此类问题。

TLS/SSL全称为：安全传输层协议（Transport Layer Security），是介于TCP和HTTP之间的一层安全协议，不影响原有的TCP协议和HTTP协议，所以使用HTTPS基本上不需要对HTTP页面进行太多的改造。

HTTPS握手过程，主要是为了验证双方身份和协商对称加密的密钥。

打开Wireshark，输入 `tcp.port == 443` 过滤请求，打开掘金网站。

整个过程如下：

![](https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20200823091745.png)

## Client Hello

抓包如下：

![](https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20200823084104.png)

客户端发起Client Hello，主要向服务器提供以下信息：

**支持的最高TLS版本号**

Version: TLS 1.2 (0x0303)。

注：上面还有一个TLS版本，它表示的是本次通信使用的TLS版本为1.0。

**一个随机数**

Random: 773ed2572a0a82ff9da714fe306e41051e76caea546b2fbea0bfc64e5eba7e77

**支持的加密套件（Cipher Suites）**

<img src="https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20200823084751.png" style="zoom:50%;" />

**支持的压缩方法**

## Server Hello

抓包如下：

![](https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20200823085430.png)

服务器收到客户端请求后，向客户端发出回应，这叫做SeverHello。服务器的回应包含以下内容：

**确认使用的TLS协议版本**

服务端确认通信使用的TLS版本：TLS 1.2。如果浏览器与服务器支持的版本不一致，则服务器关闭加密通信。

**一个随机数**

Random: 09b429492132f3f5ee2d35b7406861b9a55be4e3893f54cd71d4f9f21fd5d472

**确认一个加密套件**

从客户端提供的加密套件列表中，选择一个双方都支持的加密套件。

比如这里选择的是：`Cipher Suite: TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 (0xc030)`

**确认一个压缩方法**

这里选择的是null。

此时客户端和服务端就已经协商好了将要使用的TLS版本号，密码套件，并且客户端和服务端都拥有了两个随机数。

## Server Certificate, Certificate Status, Server Key Exchange, Server Hello Done

Server Hello发出去后，服务端会**紧接着发送**一个 `Certificate, Certificate Status, Server Key Exchange, Server Hello Done` 消息：

![](https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20200823131804.png)

只不过该消息比较大已经横跨3个包：42，43和45。比如第42个包除了Server Hello消息的数据外，还包括服务器证书的部分数据了。包43被标记为 `[TCP segment of a reassembled PDU]` 。

该消息包含了4个子信息：服务器证书，证书状态，服务器Key交换，服务器Hello完成。

**Certificate**

其中一个很重要的东西——CA证书，也就是我们平时说的数字证书。

该证书通常有两个目的：

- 用于给客户端进行身份验证。客户端拿到后会验证证书，确定是不是跟真正的服务器在通信。
- 证书中包含服务器的公钥，该公钥结合密码套件的密钥协商算法协商出预备主密钥。

**Server Key Exchange**

该信息用于密钥交换。核心是DH算法。两端通过交换DH公钥，再通过DH算法就可以分别计算出密钥K。这个密钥K就是第三个随机数，采用DH算法第三个随机数就可以不用通过网络传递而是在各自的本地计算出来。

该消息的发送需要满足一定的条件，还记得刚刚在Server Hello中说的，选择不同的密码套件会有不同的动作产生，该信息的发送就是属于不同的动作。

它发送的条件是，如果证书包含的信息不足以进行密钥交换，那么必须发送该信息。通常协商的加密套件属于下面的类型，就会发送：

- DHE_DSS
- DHE_RSA
- ECDHE_ECDSA
- ECDHE_RSA

基本上里面都有用到DH算法。

而刚刚服务器确定的是`TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384` 所以这里我们才看到该信息的发送。并且该信息是紧跟着Server Certificate信息的。

![](https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20200823134002.png)

客户端每次连接服务器的时候，服务器都会发送动态DH信息（DH参数和DH公钥，DH公钥并不是证书上的那个公钥），这些信息不存在证书中，需要通过Server Key Exchange信息来进行信息传递，**并且传递的DH信息需要使用服务器证书的私钥进行签名。这样客户端就可以用服务器证书的公钥进行解密验证是否一致确保不被篡改。**

DH公钥就是`Pubkey`，签名就是`Signatrue`，DH参数就是由`Curve Type`、`Named Curve`、`Pubkey`组成的。

**Server Hello Done**

告诉客户端服务器hello完毕，该信息的主要作用有两点：

- 服务端发送了足够的信息，接下来可以和客户端一起协商出预备主密钥
- 客户端接收到该信息后，可以进行证书校验、协商密钥等步骤

为什么需要发送Server Hello Done这个信息？如果不发送，那么客户端根本不知道计算第三个随机数需要的信息是否足够。

## Client Key Exchange, Change Cipher Spec, Encrypted Handshake Message

客户端在接受到服务器的Server Hello Done信息之后，首先验证服务器证书。如果证书不是可信机构颁布、或者证书中的域名与要访问的实际域名不一致、或者证书已经过期，就会向访问者显示一个警告，由其选择是否还要继续通信。

如果证书没有问题，客户端会立刻发送该信息，该消息包含3个子消息：`Client Key Exchange, Change Cipher Spec, Encrypted Handshake Message`。

**Client Key Exchange**

该信息的主要作用是协商出预备主密钥（即第三个随机数），第三个随机数一般有两种方式生成：

1. 由客户端生成一个随机数，然后通过服务器证书上的公钥加密，发送给服务器。服务器收到后通过私钥解密。

2. 通过服务器发送的DH参数计算出客户端的DH公钥，并传递给服务器，然后双方通过DH算法在本地计算出第三个随机数。

![](https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20200823135300.png)

通过抓包信息可以看出，这里使用的是第二种方式。因为这里只将一个DH公钥发送给了服务端。

具体客户端是如何计算出该DH公钥的可以参考DH算法的实现流程。

最终，客户端和服务器会计算出相同的预备主密钥。而该预备主密钥就作为第三个随机数。

我们知道前面已经得到了两个随机数了，最后客户端和服务端会根据这三个随机数计算出对称加密所需要的密钥。

**Change Cipher Spec**

该信息用于告诉对方，我已经计算好需要使用的对称密钥了，我们接下来的通信都需要使用该密钥进行加密之后再发送。

**Encrypted Handshake Message**

客户端将上面的信息通过协商的密钥加密后发送给服务器，服务器收到后如果有第三个随机数预备主密钥，就可以计算出对称加密密钥，从而解密该消息，如果解密不出，说明某个环节出了问题。

到此为止客户端这边就算是准备完成了。

Q：客户端是如何验证服务器身份的？

TODO：这里需要一些前置知识，需要知道CA证书是什么东西。

## New Session Ticket, Change Cipher Spec, Encrypted Handshake Message

服务器收到客户端的消息后，会计算出对称加密的密钥，然后解密客户端的Encrypted Handshake Message。如果解密成功则代表双方协商出的对称加密密钥没毛病，于是发送该消息。

**New Session Ticket**

由于SSL握手过程比较复杂和耗费时间，如果每次请求都先来这么一套显然会影响性能，因此服务器在第一次握手成功后会发送一个Session Ticket。下一次客户端发起握手时带上该Session Ticket，服务器验证通过后就可以完成一次快速的握手。这是对常规SSL握手的优化。

当然该Session Ticket是有有效期的，如下显示有效期为10分钟。

![](https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20200823141024.png)

**Change Cipher Spec**

该消息用于服务器端通知客户端，随后的信息都将用双方商定的加密方法和密钥加密后发送。

**Encrypted Handshake Message**

同样服务器也会将上面的信息通过协商的密钥加密后发送给服务器，供客户端验证。

到此为止服务器端也准备完成了，于是整个HTTPS握手就完成了，后续传输的信息都是经过密钥加密后的密文。



## DH算法

简单来讲：

1. Alice和Bob 公开同意使用一个素数p，一个基数g
2. Alice选择一个秘密整数a作为自己的私钥，然后计算出自己的公钥A并发送给Bob
3. Bob也选择一个秘密整数b作为自己的私钥，然后计算出自己的公钥B并发送给Alice
4. Alice根据Bob的公钥B，自己的私钥a以及素数p，计算出秘密s
5. Bob根据Alice的公钥A，自己的私钥b以及素数p，计算出秘密s
6. 两人都拥有了一个秘密整数s，通常作为对称加密的密钥。

上面一共涉及到两对公私钥（A-a，B-b）及素数p和基数g六个数，并由此计算出一个共享的秘密整数s。

因为私钥并不传递所以是安全的。DH算法可以在一个不安全的信道上协商出一个安全的对称加密密钥。

## 参考

[iOS安全系列之二：HTTPS进阶](https://www.jianshu.com/p/56bcc14ab052)  这一篇结合了wireshark抓包分析,更加具体.

[SSL/TLS协议运行机制的概述](https://www.ruanyifeng.com/blog/2014/02/ssl_tls.html)

[HttpDns 原理是什么](http://www.linkedkeeper.com/171.html) 

[iOS应用安全之HTTP/HTTPS详解（AFNetworking配套策略）](https://blog.csdn.net/Deft_MKJing/article/details/82868900)  这位博主介绍的是真的全而且是实践过的.TODO:自己也要重复试验一遍.

[https握手流程详解](https://juejin.im/post/6844904135230390279)  4星

[理解 Diffie-Hellman 密钥交换算法](http://wsfdl.com/algorithm/2016/02/04/%E7%90%86%E8%A7%A3Diffie-Hellman%E5%AF%86%E9%92%A5%E4%BA%A4%E6%8D%A2%E7%AE%97%E6%B3%95.html)

[Diffie–Hellman key exchange](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange)  wiki百科，已经讲的很清楚了。

[从 wireshark 抓包开始学习 https](https://cloud.tencent.com/developer/article/1004578)  不错

