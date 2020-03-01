---
title: removeObject
date: 2017-11-12 20:57:24
description: 
categories: Swift
comments: false
---

## 给Swift Array类型扩展removeObject方法
由于Array是结构体类型,而removeObject函数会改变结构体的值,所以该函数需要添加mutating关键字.“Notice the use of the mutating keyword in the declaration of SimpleStructure to mark a method that modifies the structure. The declaration of SimpleClass doesn’t need any of its methods marked as mutating because methods on a class can always modify the class.”

```
extension Array where Element: Equatable {
    mutating func removeObject(object: Element) {
        if let index = index(of: object) {
            remove(at: index)
        }
    }
}
```