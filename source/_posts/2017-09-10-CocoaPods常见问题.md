---
title: CocoaPods常见问题
date: 2017-09-10 16:07:12
description: 
categories: CocoaPods
tags: CocoaPods
---

Q:在用 Cocoapods 做第三方开源库管理的时候，有时候发现查找到的第三方库版本低于github上仓库的最新release版本.  
A:执行`pod update`更新本地仓库，完成后，即可搜索到指定的第三方库.  

Q:在使用了`pod setup`之后，发现好长时间都没有变化，无法从终端上获取`pod setup`的执行情况.  
A:这时候可以command+N新建一个窗口，通过`sudo ls`用管理员权限查看目录,然后`cd .cocoapods`文件夹，输入`du -sh`命令查看文件夹大小变化，从而确定`pod setup`的运行情况.

note:在Podfile文件中的`platform :ios, '7.0'`,如果你设置的iOS系统过低,可能有的第三方库会下载不下来.所以还是要看一下第三方库的最低版本支持说明.

Q:RuntimeError - [Xcodeproj] Unknown object version.然后是一大堆错误日志.不管是重新从svn checkout一份新的还是怎样,pod install都不能成功.  
A:一种解决办法:重新安装cocoapods.