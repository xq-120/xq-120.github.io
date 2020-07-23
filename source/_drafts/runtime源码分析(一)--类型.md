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
2. tagged pointer对象是什么以及它的布局规则
3. 对象的isa初始化过程
4. OC对象，类，元类
5. 类的加载过程，特别是类对象的创建，方法，属性，协议的获取
6. 类别的加载过程，关联对象的实现
7. load方法与initialize方法

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

```objc
NS_ROOT_CLASS
@interface NSProxy <NSObject> {
    Class	isa;
}
```

### 旧版定义

#### struct objc_object

```objc
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

#### struct objc_class

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
```

由宏OBJC_TYPES_DEFINED包裹：

```
#define OBJC_TYPES_DEFINED 1
#undef OBJC_OLD_DISPATCH_PROTOTYPES
#define OBJC_OLD_DISPATCH_PROTOTYPES 0
```

这表明上述定义都已经弃用，核心思想可能相同但实现细节已经不同了，因此需要看新版本的定义了。其他源码库也是一样，如果你只是想看一下大概原理你可以看旧版，但新版里的一些性能优化、新增功能、代码的优化重构你就不知道了，所以建议能看新版本就看新版本，当然因为上面的原因新版本难度会更大一些。

### 新版定义

#### struct objc_object

```objc
//objc-private.h

struct objc_object {
private:
    isa_t isa; //objc_object中有一个isa，但类型已经变成isa_t(union类型)

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

objc_object中有一个isa，相比旧版本，新版本类型已经变成isa_t(union类型)。有了isa就可以找到对象所属的类。

#### struct objc_class

```c++
//objc-runtime-new.h

struct objc_class : objc_object { //现在是继承自结构体objc_object
  // Class ISA;
  Class superclass;
  cache_t cache;             // formerly cache pointer and vtable
  class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags //bits里面包含了class_rw_t地址和是否自定义retain/release.../alloc等方法，是否已经realize标志。

  class_rw_t *data() const {
      return bits.data();
  }
  void setData(class_rw_t *newData) { 
      bits.setData(newData);
  }
  ...
  
}
```

superClass：指向自己的父类

cache：用于缓存指针和 vtable，加速方法的调用

bits：用于存储类的方法、属性、遵循的协议等信息。相比于旧版本，新版本将这些信息封装到一起了。

类里面也有isa，这样它也可以找到它所属的类，因此类也是一个对象，通常称为类对象。

相比旧版本：核心思想没有变化，但实现方式变了，应该是优化了一些东西，扩充了一些功能，类结构更加合理了。

#### struct class_data_bits_t

```c++
struct class_data_bits_t {
    friend objc_class;

    // Values are the FAST_ flags above.
    uintptr_t bits;
private:
		...
public:

    class_rw_t* data() const {
        return (class_rw_t *)(bits & FAST_DATA_MASK); //取bits的第[4,47]位值，得到一个地址。
    }
    void setData(class_rw_t *newData)
    {
        ASSERT(!data()  ||  (newData->flags & (RW_REALIZING | RW_FUTURE)));
        // Set during realization or construction only. No locking needed.
        // Use a store-release fence because there may be concurrent
        // readers of data and data's contents.
        uintptr_t newBits = (bits & ~FAST_DATA_MASK) | (uintptr_t)newData;
        atomic_thread_fence(memory_order_release);
        bits = newBits;
    }
  
  	// Get the class's ro data, even in the presence of concurrent realization.
    // fixme this isn't really safe without a compiler barrier at least
    // and probably a memory barrier when realizeClass changes the data field
    const class_ro_t *safe_ro() {
        class_rw_t *maybe_rw = data(); //data方法返回的不一定就是class_rw_t。
        if (maybe_rw->flags & RW_REALIZED) {
            // maybe_rw is rw
            return maybe_rw->ro;
        } else {
            // maybe_rw is actually ro
            return (class_ro_t *)maybe_rw;
        }
    }
  
