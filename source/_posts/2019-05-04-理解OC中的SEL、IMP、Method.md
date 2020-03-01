---
title: 理解OC中的SEL、IMP、Method
date: 2019-05-04 23:11:34
description: 
categories: runtime
tags: SEL、IMP、Method
---

#### SEL
Defines an opaque type that represents a method selector.

```
typedef struct objc_selector *SEL;
```
SEL是一种数据类型,代表一个方法选择器(要是觉得不好理解,可以类比一下int,int是整型类型,代表一个整数).方法选择器就是运行时中方法的名称,它是一个注册到(或映射到)Objective-C运行时里的一个C字符串.选择器由编译器生成,在系统加载类的时候由runtime自动进行映射.

按照文档说明,选择器是在编译期间生成的,在加载类的时候映射为runtime中的一个C字符串.

给对象发送一条消息`[obj xxxMethod];`,其实是调用了`objc_msgSend(obj, @selector(xxxMethod));`.

`objc_msgSend `的声明如下:
`OBJC_EXPORT id _Nullable
objc_msgSend(id _Nullable self, SEL _Nonnull op, ...);`
它的第二个参数的类型就是SEL.说明它是真实存在的.

相关的API:

```
//向runtime system注册一个方法名。如果方法名已经注册，则返回已经注册的SEL
SEL sel_registerName(const char *str)

//功能同上
SEL sel_getUid(const char *str)

//使用OC编译器指令.功能同上,也是向runtime system注册一个方法名.
@selector(<#selector#>)

//使用OC字符串构造.功能同上
SEL NSSelectorFromString(NSString *aSelectorName)

//根据Method结构体获取
SEL _Nonnull method_getName(Method _Nonnull m);

//比较两个选择器
BOOL sel_isEqual ( SEL lhs, SEL rhs );
//判断方法名是否映射到某个函数实现上
BOOL sel_isMapped(SEL sel);
```

eg:

```
SEL hasBookSEL = @selector(hasBook:); //hasBook:是Person类的一个实例方法.
//sel_getName:hasBook:, p:0x10c230538
NSLog(@"sel_getName:%s, p:%p", sel_getName(hasBookSEL), hasBookSEL);

SEL sel0 = sel_registerName("myRegisterName");
SEL sel1 = sel_getUid("myGetUid");
SEL sel2 = @selector(hello:);
SEL sel3 = NSSelectorFromString(@"mySelectorFromString"); 
//sel0:0x10c232634, sel1:0x10c232643, sel2:0x10c7be0eb, sel3:0x600003e78500.
NSLog(@"sel0:%p, sel1:%p, sel2:%p, sel3:%p", sel0, sel1, sel2, sel3);
if (sel_isMapped(sel2)) { //都是true
    NSLog(@"已经映射");
}
```
前三种方式得到的sel地址相差不大,但使用`NSSelectorFromString()`得到的sel地址却相差很大.可能的原因是前三种是在编译期间生成的.而使用`NSSelectorFromString()`得到的sel是在运行时才生成(待验证).

上述代码只是向runtime注册了方法名,并没有关联一个物理的函数实现(IMP).所以不要看到一个选择器就觉得一定有一个对应的物理函数实现.选择子SEL和物理函数实现IMP实际上是相互独立的,这给OC的动态性提供了基础.

我们都知道一条成功的消息发送最终是要调到一个物理函数的,因此消息发送的过程就是runtime根据选择子SEL找物理函数实现IMP的过程.在消息发送的过程中当一个选择子最终无法找到一个对应的IMP时,系统就会抛出著名的`unrecognized selector sent to instance`崩溃.

这就引出了一个问题,怎么通过一个SEL找到一个IMP?这个时候就需要查看类的结构体`struct objc_class`的构成了,如下:

```
struct objc_class {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class _Nullable super_class                              OBJC2_UNAVAILABLE;
    const char * _Nonnull name                               OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list * _Nullable ivars                  OBJC2_UNAVAILABLE;
    struct objc_method_list * _Nullable * _Nullable methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache * _Nonnull cache                       OBJC2_UNAVAILABLE;
    struct objc_protocol_list * _Nullable protocols          OBJC2_UNAVAILABLE;
#endif

} OBJC2_UNAVAILABLE;
/* Use `Class` instead of `struct objc_class *` */
```

