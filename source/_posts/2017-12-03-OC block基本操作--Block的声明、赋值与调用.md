---
title: OC block基本操作--Block的声明、赋值与调用
date: 2017-12-03 15:05:52
description: 
categories: OC
tags: Block
---

### Block的声明、赋值与调用

#### Block变量的声明

和C语言的函数指针声明几乎一样,只是将”*”改为”^”.如下:

`返回值类型` `(^变量名称)` `(变量类型 变量1, 变量类型 变量2, ...)`
 
例:int (^blkVar)(int a, int b);

#### Block语法
一个Block语法代表一个Block类型变量的值.它的写法和C语言函数的定义很像.仅有两点区别:

* 没有函数名.
* 在返回值前面加上”^”. 

`^` `返回值` `(变量类型 变量1, 变量类型 变量2, ...)` `{... return xx;}`. 

例:^int (int a, int b){... return 3;}.

Block作为函数参数在C语言里的写法和OC里的写法略有不同.如下:

```
//OC方法
- (int)addFunc:(int(^)(int a, int b))blkParam :(int)addA :(int)addB
{
    int c = 0;
    c = blkParam(addA, addB);
    printf("rs:%d\n", c);
    return c;
}

//C函数
int addFunc(int (^blkParam)(int a , int b), int addA, int addB)
{
    int c = 0;
    c = blkParam(addA, addB);
    printf("rs:%d\n", c);
    return c;
}
```

#### Block的调用
`blk变量(参数1, 参数2, ...);`

一个比较古怪的Block调用:直接对Block语法调用.

```
int aa = ^int(int a, int b){
        int c = a + b;
        NSLog(@"%d",c);
        return c;
    }(2,4);
NSLog(@"aa=%d",aa);
```

很早之前刚接触block的时候,总是在想Block里的代码到底啥时候执行.原来就是block什么时候被调用,什么时候就被执行.

Block也相当于一种变量类型。就像int一样，可以用来定义对应类型的变量。
因此

```
[p printWithBlcok:^ViewController *(int num) {
    NSLog(@"num*cc:%i * %i = %i",num,cc,num *cc);
    return self;
}];
```

里面的printWithBlcok:后的就是一个Block语法(也就是一个Block类型的变量的值)。在这里它作为函数的第一个参数。因此对对象p发送printWithBlcok:消息，将执行该方法的代码，该方法的第一行代码就是调用Block，因此又转而去执行Block中的代码，执行完后，再回到printWithBlcok:方法中，然后继续执行下去。