    ...
}
```

该结构体主要是用来获取class_rw_t的。

**在编译期间 **class_data_bits_t bits其实指向的是一个 `class_ro_t *` 指针，realizeClassWithoutSwift之后bits才指向class_rw_t。

可以看函数realizeClassWithoutSwift：

```c
static Class realizeClassWithoutSwift(Class cls, Class previously)
{
    runtimeLock.assertLocked();

    const class_ro_t *ro;
    class_rw_t *rw;
    Class supercls;
    Class metacls;
    bool isMeta;

    if (!cls) return nil;
    if (cls->isRealized()) return cls;
    ASSERT(cls == remapClass(cls));

    // fixme verify class is not in an un-dlopened part of the shared cache?

    ro = (const class_ro_t *)cls->data(); //这里还是一个class_ro_t指针
    if (ro->flags & RO_FUTURE) {
        // This was a future class. rw data is already allocated.
        rw = cls->data();
        ro = cls->data()->ro;
        cls->changeInfo(RW_REALIZED|RW_REALIZING, RW_FUTURE);
    } else {
        // Normal class. Allocate writeable class data.
        rw = (class_rw_t *)calloc(sizeof(class_rw_t), 1);
        rw->ro = ro;
        rw->flags = RW_REALIZED|RW_REALIZING;
        cls->setData(rw); //创建一个可读写的class_rw_t并设置。
    }
		
    ...
    
    // Attach categories
    methodizeClass(cls, previously); //给rw的methods等成员赋值及添加类别。

    return cls;
}
```

#### struct class_rw_t

看名字rw应该是read-write的意思

```c++
struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags; //realize的状态。	
    uint16_t version;
    uint16_t witness;

    const class_ro_t *ro; //指向一个只读区域

    method_array_t methods; //二维数组
    property_array_t properties;
    protocol_array_t protocols;

    Class firstSubclass;
    Class nextSiblingClass;

    char *demangledName;
    ...
}

class method_array_t : 
    public list_array_tt<method_t, method_list_t> 
{
		...
};

/***********************************************************************
* list_array_tt<Element, List>
* Generic implementation for metadata that can be augmented by categories.
*
* Element is the underlying metadata type (e.g. method_t)
* List is the metadata's list type (e.g. method_list_t)
*
* A list_array_tt has one of three values:
* - empty
* - a pointer to a single list
* - an array of pointers to lists
*
* countLists/beginLists/endLists iterate the metadata lists
* count/begin/end iterate the underlying metadata elements
**********************************************************************/
template <typename Element, typename List>
class list_array_tt {
	... 
}
```

保存了class_ro_t指针，方法列表，属性列表，遵守的协议列表。

ro存储的是**当前类在编译期就已经确定的属性、方法以及遵循的协议**。ro是只读的不可更改。rw里的方法等是可以更改的。

#### struct class_ro_t

看名字ro应该是read-only的意思

```c++
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize; //实例的大小
#ifdef __LP64__
    uint32_t reserved;
#endif

    const uint8_t * ivarLayout;
    
    const char * name;
    method_list_t * baseMethodList; //也有方法列表，这说明和class_rw_t是有区别的
    protocol_list_t * baseProtocols; //遵守的协议列表
    const ivar_list_t * ivars; //实例变量列表,指向一个只读区域。防止外部更改。

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties; //属性列表

    // This field exists only when RO_HAS_SWIFT_INITIALIZER is set.
    _objc_swiftMetadataInitializer __ptrauth_objc_method_list_imp _swiftMetadataInitializer_NEVER_USE[0];
		...
}

struct method_list_t : entsize_list_tt<method_t, method_list_t, 0x3> {
	...
};

struct ivar_list_t : entsize_list_tt<ivar_t, ivar_list_t, 0> {
    ...
};

struct property_list_t : entsize_list_tt<property_t, property_list_t, 0> {
};

/***********************************************************************
* entsize_list_tt<Element, List, FlagMask>
* Generic implementation of an array of non-fragile structs.
*
* Element is the struct type (e.g. method_t)
* List is the specialization of entsize_list_tt (e.g. method_list_t)
* FlagMask is used to stash extra bits in the entsize field
*   (e.g. method list fixup markers)
**********************************************************************/
template <typename Element, typename List, uint32_t FlagMask>
struct entsize_list_tt {
  uint32_t entsizeAndFlags;
  uint32_t count;
  Element first;
		
  ....
}
```

#### union isa_t

isa的类型定义：

```c++
//objc-private.h

struct objc_class;
struct objc_object;

