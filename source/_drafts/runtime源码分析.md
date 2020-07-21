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
5. 类对象是何时创建的，由谁创建。
6. 类别的加载，关联对象的实现
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
  class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags //bits里面包含了class_rw_t地址和是否自定义retain/release.../alloc等方法标志。

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
    methodizeClass(cls, previously); //给rw的methods等成员赋值。

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

    method_array_t methods;
    property_array_t properties;
    protocol_array_t protocols;

    Class firstSubclass;
    Class nextSiblingClass;

    char *demangledName;
    ...
}
```

保存了class_ro_t指针，方法列表，属性列表，遵守的协议列表。

ro存储的是**当前类在编译期就已经确定的属性、方法以及遵循的协议**。ro是只读的不可更改。



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
    method_list_t * baseMethodList; //也有方法列表，这说明和class_rw_t应该是有区别的
    protocol_list_t * baseProtocols; //遵守的协议列表
    const ivar_list_t * ivars; //实例变量列表,指向一个只读区域。防止外部更改。

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties; //属性列表

    // This field exists only when RO_HAS_SWIFT_INITIALIZER is set.
    _objc_swiftMetadataInitializer __ptrauth_objc_method_list_imp _swiftMetadataInitializer_NEVER_USE[0];
		...
}
```

##### TODO：struct class_ro_t和struct class_rw_t的初始化需要深究一下

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

### 对象的创建及isa的初始化

在调用`Person *per = [[Person alloc] init];` ，调用alloc时会调用到callAlloc

```c
+ (id)alloc {
    return _objc_rootAlloc(self);
}

// Base class implementation of +alloc. cls is not nil.
// Calls [cls allocWithZone:nil].
id
_objc_rootAlloc(Class cls)
{
    return callAlloc(cls, false/*checkNil*/, true/*allocWithZone*/);
}

// Call [cls alloc] or [cls allocWithZone:nil], with appropriate 
// shortcutting optimizations.
static ALWAYS_INLINE id
callAlloc(Class cls, bool checkNil, bool allocWithZone=false)
{
#if __OBJC2__
    if (slowpath(checkNil && !cls)) return nil;
    if (fastpath(!cls->ISA()->hasCustomAWZ())) {
        return _objc_rootAllocWithZone(cls, nil);
    }
#endif

    // No shortcuts available.
    if (allocWithZone) {
        return ((id(*)(id, SEL, struct _NSZone *))objc_msgSend)(cls, @selector(allocWithZone:), nil);
    }
    return ((id(*)(id, SEL))objc_msgSend)(cls, @selector(alloc));
}
```

hasCustomAWZ表示class or superclass has default alloc/allocWithZone: implementation。一般情况下我们是不会重写这两个方法的。

所以在OC2.0中会走_objc_rootAllocWithZone逻辑：

```c
id
_objc_rootAllocWithZone(Class cls, malloc_zone_t *zone __unused)
{
    // allocWithZone under __OBJC2__ ignores the zone parameter
    return _class_createInstanceFromZone(cls, 0, nil,
                                         OBJECT_CONSTRUCT_CALL_BADALLOC); //zone在OC2.0其实是被忽略的，总是为nil。
}

static ALWAYS_INLINE id
_class_createInstanceFromZone(Class cls, size_t extraBytes, void *zone,
                              int construct_flags = OBJECT_CONSTRUCT_NONE,
                              bool cxxConstruct = true,
                              size_t *outAllocatedSize = nil)
{
    ASSERT(cls->isRealized());

    // Read class's info bits all at once for performance
    bool hasCxxCtor = cxxConstruct && cls->hasCxxCtor();
    bool hasCxxDtor = cls->hasCxxDtor();
    bool fast = cls->canAllocNonpointer();
    size_t size;

    size = cls->instanceSize(extraBytes); //获取实例大小
    if (outAllocatedSize) *outAllocatedSize = size;

    id obj;
    if (zone) {
        obj = (id)malloc_zone_calloc((malloc_zone_t *)zone, 1, size);
    } else {
        obj = (id)calloc(1, size); //申请一块内存空间并初始化为0
    }
    if (slowpath(!obj)) { //堆上内存不足，申请失败。
        if (construct_flags & OBJECT_CONSTRUCT_CALL_BADALLOC) {
            return _objc_callBadAllocHandler(cls);
        }
        return nil;
    }

    if (!zone && fast) {
        obj->initInstanceIsa(cls, hasCxxDtor);
    } else {
        // Use raw pointer isa on the assumption that they might be
        // doing something weird with the zone or RR.
        obj->initIsa(cls); //初始化isa
    }

    if (fastpath(!hasCxxCtor)) {
        return obj;
    }

    construct_flags |= OBJECT_CONSTRUCT_FREE_ONFAILURE;
    return object_cxxConstructFromClass(obj, cls, construct_flags);
}
```

这里重点是`size = cls->instanceSize(extraBytes);`  获取实例大小：

```c++
size_t instanceSize(size_t extraBytes) const {
    if (fastpath(cache.hasFastInstanceSize(extraBytes))) {
        return cache.fastInstanceSize(extraBytes);
    }

    size_t size = alignedInstanceSize() + extraBytes;
    // CF requires all objects be at least 16 bytes.
    if (size < 16) size = 16;
    return size;
}

// Class's ivar size rounded up to a pointer-size boundary.
uint32_t alignedInstanceSize() const {
    return word_align(unalignedInstanceSize());
}
```

会先获取到实例实际所需的内存大小并进行内存对齐，然后至少分配16字节内存。

实例所需的内存大小：

```c++
// May be unaligned depending on class's ivars.
uint32_t unalignedInstanceSize() const {
    ASSERT(isRealized());
    return data()->ro->instanceSize;
}

class_rw_t *data() const {
    return bits.data();
}
```

