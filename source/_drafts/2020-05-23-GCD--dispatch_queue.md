---
title: iOS GCD
tags:
  - 标签1
  - 标签2
  - 标签3
categories:
  - 分类1
  - 分类2
comments: true
date: 2020-05-23 15:23:33
---

GCD 501版本的还不算复杂，但我就是不想看旧版本的于是入坑最新版本`libdispatch-1173.40.5`。

_dispatch_async_redirect_invoke

调用_dispatch_continuation_pop

调用_dispatch_continuation_invoke_inline

调用`_dispatch_continuation_with_group_invoke`或`_dispatch_client_callout`

#### 参考

[Concurrency Programming Guide](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html)  官方并发编程指南

[线程安全类的设计](https://objccn.io/issue-2-4/)  王巍译

[我所理解的 iOS 并发编程](https://blog.boolchow.com/2018/04/06/iOS-Concurrency-Programming/)  很全

[深入理解 GCD--转载](https://www.cnblogs.com/LiLihongqiang/p/7599390.html)  

[深入理解GCD](https://juejin.im/post/57cb8e8367f3560057bf4da1) 很不错

[GCD源码吐血分析(1)——GCD Queue](https://blog.csdn.net/u013378438/article/details/81031938)  GCD版本变化太大了，新版本里面的宏简直到了令人发指的地步了。作者的博客还可以，是成系列的。

[GCD源码解析(一)-dispatch_queue_create、dispatch_get_main_queue、dispatch_get_global_queue](https://www.jianshu.com/p/7702c06cda4c)  分析了队列的创建

[深入浅出 GCD 之 dispatch_group](https://xiaozhuanlan.com/topic/0863519247)  比较旧的版本的

