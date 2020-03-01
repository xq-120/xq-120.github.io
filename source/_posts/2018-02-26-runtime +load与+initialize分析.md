---
title: runtime +load与+initialize分析
date: 2018-02-26 11:31:24
description: 
categories: runtime
tags: +load
---

通过一个简单的打印实验就可以发现:

* +initialize方法和+load调用时机
* +initialize在父类,子类,类别之间的关系
* +load在父类,子类,类别之间的关系

如下表格:

|       |+load  |+initialize |
|------ |------ |----------- |
|调用时机 | 被添加到 runtime 时(类被加载时) |第一次调用该类的类方法或实例方法前调用,可能永远不调用
|调用顺序| 父类->子类->分类(多个分类间,则按加载顺序) |父类->分类(多个分类时,是最后一个的被执行)or子类
|是否需要显式调用父类实现| 否| 否
|自身没实现是否调用父类的实现| 否 |是
|分类中实现产生的影响 |类和分类都执行 |覆盖类中的方法，只执行分类的实现

另:Swizzling should always be done in +load.

## 参考:
[Objective-C 深入理解 +load 和 +initialize](https://www.jianshu.com/p/872447c6dc3f)