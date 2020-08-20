### HTTP各版本特性

#### HTTP 0.9

1991年发布

基本上只是一个草稿，设计的初衷只是为了获取简单的HTML对象。只支持GET。没怎么使用就被HTTP 1.0代替了。

#### HTTP 1.0

1996年5月发布规范 rfc1945

第一个广泛使用的版本。

**相比HTTP 0.9**：

1. 增加了对多媒体对象的处理，任何格式的内容都可以发送。这使得互联网不仅可以传输文字，还能传输图像、视频、二进制文件。

2. 增加了一些方法：POST，HEAD。丰富了浏览器与服务器的互动手段。

3. 新增了HTTP首部。比较重要的有Content-Type ，Content-Length，Content-Encoding

4. HTTP请求和响应格式的改变。

5. 其他新增功能。比如状态码的定义。

请求报文：

```
请求行
请求头

请求体
```

响应报文：

```
响应行
响应头

响应体
```

**缺点：**

HTTP/1.0 版的主要缺点是，每个TCP连接只能发送一个请求。发送数据完毕，连接就关闭，如果还要请求其他资源，就必须再新建一个连接。

TCP连接的新建成本很高，因为需要客户端和服务器三次握手，并且开始时发送速率较慢（slow start）。所以，HTTP 1.0版本的性能比较差。随着网页加载的外部资源越来越多，这个问题就愈发突出了。

为了解决这个问题，在还没出HTTP/1.1规范时，各个公司可能会有一些自己的优化手段。有些浏览器在请求时，用了一个非标准的`Connection`字段。

```
Connection: keep-alive
Connection: close
```

Connection头部，用于决定当前的事务完成后，是否会关闭网络连接。

如果该值是“keep-alive”，网络连接就是持久的，不会关闭，使得对同一个服务器的请求可以继续在该连接上完成。

如果该值是“close”表明客户端或服务器想要关闭该网络连接。

像这种做的比较好的优化可能就会被放到下一次的RFC规范中去。Connection头部就被HTTP/1.1规范所采用。

HTTP/1.0默认是close，HTTP/1.1默认是keep-alive。

#### HTTP 1.1

1999年6月发布规范 rfc2616

进一步完善了 HTTP 协议，一直用到了20年后的今天，直到现在还是最流行的版本。

**相比HTTP/1.0：**

1. 引入了持久连接（persistent connection），即TCP连接默认不关闭，可以被多个请求复用，不用声明`Connection: keep-alive`。管线机制（Pipelining）

2. 新增了一些方法：OPTIONS，PUT，DELETE，TRACE，CONNECT。

3. 新增了一大波状态码。在HTTP1.1中新增了24个错误状态响应码。

4. 完善了缓存控制机制。在 HTTP 1.0 中主要使用 header 里的 If-Modified-Since,Expires 来做为缓存判断的标准，HTTP 1.1则引入了更多的缓存控制策略例如 Entity tag，If-Unmodified-Since, If-Match, If-None-Match等更多可供选择的缓存头来控制缓存策略。

5. 新增了一大波首部。比如跟缓存控制相关的就增加了很多，还有Content-Range， Host， Range等。

6. 其他新增功能。比如传输编码（Transfer Codings），特别是块传输编码，主要是针对那些没办法提前知道内容长度的实体。范围请求，新增MIME类型 `multipart/form-data` 等。

**缺点：**

虽然 HTTP/1.1 允许复用 TCP 连接，但是同一个 TCP 连接里面，所有的数据通信是按次序进行的。服务器只有处理完一个回应，才会进行下一个回应。要是前面的回应特别慢，后面就会有许多请求排队等着。这称为「队头堵塞」（Head-of-line blocking）。

#### HTTP 2

2015年5月发布规范 rfc7540





>  为什么要复用连接?

HTTP耗时 = TCP握手

HTTPS耗时 = TCP握手 + SSL握手

```
curl -w "TCP handshake: %{time_connect}, SSL handshake: %{time_appconnect}\n" -so /dev/null https://www.alipay.com
TCP handshake: 0.035926, SSL handshake: 0.138877
```

上面命令中的w参数表示指定输出格式，time_connect变量表示TCP握手的耗时，time_appconnect变量表示SSL握手的耗时（更多变量请查看[文档](http://curl.haxx.se/docs/manpage.html)和[实例](https://josephscott.org/archives/2011/10/timing-details-with-curl/)），s参数和o参数用来关闭标准输出。另外青花瓷抓包也可以看到各阶段的耗时。

可以看到如果不复用连接，那么每次都要3次TCP握手建立TCP连接，7次SSL握手建立SSL连接，而这些连接的建立都需要耗时，特别是建立SSL连接耗时更大，差不多是TCP的2-3倍。

除了建立连接耗时外，TCP的慢启动机制也会影响传输速度。

因此从HTTP1.1版本开始就支持连接的复用了，只不过HTTP1.1版本的连接复用还不成熟，毕竟技术都有一个不断优化的过程，从来都不是一蹴而就的。

[HTTP协议头部与Keep-Alive模式详解](https://www.iteye.com/blog/a280606790-1095085)

### 参考

[HTTP 各版本差异](https://lixiaoyu.cc/2018/09/04/computer-network-12-http-version-differences/)  3.5星，博主也还不错。

[HTTP 协议入门](https://www.ruanyifeng.com/blog/2016/08/http.html)   4星，阮一峰。

[解密 HTTP/2 与 HTTP/3 的新特性](https://www.infoq.cn/article/kU4OkqR8vH123a8dLCCJ)

[HTTP/2 相比 1.0 有哪些重大改进？](https://www.zhihu.com/question/34074946)

[深入研究：HTTP2 的真正性能到底如何](https://segmentfault.com/a/1190000007219256)

[一篇文章彻底了解HTTP发展史](https://cloud.tencent.com/developer/article/1513007)

[rfc1945--Hypertext Transfer Protocol -- HTTP/1.0](https://tools.ietf.org/html/rfc1945)

[rfc2616--Hypertext Transfer Protocol -- HTTP/1.1 ](https://tools.ietf.org/html/rfc2616)

[rfc7540--Hypertext Transfer Protocol Version 2 (HTTP/2)](https://tools.ietf.org/html/rfc7540)



