---
title: OC nullability特性
date: 2019-10-01 17:13:28
description: 
categories: OC
tags: nullability
comments: true
---

### OC nullability特性

众所周知,Swift语言是严格区分可选和非可选变量的.比如`UIView`和`UIView?`.而在之前的OC中却没有这个概念,二者都使用`UIView*`表示.由于Swift编译器无法根据`UIView*`判断该变量是否是可选的.因此该类型在Swift中被解释为隐式解包`UIView!`.而在Swift中对一个值为nil的可选变量进行强制解包时将导致崩溃.因此你必须时刻警惕`UIView!`变量在某处会不会被赋值为nil.这不仅带来使用上的不便,而且也影响编程体验.因此有必要让OC兼容Swift的Optional特性,方便OC的类在Swift中的使用.于是在Xcode 6.3 开始官方便给OC添加了nullability功能.

nullability 特性包含`_Nullable` 和 `_Nonnull`这两种修饰符.`_Nullable`表明该变量可以为nil或NULL值.而`_Nonnull`表明该变量不能为nil值.两个变种写法`nullable`,`nonnull`.

示例如下:

```
#import <Foundation/Foundation.h>
#import "Person.h"

#define myCase  4

#if myCase == 1

//不使用
@interface Book : NSObject

- (instancetype)initWithName:(NSString *)name;

@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *version;
@property (nonatomic, assign) long words;
@property (nonatomic, copy) NSString *publicID;
@property (nonatomic, copy) NSString *url;
@property (nonatomic, strong) Person *author;

@end

/**
Swift接口如下:
open class Book : NSObject {

    
    public init!(name: String!)

    
    open var name: String!

    open var version: String!

    open var words: Int

    open var publicID: String!

    open var url: String!

    open var author: Person!
}
*/

#elif myCase == 2

//推荐写法.属性都不可为nil
NS_ASSUME_NONNULL_BEGIN

@interface Book : NSObject

- (instancetype)initWithName:(NSString *)name;

@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *version;
@property (nonatomic, assign) long words;
@property (nonatomic, copy) NSString *publicID;
@property (nonatomic, copy) NSString *url;
@property (nonatomic, strong) Person *author;

@end

NS_ASSUME_NONNULL_END

#elif myCase == 3

//推荐写法,个别属性可为nil
NS_ASSUME_NONNULL_BEGIN

@interface Book : NSObject

- (instancetype)initWithName:(NSString *)name;

@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *version;
@property (nonatomic, assign) long words;
@property (nonatomic, copy, nullable) NSString *publicID;
@property (nonatomic, copy, nullable) NSString *url;
@property (nonatomic, strong) Person *author;

@end

NS_ASSUME_NONNULL_END

#elif myCase == 4

//全部自己手动定义
@interface Book : NSObject

- (nonnull instancetype)initWithName:(NSString * _Nonnull)name;

@property (nonatomic, copy) NSString * _Nonnull name;
@property (nonatomic, copy, nonnull) NSString *version;
@property (nonatomic, assign) long words;
@property (nonatomic, strong, nonnull) NSNumber *barCode;
@property (nonatomic, copy, nullable) NSString *publicID;
@property (nonatomic, copy, nullable) NSString *url;
@property (nonatomic, strong, nonnull) Person *author;
@property (nonatomic, strong, nonnull) NSMutableArray<Person *> *seeker;

@end

#endif
```

#### nullability特性对OC编码的影响

在OC中即使标明为`_Nonnull`,你依然可以给它一个nil值,但编译器会给出警告,仅仅是警告.因此在OC中不要对`_Nonnull`修饰的变量抱有一定不为nil的幻想,否则该崩还得崩.

nullability特性只是一种约定,在开发的过程中我们要遵守这一约定.如果一个变量是nonnull的,我们在init方法中一定要给它一个初始值,并且在使用过程中不要给它赋值为nil.

#### nullability特性对OC,Swift混编的影响

nullability特性其实主要还是为了方便OC的类在Swift中的使用.

一个`_Nonnull`修饰的OC变量,它在OC代码中可能会被赋值为nil,当它在Swift中使用时nil会变成`<uninitialized>`对象.奇怪的是如果是nil的NSString则会被自动转换为空字符`""`.

```

NS_ASSUME_NONNULL_BEGIN

@interface Person : NSObject

@property (nonatomic, copy) NSString *name;
@property (nonatomic, copy) NSString *sex;

- (void)hello;

@end

NS_ASSUME_NONNULL_END


@interface Book : NSObject

- (nonnull instancetype)initWithName:(NSString * _Nonnull)name;

@property (nonatomic, copy) NSString * _Nonnull name;
@property (nonatomic, copy, nonnull) NSString *version;
@property (nonatomic, assign) long words;
@property (nonatomic, strong, nonnull) NSNumber *barCode;
@property (nonatomic, copy, nullable) NSString *publicID;
@property (nonatomic, copy, nullable) NSString *url;
@property (nonatomic, strong, nonnull) Person *author;
@property (nonatomic, strong, nonnull) NSMutableArray<Person *> *seeker;

@end

@objc func printBook() {
    let name = book.name
    
    let version = book.version
    print(version)
    
    let seeker: NSMutableArray = book.seeker
    seeker.add("sd") //没效果
    print(seeker) //崩溃:Thread 1: EXC_BAD_ACCESS (code=1, address=0x0)
    
    let num: NSNumber = book.barCode
//        print(num) //崩溃
    let i = num.intValue //0
    
    let author = book.author
    let authorName = author.name
    
    let publicID = book.publicID ?? "unknown"
    let url = book.url ?? "unknown"
    let string = name + "-" + version + "-" + authorName + "-" + publicID + "-" + url
    
    author.hello() //不会执行
    let des = String.init(describing: author) //空字符串
    print("des:\(des)")
//        print(author) //崩溃
    
//        let s: AnyClass = type(of: author) //崩溃
    
    var arr: [Person] = []
    arr.append(author)
    
    print("\(author)", "arr个数:\(arr.count)", arr)
    
    print("\(string)")
}
```
打印

```
 arr个数:1 []
三国---unknown-unknown
```

其中的authorName将会是空字符串`""`;而author将会是`	<uninitialized>`,其实就是一个nil对象,给它发送消息没有什么问题,但调用Swift中的某些方法比如`print(author)`或`type(of: author)`将导致野指针崩溃:`Thread 1: EXC_BAD_ACCESS (code=1, address=0x0)`.

> OC和Swift语言其他的一些设计差别

Swift中,如果一个变量没有被初始化,它是不能被使用的,编译器会报错.

```
{
	var peron: Person
	peron.hello() //Variable 'peron' used before being initialized
}
```

但OC中,声明一个变量后没有给它一个值也能够给它发送消息.

```
{
	Person *p;
    [p hello];
    Class cls = p.class;
    NSLog(@"cls:%@", cls);	 // cls:(null)
}
```
可以看出Swift对初始化是非常严格的.而OC基本没啥限制.

### 总结

nullability特性只是一种约定,在开发的过程中我们要遵守这一约定.如果一个变量是nonnull的,我们在初始化方法中一定要给它一个初始值,并且在使用过程中不要给它赋值为nil.

OC,Swift混编时,为了使用方便并减少可能的解包崩溃,推荐使用nullability特性,旧的OC代码也有必要添加.

在Swift中尽量不使用强制解包.
