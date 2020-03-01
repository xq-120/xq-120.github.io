---
title: Allocation Type介绍
date: 2018-04-04 14:06:54
description: 
categories: Instruments使用
tags: Allocations
---

### Allocation Type介绍

#### 官方文档

The following allocation type display settings filter the allocations in the detail pane based on their type.

| Setting | Description |
----------|------------
|All Heap & Anonymous VM | Shows all malloc heap allocations and interesting VM regions such as graphics- and Core Data-related. Hides mapped files, dylibs, and some large reserved VM regions.|
|All Heap Allocations | Shows all malloc heap allocations but no VM regions.|
|All VM Regions | Shows all VM regions, including mapped files, dylibs, and the VM regions that include the malloc heap.|

参考:[Allocations Instrument](https://developer.apple.com/library/content/documentation/DeveloperTools/Conceptual/InstrumentsUserGuide/Instrument-Allocations.html)

#### 论坛上的解释
#### 1
All heap allocations consist of the memory allocations your application makes. This is the allocation type you should focus on in Instruments.
 
All VM regions consist of the virtual memory the operating system reserves for your application. You cannot control the size of the virtual memory the operating system reserves.
 
All heap and anonymous vm is the sum of the All Heap Allocations and All VM Regions columns.

#### 2

While heap allocations are very important (that's where all your Swift and Objective-C objects are created), you can't ignore the importance of VM memory for most apps. I recommend investigating that category because it can grow rather large, and the memory being allocated there "by the operating system" is going to be in response to some task you've asked your application to perform.
 
For instance, when you allocate a large CGImage, a tiny object is allocated on the heap, but the many megabytes of image data is allocated in a separate VM region. In an image-heavy application, your heap may be relatively small but your VM regions may be rather large. In that case, you'd need to take a closer look at the size and number of images your application has loaded at any given moment.
 
So in general, you need to ensure you are optimizing **all** of your memory usage.

参考:[[Instruments] Allocation Types](https://forums.developer.apple.com/thread/21463)

#### 小结
All Heap Allocations:应用在堆上申请的内存空间,比如你自己的类alloc出来的对象.

All VM Regions:操作系统为你的APP预留出来的虚拟内存.比如APP加载资源文件到内存时,占用的就是这些虚拟内存.因此如果你的APP图片比较多时,这部分内存占用会很大.

#### 例子
在一个页面上加载一张140kb的图片.使用Instruments查看实际消耗的内存:
![加载图片占用的内存](http://7xqoji.com1.z0.glb.clouddn.com/%E5%8A%A0%E8%BD%BD%E5%9B%BE%E7%89%87%E5%8D%A0%E7%94%A8%E7%9A%84%E5%86%85%E5%AD%98@2x.png)
不知道为啥加载到内存后会比磁盘上要大36kb.

加载了一张52kb的图片,Instruments显示消耗64kb.感觉要比原图多占用25%.

#### 其他
##### Mac文件存储单位
Apple认为1GB=1 000 000 000字节，Microsoft认为1GB=1 073 741 824字节，网络上多用Microsoft的标准。
对于电脑而言，1 073 741 824是2的30次方，比1 000 000 000更方便处理，但硬件厂商生产硬盘等设备的时候是按后者来标注容量的。Apple希望用户买一块1TB的硬盘插到电脑上看到的就是1TB而不是931GB，这会带来更好的用户体验，于是Apple在显示容量时遵循了后者的标准。
  
<img src="http://7xqoji.com1.z0.glb.clouddn.com/Mac%E6%96%87%E4%BB%B6%E5%AD%98%E5%82%A8%E5%8D%95%E4%BD%8D@2x.png" width="264" height="451" />