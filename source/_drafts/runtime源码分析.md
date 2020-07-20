---
title: runtime源码分析
tags:
  - 标签1
  - 标签2
  - 标签3
categories:
  - 分类1
  - 分类2
comments: true
date: 2020-07-18 23:32:40
---

版本：`objc4-781`

`TARGETS -> debug-objc -> Build Settings -> Enable Hardened Runtime = NO`，解决可编译但断点不起作用问题。

问题

1. isa是什么
2. tagged pointer对象的内存布局
3. 对象的isa初始化过程
4. OC对象，类，元类
5. 类别的加载，关联对象的实现
6. load方法与initialize方法

`__attribute__((deprecated))` ：标记为已弃用，表明新版本中已经不再使用该东西了，为了兼容旧版本还保留在这里，在以后的某个版本中将会被移除。业务层不应该继续使用。

NSObject定义：

```objc
OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0, 2.0)
OBJC_ROOT_CLASS
OBJC_EXPORT
@interface NSObject <NSObject> {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wobjc-interface-ivars"
    Class isa  OBJC_ISA_AVAILABILITY; //oc2.0中已弃用，将来可能会被移除。
#pragma clang diagnostic pop
}

/* OBJC_ISA_AVAILABILITY: `isa` will be deprecated or unavailable 
 * in the future */
#if !defined(OBJC_ISA_AVAILABILITY)
#   if __OBJC2__
#       define OBJC_ISA_AVAILABILITY  __attribute__((deprecated))
#   else
#       define OBJC_ISA_AVAILABILITY  /* still available */
#   endif
#endif
```

上述的一些宏定义表明oc2.0中NSObject已经不使用`Class isa;`成员变量。isa 已经移到`struct objc_object`里面去了。

NSProxy定义：

```
NS_ROOT_CLASS
@interface NSProxy <NSObject> {
    Class	isa;
}
```

### 旧版struct objc_class定义

```objc
//runtime.h

/* Types */

#if !OBJC_TYPES_DEFINED

/// An opaque type that represents a method in a class definition.
typedef struct objc_method *Method;

/// An opaque type that represents an instance variable.
typedef struct objc_ivar *Ivar;

/// An opaque type that represents a category.
typedef struct objc_category *Category;

/// An opaque type that represents an Objective-C declared property.
typedef struct objc_property *objc_property_t;

struct objc_class {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY; //旧版本isa为一个指针指向objc_class结构体

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

#endif

//objc.h
#if !OBJC_TYPES_DEFINED
/// An opaque type that represents an Objective-C class.
typedef struct objc_class *Class; 

/// Represents an instance of a class.
struct objc_object {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;
};

/// A pointer to an instance of a class.
typedef struct objc_object *id;
#endif
```

由宏OBJC_TYPES_DEFINED包裹：

```
#define OBJC_TYPES_DEFINED 1
#undef OBJC_OLD_DISPATCH_PROTOTYPES
#define OBJC_OLD_DISPATCH_PROTOTYPES 0
```

这表明上述定义都已经弃用，因此需要看新版本的了。

### 新版struct objc_class定义

```c++
//objc-runtime-new.h

struct objc_class : objc_object { //现在是继承自结构体objc_object
  // Class ISA;
  Class superclass;
  cache_t cache;             // formerly cache pointer and vtable
  class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags

  class_rw_t *data() const {
      return bits.data();
  }
  void setData(class_rw_t *newData) { 
      bits.setData(newData);
  }
  ...
  
}

//objc-private.h

struct objc_object {
private:
    isa_t isa; //objc_object中有一个isa，但类型已经变成isa_t。

public:

    // ISA() assumes this is NOT a tagged pointer object
    Class ISA();

    // rawISA() assumes this is NOT a tagged pointer object or a non pointer ISA
    Class rawISA();

    // getIsa() allows this to be a tagged pointer object
    Class getIsa();
    
    uintptr_t isaBits() const;
    
    ...
}
```

superClass：指向自己的父类

cache：用于缓存指针和 vtable，加速方法的调用

bits：用于存储类的方法、属性、遵循的协议等信息。相比于旧版本，新版本将这些信息封装到一起了。