typedef struct objc_class *Class; //指向一个类对象
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
        uintptr_t extra_rc          : 19 //MSB。对象的引用计数超过1时会存在这里。因此extra_rc加1后就是对象的引用计数。由于只有19位所以只有对象的引用计数不太大的时候才保存在这里
    };
};
```

isa 是什么：isa 的数据结构是一个 isa_t 联合体，其中包含其所属的 Class 的地址，通过访问对象的 isa，就可以获取到指向其所属 Class 的指针。

isa 的作用：指向该对象所属类的地址。用于查找对象（或类对象）所属的类（或元类）。

#### struct method_t

别名:

```c
typedef struct method_t *Method;
typedef struct ivar_t *Ivar;
typedef struct category_t *Category;
typedef struct property_t *objc_property_t;
```

定义：

```c++
struct method_t {
    SEL name;
    const char *types;
    MethodListIMP imp; //IMP类型。

    struct SortBySELAddress :
        public std::binary_function<const method_t&,
                                    const method_t&, bool>
    {
        bool operator() (const method_t& lhs,
                         const method_t& rhs)
        { return lhs.name < rhs.name; }
    };
};
```

我们定义的方法是以一个struct method_t结构体表示的，包含：

name：方法的名称

types：方法的返回值参数类型。一个将返回值类型，参数类型编码后的C字符串。具体是怎么编码的可以查看官方文档。

imp：方法的地址

#### struct ivar_t

```c++
struct ivar_t {
#if __x86_64__
    // *offset was originally 64-bit on some x86_64 platforms.
    // We read and write only 32 bits of it.
    // Some metadata provides all 64 bits. This is harmless for unsigned 
    // little-endian values.
    // Some code uses all 64 bits. class_addIvar() over-allocates the 
    // offset for their benefit.
#endif
    int32_t *offset;
    const char *name;
    const char *type;
    // alignment is sometimes -1; use alignment() instead
    uint32_t alignment_raw;
    uint32_t size;

    uint32_t alignment() const {
        if (alignment_raw == ~(uint32_t)0) return 1U << WORD_SHIFT;
        return 1 << alignment_raw;
    }
};
```

我们定义的实例变量是以一个struct ivar_t结构体表示的，包含：

name：实例变量的名称

type：实例变量的类型

size：实例变量的大小

注意这里是不会保存实例变量的值的，实例变量的值是保存在实例对象里面的，不是保存在类里面的。

#### struct property_t

```c
struct property_t {
    const char *name;
    const char *attributes;
};
```

我们定义的属性是以一个struct property_t结构体表示的，包含：

name：属性的名称

attributes：属性的修饰符。比如atomic，strong，readwrite等。

#### struct objc_selector

别名：

```c
/// An opaque type that represents a method selector.
typedef struct objc_selector *SEL;
```

源码里面没有看到定义，但基本上可以将 SEL 理解为一个 char* 指针。代表一个方法的名称。

#### IMP

```c
/// A pointer to the function of a method implementation. 
#if !OBJC_OLD_DISPATCH_PROTOTYPES
typedef void (*IMP)(void /* id, SEL, ... */ ); //这个
#else
typedef id _Nullable (*IMP)(id _Nonnull, SEL _Nonnull, ...); 
#endif
```

IMP就是一个函数指针，指向一个函数地址。

#### struct protocol_t

```c++
struct protocol_t : objc_object {
    const char *mangledName; //协议名称
    struct protocol_list_t *protocols; //遵守的其他协议列表
    method_list_t *instanceMethods; //实例方法列表
    method_list_t *classMethods; //类方法列表
    method_list_t *optionalInstanceMethods; //可选实例方法列表
    method_list_t *optionalClassMethods; //可选类方法列表
    property_list_t *instanceProperties; //实例属性列表
    uint32_t size;   // sizeof(protocol_t)
    uint32_t flags;
    // Fields below this point are not always present on disk.
    const char **_extendedMethodTypes;
    const char *_demangledName;
    property_list_t *_classProperties; //类属性列表
    
    ...
}
```

我们定义的协议是以一个struct protocol_t结构体表示的。

#### struct category_t

```c++
struct category_t {
    const char *name;
    classref_t cls;
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
    // Fields below this point are not always present on disk.
    struct property_list_t *_classProperties;

    method_list_t *methodsForMeta(bool isMeta) {
        if (isMeta) return classMethods;
        else return instanceMethods;
    }

    property_list_t *propertiesForMeta(bool isMeta, struct header_info *hi);
    
    protocol_list_t *protocolsForMeta(bool isMeta) {
        if (isMeta) return nullptr;
        else return protocols;
    }
};
```

别名：`typedef struct category_t *Category;`