---
title: 使用reduce模拟实现map和filter
date: 2018-01-26 17:06:24
description: 
categories: Swift
tags:
---

### 使用reduce模拟实现map和filter

首先看一下系统的map和filter功能:

```
let fbArr = [1, 1, 2, 3, 5, 8, 13, 21]

let mmp = fbArr.map { (item) -> Int in
    item + item
}
print(mmp)
    
let cmp = fbArr.filter { (item) -> Bool in
    item > 5
}
print(cmp)
```
打印如下:

```
[2, 2, 4, 6, 10, 16, 26, 42]
[8, 13, 21]
```
使用reduce模拟实现map和filter:

```
extension Array {
	func map2<T>(_ transform: (Element) -> T) -> [T] {
        //初始值为空数组, 数组+数组
        return reduce([], { (result, item) -> [T] in
            return result + [transform(item)]
        })
        //简化一下
        return reduce([], {
            $0 + [transform($1)]
        })
        //再简化一下
        return reduce([]) { $0 + [transform($1)] }
    }
    
    func filter2(_ filter: (Element) -> Bool) -> [Element] {
        return reduce([], { (result, item) -> [Element] in
            return filter(item) ? result + [item] : result
        })
        //简化一下
        return reduce([]) { filter($1) ? $0 + [$1] : $0 }
    }
}
```
说实话,Swift对闭包的简写有时候感觉有点过了,初次看到这样的写法`reduce([]) { filter($1) ? $0 + [$1] : $0 }`,有时还得缓缓才知道是啥意思.

使用:

```
let ammp = fbArr.map2 { (item) -> Int in
    item + item
}
print(ammp)
    
let bm = fbArr.filter2 { (item) -> Bool in
    item > 10
}
print(bm)
```
打印如下:

```
[2, 2, 4, 6, 10, 16, 26, 42]
[8, 13, 21]
```

虽然仅使用reduce就可以做到map和filter的功能,但是随着数组的增长,复杂度会很大.