可以看到：

1. objc_class已经是继承自objc_object了
2. isa的类型已经变了，变成了一个联合体

相比旧版本：核心思想没有变化，但实现方式变了，应该是优化了一些东西，扩充了一些功能，类结构更加合理了。

### isa

```c++
//objc-private.h

struct objc_class;
struct objc_object;

typedef struct objc_class *Class;
  typedef struct objc_object *id; //这里和旧版本一样

namespace {
    struct SideTable;
};

#include "isa.h"

union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;
#if defined(ISA_BITFIELD)
    struct {
        ISA_BITFIELD;  // defined in isa.h
    };
#endif
};
```

ISA_BITFIELD宏定义：

```
    // extra_rc must be the MSB-most field (so it matches carry/overflow flags)
    // nonpointer must be the LSB (fixme or get rid of it)
    // shiftcls must occupy the same bits that a real class pointer would
    // bits + RC_ONE is equivalent to extra_rc + 1
    // RC_HALF is the high bit of extra_rc (i.e. half of its range)

    // future expansion:
    // uintptr_t fast_rr : 1;     // no r/r overrides
    // uintptr_t lock : 2;        // lock for atomic property, @synch
    // uintptr_t extraBytes : 1;  // allocated with extra bytes

# if __arm64__
#   define ISA_MASK        0x0000000ffffffff8ULL
#   define ISA_MAGIC_MASK  0x000003f000000001ULL
#   define ISA_MAGIC_VALUE 0x000001a000000001ULL
#   define ISA_BITFIELD                                                      \
      uintptr_t nonpointer        : 1;                                       \
      uintptr_t has_assoc         : 1;                                       \
      uintptr_t has_cxx_dtor      : 1;                                       \
      uintptr_t shiftcls          : 33; /*MACH_VM_MAX_ADDRESS 0x1000000000*/ \
      uintptr_t magic             : 6;                                       \
      uintptr_t weakly_referenced : 1;                                       \
      uintptr_t deallocating      : 1;                                       \
      uintptr_t has_sidetable_rc  : 1;                                       \
      uintptr_t extra_rc          : 19
#   define RC_ONE   (1ULL<<45)
#   define RC_HALF  (1ULL<<18)
```

展开一下：

```c++
union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls; //一个指针指向objc_class结构体。这样就可以兼容旧版本。
    uintptr_t bits;
    //这里展开的是arm64的
    struct {
        uintptr_t nonpointer        : 1; //LSB。0 is raw isa, 1 is non-pointer isa.                                      
      	uintptr_t has_assoc         : 1; //对象是否有关联对象                                      
        uintptr_t has_cxx_dtor      : 1; //对象是否有C++或ARC析构函数                                      
        uintptr_t shiftcls          : 33; /*MACH_VM_MAX_ADDRESS 0x1000000000*/ //Class地址。
        uintptr_t magic             : 6;  //一个魔法数。用于区分是否初始化                                     
        uintptr_t weakly_referenced : 1;  //是否有弱引用指针。                                      
        uintptr_t deallocating      : 1; //是否正在被销毁                                      
        uintptr_t has_sidetable_rc  : 1; //是否有sidetable引用计数。对象的引用计数太大而不能在内部保存时会将引用计数保存到sidetable                                      
        uintptr_t extra_rc          : 19 //MSB。加1后就是对象的引用计数。由于只有19位所以只有对象的引用计数不太大的时候才保存在这里
    };
};
```

isa 是什么：isa 的数据结构是一个 isa_t 联合体，其中包含其所属的 Class 的地址，通过访问对象的 isa，就可以获取到指向其所属 Class 的指针。

isa 的作用：指向该对象所属类的地址。用于查找对象（或类对象）所属的类（或元类）。

### Tagged Pointer

```
// Tagged pointer objects.

#if __LP64__
#define OBJC_HAVE_TAGGED_POINTERS 1
#endif
```

Tagged pointer对象是在64位系统才有的。

objc_tag_index_t：

用于查找对应的对象类型。

