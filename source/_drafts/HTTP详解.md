### HTTP各版本特性

#### HTTP 1.0



#### HTTP 1.1



#### HTTP 2.0



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

[rfc2616--Hypertext Transfer Protocol -- HTTP/1.1 ](https://tools.ietf.org/html/rfc2616)

[rfc1945--Hypertext Transfer Protocol -- HTTP/1.0](https://tools.ietf.org/html/rfc1945)

[一篇文章彻底了解HTTP发展史](https://cloud.tencent.com/developer/article/1513007)