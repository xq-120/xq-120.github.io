---
title: Swift String基本操作
date: 2017-11-26 22:55:24
description: 
categories: Swift
tags:
---

## Swift String基本操作

```
//创建字符串  
let s1: String = "dsfdsf"

//遍历字符串
for char in s1 {
	print(char)
}

//字符串拼接
let s2 = "dasdfds"
let s3 = s1 + s2
print(s3)

//计算字符串长度
let length: Int = s3.count
print("长度:\(length)")

//判断一个字符串是否为空
let rs1 = s3.isEmpty
print("是否为空:\(rs1)")

//将一个字符串转换为大/小写
let s4: String = "fsdfdsfsd"
let s5 = s4.uppercased() //返回一个大写的字符串,原字符串不受影响
print("s4:\(s4)")
print("s5:\(s5)")

//字符串截取.字符串截取需要使用String.Index中介
let s7: String = "dasd3243232423sfsfdsf13"
//截取前4个字符
let s8: Substring = s7.prefix(4) //返回的是一个Substring类型,因此不能赋值给一个String类型的变量,如果要赋值给lable,需要转换为String类型
let label = UILabel.init()
label.text = String.init(s8) 

//String和Substring可以直接相加操作,编译器已经做了强制转换.
let s9: String = s7 + s8

//截取后5个字符
let s10: Substring = s7.suffix(5)
print("s10:\(s10)")

//从第四个(包含)开始截取,截取10个长度的字符
let offset4Index: String.Index = s7.index(s7.startIndex, offsetBy: 4)
let dest14Index = s7.index(offset4Index, offsetBy: 10) //注意:起点+长度不能越界.
let s11: Substring = s7[offset4Index..<dest14Index] //返回的是一个Substring类型,因此不能赋值给一个String类型的变量. "..<"表示包含左边,但不包含右边,"..."表示闭区间,两边都包含.
print("s11:\(s11)")


//移除最后一个字符.被操作的字符串不能为空字符串.
var s12: String = "sfdsfds"
s12.removeLast()
print("s12:\(s12)"

```