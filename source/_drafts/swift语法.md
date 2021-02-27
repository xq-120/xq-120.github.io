#associatedtype

用于对协议中使用的类型进行约束。一点点类似于OC中的 `id<UITableViewDataSource>` 。

# 协议中的where Self

示例：

```swift
/// Represents the value that identified for differentiate.
public protocol ContentIdentifiable {
    /// A type representing the identifier.
    associatedtype DifferenceIdentifier: Hashable

    /// An identifier value for difference calculation.
    var differenceIdentifier: DifferenceIdentifier { get }
}

public extension ContentIdentifiable where Self: Hashable {
    /// The `self` value as an identifier for difference calculation.
    @inlinable
    var differenceIdentifier: Self {
        return self
    }
}
```

这里的Self表示的是实现了该协议的类或子类。因为在使用协议的时候，并不能知道最后是哪个类实现了该协议。但在定义协议的时候我们又需要知道这个类，或者需要对这个类做一些限制。所以用Self作为一个占位符。

这里的 `where Self: Hashable` 表示的是ContentIdentifiable扩展只在实现了Hashable协议的类中才可见。

又比如：

```swift
/// Represents a value that can compare whether the content are equal.
public protocol ContentEquatable {
    /// Indicate whether the content of `self` is equals to the content of
    /// the given source value.
    ///
    /// - Parameters:
    ///   - source: A source value to be compared.
    ///
    /// - Returns: A Boolean value indicating whether the content of `self` is equals
    ///            to the content of the given source value.
    func isContentEqual(to source: Self) -> Bool
}
```

参考：

[what is 'where self' in protocol extension](https://stackoverflow.com/questions/46018677/what-is-where-self-in-protocol-extension)

[swift 协议和类方法中的 Self](https://www.jianshu.com/p/d10c6b64ab30)

# 命名空间

在 Swift 中的命名空间是基于一个 Module 的,也就是一个 Module 是一个命名空间.这也就是为什么现在的 Snapkit 使用的时候会`view.snp.xxxx`,Kingfisher 使用的时候要用`imageView.kf.xxxx`.

但是在我们自己平时在写一些工具类或者 Foundation框架的类的扩展的时候,有可能会有一些方法与系统或者别人写的方法重名,这就很尴尬了!

比如：A，B，C三个库都对UIViewController进行了扩展，而且刚好扩展里的方法重名了，在我们的工程该如何调到合适的方法？

又比如：A，B两个库里都定义了Person类。在我们的工程里如何选择合适的类进行继承？

```
open class Person: NSObject {
   public var name: String = ""
}

open class Person: NSObject {
   public var age: String = ""
}
```

参考：

[Swift工具类命名空间隔离](https://www.jianshu.com/p/bb5e6305f8ca)



Q1:父类实现了某个协议，子类能不能也实现它？