```c
// Tagged pointer layout and usage is subject to change on different OS versions.

// Tag indexes 0..<7 have a 60-bit payload.
// Tag index 7 is reserved.
// Tag indexes 8..<264 have a 52-bit payload.
// Tag index 264 is reserved.

#if __has_feature(objc_fixed_enum)  ||  __cplusplus >= 201103L
enum objc_tag_index_t : uint16_t
#else
typedef uint16_t objc_tag_index_t;
enum
#endif
{
    // 60-bit payloads
    OBJC_TAG_NSAtom            = 0, 
    OBJC_TAG_1                 = 1, 
    OBJC_TAG_NSString          = 2, 
    OBJC_TAG_NSNumber          = 3, 
    OBJC_TAG_NSIndexPath       = 4, 
    OBJC_TAG_NSManagedObjectID = 5, 
    OBJC_TAG_NSDate            = 6,

    // 60-bit reserved
    OBJC_TAG_RESERVED_7        = 7, 

    // 52-bit payloads
    OBJC_TAG_Photos_1          = 8,
    OBJC_TAG_Photos_2          = 9,
    OBJC_TAG_Photos_3          = 10,
    OBJC_TAG_Photos_4          = 11,
    OBJC_TAG_XPC_1             = 12,
    OBJC_TAG_XPC_2             = 13,
    OBJC_TAG_XPC_3             = 14,
    OBJC_TAG_XPC_4             = 15,
    OBJC_TAG_NSColor           = 16,
    OBJC_TAG_UIColor           = 17,
    OBJC_TAG_CGColor           = 18,
    OBJC_TAG_NSIndexSet        = 19,

    OBJC_TAG_First60BitPayload = 0, 
    OBJC_TAG_Last60BitPayload  = 6, 
    OBJC_TAG_First52BitPayload = 8, 
    OBJC_TAG_Last52BitPayload  = 263, 

    OBJC_TAG_RESERVED_264      = 264
};
#if __has_feature(objc_fixed_enum)  &&  !defined(__cplusplus)
typedef enum objc_tag_index_t objc_tag_index_t;
#endif
```

tag bit位：

tag bit位在x86_64和arm64是不同的

```c
#if (TARGET_OS_OSX || TARGET_OS_IOSMAC) && __x86_64__
    // 64-bit Mac - tag bit is LSB
#   define OBJC_MSB_TAGGED_POINTERS 0 //tag bit在x86_64上是最低有效位
#else
    // Everything else - tag bit is MSB
#   define OBJC_MSB_TAGGED_POINTERS 1 //tag bit在arm64上是最高有效位
#endif

#if OBJC_MSB_TAGGED_POINTERS
#   define _OBJC_TAG_MASK (1UL<<63)  //用于判断是否是Tagged pointer
#   define _OBJC_TAG_INDEX_SHIFT 60
#   define _OBJC_TAG_SLOT_SHIFT 60
#   define _OBJC_TAG_PAYLOAD_LSHIFT 4
#   define _OBJC_TAG_PAYLOAD_RSHIFT 4
#   define _OBJC_TAG_EXT_MASK (0xfUL<<60)
#   define _OBJC_TAG_EXT_INDEX_SHIFT 52
#   define _OBJC_TAG_EXT_SLOT_SHIFT 52
#   define _OBJC_TAG_EXT_PAYLOAD_LSHIFT 12
#   define _OBJC_TAG_EXT_PAYLOAD_RSHIFT 12
#else
#   define _OBJC_TAG_MASK 1UL
#   define _OBJC_TAG_INDEX_SHIFT 1
#   define _OBJC_TAG_SLOT_SHIFT 0
#   define _OBJC_TAG_PAYLOAD_LSHIFT 0
#   define _OBJC_TAG_PAYLOAD_RSHIFT 4
#   define _OBJC_TAG_EXT_MASK 0xfUL
#   define _OBJC_TAG_EXT_INDEX_SHIFT 4
#   define _OBJC_TAG_EXT_SLOT_SHIFT 4
#   define _OBJC_TAG_EXT_PAYLOAD_LSHIFT 0
#   define _OBJC_TAG_EXT_PAYLOAD_RSHIFT 12
#endif
```

示例：

