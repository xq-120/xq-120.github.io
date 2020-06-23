### Block定义
定义:能够截获自动变量**值**的匿名函数.本质上也是一个OC对象，它内部也有个isa指针。

### Block实现

#### 不截获任何变量值

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    void (^blk)(void) = ^{
        printf("block.");
    };
    blk();
    return 0;
}
```

`clang -rewrite-objc 文件名`得到一个`.cpp`的c++源文件.这个时候就可以看到上述代码的C语言实现了.

```c++
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};

...

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {

    printf("block.");
}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
int main(int argc, const char * argv[]) {
    void (*blk)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));
    ((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);
    return 0;
}
static struct IMAGE_INFO { unsigned version; unsigned flag; } _OBJC_IMAGE_INFO = { 0, 2 };
```
匿名函数变成了一个普通的C语言函数:

```c
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {

    printf("block.");
}
```
参数是block自身.

将Block语法生成的Block赋值给Block类型变量blk,等同于将`__main_block_impl_0`结构体实例的指针赋值给变量blk.

`__main_block_impl_0 `结构体的构造函数第一个参数就是匿名函数对应的C语言函数`void __main_block_func_0(struct __main_block_impl_0 *__cself);`

`blk();`Block调用变成了一个函数指针的调用:

`(*blk->impl.FuncPtr)(blk);`

上述是一个没有截获自动变量值的block例子.下面看一下当block截获自动变量值时的情况.

#### 截获 static 局部变量



#### 截获全局变量



#### 截获自动变量值

```objc
int main(int argc, const char * argv[]) {
    int va = 3;
    int vb = 4;
    
    void (^blk)(void) = ^{
        int vc = va + vb;
        printf("vc = %d\n", vc);
    };
    
    va = 10;
    vb = 5;
    
    blk();
    
    return 0;
}
```

同样的操作之后:

```c++
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};

...

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int va;
  int vb;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _va, int _vb, int flags=0) : va(_va), vb(_vb) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  int va = __cself->va; // bound by copy
  int vb = __cself->vb; // bound by copy

        int vc = va + vb;
        printf("vc = %d\n", vc);
    }

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
int main(int argc, const char * argv[]) {
    int va = 3;
    int vb = 4;

    void (*blk)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, va, vb));

    va = 10;
    vb = 5;

    ((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);

    return 0;
}
static struct IMAGE_INFO { unsigned version; unsigned flag; } _OBJC_IMAGE_INFO = { 0, 2 };
```
和第一种情况不同点在于:  
**结构体`__main_block_impl_0`多了两个成员变量`va`和 `vb`并且构造函数也多了两个参数**,**在实例化结构体`__main_block_impl_0`时将va和vb作为参数传给构造函数**.Block语法表达式所使用的自动变量值被保存到Block的结构体实例中了.这样就完成了对自动变量值的截获.

由于va和vb是作为参数传给构造函数的(传的是值),因此block语法块内修改va和vb的值(即在`static void __main_block_func_0(struct __main_block_impl_0 *__cself);`函数里修改va和vb的值)并不会影响外面变量的值,不过现在你也修改不了因为系统会提示错误`Variable is not assignable (missing __block type specifier).`.

而后面`va = 10;vb = 5;`va和vb的值的改变,也不会影响到block实例截获的值.因为初始化block实例的时候传的是值.

思考：为什么Block里截获的变量不加__block修饰符就不能在block里修改？

#### 截获自动变量值并修改

```objc
int main(int argc, const char * argv[]) {
    __block int va = 3;
    __block int vb = 4;
    __block int vc = 5;
    int vd = 6;
    
    void (^blk)(void) = ^{
        vc = 12;
        int ret = va + vb + vc + vd;
        printf("ret = %d\n", ret);
    };
    
    va = 10;
    vb = 5;
    
    blk();
    
    printf("va = %d, vb = %d, vc = %d, vd = %d\n", va, vb, vc, vd);
    
    return 0;
}
```
打印如下:

```
ret = 33
va = 10, vb = 5, vc = 12, vd = 6
Program ended with exit code: 0
```

同样的操作之后:

```c++
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};

...

