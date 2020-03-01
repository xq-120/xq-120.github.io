---
title: Swift addTarget函数的Selector参数
date: 2017-09-15 10:12:24
description: 
categories: Swift
tags:
---

Swift里addTarget函数里的Selector参数和OC的不太一样.
`open func addTarget(_ target: Any?, action: Selector, for controlEvents: UIControlEvents)`
根据Selector的定义

```
public struct Selector : ExpressibleByStringLiteral {

    /// Create a selector from a string.
    public init(_ str: String)

    /// Create an instance initialized to `value`.
    public init(unicodeScalarLiteral value: String)

    /// Construct a selector from `value`.
    public init(extendedGraphemeClusterLiteral value: String)

    /// Create an instance initialized to `value`.
    public init(stringLiteral value: String)
}
```

它是一个结构体,并遵守`ExpressibleByStringLiteral`协议,因此可以直接传递一个字面值字符串.  
不过这个字符串需要遵循一定的规则:

1. 如果函数没有参数,则字符串的格式为"函数名".
2. 如果函数有参数但省略了参数标签,则字符串的格式为"函数名:".
3. 如果函数有参数并且没有省略参数标签,则字符串的格式为"函数名With参数标签:".ps:如果函数使用的是默认参数标签即使用参数名称作为参数标签的,那么字符串的格式为"函数名With参数名称:"

举个例子:  

1. 没有参数
```
    func numberSliderChanged() -> Void {
    }
```
对应写法:`numberSlider.addTarget(self, action: "numberSliderChanged", for: .valueChanged)`
2. 省略参数标签
```
    func numberSliderChanged(_ sender: UISlider) -> Void {
        print("---", sender.value)
    }
```
对应写法:`numberSlider.addTarget(self, action: "numberSliderChanged:", for: .valueChanged)`
3. 写明了参数标签slider.
```
    func numberSliderChanged(slider sender: UISlider) -> Void {
        print(sender.value)
    }
```
对应写法:`numberSlider.addTarget(self, action: "numberSliderChangedWithSlider:", for: .valueChanged)`
4. 使用默认参数标签
  ```
    func numberSliderChanged(sender: UISlider) -> Void {
        print("sds", sender.value)
    }
```
对应写法:`numberSlider.addTarget(self, action: "numberSliderChangedWithSender:", for: .valueChanged)`

然而这种写法已经废弃,目前swift3.1推荐的是使用`#selector`,因此上述4写法等价于:
`numberSlider.addTarget(self, action: #selector(numberSliderChanged(sender:)), for: .valueChanged)`.
当然你也可以传递一个Selector类型的变量:
`numberSlider.addTarget(self, action: Selector.init(stringLiteral: "numberSliderChangedWithSender:"), for: .valueChanged)`.不过这种写法依然不是推荐的写法.