可以看到它里面有一个`struct objc_method_list * _Nullable * _Nullable methodLists`方法列表的成员变量,继续查看方法列表结构体,如下:

```
struct objc_method {
    SEL _Nonnull method_name                                 OBJC2_UNAVAILABLE;
    char * _Nullable method_types                            OBJC2_UNAVAILABLE;
    IMP _Nonnull method_imp                                  OBJC2_UNAVAILABLE;
}                                                            OBJC2_UNAVAILABLE;

struct objc_method_list {
    struct objc_method_list * _Nullable obsolete             OBJC2_UNAVAILABLE;

    int method_count                                         OBJC2_UNAVAILABLE;
#ifdef __LP64__
    int space                                                OBJC2_UNAVAILABLE;
#endif
    /* variable length structure */
    struct objc_method method_list[1]                        OBJC2_UNAVAILABLE;
}                                                            OBJC2_UNAVAILABLE;

```
我们终于看到了一个`struct objc_method`的结构体,这就是后面要说到的Method类型.看到Method类型的成员变量我们就知道OC语言的设计者是怎么看待在.m文件中写的那一个个物理函数了:

一个物理函数={选择子(方法名)+方法类型(将返回值类型,参数类型编码后的一个C字符串)+函数指针IMP}=一个Method实例.

由上可知,一个类拥有一个方法列表,给类的实例发送一条消息的过程简单点讲就是根据该消息选择子在方法列表里找对应的Method,找到了就调用Method实例里的IMP,执行函数就完事了.当然实际情况要稍微复杂些:如果在自身类里没找到会继续沿着继承链往上找,最后可能还会进入消息转发.这里就不展开讲了,网上有很多.

给类动态的添加一个方法:

```
- (void)viewDidLoad {
	BOOL rs = class_addMethod(Person.class, sel2, (IMP)hello, "B@:@");
	if (rs) {
	    NSLog(@"添加方法成功");
	}
	Person *p = [Person new];
	[p performSelector:sel2 withObject:@"小明"];
}

BOOL hello(id self, SEL _cmd, NSString *name) {
    NSLog(@"%@, hello:%@", self, name);
    return YES;
}
```


#### IMP

A pointer to the function of a method implementation.

```
typedef id _Nullable (*IMP)(id _Nonnull, SEL _Nonnull, ...); 
```
IMP是一个函数指针,指向方法实现的具体函数首地址.

eg:

```
IMP imp0 = class_getMethodImplementation(Person.class, @selector(hasBook:));
NSLog(@"%p", imp0);
    
Method method = class_getInstanceMethod(Person.class, @selector(hasBook:));
IMP imp1 = method_getImplementation(method);
NSLog(@"%p", imp1);
    
/**
 1. 需要将Enable Strict Checking of objc_msgSend Calls 设置为NO
 2. 必须将imp1类型强转后再调用.
 id rs1 = imp1([Person new], @selector(hasBook:), @"wind");
 会导致崩溃:Thread 1: EXC_BAD_ACCESS (code=1, address=0x7d8)
 */
/**
 通过IMP直接调用函数.这里传入了什么SEL形参值,那么执行的函数读取_cmd的值就是什么.甚至可以传NULL,但最好还是传原始方法选择器.
 imp1是和Person类相关联的.因此第0个参数传Person类型对象,第1个参数传Person里的方法,才有意义.
*/
BOOL rs = ((BOOL (*)(id, SEL, NSString *))imp1)([Person new], @selector(hasBook:), @"Gone with the wind");
if (rs) {
    NSLog(@"has book: Gone with the wind");
} else {
    NSLog(@"not found book:Gone with the wind");
}
    
/**
 使用imp_implementationWithBlock()生成一个IMP,并调用IMP.由于imp2是通过imp_implementationWithBlock()函数得来和上面的imp0的产生方式完全不同,因此imp2并不和任何类有啥关联,故它的前两个参数基本上可任意传.
 */
IMP imp2 = imp_implementationWithBlock(^BOOL(id object, NSString *arg) {
    NSLog(@"object:%@, arg:%@", object, arg);
    return NO;
});
BOOL rss = ((BOOL (*)(id, SEL, NSString *))imp2)([Person new], @selector(placeholderSEL:), @"Gone with the wind");
if (rss) {
    NSLog(@"has book: Gone with the wind");
} else {
    NSLog(@"not found book:Gone with the wind");
}
    
rss = [Person.new hasBook:@"Gone with the wind"];
```
通过IMP直接调用对应函数,需要注意两点:

