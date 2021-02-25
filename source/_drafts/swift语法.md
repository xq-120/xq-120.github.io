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

