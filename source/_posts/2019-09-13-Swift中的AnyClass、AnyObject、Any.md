---
title: Swift中的AnyClass、AnyObject、Any
date: 2019-09-13 12:39:40
description: 
categories: Swift
tags: AnyClass、AnyObject、Any
comments: true
---

1.AnyObject
     
定义:
public typealias AnyObject
 
说明:
The protocol to which all classes implicitly conform.
 
AnyObject can be used as the concrete type for an instance of any class, class type, or class-only protocol.
 
The flexible behavior of the `AnyObject` protocol is similar to
Objective-C's `id` type.

AnyObject:用于表示任意类,元类,类协议的实例.
 
对于"123"或123这些基础数据类型在Swift中是结构体类型,所以这里需要将其强转为AnyObject类型,此时它们的类型将是
OC的NSTaggedPointerString和__NSCFNumber等对象类型.不强转的话编译会出错.

eg:
 
```
let anyA: AnyObject = People.init()
let anyB: AnyObject = People.self
let arr1: [AnyObject] = ["123" as AnyObject, 123 as AnyObject, [123] as AnyObject]
 
let arr2: [AnyObject] = [NSString("123"),
                             NSNumber(123),
                             true as AnyObject,
                             People.init(),
                             People.self
                             ]
print(arr2)

...
[123, 123, 1, <OSHybridDemo.People: 0x600002348b90>, OSHybridDemo.People]
```
 
2.AnyClass
 
定义:
typealias AnyClass = AnyObject.Type
 
说明:
The protocol to which all class types implicitly conform.
 
You can use the AnyClass protocol as the concrete type for an instance of any class.
 
AnyClass:用于表示任意元类的实例.
 
 ```
let anyB: AnyObject = People.self
let anyC: AnyClass = People.self
    
let obj1 = (anyB as! People.Type).init()
let obj2 = (anyC as! People.Type).init()
 ```
 
3.Any

Any:用于表示任意类型.

之所以出现Any主要是由于AnyObject只能表示类的实例.而 Swift 中所有的基本类型，包括 Array 和 Dictionary 这些传统意义上会是 class 的东西，统统都是 struct 类型，并不能由 AnyObject 来表示，于是 Apple 提出了一个更为特殊的 Any，除了 class 以外，它还可以表示包括 struct 和 enum 在内的所有类型。

```
let arr: [Any] = ["123", 123]
print(arr)

...
["123", 123]
```

三者之间的关系:

![](http://a3.qpic.cn/psb?/V1266oZg2CElNW/BzT6fN6Vr7t3wco9dk3GcbnrUQhnHYvdiJd4Jrs35DA!/b/dLYAAAAAAAAA&ek=1&kp=1&pt=0&bo=MAJWAgAAAAADN3Q!&tl=1&vuin=1185530890&tm=1568347200&sce=60-2-2&rf=viewer_4)
