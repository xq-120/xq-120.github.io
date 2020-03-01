---
title: 从JSONSerialization崩溃到搞懂Swift的try-catch
date: 2019-03-03 13:21:39
description: 
categories: Swift
tags: 错误处理
---

今天学习了下Swift中的try-catch,突然发现在json序列化时try-catch不好使了,下面的代码依然会导致崩溃,try-catch并不能catch住.

```
import Foundation

class People: NSObject, Decodable {
    @objc var firstName: String
    @objc var lastName: String
    @objc var age: Int = 0
    @objc var height: Float = 0
    @objc var weight: Float = 0
    @objc var sex: Bool = false
    
    @objc init(firstName: String, lastName: String) {
        self.firstName = firstName
        self.lastName = lastName
        super.init()
    }
}

let dict = ["one": People.init(firstName: "x", lastName: "q")]
do {
    let jsonData = try JSONSerialization.data(withJSONObject: dict, options: [])
    print(jsonData)
} catch let error {
    print(error)
}
```


看了下方法`class func data(withJSONObject obj: Any, options opt: JSONSerialization.WritingOptions = []) throws -> Data`的说明:
>If obj will not produce valid JSON, an exception is thrown. This exception is thrown prior to parsing and represents a programming error, not an internal error. You should check whether the input will produce valid JSON before calling this method by using isValidJSONObject(_:).

大意是如果obj参数不是一个合法的JSON,那么将抛出一个异常.该异常代表的是一个编程错误而不是解析时的内部错误,它在解析之前就被抛出了.估计这种异常是无法捕获的.所以最好先调用`isValidJSONObject `判断下是不是一个合法的JSON,再进行序列化.

类似的这样的错误也是捕捉不到的.

```
var ot: String!
do {
    let os = try ot!
    print(os)
} catch {
    print(error)
}
```
上述代码依旧崩溃.

看来以后使用try-catch得注意下了.

### try-catch能够捕获什么样的错误呢?  
编程错误也就是异常都是不能捕获的.比如对nil强制解包,数组越界,非法参数等.
只有可恢复的错误才可以捕获.

[Error-Handling in Swift-Language
](https://stackoverflow.com/questions/24010569/error-handling-in-swift-language)

但在OC中是可以捕获异常的:

```
@try {
    NSLog(@"@try");
    
    NSString *s = @"xx";
    s = [s stringByAppendingString:nil];
    
    NSLog(@"%@", s);
}
@catch (NSException *exception) {
    NSLog(@"exception:%@", exception);
}
@finally {
    NSLog(@"@finally");
}
```
上述代码加了try-catch后就不会崩溃.

Swfit中的try-catch只是用来进行错误处理的,因此只能捕获错误,不能捕获异常.

### Swift的错误处理（Error handling）

错误处理（Error handling）是响应错误以及从错误中恢复的过程。Swift 提供了在运行时对可恢复错误的抛出、捕获、传递和操作的一等公民支持。

某些操作无法保证总是执行完所有代码或总是生成有用的结果。可选类型可用来表示值缺失，但是当某个操作失败时，最好能得知失败的原因，从而可以作出相应的应对。

举个例子，假如有个从磁盘上的某个文件读取数据并进行处理的任务，该任务会有多种可能失败的情况，包括指定路径下文件并不存在，文件不具有可读权限，或者文件编码格式不兼容。区分这些不同的失败情况可以让程序解决并处理某些错误，然后把它解决不了的错误报告给用户。

>Swift中的错误处理类似于其他语言中的异常处理,比如都使用try，catch和throw关键字。和其他语言中（包括 Objective-C ）的异常处理不同的是，Swift 中的错误处理并不涉及unwinding调用栈(即调用栈展开)，这是一个计算代价高昂的过程。就此而言，throw语句的性能是可以和return语句相媲美的。

由此可见Swift中的错误处理和异常处理不是同一个东西,只不过使用的关键字相同.