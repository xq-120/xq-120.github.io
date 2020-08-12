---
title: Swift中的Any AnyObject AnyClass
tags:
  - Any
categories:
  - Swift
comments: true
date: 2020-08-11 09:55:03
---



Any: 可以表示任意类型，甚至方法类型（func）

AnyObject: 表示任何class类型的实例对象（类似OC中的id类型）

AnyClass：表示任意类的元类型.任意类的类型都隐式遵守这个协议.  AnyObject.Type中的.Type就是获取元类型, 辟如你有一个Student类, Student.Type就是获取Student的元类型.

在 Swift 中所有的基本类型，包括 Array 和 Dictionary 这些传统意义上会是 class 的东西，统统都是 struct 类型，并不能由 AnyObject 来表示。`let a: AnyObject = Int.init(2.0)` 会编译错误，需要强转为AnyObject，此时a的实际类型将不再是Int而是`__NSCFNumber` ，这表明系统会将这些struct转为对应的OC类。

```swift
let a: AnyObject = Int.init(2.0) as AnyObject
print("a type:\(type(of: a))") //a type:__NSCFNumber
let b = 3
let c = a as! Int + b
```

在使用AnyObject 对象时要特别注意如果你需要调用它的属性则最好先downcast为实际的类型，如果不downcast那么系统可能获取的是其他类的同名属性，得到的值将是nil。

```swift
let node1 = BinaryTreeNode.init(1)
let node2 = BinaryTreeNode.init(2)
node1.left = node2
let curr: AnyObject = node1
let left: AnyObject? = curr.left  //不会报错
let left1: AnyObject? = (curr as! BinaryTreeNode).left
print("left:\(left), left1:\(left1!)") //left:nil, left1:AlgorithmUtils.BinaryTreeNode
```