1. 需要将Enable Strict Checking of objc_msgSend Calls 设置为NO
2. 必须将IMP类型强转后再调用.否则会导致崩溃:`Thread 1: EXC_BAD_ACCESS (code=1, address=0x7d8)`



#### Method

```
/// An opaque type that represents a method in a class definition.
typedef struct objc_method *Method;

struct objc_method {
    SEL _Nonnull method_name                                 OBJC2_UNAVAILABLE;
    char * _Nullable method_types                            OBJC2_UNAVAILABLE;
    IMP _Nonnull method_imp                                  OBJC2_UNAVAILABLE;
}                                                            OBJC2_UNAVAILABLE;
```
Method是一种数据类型,代表类里面定义的一个方法.个人觉得将一个物理方法抽象为一个Method实例,简直是秀啊.这样只需要更改Method实例里的IMP成员变量的值,不就调到另外一个物理函数了吗,这样不就动态起来了吗,果然是优秀的设计!事实上Method系列API就提供了这些骚操作:

```
//获取Method里的IMP
OBJC_EXPORT IMP _Nonnull
method_getImplementation(Method _Nonnull m);

//设置一个新的IMP
OBJC_EXPORT IMP _Nonnull
method_setImplementation(Method _Nonnull m, IMP _Nonnull imp);

//方法交换.就是那个黑魔法.
OBJC_EXPORT void
method_exchangeImplementations(Method _Nonnull m1, Method _Nonnull m2);
```

由于OBJC2_UNAVAILABLE,即使得到了Method指针也不能直接访问结构体里的成员变量.只能通过API来获取.

eg:

```
Method method = class_getInstanceMethod(Person.class, @selector(hasBook:));

SEL name = method_getName(method);
```

相关API:

```
// 获取实例方法
Method class_getInstanceMethod ( Class cls, SEL name );

// 获取类方法
Method class_getClassMethod ( Class cls, SEL name );

// 获取所有方法的数组
Method * class_copyMethodList ( Class cls, unsigned int *outCount );

//获取方法名
OBJC_EXPORT SEL _Nonnull
method_getName(Method _Nonnull m) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);

//获取IMP  
OBJC_EXPORT IMP _Nonnull
method_getImplementation(Method _Nonnull m) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);

//获取函数参数和返回值类型.(包含在返回的字符串中)
OBJC_EXPORT const char * _Nullable
method_getTypeEncoding(Method _Nonnull m) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);
    
OBJC_EXPORT void
method_getReturnType(Method _Nonnull m, char * _Nonnull dst, size_t dst_len) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);
```

eg:

```
- (void)methodApi
{
    Method method = class_getInstanceMethod(Person.class, @selector(hasBook:));
    SEL name = method_getName(method);
    NSLog(@"sel:%@", NSStringFromSelector(name));
    
    //打印方法的参数和返回值类型编码
    const char *types = method_getTypeEncoding(method);
    NSLog(@"TypeEncoding:%s", types); //"B24@0:8@16",不看数字就是"B@:@",即"BOOL id SEL id".表明返回值为BOOL类型,第0个参数的类型是对象类型,第1个参数的类型是SEL类型,第2个参数的类型也是对象类型.中间夹杂的数字估计是偏移量.第0个参数偏移量自然是0,长度8字节.于是第二个参数的偏移量自然是8.以此类推.
    
    //打印参数个数
    NSUInteger args = method_getNumberOfArguments(method);
    NSLog(@"参数个数:%lu", args);
    
    //打印返回值的类型编码
    char returnType;
    method_getReturnType(method, &returnType, 1);
    char *rt = method_copyReturnType(method);
    NSLog(@"returnType:%c %s", returnType, rt);
    free(rt); //必须调用free,否则内存泄漏.
    
    //打印参数的类型编码
    for (int i = 0; i < args; i++) {
        char argType;
        method_getArgumentType(method, i, &argType, 1);
        NSLog(@"argType%d:%c", i, argType);
    }
    for (int i = 0; i < args; i++) {
        char *arg = method_copyArgumentType(method, i);
        NSLog(@"argType%d:%s", i, arg);
        free(arg); //必须调用free,否则内存泄漏.
    }
    
    //也可以使用NSMethodSignature对象来获取某个参数的类型编码或返回值编码
    NSMethodSignature *methodSig = [Person instanceMethodSignatureForSelector:name];
}
```

#### 总结

根据上面的简单介绍,我们知道了:  
SEL是一种数据类型,代表一个方法选择器.  
IMP是一个函数指针,指向方法实现的具体函数首地址.  
Method是一种数据类型,代表类里面定义的一个方法.  
正是抽象出了这么几种数据类型,才让我们能够如此方便的运用runtime,使用各种黑科技.

最后以解决一个小问题,再次温习一下这三个概念,并结束本文.

问题:假设有一个SDK,我们仅知道这个SDK里面有一个私有类名叫"Person",以及它的一个方法:`- (BOOL)hasBook:(NSString *)book;`,假设该方法的实现如下:

```
- (BOOL)hasBook:(NSString *)book
{
    if ([book isEqualToString:@"Gone with the wind"]) {
        return YES;
    }
    return NO;
}
```
现在想要重写该方法的内部实现要求:当传入的参数为"Romance Of The Three Kingdoms"返回YES,否则还走原来的逻辑.如何做才可以办到?

解:其实如果Person不是私有类,我们只需要继承它,然后重写`hasBook:`方法,在里面来个if-else就完事了.但是现在Person是私有类,没办法继承,此路不通.同样的类别也不行,再说类别里的方法会覆盖原来的实现,基本上没法调用到原来的实现,所以类别也是行不通的.目前看来只有runtime可能有点用,现在就让我们使用上述API来搞定这个问题.

通过runtime我们可以获取到一个类的Method,有了Method就可以获取到原始IMP,然后我们还可以给Method设置一个新的IMP,在新的IMP里调用原始IMP.这样上述问题就解决了.

咱们先写个调用Person的hasBook:的方法,方便后面测试:

```
- (BOOL)invokeHasBookWithPerson:(id)person param:(NSString *)param
{
    SEL seletor = NSSelectorFromString(@"hasBook:");
    NSMethodSignature *methodSig = [person methodSignatureForSelector:seletor];
    NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:methodSig];
    [invocation setArgument:&param atIndex:2];
    [invocation setSelector:seletor];
    [invocation setTarget:person];
    [invocation invoke];
    //调用后才可以去获取返回值.
    BOOL rs = NO;
    [invocation getReturnValue:&rs];
    return rs;
}
```
这里为啥使用NSInvocation而不直接使用performSelector,是因为:
>`- (id)performSelector:(SEL)aSelector;`
`- (id)performSelector:(SEL)aSelector withObject:(id)object;`
performSelector系列方法,被执行的选择器的返回值必须是对象类型.否则将崩溃.参数也必须是对象类型,否则被执行的选择器得到的将是无效参数.如果是非对象类型,则只能使用NSInvocation.

继续扯回来,这样正常的写法`[Person.new hasBook:@"xxx"];`变为:

```
id person = [NSClassFromString(@"Person") new];
BOOL rst1 = [self invokeHasBookWithPerson:person param:@"Romance Of The Three Kingdoms"];
BOOL rst2 = [self invokeHasBookWithPerson:person param:@"Gone with the wind"];
NSLog(@"rst1:%d, rst2:%d", rst1, rst2);
```
打印为rst1:0, rst2:1.说明目前走的是以前的实现.

