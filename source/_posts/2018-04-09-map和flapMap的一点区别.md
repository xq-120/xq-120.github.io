---
title: map和flapMap的一点区别
date: 2018-04-09 11:24:37
description: 
categories: Swift
tags: map flapMap
comments: false
---

### map和flapMap的一点区别

map的声明:
`public func map<T>(_ transform: (Element) throws -> T) rethrows -> [T]`
它返回的是一个数组.闭包的返回值是一个新数组的元素.  
如果闭包返回的是一个optional类型元素则map返回的也将是一个[optional(T)]数组.

flapMap的声明:  
头文件里只看到这一个

```
@available(swift, deprecated: 4.1, renamed: "compactMap(_:)", message: "Please use compactMap(_:) for the case where closure returns an optional value")
public func flatMap(_ transform: (Element) throws -> String?) rethrows -> [String]
```
提示器会有三个:  
![flapMap声明](http://7xqoji.com1.z0.glb.clouddn.com/flapMap%E5%A3%B0%E6%98%8E@2x.png)
虽然下面两种已经被废弃,但现在还能使用.  
它返回的是一个数组.闭包的返回值可以是一个序列也可以是一个新数组的元素.

区别:  

1. flatMap可以用来展平一个数组:

	```
	let links: [[String]] = [
	    ["http://a1", "http://a2", "http://a3"],
	    ["http://b1", "http://b2", "http://b3"],
	    ["http://c1", "http://c2", "http://c3"],
	]
	print(links)
	
	let flapLink = links.flatMap { (exlinks: [String]) -> [String] in
	    return exlinks
	}
	print(flapLink)  //["http://a1", "http://a2", "http://a3", "http://b1", "http://b2", "http://b3", "http://c1", "http://c2", "http://c3"]
	```
	
2. flatMap可以将转化失败的nil元素剔除,而map则不能:

	```
	let wq = ["123", "fsd", "dfsd", "34"]
	let wqm = wq.map { (element) -> Int? in
	    return Int(element)
	}
	print(wqm)  //[Optional(123), nil, nil, Optional(34)]
	
	let wqfm = wq.flatMap { (element) -> Int? in
	    return Int(element)
	}
	print(wqfm) //[123, 34]
	```

由于最下面两种用法已经被废弃,Apple推荐使用compactMap,因此flatMap的用法主要是第一种了.