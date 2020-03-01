---
title: Swift实现removeObject
date: 2017-09-15 10:12:24
description: 
categories: Swift
tags:
---

## Swift实现removeObject
如果类没有继承自NSObject,那么类必须要实现`Equatable`协议.否则不需要.

```
class Observer {
    var name: String!
    var subject: Subject!
    func observer(name: String, subject: Subject) {
        self.name = name
        self.subject = subject
    }
    
    func update() {
        
    }
}

extension Observer: Equatable {
    static func ==(lhs: Observer, rhs: Observer) -> Bool {
        if lhs === rhs { //地址相等. ps:"=="值相等.
            return true
        } else {
            return false
        }
    }
}

```
然后在通过扩展给Array增加一个`removeObject(object: Element)`方法:

```
extension Array where Element: Equatable {
    mutating func removeObject(object: Element) {
        if let index = index(of: object) {
            remove(at: index)
        }
    }
}
```
由于Array是结构体类型,而removeObject函数会改变结构体的值,所以该函数需要添加mutating关键字.“Notice the use of the mutating keyword in the declaration of SimpleStructure to mark a method that modifies the structure. The declaration of SimpleClass doesn’t need any of its methods marked as mutating because methods on a class can always modify the class."