可以看到是调用了`struct objc_class`的data方法先获取到class_rw_t-->class_ro_t-->instanceSize。实例的大小在类加载的时候就已经计算好了，这里只是获取一下。

分配内存后，接着是初始化isa：

```c
inline void 
objc_object::initIsa(Class cls)
{
    initIsa(cls, false, false); //默认情况下都是isa都是nonpointer
}

inline void 
objc_object::initIsa(Class cls, bool nonpointer, bool hasCxxDtor) 
{ 
    ASSERT(!isTaggedPointer()); 
    
    if (!nonpointer) {
        isa = isa_t((uintptr_t)cls);
    } else {
        ASSERT(!DisableNonpointerIsa);
        ASSERT(!cls->instancesRequireRawIsa());

        isa_t newisa(0);

#if SUPPORT_INDEXED_ISA
        ASSERT(cls->classArrayIndex() > 0);
        newisa.bits = ISA_INDEX_MAGIC_VALUE;
        // isa.magic is part of ISA_MAGIC_VALUE
        // isa.nonpointer is part of ISA_MAGIC_VALUE
        newisa.has_cxx_dtor = hasCxxDtor;
        newisa.indexcls = (uintptr_t)cls->classArrayIndex();
#else
        newisa.bits = ISA_MAGIC_VALUE; 
        // isa.magic is part of ISA_MAGIC_VALUE
        // isa.nonpointer is part of ISA_MAGIC_VALUE
        newisa.has_cxx_dtor = hasCxxDtor;
        newisa.shiftcls = (uintptr_t)cls >> 3; //cls是一个指针指向一个类对象的地址，这里将它的值（也就是类对象地址）右移3位，保存到shiftcls字段。
#endif

        // This write must be performed in a single store in some cases
        // (for example when realizing a class because other threads
        // may simultaneously try to use the class).
        // fixme use atomics here to guarantee single-store and to
        // guarantee memory order w.r.t. the class index table
        // ...but not too atomic because we don't want to hurt instantiation
        isa = newisa;
    }
}
```

大致流程：

1. 如果是非nonpointer则isa的值就是类对象cls的地址，使用的是isa_t里面的cls指针。由于类对象的地址基本上不会超过40位，所以有很多位是空的。为了优化内存空间，默认情况下isa都是nonpointer的。因此一般不会走到这个分支。
2. 如果是nonpointer，则isa就是经过优化的，使用的是isa_t里面的匿名结构体。
3. 使用ISA_MAGIC_VALUE初始化bits，这里其实就是设置了匿名结构体里的nonpointer为1，设置了magic为X。
4. 设置shiftcls为(uintptr_t)cls >> 3。cls是一个指针指向一个类对象的地址，这里将它的值（也就是类对象地址）右移3位，保存到shiftcls字段。这里为啥可以丢弃最后3位，也是因为一个对象的地址这个数值其实是占不满64位的最后3位是无用信息可以丢弃。有兴趣的可以深究下。
5. 将newisa变量赋值给isa字段，完成isa的初始化。

由此可知如果isa的最低位为1，那么isa就是nonpointer，否则为纯pointer。对于nonpointer的isa里面除了保存类对象地址外，还会保存很多其他信息比如对象的引用计数，是否有关联对象，是否有weak指针等，这样就可以减少一些哈希表的操作，提高了效率。

### Tagged Pointer对象

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

011 = 3，查找上面的OBJC_TAG枚举得知为OBJC_TAG_NSNumber，即为NSNumber类型。

其他60位就是payload了，payload里面包含真正的数据以及数据的一些其他信息比如长度等。

标志位--OBJC_TAG--数据--元数据

### 参考

[load 方法全程跟踪](https://www.desgard.com/iOS-Source-Probe/Objective-C/Runtime/load 方法全程跟踪.html)

[读 objc4 源码，深入理解 Objective-C Runtime](https://www.jianshu.com/p/e3d521b2cb43)  提出了一些问题可以看看

[RuntimePDF](https://github.com/DeveloperErenLiu/RuntimePDF)  runtime的系列文章

[iOS底层原理总结 - 探寻Runtime本质（二）](https://www.jianshu.com/p/27ee04f3ed7b)  runtime的系列文章。

[深入解析 ObjC 中方法的结构](https://draveness.me/method-struct/)  也是oc类的系列文章



tagged pointer

[Tagged Pointer小记](https://crmo.github.io/2018/07/04/Tagged Pointer小记/)  讲了tagged的内存布局

[Objective-C中的伪指针](https://jinxuebin.cn/2019/06/Objective-C中的伪指针/)

[【译】采用Tagged Pointer的字符串](http://www.cocoachina.com/ios/20150918/13449.html) 外国人写的，非常棒，深入探讨了tagged pointer的实现细节

[Let's Build Tagged Pointers](https://www.mikeash.com/pyblog/friday-qa-2012-07-27-lets-build-tagged-pointers.html)  原理性的一些讲解，深入的可以看看。

[Non-pointer isa](http://www.sealiesoftware.com/blog/archive/2013/09/24/objc_explain_Non-pointer_isa.html)  很详细的解释了OC对象的isa字段为什么不再是一个指针类型。主要是为了做优化。



内存对齐：

[32位与64位系统基本数据类型的字节数](https://blog.csdn.net/u012611878/article/details/52455576)

[C/C++内存对齐详解](https://zhuanlan.zhihu.com/p/30007037)



引用计数

[探寻Objective-C引用计数本质](https://crmo.github.io/2018/05/26/探寻Objective-C引用计数本质/)

[Objective-C 引用计数原理](https://www.jianshu.com/p/2acbde294266)  主要是tagged部分

