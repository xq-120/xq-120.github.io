---
title: VPS shadowsocks搭建
date: 2018-11-25 22:19:02
description: 
categories: vps
---

#### VPS shadowsocks搭建
本文主要是方便自己以后查找,如有侵权立删.

主要参考:
[科学上网的终极姿势-在-vultr-vps-上搭建-shadowsocks](https://medium.com/@zoomyale/%E7%A7%91%E5%AD%A6%E4%B8%8A%E7%BD%91%E7%9A%84%E7%BB%88%E6%9E%81%E5%A7%BF%E5%8A%BF-%E5%9C%A8-vultr-vps-%E4%B8%8A%E6%90%AD%E5%BB%BA-shadowsocks-fd57c807d97e)

### 一、部署 Shadowsocks
Shadowsocks 需要同时具备客户端和服务器端，所以它的部署也需要分两步。

#### 1.1部署 Shadowsocks 服务器端

#### 1.2 TCP Fast Open

为了更好的连接速度，需要多做几步。首先是打开 TCP Fast Open.

```
{
    "server":"0.0.0.0",
    "local_address":"127.0.0.1",
    "local_port":1080,
    "port_password":{
        "1234":"yourpassword",
        "2234":"yourpassword",
        "3234":"yourpassword"
    },
    "timeout":300,
    "method":"aes-256-gcm",
    "fast_open":true
}
```

#### 1.3安装 BBR
TCP BBR 是 Google 于 2016 年发布的，一种避免网络拥塞的算法。目的是要尽量跑满带宽, 并且尽可能避免排队的情况。

实际使用下来，BBR 的速度提升效果并不比「锐速」（据说已倒闭）、kcptun（多倍发包耗流量）等差。而且 BBR 已植入 Linux 4.9+ 版本的内核中，服务器开启后就能有显著的效果提升。

#### 二、安装期间的问题
主要是CentOS 7的防火墙,CentOS 7使用的是firewall防火墙,而博文里使用的是iptables防火墙.这在添加端口的方式有所不同.

#### 附:CentOS 7 firewall 命令

查看firewall的状态`firewall-cmd --state` (关闭后显示notrunning，开启后显示running）

查看已经开放的端口`firewall-cmd --list-ports`,更详细的检查防火墙规则`firewall-cmd --list-all`

开启端口`firewall-cmd --zone=public --add-port=80/tcp --permanent`

命令含义：

–zone #作用域

–add-port=80/tcp #添加端口，格式为：端口/通讯协议

–permanent #永久生效，没有此参数重启后失效

reload防火墙`firewall-cmd --reload #重启firewall`

对应的文件路径在`/etc/firewalld/zones/public.xml`

参考:
[CentOS 7中firewall防火墙详解和配置以及切换为iptables防火墙](https://blog.csdn.net/xlgen157387/article/details/52672988)