```objc
- (void)test_tagged_pointer {
    Animal *obj = [[Animal alloc] init];
    NSNumber *numberInt = [NSNumber numberWithInt:1];
    NSNumber *numberShort = [NSNumber numberWithShort:1];
    NSString *str = [NSString stringWithFormat:@"%s", "abcdefghi"];
    NSString *greaterStr = [NSString stringWithFormat:@"%s", "abcdefghij"];
    
    uintptr_t numberIntDecode = _objc_decodeTaggedPointer((__bridge const void * _Nullable)(numberInt));
    uintptr_t numberShortDecode = _objc_decodeTaggedPointer((__bridge const void * _Nullable)(numberShort));
    uintptr_t strDecode = _objc_decodeTaggedPointer((__bridge const void * _Nullable)(str));
    
    NSLog(@"obj:%p", obj);
    NSLog(@"numberInt:%p  decode-->%#lx", numberInt, numberIntDecode);
    NSLog(@"numberShort:%p  decode-->%#lx", numberShort, numberShortDecode);
    NSLog(@"str:%p  decode-->%lx", str, strDecode);
    NSLog(@"greaterStr:%p", greaterStr);

    printf("\n");
    printf("\n");
}
```

真机运行：

```
2020-07-20 18:13:43.258843+0800 runtime方法使用Demo[1252:281045] obj:0x2832f8040
2020-07-20 18:13:43.258898+0800 runtime方法使用Demo[1252:281045] numberInt:0xb7427616a082ae26  decode-->0xb000000000000012
2020-07-20 18:13:43.258945+0800 runtime方法使用Demo[1252:281045] numberShort:0xb7427616a082ae25  decode-->0xb000000000000011
2020-07-20 18:13:43.258988+0800 runtime方法使用Demo[1252:281045] str:0xa7ca783ea4d8fa2d  decode-->a0880e28045a5419
2020-07-20 18:13:43.259030+0800 runtime方法使用Demo[1252:281045] greaterStr:0x2830e9420
```

最高位为1说明是Tagged pointer对象。是Tagged pointer对象后，我们才可以解码该地址，解码后可以看到TaggedPointer的真实布局。

最高4位为b(1011)：

101 = 3，查找上面的OBJC_TAG枚举得知为OBJC_TAG_NSNumber，即为NSNumber类型。

其他60位就是payload了，payload里面包含真正的数据以及数据的一些其他信息比如长度等。

标志位--OBJC_TAG--数据--元数据

### 参考

[load 方法全程跟踪](https://www.desgard.com/iOS-Source-Probe/Objective-C/Runtime/load 方法全程跟踪.html)

[读 objc4 源码，深入理解 Objective-C Runtime](https://www.jianshu.com/p/e3d521b2cb43)

[探秘Runtime - Runtime源码分析](https://www.jianshu.com/p/3019605a4fc9)

[Tagged Pointer小记](https://crmo.github.io/2018/07/04/Tagged Pointer小记/)  讲了tagged的内存布局

[Objective-C中的伪指针](https://jinxuebin.cn/2019/06/Objective-C中的伪指针/)

[【译】采用Tagged Pointer的字符串](http://www.cocoachina.com/ios/20150918/13449.html) 外国人写的，非常棒，深入探讨了tagged pointer的实现细节

[Let's Build Tagged Pointers](https://www.mikeash.com/pyblog/friday-qa-2012-07-27-lets-build-tagged-pointers.html)  原理性的一些讲解，深入的可以看看。

[Non-pointer isa](http://www.sealiesoftware.com/blog/archive/2013/09/24/objc_explain_Non-pointer_isa.html)  很详细的解释了OC对象的isa字段为什么不再是一个指针类型。主要是为了做优化。

内存对齐：

[32位与64位系统基本数据类型的字节数](https://blog.csdn.net/u012611878/article/details/52455576)

[C/C++内存对齐详解](https://zhuanlan.zhihu.com/p/30007037)



[探寻Objective-C引用计数本质](https://crmo.github.io/2018/05/26/探寻Objective-C引用计数本质/)

[Objective-C 引用计数原理](https://www.jianshu.com/p/2acbde294266)  主要是tagged部分