struct __Block_byref_va_0 {
  void *__isa;
__Block_byref_va_0 *__forwarding;
 int __flags;
 int __size;
 int va;
};
struct __Block_byref_vb_1 {
  void *__isa;
__Block_byref_vb_1 *__forwarding;
 int __flags;
 int __size;
 int vb;
};
struct __Block_byref_vc_2 {
  void *__isa;
__Block_byref_vc_2 *__forwarding;
 int __flags;
 int __size;
 int vc;
};

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int vd;
  __Block_byref_vc_2 *vc; // by ref
  __Block_byref_va_0 *va; // by ref
  __Block_byref_vb_1 *vb; // by ref
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _vd, __Block_byref_vc_2 *_vc, __Block_byref_va_0 *_va, __Block_byref_vb_1 *_vb, int flags=0) : vd(_vd), vc(_vc->__forwarding), va(_va->__forwarding), vb(_vb->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  __Block_byref_vc_2 *vc = __cself->vc; // bound by ref
  __Block_byref_va_0 *va = __cself->va; // bound by ref
  __Block_byref_vb_1 *vb = __cself->vb; // bound by ref
  int vd = __cself->vd; // bound by copy

        (vc->__forwarding->vc) = 12;
        int ret = (va->__forwarding->va) + (vb->__forwarding->vb) + (vc->__forwarding->vc) + vd;
        printf("ret = %d\n", ret);
    }
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->vc, (void*)src->vc, 8/*BLOCK_FIELD_IS_BYREF*/);_Block_object_assign((void*)&dst->va, (void*)src->va, 8/*BLOCK_FIELD_IS_BYREF*/);_Block_object_assign((void*)&dst->vb, (void*)src->vb, 8/*BLOCK_FIELD_IS_BYREF*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->vc, 8/*BLOCK_FIELD_IS_BYREF*/);_Block_object_dispose((void*)src->va, 8/*BLOCK_FIELD_IS_BYREF*/);_Block_object_dispose((void*)src->vb, 8/*BLOCK_FIELD_IS_BYREF*/);}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};
int main(int argc, const char * argv[]) {
    __attribute__((__blocks__(byref))) __Block_byref_va_0 va = {(void*)0,(__Block_byref_va_0 *)&va, 0, sizeof(__Block_byref_va_0), 3};
    __attribute__((__blocks__(byref))) __Block_byref_vb_1 vb = {(void*)0,(__Block_byref_vb_1 *)&vb, 0, sizeof(__Block_byref_vb_1), 4};
    __attribute__((__blocks__(byref))) __Block_byref_vc_2 vc = {(void*)0,(__Block_byref_vc_2 *)&vc, 0, sizeof(__Block_byref_vc_2), 5};
    int vd = 6;

    void (*blk)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, vd, (__Block_byref_vc_2 *)&vc, (__Block_byref_va_0 *)&va, (__Block_byref_vb_1 *)&vb, 570425344));

    (va.__forwarding->va) = 10;
    (vb.__forwarding->vb) = 5;

    ((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);

    printf("va = %d, vb = %d, vc = %d, vd = %d\n", (va.__forwarding->va), (vb.__forwarding->vb), (vc.__forwarding->vc), vd);

    return 0;
}
static struct IMAGE_INFO { unsigned version; unsigned flag; } _OBJC_IMAGE_INFO = { 0, 2 };
```

可以看到代码量迅速增多.

**`__forwarding`作用**

注意到使用__block修饰的变量

```
__block int va = 3;
```

被截获后，最终会变成一个结构体实例：

```c
struct __Block_byref_va_0 {
  void *__isa;
__Block_byref_va_0 *__forwarding;
 int __flags;
 int __size;
 int va;
};
```

__isa：对象的类型

__forwarding：指针变量，指向自身。

va：用于保存变量的值

可以看到外部使用的时候是通过`va.__forwarding->va`来访问变量的值的。为什么不直接通过`va->va`来访问呢？

这是因为，当`__block`变量被拷贝到堆上时，`__forwarding`也会更改为指向堆上的地址，如果原来的变量访问的时候还去访问栈上的话，就会出现va的值不一致的情况，所以统一先根据`__forwarding`找到正确的地址后再取值。

#### 截获__strong对象指针变量

```objc
int main(int argc, const char * argv[]) {
    
    void (^blk)(id obj) = NULL;
    
    {
        id array = [[NSMutableArray alloc] init];
        blk = [^(id obj){
            [array addObject:obj];
            NSLog(@"array count = %ld", [array count]);
        } copy];
    }
    
    blk([NSObject new]);
    blk([NSObject new]);
    blk([NSObject new]);
    
    return 0;
}
```

```c++
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};

