---
title: NSData与它的属性bytes
date: 2018-07-07 17:39:16
description: 
categories: NSData
tags: 
comments:
---

#### NSData与它的属性bytes

```
NSString* enString= @"abc";
NSData* utf8EnData = [enString dataUsingEncoding:NSUTF8StringEncoding];

```

bytes属性:

```
/*
 The -bytes method returns a pointer to a contiguous region of memory managed by the receiver.
 If the regions of memory represented by the receiver are already contiguous, it does so in O(1) time, otherwise it may take longer
 Using -enumerateByteRangesUsingBlock: will be efficient for both contiguous and discontiguous data.
 */
@property (readonly) const void *bytes NS_RETURNS_INNER_POINTER;
```

bytes属性指向的是NSData对象装载的内容.NSData装载的二进制内容在内存中的分布可能是连续的一片,也可能是不连续的.使用`- (void) enumerateByteRangesUsingBlock:(void (NS_NOESCAPE ^)(const void *bytes, NSRange byteRange, BOOL *stop))block`遍历所有的分布区域.



NSData对象本身的地址与装载的二进制内容的地址,如下图:
![](http://7xqoji.com1.z0.glb.clouddn.com/NSData%E4%B8%8Ebytes%E5%B1%9E%E6%80%A7%E7%9A%84%E5%85%B3%E7%B3%BB@2x.png)
可以看到NSData对象本身的地址与二进制内容所在的地址是不同的.二者内存位置相距还挺远的.

查看NSData对象装载的二进制内容方法:
![](http://7xqoji.com1.z0.glb.clouddn.com/%E6%9F%A5%E7%9C%8BNSData%E7%9A%84%E5%86%85%E5%AD%98%E6%95%B0%E6%8D%AE%E6%96%B9%E6%B3%95@2x.png)