接下来就是获取原始IMP,以及设置新IMP.

```
Method method = class_getInstanceMethod(NSClassFromString(@"Person"), NSSelectorFromString(@"hasBook:"));
//获取原始IMP
IMP imp1 = method_getImplementation(method);
NSLog(@"imp1:%p", imp1);

//设置新IMP
method_setImplementation(method, imp_implementationWithBlock(^BOOL(id object, NSString *arg) {
    NSLog(@"object:%@, arg:%@", object, arg);
    if ([arg isEqualToString:@"Romance Of The Three Kingdoms"]) {
        return YES;
    }
    //调用原来的实现.
    return ((BOOL (*)(id, SEL, NSString *))imp1)(object, NSSelectorFromString(@"hasBook:"), arg);
}));
    
//测试效果
id person = [NSClassFromString(@"Person") new];
BOOL rst1 = [self invokeHasBookWithPerson:person param:@"Romance Of The Three Kingdoms"];
BOOL rst2 = [self invokeHasBookWithPerson:person param:@"Gone with the wind"];
BOOL rst3 = [self invokeHasBookWithPerson:person param:@"xxx"];
NSLog(@"rst1:%d, rst2:%d", rst1, rst2);
```
打印结果:rst1:1, rst2:1, rst3:0.是预期的效果.问题解决.

其实我们还可以对上面的重写操作封装一下,这样如果其他类也有这样的需求,那么只需要改动Block的实现就可以了:

```
BOOL overideImplemention(Class class, SEL selector, id (^impBlk)(Class cls, SEL sel, IMP imp)) {
    
    if (class == Nil || selector == NULL || impBlk == nil) {
        return NO;
    }
    
    Method method = class_getInstanceMethod(class, selector);
    if (method == NULL) {
        return NO;
    }
    IMP originalIMP = method_getImplementation(method);
    method_setImplementation(method, imp_implementationWithBlock(impBlk(class, selector, originalIMP)));
    return YES;
}


//使用
{
	BOOL isOveride = overideImplemention(NSClassFromString(@"Person"), NSSelectorFromString(@"hasBook:"), ^id(__unsafe_unretained Class cls, SEL sel, IMP imp) {
        return ^BOOL(id object, NSString *arg) {
            if (![object isKindOfClass:cls]) {
                return NO;
            }
            
            if ([arg isEqualToString:@"Romance Of The Three Kingdoms"]) {
                return YES;
            }
            
            //调用原始方法
            return ((BOOL (*)(id, SEL, NSString *))imp)(object, sel, arg);
        };
    });
    if (isOveride) {
        NSLog(@"重写成功!");
    }
    BOOL rst4 = [self invokeHasBookWithPerson:person param:@"Romance Of The Three Kingdoms"];
    BOOL rst5 = [self invokeHasBookWithPerson:person param:@"Gone with the wind"];
    BOOL rst6 = [self invokeHasBookWithPerson:person param:@"Journey to the West"];
    NSLog(@"rst4:%d, rst5:%d, rst6:%d", rst4, rst5, rst6);
}
```
打印:rst4:1, rst5:1, rst6:0.也是一样的效果.
经过重写,原SDK里的`hasBook:`消息也会走到我们新设置的IMP,如果我们不在新IMP里调用原IMP,那么程序将不再执行以前的实现.这就是runtime,是不是很强大.

#### 参考
[SEL的官方文档说明](https://developer.apple.com/documentation/objectivec/sel?language=occ)

[从源代码看 ObjC 中消息的发送](https://draveness.me/message)

[尝试手写一个更好用的performSelector-msgSend](https://awhisper.github.io/2015/12/31/%E5%B0%9D%E8%AF%95%E6%89%8B%E5%86%99%E4%B8%80%E4%B8%AA%E6%9B%B4%E5%A5%BD%E7%94%A8%E7%9A%84performSelector-msgSend/)