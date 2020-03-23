---
title: pod install慢解决
date: 2019-11-06 00:22:59
description: 
categories: CocoaPods
tags: 
---

今天在安装WCDB时,不管是pod install

```
-> Installing WCDB (1.0.6)
 > Git download
 > Git download
     $ /usr/local/bin/git clone https://github.com/Tencent/wcdb.git
     /var/folders/5z/1pxqzfcn77s2n7z4gmr63sdr0000gn/T/d20191216-6636-ji8u78 --template= --single-branch --depth 1 --branch v1.0.6
     Cloning into '/var/folders/5z/1pxqzfcn77s2n7z4gmr63sdr0000gn/T/d20191216-6636-ji8u78'...
     error: RPC failed; curl 18 transfer closed with outstanding read data remaining
     fatal: The remote end hung up unexpectedly
     fatal: early EOF
     fatal: index-pack failed

[!] Error installing WCDB
[!] /usr/local/bin/git clone https://github.com/Tencent/wcdb.git /var/folders/5z/1pxqzfcn77s2n7z4gmr63sdr0000gn/T/d20191216-6636-ji8u78 --template= --single-branch --depth 1 --branch v1.0.6

Cloning into '/var/folders/5z/1pxqzfcn77s2n7z4gmr63sdr0000gn/T/d20191216-6636-ji8u78'...
error: RPC failed; curl 18 transfer closed with outstanding read data remaining
fatal: The remote end hung up unexpectedly
fatal: early EOF
fatal: index-pack failed
```
还是直接git clone,都下载不了.

```
git clone https://github.com/Tencent/wcdb.git
Cloning into 'wcdb'...
remote: Enumerating objects: 1704, done.
remote: Counting objects: 100% (1704/1704), done.
remote: Compressing objects: 100% (1192/1192), done.
error: RPC failed; curl 56 LibreSSL SSL_read: SSL_ERROR_SYSCALL, errno 54
fatal: The remote end hung up unexpectedly
fatal: early EOF
fatal: index-pack failed
```
从错误原因不难看出是网络的问题，应该是github被墙了。

最后解决办法:

1. `pod repo update`更新仓库。

   这一步不是必须，除非你的仓库很旧，或者第二步没效果。这一步不翻墙的话同样很慢大概10kb/s的样子，但是比起后面下载源码库尤其是一些必须要翻墙才能下载的（比如libwebp），那简直小巫见大巫了。`pod repo update`会从github上把最新的仓库更新到本地：`~/.cocoapods/repos/master`，里面保存了所有第三方库的spec说明书。

2. 从同事那里拷贝源码库(路径为`~/Library/Caches/CocoaPods/Pods/Release`)和源码库的说明书(路径为`~/Library/Caches/CocoaPods/Pods/Specs/Release`),也分别放在自己的上述两个目录下.再pod install解决.

从这里可以看出pod install都干了些啥。

参考:[无法Pod install第三方库的解决办法](https://www.jianshu.com/p/e2947189c004)

ps:
pod install下载下来的库的本地存储路径为
`~/Library/Caches/CocoaPods`,该目录包含一个搜索索引json文件,以及下载过的库的源代码和对应的Spec.
如果Podfile文件中库的版本一致则不会再从github下载,而是直接使用缓存下来的库的源码.因此如果某个库pod install很慢时,可以拷贝别人的版本一致的库到该目录下,减少等待时间.
在清理Mac磁盘存储空间时最好不要删除`~/Library/Caches/CocoaPods`里面的缓存,否则下次pod install会从github上重新下载源码库,这个等待时间你懂得.

#### 让git clone走代理

慢的根本原因:github.global.ssl.fastly.net域名被限制了,导致git clone从上面拉源码库的时候非常慢,呵呵.
解决办法:

```
git config --global http.https://github.com.proxy https://127.0.0.1:xxxxx
git config --global https.https://github.com.proxy https://127.0.0.1:xxxxx
```
或者

```
git config --global http.https://github.com.proxy socks5://127.0.0.1:xxxxx
git config --global https.https://github.com.proxy socks5://127.0.0.1:xxxxx
```
`xxxxx`为你的HTTP(S)代理服务器的端口.为什么有两种设置,不知道.反正都试一下.

对应的取消设置为:

```
git config --global --unset http.https://github.com.proxy
git config --global --unset https.https://github.com.proxy
```
这个时候git clone可以达到500kb.

ps:

Git 自带一个 git config 的工具来帮助设置控制 Git 外观和行为的配置变量。 这些变量存储在三个不同的位置：

1. /etc/gitconfig 文件: 包含系统上每一个用户及他们仓库的通用配置。 如果使用带有 --system 选项的 git config 时，它会从此文件读写配置变量。

2. ~/.gitconfig 或 ~/.config/git/config 文件：只针对当前用户。 可以传递 --global 选项让 Git 读写此文件。

3. 当前使用仓库的 Git 目录中的 config 文件（就是 .git/config）：针对该仓库。

每一个级别覆盖上一级别的配置，所以 .git/config 的配置变量会覆盖 /etc/gitconfig 中的配置变量。

参考:

[1.6 起步 - 初次运行 Git 前的配置](https://git-scm.com/book/zh/v2/%E8%B5%B7%E6%AD%A5-%E5%88%9D%E6%AC%A1%E8%BF%90%E8%A1%8C-Git-%E5%89%8D%E7%9A%84%E9%85%8D%E7%BD%AE)

[8.1 自定义 Git - 配置 Git](https://git-scm.com/book/zh/v2/%E8%87%AA%E5%AE%9A%E4%B9%89-Git-%E9%85%8D%E7%BD%AE-Git)

#### 让Mac终端走代理

`vim ~/.zshrc`打开`.zshrc`文件.(我这里用的是zsh)
添加

```
alias proxy='export all_proxy=socks5://127.0.0.1:xxxxx'
alias unproxy='unset all_proxy'
```

`proxy`开启代理.对应的`unproxy`是关闭.  
执行`curl cip.cc`或`curl  https://www.dropbox.com`检测是否翻墙成功.
终端没走代理前git clone都是失败的,走代理后git clone可以工作但只有5kb左右.因此还是需要单独配置上面的git clone.

参考: [MaxOSX终端走shadowsocks代理](https://zhuanlan.zhihu.com/p/47849525).  
我自己试了一下能用,但晚上11点的速度也不是很快,可能不是独享网络的原因吧.早上6点又试了一下很快. 

另:CocoaPods升级到1.8.0+后,版本变化导致的一些问题.  
参考:[脱离CocoaPods 1.8.0的trunk](https://zhaoxin.pro/15695124897584.html)  
在国内还是暂时不要使用trunk,它会让你变得更慢而且还会失败.由于CocoaPods在1.8.0后默认使用trunk,我们可以移除trunk,继续使用master,但此时需要在Podfile文件中指定源
`source 'https://github.com/CocoaPods/Specs.git'`.