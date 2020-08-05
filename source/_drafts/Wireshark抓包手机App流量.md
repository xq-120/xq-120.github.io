---
title: Wireshark抓包手机App流量
tags:
  - 标签1
  - 标签2
  - 标签3
categories:
  - 分类1
  - 分类2
comments: true
date: 2020-08-05 11:05:21
---



### Wireshark抓包手机App流量

要想通过Wireshark抓包手机上的App流量，前提条件就是手机使用的流量必须要经过Wireshark。如果抓不到你就得反思一下哪个环节出问题导致流量没有经过Wireshark，一般情况下抓包软件是没有问题的。

我们可以先测试一下抓包电脑上流量，验证Wireshark是没有问题的，打开Wireshark，选好接口双击就可以抓包了，这个时候可以打开浏览器随便打开个网页(最好是HTTP的，不要HTTPS的因为不是很直观，不过现在基本上都是HTTPS了有点难搞)，这时Wireshark应该可以抓到很多协议的流量比如TCP，HTTP等等。

接下来就是抓包手机App流量了，首先要解决的问题就是如何让手机上的流量经过Wireshark，因为Wireshark是装在电脑上的，所以我们必须要把电脑的网络共享给我们的手机，手机连上后使用的其实是电脑的流量，所以Wireshark才能监控到。

问题变为如何把电脑的网络共享给我们的手机。

电脑上网方式的不同，设置也会不同。接下来以电脑使用WIFI上网为例共享电脑的WiFi给我们的手机使用。

打开电脑的”系统偏好设置“-->"共享"，设置如下：

<img src="https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/3851CD634D0246967E6E3D22DEC203C5.jpg" style="zoom:50%;" />

上图表示：你要共享什么，共享的东西以什么样的方式提供给其他设备使用。

1. 这里电脑将共享自己的WIFI网络
2. 这里电脑将以蓝牙的方式将网络共享给其他设备

其他设备通过蓝牙连接电脑蓝牙即可上网。如果你的电脑是通过网线上网的，那么你还可以通过WIFI的方式将网络共享出去，方式可以有很多。

手机连上后，可以关掉手机自身的WIFI和蜂窝，验证是否共享成功。这个时候应该是可以上网的。

Wireshark接口选择Bluetooth PAN就可以抓包了。

<img src="https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/CFEA70C1E2A1D4C65779DBB850E532FB.jpg" style="zoom:50%;" />

ps：不知道为什么Mac电脑通过蓝牙共享出去的网络非常慢，不知道是我自身网络的原因还是其他原因。导致抓包软件没有反应还以为是抓包软件有问题，原来是网很慢没有流量经过。建议通过以iPhone USB或者WIFI的方式共享给其他设备，速度稍微要快一点。如果是以iPhone USB的方式共享的网络，手机可以关闭自身的WIFI和蜂窝，蓝牙等，验证是否共享成功，如果能够上网则表明成功，抓包的时候Wireshark接口需要重新选择，具体是哪一个不好确定，挨个试一下。

效果：

这里抓了一个图片下载的HTTP请求。

<img src="https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20200805174709.png" style="zoom:50%;" />

前三个包可以看到是TCP的三次握手。握手成功后客户端开始发起HTTP请求。

### 参考

[【技术流】以TCP/IP协议为例，如何通过wireshark抓包分析？](https://zhuanlan.zhihu.com/p/36414915)

[【技术流】Wireshark对HTTPS数据的解密](https://zhuanlan.zhihu.com/p/36669377)

[你听说过 Wireshark 抓包么？](https://zhuanlan.zhihu.com/p/143804751)

[【腾讯TMQ】从 wireshark 抓包开始学习 https](https://cloud.tencent.com/developer/article/1004578)