...

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  id array;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, id _array, int flags=0) : array(_array) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself, id obj) {
  id array = __cself->array; // bound by copy

            ((void (*)(id, SEL, ObjectType _Nonnull))(void *)objc_msgSend)((id)array, sel_registerName("addObject:"), (id)obj);
            NSLog((NSString *)&__NSConstantStringImpl__var_folders_46_lys08y0137d41lbysrxxd0h80000gn_T_main_b3a0b2_mi_0, ((NSUInteger (*)(id, SEL))(void *)objc_msgSend)((id)array, sel_registerName("count")));
        }
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->array, (void*)src->array, 3/*BLOCK_FIELD_IS_OBJECT*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->array, 3/*BLOCK_FIELD_IS_OBJECT*/);}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};
int main(int argc, const char * argv[]) {

    void (*blk)(id obj) = __null;

    {
        id array = ((NSMutableArray *(*)(id, SEL))(void *)objc_msgSend)((id)((NSMutableArray *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSMutableArray"), sel_registerName("alloc")), sel_registerName("init"));
        blk = (void (*)(id))((id (*)(id, SEL))(void *)objc_msgSend)((id)((void (*)(id))&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, array, 570425344)), sel_registerName("copy"));
    }

    ((void (*)(__block_impl *, id))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk, ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("new")));
    ((void (*)(__block_impl *, id))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk, ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("new")));
    ((void (*)(__block_impl *, id))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk, ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("new")));

    return 0;
}
static struct IMAGE_INFO { unsigned version; unsigned flag; } _OBJC_IMAGE_INFO = { 0, 2 };
```

#### 截获__weak对象指针变量

```objc
int main(int argc, const char * argv[]) {
    
    void (^blk)(id obj) = NULL;
    
    {
        id array = [[NSMutableArray alloc] init];
        id __weak array2 = array;
        blk = [^(id obj){
            [array2 addObject:obj];
            NSLog(@"array count = %ld", [array2 count]);
        } copy];
    }
    
    blk([NSObject new]);
    blk([NSObject new]);
    blk([NSObject new]);
    
    return 0;
}
```
clang命令:
`clang -rewrite-objc -fobjc-arc -stdlib=libc++ -mmacosx-version-min=10.7 -fobjc-runtime=macosx-10.7 -Wno-deprecated-declarations main.m`

PS:其他报`UIKit/UIKit.h`找不到错的.
使用命令:`xcrun -sdk iphonesimulator clang -rewrite-objc AppDelegate.m`

[clanclang编译错误: fatal error: 'UIKit/UIKit.h' file not found（本人亲测有效）](https://www.jianshu.com/p/1114678bee78)

```c++
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __weak id array2;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __weak id _array2, int flags=0) : array2(_array2) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself, __strong id obj) {
  __weak id array2 = __cself->array2; // bound by copy

            ((void (*)(id, SEL, ObjectType  _Nonnull __strong))(void *)objc_msgSend)((id)array2, sel_registerName("addObject:"), (id)obj);
            NSLog((NSString *)&__NSConstantStringImpl__var_folders_46_lys08y0137d41lbysrxxd0h80000gn_T_main_39f7ce_mi_0, ((NSUInteger (*)(id, SEL))(void *)objc_msgSend)((id)array2, sel_registerName("count")));
        }
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->array2, (void*)src->array2, 3/*BLOCK_FIELD_IS_OBJECT*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->array2, 3/*BLOCK_FIELD_IS_OBJECT*/);}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};
int main(int argc, const char * argv[]) {

    void (*blk)(id obj) = __null;

    {
        id array = ((NSMutableArray *(*)(id, SEL))(void *)objc_msgSend)((id)((NSMutableArray *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSMutableArray"), sel_registerName("alloc")), sel_registerName("init"));
        id __attribute__((objc_ownership(weak))) array2 = array;
        blk = (void (*)(__strong id))((id (*)(id, SEL))(void *)objc_msgSend)((id)((void (*)(__strong id))&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, array2, 570425344)), sel_registerName("copy"));
    }

    ((void (*)(__block_impl *, __strong id))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk, ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("new")));
    ((void (*)(__block_impl *, __strong id))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk, ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("new")));
    ((void (*)(__block_impl *, __strong id))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk, ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("new")));

    return 0;
}
static struct IMAGE_INFO { unsigned version; unsigned flag; } _OBJC_IMAGE_INFO = { 0, 2 };
```

问题:

* Block是如何截获自动变量的值? ok
* 为什么在Block语法外声明的自动变量需要添加__block修饰符才能在Block内修改? ok
* 为什么使用__weak修饰的对象指针变量可以避免Block循环引用?
* 避免Block循环引用有哪几种方式?
* 如何检测Block循环引用导致的内存泄漏?(FBRetainCycleDetector(facebook开源工具库), MLeaksFinder(WeRead开源))



### 参考

[Block 的递归调用 - How to implement recursive call block?](http://linlexus.com/how-to-implement-recursive-call-block/)   block 递归调用，超出你的想象。

[[iOS] Blocks 中的递归](http://blog.codescv.com/ios/blocks-recursion.html)

