---
title: Swift Array基本操作
date: 2017-12-01 23:17:24
description: 
categories: Swift
tags:
---

## Swift Array基本操作
Array使用了copy-on-write技术.Array是结构体类型,因此是值类型.和OC里的NSArray引用类型是不同的.

```
//定义一个数组
let arr1: Array<String> = ["213", "fdsf", "sdfdsg"]  //let定义的是常量,所以arr1不能再进行append操作.

//追加元素
//追加一个元素
var arr2: Array<String> = ["llllfs", "dqafas", "21231"]
arr2 += ["sfsdfsd"]
print("arr2:\(arr2)")
var arr3: Array<String> = arr2
arr3.append("hhehehe")  //append()会改变自身的值.但arr2的值并不会改变,和OC不一样.
print("arr2:\(arr2)")
print("arr3:\(arr3)")

//追加一个数组的元素
arr3.append(contentsOf: ["sfsdf", "dfsfds", "dgfdg"])
print("arr3:\(arr3)")
var arr4: Array<People> = [People]()
for index: UInt in 0..<4 {
    let p = People.init(name: "渣渣辉\(index)", sex: true, age: index * UInt(arc4random()%20))
    arr4.append(p)
}
print("arr4:\(arr4)")

//移除元素
var arr5: Array = arr4
//移除最后一个元素
arr5.removeLast() //改变自身.被操作的数组不能为空.
print("arr5:\(arr5)")

//移除元素,原数组不受影响,得到一个类型为ArraySlice的新数组.
var arr6: Array = Array.init(arr5.dropLast()) //ArraySlice转Array
print("arr5:\(arr5)")
print("arr6:\(arr6)")
var arr7: ArraySlice = ArraySlice.init(arr4)  //Array转ArraySlice

//移除某个范围里的元素
var arr8: Array = [1, 2, 3, 4, 5, 6]
arr8.removeSubrange(1...) //移除从第1个位置处开始一直到末尾的元素
print("arr8:\(arr8)")
var arr9: Array = [1]
arr9.removeSubrange(1...) //如果数组只有一个元素,但移除从第一个位置处的元素并不会越界崩溃.但是移除从第二个位置处的元素将会导致越界崩溃
print("arr9:\(arr9)") //arr9:[1]
var arr10: Array = [1, 2, 3, 4, 5, 6]
arr10.removeSubrange(1...3) //移除从第1个位置处开始随后的3个元素
print("arr10:\(arr10)")
var arr11: Array = [1, 2, 3, 4, 5, 6]
arr11.removeSubrange(1..<3) //移除从第1个位置处开始随后的2个元素
print("arr11:\(arr11)")
//便捷操作
public mutating func removeLast(_ n: Int) //移除后几个
public mutating func removeFirst(_ n: Int) //移除前几个
public mutating func remove(at index: Int) -> Array.Element //移除指定某个位置的元素
public mutating func removeAll(keepingCapacity keepCapacity: Bool = default) //移除所有元素

let tanwanlanyue = ["渣渣辉", "纯扰春", "咕天落", "森红雷"]
//“迭代除了第一个元素以外的数组其余部分”
for item in tanwanlanyue.dropFirst() {
    print(item)
}
//“迭代除了最后 1 个元素以外的数组”
for item in tanwanlanyue.dropLast() {
    print(item)
}
//“遍历数组中的元素和对应的下标”
for (index, element) in tanwanlanyue.enumerated() {
    print(index, element)
}
//“寻找一个指定元素的位置”
let idx = tanwanlanyue.index(of: "咕天落")
let i: Int = idx! + 1
print(i)
//使用Where闭包
let index = tanwanlanyue.index {
    $0 == "纯扰春"
}
let j: Int = index! + 3
print(j)
if let sidx = tanwanlanyue.index(where: { (item) -> Bool in
    item.hasSuffix("落")
}) {
    print(sidx)
}
let didx = tanwanlanyue.index(where: { (element) -> Bool in
    return element.hasPrefix("渣渣")
})
print(didx)
//“对数组中的所有元素进行变形,得到一个新数组”
let tan2 = tanwanlanyue.map { (item) -> String in
    let idx = tanwanlanyue.index(of: item)
    return item + "\(idx!)"
}
print(tan2)
//“筛选出符合某个标准的元素,得到一个新数组”
let tan3 = tan2.filter { (item) -> Bool in
    let ch = item.last!
    return ch > "1"
}
print(tan3)
    
let names = ["Paula", "Elena", "Zoe"]
var lastNameEndingInA: String?
//(for in) + where条件,仅遍历符合条件的元素
for name in names.reversed() where name.hasSuffix("a") {
    lastNameEndingInA = name
    break
}
print(lastNameEndingInA)
for name in names.reversed() where name.hasSuffix("a") {
    print(name)
}
    
let last = names.last { $0.hasSuffix("a")
}
print(last)

//插入
略.

//替换
略.

//翻转
略.

//遍历
let names = ["Paula", "Elena", "Zoe"]
var lastNameEndingInA: String?
//(for in) + where条件:仅遍历符合条件的元素
for name in names.reversed() where name.hasSuffix("a") {
    lastNameEndingInA = name
    break
}
print(lastNameEndingInA)
for name in names.reversed() where name.hasSuffix("a") {
    print(name)
}
```
Swfit提供了三种类型的数组:

* public struct ContiguousArray<Element> : RandomAccessCollection, MutableCollection

* public struct ArraySlice<Element> : RandomAccessCollection, MutableCollection

* public struct Array<Element> : RandomAccessCollection, MutableCollection