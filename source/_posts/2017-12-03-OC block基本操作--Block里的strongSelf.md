---
title: OC block基本操作--Block里的strongSelf
date: 2017-12-03 13:15:20
description: 
categories: OC
tags: Block
---

### Block里的strongSelf
有时我们在看源码时,会发现作者会在block里第一行strong一下weakSelf.以下便是对这种写法的一个探究.

一般在使用block的时候,都会在block外,声明一个weakSelf,来确保block不去持有self.但是这也会引发一个问题,就是如果在block执行之前,self就已经销毁了,那么block里的[weakSelf method],method方法就不会被执行,如果有返回值的话,会返回nil.如果Block里有这样的代码:[arrM addObject:nil],是会崩溃的.所以,如何让block在执行时,self不销毁?

解决办法就是:在block代码块里第一行将weakSelf赋值给一个strong类型指针变量,通常是` __strong typeof(weakSelf) strongSelf = weakSelf;`.有了strongSelf这样一个强引用指针,以后的代码执行时,便不会出现self==nil的情况,且当block执行完后,局部变量strongSelf清除,self对象被释放.一切看起来似乎很完美.但是如果在
__strong typeof(weakSelf) strongSelf = weakSelf;
执行之前,self就已经销毁,那么strongSelf将等于nil.上面的崩溃还是会发生.所以必须判断一下strongSelf是否为nil.

综上完整的写法应该如下:

```
__weak typeof(self) weakSelf = self;
void (^success)(id data, NSString *statusInfo) = ^(NSArray *data, NSString *statusInfo) {
    __strong typeof(weakSelf) strongSelf = weakSelf;
    if (strongSelf == nil) return;
    
    NSLog(@"block回调了,strongSelf:%@", strongSelf); 
    NSString *sd = [strongSelf printSomething]; //  如果没有前面的判断,这里有可能会因为strongSelf==nil,而导致返回值为nil.从而后面的[mA addObject:sd];会崩溃.
    
    NSMutableArray *mA = [[NSMutableArray alloc] initWithArray:data];
    [mA addObject:@"dsfds"];
    [mA addObject:sd];
    NSLog(@"%@", mA);
    NSLog(@"%@", statusInfo);
};
```

> 思考1:在Block里写`__strong typeof(weakSelf) strongSelf = weakSelf;`会不会导致self-Block引用循环?

答案是不会的,因为strongSelf是Block代码块里定义的一个局部变量,当Block执行完后,局部变量就会销毁,这样就不会有强引用的指针指向self对象.self与Block之间也就不会形成一个引用循环.

> 思考2:是不是Block里只能使用weakSelf,不能使用self?

这个并不是绝对的,使用weakSelf当然是不会有问题的.而如果self不持有Block,那么即使Block里使用self,也不会形成self-Block引用循环,这种情况下在Block里直接使用self也是没有问题的,只不过此时self对象的销毁将依赖于Block的销毁,换句话说就是self对象的生命周期可能会被延长.  

举个例子:  
vcB里有一个实例变量指向C对象,vcB里用该实例变量调用C类里的方法,参数block代码块里直接使用self(即vcB对象).

```
@interface SecondViewController ()

@property (nonatomic, strong) Animal *myAni;

@end

@implementation SecondViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.view.backgroundColor = [UIColor whiteColor];
    
    self.myAni = [Animal new];
    void (^blk)(Animal *aAnimal) = ^(Animal *aAnimal) {
        NSLog(@"%@", aAnimal);
        NSLog(@"self:%@", self); //Block截获self.
    };
    [self.myAni printHello:blk];
}

@interface Animal : NSObject

- (void)printHello:(void(^)(Animal *aAnimal))paramBlk;

@end

@interface Animal ()

@property (nonatomic, copy) void (^blk)(Animal *aAnimal);

@end

@implementation Animal

- (void)printHello:(void (^)(Animal *))paramBlk
{
//    self.blk = paramBlk;   
    paramBlk(self);
}

- (void)dealloc
{
    NSLog(@"%@销毁", self);
}

@end
```

他们之间的关系就如下图所示,vcB持有C对象,参数block持有vcB对象,从下图可以看出并没有构成一个循环引用,所以虽然在block里直接使用self,但却并没有什么问题.
![Block里直接使用self对象关系图](http://7xqoji.com1.z0.glb.clouddn.com/Block%E9%87%8C%E7%9B%B4%E6%8E%A5%E4%BD%BF%E7%94%A8self%E5%AF%B9%E8%B1%A1%E5%85%B3%E7%B3%BB%E5%9B%BE.png)
但是如果C类里有个block实例变量,指向了该参数block,那么将导致循环引用.