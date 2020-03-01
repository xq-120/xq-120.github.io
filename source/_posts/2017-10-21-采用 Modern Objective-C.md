---
title: 采用 Modern Objective-C
date: 2017-10-21 13:21:24
description: 
categories: OC编码规范
tags: 
comments: false
---

## 采用 Modern Objective-C
参考:[Adopting Modern Objective-C](https://developer.apple.com/library/content/releasenotes/ObjectiveC/ModernizationObjC/AdoptingModernObjective-C/AdoptingModernObjective-C.html)

These modernizations **improve type safety, memory management, performance, and other aspects of Objective-C**, making it easier for you to write correct code. It’s important to adopt these changes in your existing and future code to help it become more consistent, readable, and resilient.

### instancetype
Use the instancetype keyword as the return type of methods that return an instance of the class they are called on (or a subclass of that class).  
Using instancetype instead of id in appropriate places **improves type safety** in your Objective-C code.  
使用instancetype后,如果给对象发送一条不能识别的消息那么编译器就会提示错误,这样就可以提前解决错误,因此提高了类型安全.参考官方文档上的例子说明.

Note that you should replace id with instancetype for return values only, not elsewhere in your code. Unlike id, the instancetype keyword can be used only as the result type in a method declaration.

### `@property`语法
使用properties代替实例变量有许多好处.

* 自动合成getter和setter.
* 更好的声明一系列方法的意图.
* Property关键字可以传递额外的信息.比如assign (vs copy), weak, atomic (vs nonatomic), and so on.

Property methods follow a simple naming convention. The getter is the name of the property (for example, date), and the setter is the name of the property with the set prefix, written in camel case (for example, setDate). The naming convention for Boolean properties is to declare them with a named getter starting with the word “is”:
`@property (readonly, getter=isBlue) BOOL blue;`

### 枚举宏
使用NS_ENUM 和 NS_OPTIONS宏.when you use enum to define a bitmask use the NS_OPTIONS macro.

```
typedef NS_ENUM(NSInteger, UITableViewCellStyle) {

        UITableViewCellStyleDefault,

        UITableViewCellStyleValue1,

        UITableViewCellStyleValue2,

        UITableViewCellStyleSubtitle

};
```

```
typedef NS_OPTIONS(NSUInteger, UIViewAutoresizing) {

        UIViewAutoresizingNone                 = 0,

        UIViewAutoresizingFlexibleLeftMargin   = 1 << 0,

        UIViewAutoresizingFlexibleWidth        = 1 << 1,

        UIViewAutoresizingFlexibleRightMargin  = 1 << 2,

        UIViewAutoresizingFlexibleTopMargin    = 1 << 3,

        UIViewAutoresizingFlexibleHeight       = 1 << 4,

        UIViewAutoresizingFlexibleBottomMargin = 1 << 5

};
```