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
2. OC对象，类，元类
3. 类别的加载，关联对象的实现
4. load方法与initialize方法

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

上述的一些宏定义表明oc2.0中NSObject已经不使用`Class isa;`成员变量。

NSProxy定义：

```
NS_ROOT_CLASS
@interface NSProxy <NSObject> {
    Class	isa;
}
```

### 旧版struct objc_class

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

### 新版struct objc_class

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

展开一下：

```c++
union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls; //一个指针指向objc_class结构体。这样就可以兼容旧版本。
    uintptr_t bits;
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

对于非指针isa，系统提供了几种

```c
// Extract isa pointer from an isa field.
// (Class)(isa & mask) == class pointer
OBJC_EXPORT const uintptr_t objc_debug_isa_class_mask
    OBJC_AVAILABLE(10.10, 7.0, 9.0, 1.0, 2.0);

// Extract magic cookie from an isa field.
// (isa & magic_mask) == magic_value
OBJC_EXPORT const uintptr_t objc_debug_isa_magic_mask
    OBJC_AVAILABLE(10.10, 7.0, 9.0, 1.0, 2.0);
OBJC_EXPORT const uintptr_t objc_debug_isa_magic_value
    OBJC_AVAILABLE(10.10, 7.0, 9.0, 1.0, 2.0);
```

通过(isa & mask)可以得到class的地址。

问题：

对于tagged pointer对象的内存布局。

### 参考

[load 方法全程跟踪](https://www.desgard.com/iOS-Source-Probe/Objective-C/Runtime/load 方法全程跟踪.html)

[读 objc4 源码，深入理解 Objective-C Runtime](https://www.jianshu.com/p/e3d521b2cb43)

[探秘Runtime - Runtime源码分析](https://www.jianshu.com/p/3019605a4fc9)

[objc 中的Tagged Pointer的应用](https://www.jianshu.com/p/31678563d00d)

[Non-pointer isa](http://www.sealiesoftware.com/blog/archive/2013/09/24/objc_explain_Non-pointer_isa.html)  很详细的解释了OC对象的isa字段为什么不再是一个指针类型。主要是为了做优化。

[探寻Objective-C引用计数本质](https://crmo.github.io/2018/05/26/探寻Objective-C引用计数本质/)

[Objective-C 引用计数原理](https://www.jianshu.com/p/2acbde294266)  主要是tagged部分

内存对齐：

[32位与64位系统基本数据类型的字节数](https://blog.csdn.net/u012611878/article/details/52455576)

[C/C++内存对齐详解](https://zhuanlan.zhihu.com/p/30007037)

