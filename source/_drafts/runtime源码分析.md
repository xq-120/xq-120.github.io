---
title: runtime源码分析
tags:
  - runtime
categories:
  - OC
comments: true
date: 2020-07-18 23:32:40
---

版本：`objc4-781`

`TARGETS -> debug-objc -> Build Settings -> Enable Hardened Runtime = NO`，解决可编译但断点不起作用问题。

## 问题

1. isa是什么
2. tagged pointer对象是什么以及它的布局规则
3. 对象的创建及isa初始化过程
4. OC对象，类，元类
5. 类的加载过程，特别是类对象的创建，方法，属性，协议的获取
6. 类别的加载过程，关联对象的实现
7. load方法与initialize方法
8. load与initialize是否有线程安全问题？

## 对象的创建及isa的初始化

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
    bool fast = cls->canAllocNonpointer(); //TODO:这里没看懂怎么得来的，不过看命名大多数情况下应该是true
    size_t size;

    size = cls->instanceSize(extraBytes); //获取实例大小
    if (outAllocatedSize) *outAllocatedSize = size;

    id obj;
    if (zone) {
        obj = (id)malloc_zone_calloc((malloc_zone_t *)zone, 1, size);
    } else {
        obj = (id)calloc(1, size); //申请1个大小为size的内存空间并初始化为0
    }
    if (slowpath(!obj)) { //堆上内存不足，申请失败。
        if (construct_flags & OBJECT_CONSTRUCT_CALL_BADALLOC) {
            return _objc_callBadAllocHandler(cls);
        }
        return nil;
    }

    if (!zone && fast) { //一般情况下我们的对象走initInstanceIsa
        obj->initInstanceIsa(cls, hasCxxDtor); //Nonpointer的初始化isa
    } else {
        // Use raw pointer isa on the assumption that they might be
        // doing something weird with the zone or RR.
        obj->initIsa(cls); //非Nonpointer(即纯pointer)的初始化isa
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

可以看到是调用了`struct objc_class`的data方法先获取到class_rw_t-->class_ro_t-->instanceSize。实例的大小编译期间就已经计算好了，这里只是获取一下。

分配内存后，接着是初始化isa：

```c
inline void 
objc_object::initInstanceIsa(Class cls, bool hasCxxDtor)
{
    ASSERT(!cls->instancesRequireRawIsa());
    ASSERT(hasCxxDtor == cls->hasCxxDtor());

    initIsa(cls, true, hasCxxDtor);
}

inline void 
objc_object::initIsa(Class cls)
{
    initIsa(cls, false, false); 
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

由此可知如果isa的最低位为1，那么isa就是nonpointer，否则为非nonpointer（即纯pointer）。对于nonpointer的isa里面除了保存类对象地址外，还会保存很多其他信息比如对象的引用计数，是否有关联对象，是否有weak指针等，这样就可以减少一些哈希表的操作，提高了效率。

这里说一下isa为啥可以变为结构体类型，我们都知道32位的指针就可以寻址4GB的内存大小了，苹果手机目前最大内存也就3GB，因此一个64位的指针变量最多用到32位，这样就有一半是空着的，那哪行啊，所以isa的类型就从原来的纯指针类型变为了一个联合体类型，类对象的地址就保存在shiftcls字段中，其他31位可以存放其他信息，充分利用了内存空间。另外isa的shiftcls在arm64中占33位，在`__x86_64__` 中占44位。33位可以寻址8GB，因此shiftcls坚挺个10年应该没有什么问题。即使以后苹果也上8GB甚至10GB内存，shiftcls只需要改为44位就可以了，44位已经可以寻址16T内存了估计可以用到苹果破产。

## Tagged Pointer对象

```
// Tagged pointer objects.

#if __LP64__
#define OBJC_HAVE_TAGGED_POINTERS 1
#endif
```

Tagged pointer对象是在64位系统才有的。对于一些小的对象比如整数对象，字符串对象等，他们的值一般情况下不会太大，如果还像普通对象那样从堆上申请内存，引用计数，释放内存一套下来效率必然低下。而64位的指针变量本身就可以保存较大的数值，因此可以把一些小对象的值直接保存到指针变量里，这样的对象就称为Tagged pointer对象。可以看到Tagged pointer对象已经不是传统意义上的对象了，什么申请内存，引用计数，释放内存统统不存在了。

既然是将对象的值直接保存在指针变量里，自然就需要一套编码规则或者说协议规则。

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

tag bit位：用于判断指针变量指向的是否是一个Tagged pointer对象。

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

以numberInt：0xb7427616a082ae26为例

最高位为1说明是Tagged pointer对象。是Tagged pointer对象后，我们才可以解码该地址，解码后可以看到TaggedPointer的真实布局。

这里解码为0xb000000000000012

最高4位为b(1011)：

011 = 3，查找上面的OBJC_TAG枚举得知为OBJC_TAG_NSNumber，即为NSNumber类型。

其他60位就是payload了，payload里面包含真正的数据以及数据的一些其他信息比如长度等。

这里最低4位是NSNumber的类型：比如，char是0、short是1、int是2、float是4。2代表为int类型。

中间56位就是真正的数据了，这里为1。

可以看到Tagged Pointer对象的编码规则基本上如下：

```
标志位--OBJC_TAG--数据--元数据
```

## runtime的加载

```c
/***********************************************************************
* _objc_init
* Bootstrap initialization. Registers our image notifier with dyld.
* Called by libSystem BEFORE library initialization time
**********************************************************************/

void _objc_init(void)
{
    static bool initialized = false;
    if (initialized) return;
    initialized = true;
    
    // fixme defer initialization until an objc-using image is found?
    environ_init();
    tls_init();
    static_init();
    runtime_init();
    exception_init();
    cache_init();
    _imp_implementationWithBlock_init();

    _dyld_objc_notify_register(&map_images, load_images, unmap_image);
}

void runtime_init(void)
{
    objc::unattachedCategories.init(32);
    objc::allocatedClasses.init();
}
```

在_objc_init函数中runtime会向dyld注册回调函数。

该函数主要作用：

1. 各种初始化。比如初始化了unattachedCategories和allocatedClasses两张表。
2. 往dyld中注册了3个通知回调函数`map_images` , `load_images` , `unmap_image`。

unattachedCategories表：用于存储所有类的类别，当类别添加到类上后，该类别将会从表中移除。

allocatedClasses表：

```c++
/***********************************************************************
* allocatedClasses
* A table of all classes (and metaclasses) which have been allocated
* with objc_allocateClassPair.
**********************************************************************/
namespace objc {
static ExplicitInitDenseSet<Class> allocatedClasses;
}
```

包含了所有objc_allocateClassPair创建的类和元类(基本上是运行时创建的类)。

`map_images` 里面做了很多事情包括类的加载，类别的加载，类别添加到类上，协议的加载等等。

`load_images` 里面会调用我们实现的`+load`方法。

_dyld_objc_notify_register未公开，只能看到它的说明：

```c
//
// Note: only for use by objc runtime
// Register handlers to be called when objc images are mapped, unmapped, and initialized.
// Dyld will call back the "mapped" function with an array of images that contain an objc-image-info section.
// Those images that are dylibs will have the ref-counts automatically bumped, so objc will no longer need to
// call dlopen() on them to keep them from being unloaded.  During the call to _dyld_objc_notify_register(),
// dyld will call the "mapped" function with already loaded objc images.  During any later dlopen() call,
// dyld will also call the "mapped" function.  Dyld will call the "init" function when dyld would be called
// initializers in that image.  This is when objc calls any +load methods in that image.
//
void _dyld_objc_notify_register(_dyld_objc_notify_mapped    mapped,
                                _dyld_objc_notify_init      init,
                                _dyld_objc_notify_unmapped  unmapped);
```

mapped函数在调用_dyld_objc_notify_register时会回调一次，后面如果有调用dlopen时还会再回调。

init函数在初始化image的时候会回调，通常一个APP会包含很多个image，所以该回调会回调很多次。

**image**

image可以理解为一个个编译后的二进制库（比如系统的动态库，自己的工程代码编译后的二进制）。比如：

```
objc[6706]: IMAGES: loading image for /usr/lib/system/introspection/libdispatch.dylib (has class properties)

objc[6706]: IMAGES: loading image for /usr/lib/system/libxpc.dylib (has class properties) (preoptimized)

objc[6706]: IMAGES: loading image for /Users/xuequan/Library/Developer/Xcode/DerivedData/objc-cqxbandkuogzvugfeofcgrgddgkz/Build/Products/Debug/debug-objc (has class properties)
```

### map_images

```c
/***********************************************************************
* map_images
* Process the given images which are being mapped in by dyld.
* Calls ABI-agnostic code after taking ABI-specific locks.
*
* Locking: write-locks runtimeLock
**********************************************************************/
void
map_images(unsigned count, const char * const paths[],
           const struct mach_header * const mhdrs[])
{
    mutex_locker_t lock(runtimeLock);
    return map_images_nolock(count, paths, mhdrs);
}
```

大致过程：

1. 加锁
2. 调用map_images_nolock

因此主要是map_images_nolock逻辑。

#### map_images_nolock

```c
void 
map_images_nolock(unsigned mhCount, const char * const mhPaths[],
                  const struct mach_header * const mhdrs[])
{
    static bool firstTime = YES;
    header_info *hList[mhCount];
    uint32_t hCount;
    size_t selrefCount = 0;

    // Perform first-time initialization if necessary.
    // This function is called before ordinary library initializers. 
    // fixme defer initialization until an objc-using image is found?
    if (firstTime) {
        preopt_init();
    }

    if (PrintImages) {
        _objc_inform("IMAGES: processing %u newly-mapped images...\n", mhCount);
    }


    // Find all images with Objective-C metadata.
    hCount = 0;

    // Count classes. Size various table based on the total.
    int totalClasses = 0;
    int unoptimizedTotalClasses = 0;
    {
        uint32_t i = mhCount;
        while (i--) {
            const headerType *mhdr = (const headerType *)mhdrs[i];

            auto hi = addHeader(mhdr, mhPaths[i], totalClasses, unoptimizedTotalClasses);
            if (!hi) {
                // no objc data in this entry
                continue;
            }
            
            if (mhdr->filetype == MH_EXECUTE) {
                // Size some data structures based on main executable's size
#if __OBJC2__
                size_t count;
                _getObjc2SelectorRefs(hi, &count);
                selrefCount += count;
                _getObjc2MessageRefs(hi, &count);
                selrefCount += count;
#else
                _getObjcSelectorRefs(hi, &selrefCount);
#endif
                ...
            }
            
            hList[hCount++] = hi;
        }
    }

    // Perform one-time runtime initialization that must be deferred until 
    // the executable itself is found. This needs to be done before 
    // further initialization.
    // (The executable may not be present in this infoList if the 
    // executable does not contain Objective-C code but Objective-C 
    // is dynamically loaded later.
    if (firstTime) {
        sel_init(selrefCount); //初始化全局namedSelectors选择子表
        arr_init(); //这里会初始化三个全局的哈希表：自动释放池表，SideTables表，关联对象表
        ...
    }

    if (hCount > 0) {
        _read_images(hList, hCount, totalClasses, unoptimizedTotalClasses);
    }

    firstTime = NO;
    
    // Call image load funcs after everything is set up.
    for (auto func : loadImageFuncs) {
        for (uint32_t i = 0; i < mhCount; i++) {
            func(mhdrs[i]);
        }
    }
}
```

大致流程：

1. 初始化全局namedSelectors选择子表。这说明runtime内部维护了一个巨大的选择子表。一个选择子代表一个方法名。名字相同但参数类型不同的方法对应的是同一个选择子。
2. 初始化三个全局的哈希表：自动释放池表，SideTables表，关联对象表。
3. 调用_read_images读取image。

#### _read_images

这个方法很长做了很多事情。

大致流程：

1. fix up selector references
2. discover classes
3. Fix up remapped classes
4. fix up objc_msgSend_fixup
5. discover protocols
6. Fix up @protocol references
7. Discover categories.
8. Realize non-lazy classes
9. realize future classes

接下来分段查看源码

> fix up selector references

代码：

```c
void _read_images(header_info **hList, uint32_t hCount, int totalClasses, int unoptimizedTotalClasses)
{
    header_info *hi;
    uint32_t hIndex;
    size_t count;
    size_t i;
    Class *resolvedFutureClasses = nil;
    size_t resolvedFutureClassCount = 0;
    static bool doneOnce;
    bool launchTime = NO;
    TimeLogger ts(PrintImageTimes);

    runtimeLock.assertLocked();
    
    #define EACH_HEADER \
    hIndex = 0;         \
    hIndex < hCount && (hi = hList[hIndex]); \
    hIndex++

    if (!doneOnce) {
        doneOnce = YES;
        launchTime = YES;

        if (DisableTaggedPointers) {
            disableTaggedPointers();
        }
        
        initializeTaggedPointerObfuscator();

        if (PrintConnecting) {
            _objc_inform("CLASS: found %d classes during launch", totalClasses);
        }

        // namedClasses
        // Preoptimized classes don't go in this table.
        // 4/3 is NXMapTable's load factor
        int namedClassesSize = 
            (isPreoptimized() ? unoptimizedTotalClasses : totalClasses) * 4 / 3;
        gdb_objc_realized_classes =
            NXCreateMapTable(NXStrValueMapPrototype, namedClassesSize); //初始化一个全局的name-class哈希表

        ts.log("IMAGE TIMES: first time tasks");
    }
    
    // Fix up @selector references
    static size_t UnfixedSelectors;
    {
        mutex_locker_t lock(selLock);
        for (EACH_HEADER) {
            if (hi->hasPreoptimizedSelectors()) continue;

            bool isBundle = hi->isBundle();
            SEL *sels = _getObjc2SelectorRefs(hi, &count);
            UnfixedSelectors += count;
            for (i = 0; i < count; i++) {
                const char *name = sel_cname(sels[i]);
                SEL sel = sel_registerNameNoLock(name, isBundle); //将方法选择器添加到namedSelectors选择子表中
                if (sels[i] != sel) {
                    sels[i] = sel;
                }
            }
        }
    }

    ts.log("IMAGE TIMES: fix up selector references");
}    
```

大致逻辑：

1. 初始化一个全局的name-class哈希表gdb_objc_realized_classes，里面包含了所有已实现的类，可以通过类名称来获取对应类对象。
2. 将所有选择子注册到namedSelectors选择子表中。



> Discover classes. Fix up unresolved future classes. Mark bundle classes

```c
// Discover classes. Fix up unresolved future classes. Mark bundle classes.
bool hasDyldRoots = dyld_shared_cache_some_image_overridden();

for (EACH_HEADER) {
    if (! mustReadClasses(hi, hasDyldRoots)) {
        // Image is sufficiently optimized that we need not call readClass()
        continue;
    }
		
    classref_t const *classlist = _getObjc2ClassList(hi, &count);

    bool headerIsBundle = hi->isBundle();
    bool headerIsPreoptimized = hi->hasPreoptimizedClasses();

    for (i = 0; i < count; i++) {
        Class cls = (Class)classlist[i];
        Class newCls = readClass(cls, headerIsBundle, headerIsPreoptimized);

        if (newCls != cls  &&  newCls) {
            // Class was moved but not deleted. Currently this occurs 
            // only when the new class resolved a future class.
            // Non-lazily realize the class below.
            resolvedFutureClasses = (Class *)
                realloc(resolvedFutureClasses, 
                        (resolvedFutureClassCount+1) * sizeof(Class));
            resolvedFutureClasses[resolvedFutureClassCount++] = newCls;
        }
    }
}

ts.log("IMAGE TIMES: discover classes");

//GETSECT(_getObjc2ClassList,           classref_t const,      "__objc_classlist");
```

大致流程：

1. 调用`_getObjc2ClassList` 从_DATA的"__objc_classlist" section获取类
2. 调用readClass，将类读取到runtime。主要是将类添加到gdb_objc_realized_classes 和 allocatedClasses表里去。一般情况下只有运行时创建的类才被添加到allocatedClasses表中，其他在编译时就已经确定的类runtime都能够知道所以就不会添加到allocatedClasses，运行时创建的类添加到allocatedClasses表后，runtime再次遇到时也就能够识别了。

这里主要是`_getObjc2ClassList`，可以`clang -rewrite-objc Person.m` 就能看到我们写的OC类会保存在一个数组中，数组保存在`_DATA`的"`__objc_classlist`" section处。我们自己也可也在`__DATA`区存储东西的。

```c++
extern "C" __declspec(dllexport) struct _class_t OBJC_CLASS_$_Person __attribute__ ((used, section ("__DATA,__objc_data"))) = {
	0, // &OBJC_METACLASS_$_Person,
	0, // &OBJC_CLASS_$_NSObject,
	0, // (void *)&_objc_empty_cache,
	0, // unused, was (void *)&_objc_empty_vtable,
	&_OBJC_CLASS_RO_$_Person,
};
static void OBJC_CLASS_SETUP_$_Person(void ) {
	OBJC_METACLASS_$_Person.isa = &OBJC_METACLASS_$_NSObject;
	OBJC_METACLASS_$_Person.superclass = &OBJC_METACLASS_$_NSObject;
	OBJC_METACLASS_$_Person.cache = &_objc_empty_cache;
	OBJC_CLASS_$_Person.isa = &OBJC_METACLASS_$_Person;
	OBJC_CLASS_$_Person.superclass = &OBJC_CLASS_$_NSObject;
	OBJC_CLASS_$_Person.cache = &_objc_empty_cache;
}
#pragma section(".objc_inithooks$B", long, read, write)
__declspec(allocate(".objc_inithooks$B")) static void *OBJC_CLASS_SETUP[] = {
	(void *)&OBJC_CLASS_SETUP_$_Person,
};
static struct _class_t *L_OBJC_LABEL_CLASS_$ [1] __attribute__((used, section ("__DATA, __objc_classlist,regular,no_dead_strip")))= { //类保存在这个数组里
	&OBJC_CLASS_$_Person,
};
static struct _class_t *_OBJC_LABEL_NONLAZY_CLASS_$[] = {
		&OBJC_CLASS_$_Person,
};
```

强烈建议深究 `__attribute__` 的用法。



> discover protocols

代码：

```c
// Discover protocols. Fix up protocol refs.
for (EACH_HEADER) {
    extern objc_class OBJC_CLASS_$_Protocol;
    Class cls = (Class)&OBJC_CLASS_$_Protocol;
    ASSERT(cls);
    NXMapTable *protocol_map = protocols();
    bool isPreoptimized = hi->hasPreoptimizedProtocols();

    // Skip reading protocols if this is an image from the shared cache
    // and we support roots
    // Note, after launch we do need to walk the protocol as the protocol
    // in the shared cache is marked with isCanonical() and that may not
    // be true if some non-shared cache binary was chosen as the canonical
    // definition
    if (launchTime && isPreoptimized && cacheSupportsProtocolRoots) {
        if (PrintProtocols) {
            _objc_inform("PROTOCOLS: Skipping reading protocols in image: %s",
                         hi->fname());
        }
        continue;
    }

    bool isBundle = hi->isBundle();

    protocol_t * const *protolist = _getObjc2ProtocolList(hi, &count);
    for (i = 0; i < count; i++) {
        readProtocol(protolist[i], cls, protocol_map, 
                     isPreoptimized, isBundle); //将协议注册到protocol_map表中
    }
}

ts.log("IMAGE TIMES: discover protocols");

//GETSECT(_getObjc2ProtocolList,        protocol_t * const,    "__objc_protolist");
```

大致流程：读取所有的协议protocol_t，并将其注册到protocol_map表中。

协议也是从`__DATA` 区的`__objc_protolist` 段获取的。



> Discover categories.

代码：

```c
// Discover categories.
for (EACH_HEADER) {
    bool hasClassProperties = hi->info()->hasCategoryClassProperties();

    auto processCatlist = [&](category_t * const *catlist) {
        for (i = 0; i < count; i++) {
            category_t *cat = catlist[i];
            Class cls = remapClass(cat->cls); //从remapped_class_map表中获取类
            locstamped_category_t lc{cat, hi};

            if (!cls) {
                // Category's target class is missing (probably weak-linked).
                // Ignore the category.
                if (PrintConnecting) {
                    _objc_inform("CLASS: IGNORING category \?\?\?(%s) %p with "
                                 "missing weak-linked target class",
                                 cat->name, cat);
                }
                continue;
            }

            // Process this category.
            if (cls->isStubClass()) {
                // Stub classes are never realized. Stub classes
                // don't know their metaclass until they're
                // initialized, so we have to add categories with
                // class methods or properties to the stub itself.
                // methodizeClass() will find them and add them to
                // the metaclass as appropriate.
                if (cat->instanceMethods ||
                    cat->protocols ||
                    cat->instanceProperties ||
                    cat->classMethods ||
                    cat->protocols ||
                    (hasClassProperties && cat->_classProperties))
                {
                    objc::unattachedCategories.addForClass(lc, cls);
                }
            } else {
                // First, register the category with its target class.
                // Then, rebuild the class's method lists (etc) if
                // the class is realized.
                //重点是这一段
                if (cat->instanceMethods ||  cat->protocols
                    ||  cat->instanceProperties)
                {
                    if (cls->isRealized()) { //这个时候一般类还没有被实现，所以大部分情况下不会走这里
                        attachCategories(cls, &lc, 1, ATTACH_EXISTING);
                    } else {
                        objc::unattachedCategories.addForClass(lc, cls);
                    }
                }

                if (cat->classMethods  ||  cat->protocols
                    ||  (hasClassProperties && cat->_classProperties))
                {
                    if (cls->ISA()->isRealized()) {
                        attachCategories(cls->ISA(), &lc, 1, ATTACH_EXISTING | ATTACH_METACLASS);
                    } else {
                        objc::unattachedCategories.addForClass(lc, cls->ISA());
                    }
                }
            }
        }
    };
    processCatlist(_getObjc2CategoryList(hi, &count));
    processCatlist(_getObjc2CategoryList2(hi, &count));
}

ts.log("IMAGE TIMES: discover categories");

//GETSECT(_getObjc2CategoryList,    category_t * const,  "__objc_catlist");
```

大致逻辑：

1. 从`__DATA` 区的`__objc_catlist` 段获取类别数组。
2. 获取类别对应的类。
3. 如果类别中有实例方法，协议，实例属性则将cls(类)--lc键值对添加到unattachedCategories表中。最终会将实例方法，协议，实例属性等添加到类上。
4. 如果类别中有类方法，协议，类属性则将cls->ISA()(元类)--lc键值对添加到unattachedCategories表中。最终会将类方法，协议，类属性等添加到元类上。

主要的逻辑就是读取所有的类别到unattachedCategories表中。

> Realize non-lazy classes

代码：

```c
// Category discovery MUST BE Late to avoid potential races
// when other threads call the new category code before
// this thread finishes its fixups.

// +load handled by prepare_load_methods()

// Realize non-lazy classes (for +load methods and static instances)
for (EACH_HEADER) {
    classref_t const *classlist = 
        _getObjc2NonlazyClassList(hi, &count);
    for (i = 0; i < count; i++) {
        Class cls = remapClass(classlist[i]);
        if (!cls) continue;

        addClassTableEntry(cls);

        if (cls->isSwiftStable()) {
            if (cls->swiftMetadataInitializer()) {
                _objc_fatal("Swift class %s with a metadata initializer "
                            "is not allowed to be non-lazy",
                            cls->nameForLogging());
            }
            // fixme also disallow relocatable classes
            // We can't disallow all Swift classes because of
            // classes like Swift.__EmptyArrayStorage
        }
        realizeClassWithoutSwift(cls, nil);
    }
}

ts.log("IMAGE TIMES: realize non-lazy classes");

//GETSECT(_getObjc2NonlazyClassList,    classref_t const,      "__objc_nlclslist");
```

大致流程：

1. 从`__DATA` 区的`__objc_nlclslist` 段获取类数组。由于获取的是非懒加载的类，所以懒加载的类此时并不会被实现。
2. 调用realizeClassWithoutSwift实现类。主要是类的ro 和 rw，以及将类别添加到类上。

**懒加载的类和非懒加载的类**

非懒加载的类指的是实现了+load方法的类，非懒加载的类在runtime启动时就会被实现。具体在`_read_images` 里实现。

懒加载的类就是没有实现+load方法的类，懒加载的类是在第一次收到消息时才会被实现。具体在`lookUpImpOrForward` 里调用`realizeClassMaybeSwiftAndLeaveLocked` 实现。

这里面的逻辑主要是realizeClassWithoutSwift了

##### realizeClassWithoutSwift

实现：

```c
/***********************************************************************
* realizeClassWithoutSwift
* Performs first-time initialization on class cls, 
* including allocating its read-write data.
* Does not perform any Swift-side initialization.
* Returns the real class structure for the class. 
* Locking: runtimeLock must be write-locked by the caller
**********************************************************************/
static Class realizeClassWithoutSwift(Class cls, Class previously)
{
    runtimeLock.assertLocked();

    const class_ro_t *ro;
    class_rw_t *rw;
    Class supercls;
    Class metacls;
    bool isMeta;

    if (!cls) return nil; //递归退出条件.NSObject的父类就是nil.所以递归必定会退出。
    if (cls->isRealized()) return cls; //递归退出条件.类已经实现
    ASSERT(cls == remapClass(cls));

    // fixme verify class is not in an un-dlopened part of the shared cache?

    ro = (const class_ro_t *)cls->data();
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
        cls->setData(rw); //设置rw。
    }

    isMeta = ro->flags & RO_META;
#if FAST_CACHE_META
    if (isMeta) cls->cache.setBit(FAST_CACHE_META);
#endif
    rw->version = isMeta ? 7 : 0;  // old runtime went up to 6


    // Choose an index for this class.
    // Sets cls->instancesRequireRawIsa if indexes no more indexes are available
    cls->chooseClassArrayIndex();

    if (PrintConnecting) {
        _objc_inform("CLASS: realizing class '%s'%s %p %p #%u %s%s",
                     cls->nameForLogging(), isMeta ? " (meta)" : "", 
                     (void*)cls, ro, cls->classArrayIndex(),
                     cls->isSwiftStable() ? "(swift)" : "",
                     cls->isSwiftLegacy() ? "(pre-stable swift)" : "");
    }

    // Realize superclass and metaclass, if they aren't already.
    // This needs to be done after RW_REALIZED is set above, for root classes.
    // This needs to be done after class index is chosen, for root metaclasses.
    // This assumes that none of those classes have Swift contents,
    //   or that Swift's initializers have already been called.
    //   fixme that assumption will be wrong if we add support
    //   for ObjC subclasses of Swift classes.
    // 递归的实现整个继承链上的class（类及元类）
    supercls = realizeClassWithoutSwift(remapClass(cls->superclass), nil);
    metacls = realizeClassWithoutSwift(remapClass(cls->ISA()), nil);

#if SUPPORT_NONPOINTER_ISA
    if (isMeta) {
        // Metaclasses do not need any features from non pointer ISA
        // This allows for a faspath for classes in objc_retain/objc_release.
        cls->setInstancesRequireRawIsa();
    } else {
        // Disable non-pointer isa for some classes and/or platforms.
        // Set instancesRequireRawIsa.
        bool instancesRequireRawIsa = cls->instancesRequireRawIsa();
        bool rawIsaIsInherited = false;
        static bool hackedDispatch = false;

        if (DisableNonpointerIsa) {
            // Non-pointer isa disabled by environment or app SDK version
            instancesRequireRawIsa = true;
        }
        else if (!hackedDispatch  &&  0 == strcmp(ro->name, "OS_object"))
        {
            // hack for libdispatch et al - isa also acts as vtable pointer
            hackedDispatch = true;
            instancesRequireRawIsa = true;
        }
        else if (supercls  &&  supercls->superclass  &&
                 supercls->instancesRequireRawIsa())
        {
            // This is also propagated by addSubclass()
            // but nonpointer isa setup needs it earlier.
            // Special case: instancesRequireRawIsa does not propagate
            // from root class to root metaclass
            instancesRequireRawIsa = true;
            rawIsaIsInherited = true;
        }

        if (instancesRequireRawIsa) {
            cls->setInstancesRequireRawIsaRecursively(rawIsaIsInherited);
        }
    }
// SUPPORT_NONPOINTER_ISA
#endif

    // Update superclass and metaclass in case of remapping
    cls->superclass = supercls;
    cls->initClassIsa(metacls);

    // Reconcile instance variable offsets / layout.
    // This may reallocate class_ro_t, updating our ro variable.
    if (supercls  &&  !isMeta) reconcileInstanceVariables(cls, supercls, ro);

    // Set fastInstanceSize if it wasn't set already.
    cls->setInstanceSize(ro->instanceSize); //设置实例大小

    // Copy some flags from ro to rw
    if (ro->flags & RO_HAS_CXX_STRUCTORS) {
        cls->setHasCxxDtor();
        if (! (ro->flags & RO_HAS_CXX_DTOR_ONLY)) {
            cls->setHasCxxCtor();
        }
    }
    
    // Propagate the associated objects forbidden flag from ro or from
    // the superclass.
    if ((ro->flags & RO_FORBIDS_ASSOCIATED_OBJECTS) ||
        (supercls && supercls->forbidsAssociatedObjects()))
    {
        rw->flags |= RW_FORBIDS_ASSOCIATED_OBJECTS;
    }

    // Connect this class to its superclass's subclass lists
    if (supercls) {
        addSubclass(supercls, cls);
    } else {
        addRootClass(cls);
    }

    // Attach categories
    methodizeClass(cls, previously);

    return cls;
}
```

大致逻辑：

1. 分配rw内存，初始化 rw->ro = ro，rw->flags = RW_REALIZED|RW_REALIZING;
2. 递归的实现该类的继承链往上的class（类及元类）
3. 将类别里的信息添加到类上。

这里最重要的就是methodizeClass的逻辑了。

##### methodizeClass

实现：

```c
/***********************************************************************
* methodizeClass
* Fixes up cls's method list, protocol list, and property list.
* Attaches any outstanding categories.
* Locking: runtimeLock must be held by the caller
**********************************************************************/
static void methodizeClass(Class cls, Class previously)
{
    runtimeLock.assertLocked();

    bool isMeta = cls->isMetaClass();
    auto rw = cls->data();
    auto ro = rw->ro;

    // Methodizing for the first time
    if (PrintConnecting) {
        _objc_inform("CLASS: methodizing class '%s' %s", 
                     cls->nameForLogging(), isMeta ? "(meta)" : "");
    }

    // Install methods and properties that the class implements itself.
    method_list_t *list = ro->baseMethods();
    if (list) {
        prepareMethodLists(cls, &list, 1, YES, isBundleClass(cls));
        rw->methods.attachLists(&list, 1); //直接把方法列表数组作为一个整体添加到rw的methods里面去了，因此rw->methods是一个二维数组。
    }

    property_list_t *proplist = ro->baseProperties;
    if (proplist) {
        rw->properties.attachLists(&proplist, 1);
    }

    protocol_list_t *protolist = ro->baseProtocols;
    if (protolist) {
        rw->protocols.attachLists(&protolist, 1);
    }

    // Root classes get bonus method implementations if they don't have 
    // them already. These apply before category replacements.
    if (cls->isRootMetaclass()) {
        // root metaclass
        addMethod(cls, @selector(initialize), (IMP)&objc_noop_imp, "", NO);
    }

    // Attach categories.
    if (previously) {
        if (isMeta) {
            objc::unattachedCategories.attachToClass(cls, previously,
                                                     ATTACH_METACLASS);
        } else {
            // When a class relocates, categories with class methods
            // may be registered on the class itself rather than on
            // the metaclass. Tell attachToClass to look for those.
            objc::unattachedCategories.attachToClass(cls, previously,
                                                     ATTACH_CLASS_AND_METACLASS);
        }
    }
    objc::unattachedCategories.attachToClass(cls, cls,
                                             isMeta ? ATTACH_METACLASS : ATTACH_CLASS);
}
```

大致流程：

1. 把ro中的方法列表，协议列表，属性列表添加到rw的对应字段中。这说明ro是只读的，rw才是读写的。
2. 从unattachedCategories表中根据cls键取出它的类别，然后把类别信息添加到类上。

先看一下ro中的方法列表，协议列表，属性列表是如何添加到rw的对应字段中的。主要是attachLists的逻辑。

attachLists实现：

```c
void attachLists(List* const * addedLists, uint32_t addedCount) {
    if (addedCount == 0) return;

    if (hasArray()) {
        // many lists -> many lists
        uint32_t oldCount = array()->count;
        uint32_t newCount = oldCount + addedCount;
        setArray((array_t *)realloc(array(), array_t::byteSize(newCount)));
        array()->count = newCount;
        memmove(array()->lists + addedCount, array()->lists, 
                oldCount * sizeof(array()->lists[0])); //把旧的列表移到后面
        memcpy(array()->lists, addedLists, 
               addedCount * sizeof(array()->lists[0]));//把新的列表拷贝到前面，因此addedLists总是往前插入的而不是追加到数组末尾
    }
    else if (!list  &&  addedCount == 1) {
        // 0 lists -> 1 list
        list = addedLists[0];
    } 
    else {
        // 1 list -> many lists
        List* oldList = list;
        uint32_t oldCount = oldList ? 1 : 0;
        uint32_t newCount = oldCount + addedCount;
        setArray((array_t *)malloc(array_t::byteSize(newCount)));
        array()->count = newCount;
        if (oldList) array()->lists[addedCount] = oldList;
        memcpy(array()->lists, addedLists, 
               addedCount * sizeof(array()->lists[0]));
    }
}
```

attachLists的作用就是把addedLists插入到表的头部。理解了这个后面的attachCategories就很容易了。

接下来看一下类别是如何添加到类中的，这里主要是attachToClass的逻辑：

实现：

```c
void attachToClass(Class cls, Class previously, int flags)
{
    runtimeLock.assertLocked();
    ASSERT((flags & ATTACH_CLASS) ||
           (flags & ATTACH_METACLASS) ||
           (flags & ATTACH_CLASS_AND_METACLASS));

    auto &map = get();
    auto it = map.find(previously);

    if (it != map.end()) {
        category_list &list = it->second;
        if (flags & ATTACH_CLASS_AND_METACLASS) {
            int otherFlags = flags & ~ATTACH_CLASS_AND_METACLASS;
            attachCategories(cls, list.array(), list.count(), otherFlags | ATTACH_CLASS);
            attachCategories(cls->ISA(), list.array(), list.count(), otherFlags | ATTACH_METACLASS);
        } else {
            attachCategories(cls, list.array(), list.count(), flags);
        }
        map.erase(it);
    }
}
```

大致逻辑：

1. 从unattachedCategories表中取出cls的类别
2. 调用attachCategories将类别添加到类上。
3. 添加后将类别从unattachedCategories表中移除。

attachCategories实现：

```c
// Attach method lists and properties and protocols from categories to a class.
// Assumes the categories in cats are all loaded and sorted by load order, 
// oldest categories first.
static void
attachCategories(Class cls, const locstamped_category_t *cats_list, uint32_t cats_count,
                 int flags)
{
    if (slowpath(PrintReplacedMethods)) {
        printReplacements(cls, cats_list, cats_count);
    }
    if (slowpath(PrintConnecting)) {
        _objc_inform("CLASS: attaching %d categories to%s class '%s'%s",
                     cats_count, (flags & ATTACH_EXISTING) ? " existing" : "",
                     cls->nameForLogging(), (flags & ATTACH_METACLASS) ? " (meta)" : "");
    }

    /*
     * Only a few classes have more than 64 categories during launch.
     * This uses a little stack, and avoids malloc.
     *
     * Categories must be added in the proper order, which is back
     * to front. To do that with the chunking, we iterate cats_list
     * from front to back, build up the local buffers backwards,
     * and call attachLists on the chunks. attachLists prepends the
     * lists, so the final result is in the expected order.
     */
    constexpr uint32_t ATTACH_BUFSIZ = 64;
    method_list_t   *mlists[ATTACH_BUFSIZ];
    property_list_t *proplists[ATTACH_BUFSIZ];
    protocol_list_t *protolists[ATTACH_BUFSIZ];

    uint32_t mcount = 0;
    uint32_t propcount = 0;
    uint32_t protocount = 0;
    bool fromBundle = NO;
    bool isMeta = (flags & ATTACH_METACLASS);
    auto rw = cls->data();

    for (uint32_t i = 0; i < cats_count; i++) {
        auto& entry = cats_list[i]; //从数组中取出一个locstamped_category_t，因为一个类可能有多个类别

        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);//取出类别的方法数组
        if (mlist) {
            if (mcount == ATTACH_BUFSIZ) { //等于64个时buff满了就添加到rw里，一般没这么多个类别。
                prepareMethodLists(cls, mlists, mcount, NO, fromBundle);
                rw->methods.attachLists(mlists, mcount);
                mcount = 0;
            }
            mlists[ATTACH_BUFSIZ - ++mcount] = mlist; //倒序放入mlists
            fromBundle |= entry.hi->isBundle();
        }

        property_list_t *proplist =
            entry.cat->propertiesForMeta(isMeta, entry.hi);
        if (proplist) {
            if (propcount == ATTACH_BUFSIZ) {
                rw->properties.attachLists(proplists, propcount);
                propcount = 0;
            }
            proplists[ATTACH_BUFSIZ - ++propcount] = proplist;
        }

        protocol_list_t *protolist = entry.cat->protocolsForMeta(isMeta);
        if (protolist) {
            if (protocount == ATTACH_BUFSIZ) {
                rw->protocols.attachLists(protolists, protocount);
                protocount = 0;
            }
            protolists[ATTACH_BUFSIZ - ++protocount] = protolist;
        }
    }

    if (mcount > 0) {
        prepareMethodLists(cls, mlists + ATTACH_BUFSIZ - mcount, mcount, NO, fromBundle);
        rw->methods.attachLists(mlists + ATTACH_BUFSIZ - mcount, mcount);
        if (flags & ATTACH_EXISTING) flushCaches(cls);
    }

    rw->properties.attachLists(proplists + ATTACH_BUFSIZ - propcount, propcount);

    rw->protocols.attachLists(protolists + ATTACH_BUFSIZ - protocount, protocount);
}
```

大致流程：

1. 声名了一个临时变量mlists 用于保存所有类别的方法列表。因此mlists是一个二维数组
2. 取出类别的方法数组倒序放入mlists
3. 调用attachLists将mlists往前插入到rw里。

从数据段取出的类别数组------->放入unattachedCategories表中-------->从表中取出cls的类别数组即我们的参数cats_list。

因此cats_list数组的元素顺序是[cat1, cat2, cat3]，也就是我们的文件编译时的顺序。

mlists:  [[cat3_methodLists], [cat2_methodLists], [cat1_methodLists]]

rw添加前： [[base_methodLists]]

rw添加后： [[cat3_methodLists], [cat2_methodLists], [cat1_methodLists], [base_methodLists]]

如果类别和类之间有重名的方法，则查找到的将是cat3里面的，原类里面的方法在后面并没有被删除，因此我们可以遍历方法列表找到它调用它。

### load_images （+load的调用）

在方法load_images里面被调用

```c
void
load_images(const char *path __unused, const struct mach_header *mh)
{
    // Return without taking locks if there are no +load methods here.
    if (!hasLoadMethods((const headerType *)mh)) return;

    recursive_mutex_locker_t lock(loadMethodLock);

    // Discover load methods
    {
        mutex_locker_t lock2(runtimeLock);
        prepare_load_methods((const headerType *)mh);
    }

    // Call +load methods (without runtimeLock - re-entrant)
    call_load_methods();
}
```

大致流程：

1. Discover load methods。找到所有load方法
2. Call +load methods。调用所有load方法

先看下准备load方法，prepare_load_methods。

#### prepare_load_methods

```c
void prepare_load_methods(const headerType *mhdr)
{
    size_t count, i;

    runtimeLock.assertLocked();

    classref_t const *classlist = 
        _getObjc2NonlazyClassList(mhdr, &count); //获取所有的类
    for (i = 0; i < count; i++) {
        schedule_class_load(remapClass(classlist[i]));
    }

    category_t * const *categorylist = _getObjc2NonlazyCategoryList(mhdr, &count); //获取所有的类别
    for (i = 0; i < count; i++) { 
        category_t *cat = categorylist[i];
        Class cls = remapClass(cat->cls);
        if (!cls) continue;  // category for ignored weak-linked class
        if (cls->isSwiftStable()) {
            _objc_fatal("Swift class extensions and categories on Swift "
                        "classes are not allowed to have +load methods");
        }
        realizeClassWithoutSwift(cls, nil); //load_images调用时类都已经实现，所以一般不会走这个方法逻辑。
        ASSERT(cls->ISA()->isRealized());
        add_category_to_loadable_list(cat); ////按顺序添加cat-load对到loadable_categories数组中
    }
}

/***********************************************************************
* prepare_load_methods
* Schedule +load for classes in this image, any un-+load-ed 
* superclasses in other images, and any categories in this image.
**********************************************************************/
// Recursively schedule +load for cls and any un-+load-ed superclasses.
// cls must already be connected.
static void schedule_class_load(Class cls)
{
    if (!cls) return;
    ASSERT(cls->isRealized());  // _read_images should realize

    if (cls->data()->flags & RW_LOADED) return;

    // Ensure superclass-first ordering
    schedule_class_load(cls->superclass); //递归的调用，父类优先

    add_class_to_loadable_list(cls); //按顺序添加cls-load对到loadable_classes数组中
    cls->setInfo(RW_LOADED);  //设置标志位RW_LOADED，用于退出递归
}

/***********************************************************************
* add_class_to_loadable_list
* Class cls has just become connected. Schedule it for +load if
* it implements a +load method.
**********************************************************************/
void add_class_to_loadable_list(Class cls)
{
    IMP method;

    loadMethodLock.assertLocked();

    method = cls->getLoadMethod();
    if (!method) return;  // Don't bother if cls has no +load method
    
    if (PrintLoading) {
        _objc_inform("LOAD: class '%s' scheduled for +load", 
                     cls->nameForLogging());
    }
    
    if (loadable_classes_used == loadable_classes_allocated) {
        loadable_classes_allocated = loadable_classes_allocated*2 + 16;
        loadable_classes = (struct loadable_class *)
            realloc(loadable_classes,
                              loadable_classes_allocated *
                              sizeof(struct loadable_class));
    }
    
    loadable_classes[loadable_classes_used].cls = cls;
    loadable_classes[loadable_classes_used].method = method;
    loadable_classes_used++;
}

/***********************************************************************
* add_category_to_loadable_list
* Category cat's parent class exists and the category has been attached
* to its class. Schedule this category for +load after its parent class
* becomes connected and has its own +load method called.
**********************************************************************/
void add_category_to_loadable_list(Category cat)
{
    IMP method;

    loadMethodLock.assertLocked();

    method = _category_getLoadMethod(cat);

    // Don't bother if cat has no +load method
    if (!method) return;

    if (PrintLoading) {
        _objc_inform("LOAD: category '%s(%s)' scheduled for +load", 
                     _category_getClassName(cat), _category_getName(cat));
    }
    
    if (loadable_categories_used == loadable_categories_allocated) {
        loadable_categories_allocated = loadable_categories_allocated*2 + 16;
        loadable_categories = (struct loadable_category *)
            realloc(loadable_categories,
                              loadable_categories_allocated *
                              sizeof(struct loadable_category));
    }

    loadable_categories[loadable_categories_used].cat = cat;
    loadable_categories[loadable_categories_used].method = method;
    loadable_categories_used++;
}
```

准备的过程大致如下：

1. 添加所有cls-load对到loadable_classes数组中。父类的在前子类的在后
2. 添加所有cat-load对到loadable_categories数组中。元素的顺序是类别的编译顺序。

再看一下调用方法：call_load_methods

#### call_load_methods

```c

/***********************************************************************
* call_load_methods
* Call all pending class and category +load methods.
* Class +load methods are called superclass-first. 
* Category +load methods are not called until after the parent class's +load.
* 
* This method must be RE-ENTRANT, because a +load could trigger 
* more image mapping. In addition, the superclass-first ordering 
* must be preserved in the face of re-entrant calls. Therefore, 
* only the OUTERMOST call of this function will do anything, and 
* that call will handle all loadable classes, even those generated 
* while it was running.
*
* The sequence below preserves +load ordering in the face of 
* image loading during a +load, and make sure that no 
* +load method is forgotten because it was added during 
* a +load call.
* Sequence:
* 1. Repeatedly call class +loads until there aren't any more
* 2. Call category +loads ONCE.
* 3. Run more +loads if:
*    (a) there are more classes to load, OR
*    (b) there are some potential category +loads that have 
*        still never been attempted.
* Category +loads are only run once to ensure "parent class first" 
* ordering, even if a category +load triggers a new loadable class 
* and a new loadable category attached to that class. 
*
* Locking: loadMethodLock must be held by the caller 
*   All other locks must not be held.
**********************************************************************/
void call_load_methods(void)
{
    static bool loading = NO;
    bool more_categories;

    loadMethodLock.assertLocked();

    // Re-entrant calls do nothing; the outermost call will finish the job.
    if (loading) return;
    loading = YES;

    void *pool = objc_autoreleasePoolPush();

    do {
        // 1. Repeatedly call class +loads until there aren't any more
        while (loadable_classes_used > 0) { //先调用所有类的load方法
            call_class_loads();
        }

        // 2. Call category +loads ONCE
        more_categories = call_category_loads(); //再调用所有类别的load方法

        // 3. Run more +loads if there are classes OR more untried categories
    } while (loadable_classes_used > 0  ||  more_categories);

    objc_autoreleasePoolPop(pool);

    loading = NO;
}
```

看一下如何调用类里面的load：

##### call_class_loads

实现：

```c
/***********************************************************************
* call_class_loads
* Call all pending class +load methods.
* If new classes become loadable, +load is NOT called for them.
*
* Called only by call_load_methods().
**********************************************************************/
static void call_class_loads(void)
{
    int i;
    
    // Detach current loadable list.
    struct loadable_class *classes = loadable_classes;
    int used = loadable_classes_used;
    loadable_classes = nil;
    loadable_classes_allocated = 0;
    loadable_classes_used = 0; //先标记为调用完了
    
    // Call all +loads for the detached list.
    for (i = 0; i < used; i++) {
        Class cls = classes[i].cls;
        load_method_t load_method = (load_method_t)classes[i].method;
        if (!cls) continue; 

        if (PrintLoading) {
            _objc_inform("LOAD: +[%s load]\n", cls->nameForLogging());
        }
        (*load_method)(cls, @selector(load)); //就是通过函数指针直接调用的
    }
    
    // Destroy the detached list.
    if (classes) free(classes); //释放数组
}
```

循环通过IMP，调用load方法。

看一下如何调用类别里的load：

##### call_category_loads

实现：

```c
/***********************************************************************
* call_category_loads
* Call some pending category +load methods.
* The parent class of the +load-implementing categories has all of 
*   its categories attached, in case some are lazily waiting for +initalize.
* Don't call +load unless the parent class is connected.
* If new categories become loadable, +load is NOT called, and they 
*   are added to the end of the loadable list, and we return TRUE.
* Return FALSE if no new categories became loadable.
*
* Called only by call_load_methods().
**********************************************************************/
static bool call_category_loads(void)
{
    int i, shift;
    bool new_categories_added = NO;
    
    // Detach current loadable list.
    struct loadable_category *cats = loadable_categories;
    int used = loadable_categories_used;
    int allocated = loadable_categories_allocated;
    loadable_categories = nil;
    loadable_categories_allocated = 0;
    loadable_categories_used = 0;

    // Call all +loads for the detached list.
    for (i = 0; i < used; i++) {
        Category cat = cats[i].cat;
        load_method_t load_method = (load_method_t)cats[i].method;
        Class cls;
        if (!cat) continue;

        cls = _category_getClass(cat);
        if (cls  &&  cls->isLoadable()) {
            if (PrintLoading) {
                _objc_inform("LOAD: +[%s(%s) load]\n", 
                             cls->nameForLogging(), 
                             _category_getName(cat));
            }
            (*load_method)(cls, @selector(load)); //load方法的两个隐藏参数
            cats[i].cat = nil;
        }
    }

    // Compact detached list (order-preserving)
    shift = 0;
    for (i = 0; i < used; i++) {
        if (cats[i].cat) {
            cats[i-shift] = cats[i];
        } else {
            shift++;
        }
    }
    used -= shift;

    // Copy any new +load candidates from the new list to the detached list.
    new_categories_added = (loadable_categories_used > 0);
    for (i = 0; i < loadable_categories_used; i++) {
        if (used == allocated) {
            allocated = allocated*2 + 16;
            cats = (struct loadable_category *)
                realloc(cats, allocated *
                                  sizeof(struct loadable_category));
        }
        cats[used++] = loadable_categories[i];
    }

    // Destroy the new list.
    if (loadable_categories) free(loadable_categories);

    // Reattach the (now augmented) detached list. 
    // But if there's nothing left to load, destroy the list.
    if (used) {
        loadable_categories = cats;
        loadable_categories_used = used;
        loadable_categories_allocated = allocated;
    } else {
        if (cats) free(cats);
        loadable_categories = nil;
        loadable_categories_used = 0;
        loadable_categories_allocated = 0;
    }

    if (PrintLoading) {
        if (loadable_categories_used != 0) {
            _objc_inform("LOAD: %d categories still waiting for +load\n",
                         loadable_categories_used);
        }
    }

    return new_categories_added;
}
```

循环通过IMP，调用load方法。

#### 总结

1. 调用时机：类被实现后，load_images通知回调中被调用。
2. 所有类和类别的load方法都会调用。是通过IMP函数指针直接调用的。
3. 调用顺序为：先类再类别。类的顺序为先父类再子类。类别的顺序为编译顺序。
4. 只执行一次。由于runtime会保证所有load方法都会执行，所以最好不要在实现中调用super否则会造成多次执行，引发一些问题。

## +initialize的调用

第一次调用该类的类方法或实例方法前调用,可能永远不调用。

给对象发送消息时会调用lookUpImpOrForward，而lookUpImpOrForward会查看initialize 方法有没有执行过，没有则先执行initialize.

### lookUpImpOrForward

```c
IMP lookUpImpOrForward(id inst, SEL sel, Class cls, int behavior)
{
  const IMP forward_imp = (IMP)_objc_msgForward_impcache;
  IMP imp = nil;
  Class curClass;

  runtimeLock.assertUnlocked();

  // Optimistic cache lookup
  if (fastpath(behavior & LOOKUP_CACHE)) {
      imp = cache_getImp(cls, sel);
      if (imp) goto done_nolock;
  }
  
  ...
  
  if (slowpath((behavior & LOOKUP_INITIALIZE) && !cls->isInitialized())) {
      cls = initializeAndLeaveLocked(cls, inst, runtimeLock);
      // runtimeLock may have been dropped but is now locked again

      // If sel == initialize, class_initialize will send +initialize and 
      // then the messenger will send +initialize again after this 
      // procedure finishes. Of course, if this is not being called 
      // from the messenger then it won't happen. 2778172
  }
  
  ...
```

调用initializeAndLeaveLocked执行+initialize方法。

#### initializeNonMetaClass

```c
/***********************************************************************
* class_initialize.  Send the '+initialize' message on demand to any
* uninitialized class. Force initialization of superclasses first.
**********************************************************************/
void initializeNonMetaClass(Class cls)
{
    ASSERT(!cls->isMetaClass());

    Class supercls;
    bool reallyInitialize = NO;

    // Make sure super is done initializing BEFORE beginning to initialize cls.
    // See note about deadlock above.
    supercls = cls->superclass;
    if (supercls  &&  !supercls->isInitialized()) {
        initializeNonMetaClass(supercls); //递归调用，先执行父类的+initialize方法。因此在子类的实现中并不需要使用super。如果使用super反而会造成重复执行造成一些问题。
    }
    
    // Try to atomically set CLS_INITIALIZING.
    SmallVector<_objc_willInitializeClassCallback, 1> localWillInitializeFuncs;
    {
        monitor_locker_t lock(classInitLock);
        if (!cls->isInitialized() && !cls->isInitializing()) {
            cls->setInitializing();
            reallyInitialize = YES;

            // Grab a copy of the will-initialize funcs with the lock held.
            localWillInitializeFuncs.initFrom(willInitializeFuncs);
        }
    }
    
    if (reallyInitialize) {
        // We successfully set the CLS_INITIALIZING bit. Initialize the class.
        
        // Record that we're initializing this class so we can message it.
        _setThisThreadIsInitializingClass(cls);

        if (MultithreadedForkChild) {
            // LOL JK we don't really call +initialize methods after fork().
            performForkChildInitialize(cls, supercls);
            return;
        }
        
        for (auto callback : localWillInitializeFuncs)
            callback.f(callback.context, cls);

        // Send the +initialize message.
        // Note that +initialize is sent to the superclass (again) if 
        // this class doesn't implement +initialize. 2157218
        if (PrintInitializing) {
            _objc_inform("INITIALIZE: thread %p: calling +[%s initialize]",
                         objc_thread_self(), cls->nameForLogging());
        }

        // Exceptions: A +initialize call that throws an exception 
        // is deemed to be a complete and successful +initialize.
        //
        // Only __OBJC2__ adds these handlers. !__OBJC2__ has a
        // bootstrapping problem of this versus CF's call to
        // objc_exception_set_functions().
#if __OBJC2__
        @try
#endif
        {
            callInitialize(cls); //调用

            if (PrintInitializing) {
                _objc_inform("INITIALIZE: thread %p: finished +[%s initialize]",
                             objc_thread_self(), cls->nameForLogging());
            }
        }
#if __OBJC2__
        @catch (...) {
            if (PrintInitializing) {
                _objc_inform("INITIALIZE: thread %p: +[%s initialize] "
                             "threw an exception",
                             objc_thread_self(), cls->nameForLogging());
            }
            @throw;
        }
        @finally
#endif
        {
            // Done initializing.
            lockAndFinishInitializing(cls, supercls);
        }
        return;
    }
    
    else if (cls->isInitializing()) {
        // We couldn't set INITIALIZING because INITIALIZING was already set.
        // If this thread set it earlier, continue normally.
        // If some other thread set it, block until initialize is done.
        // It's ok if INITIALIZING changes to INITIALIZED while we're here, 
        //   because we safely check for INITIALIZED inside the lock 
        //   before blocking.
        if (_thisThreadIsInitializingClass(cls)) {
            return;
        } else if (!MultithreadedForkChild) {
            waitForInitializeToComplete(cls);
            return;
        } else {
            // We're on the child side of fork(), facing a class that
            // was initializing by some other thread when fork() was called.
            _setThisThreadIsInitializingClass(cls);
            performForkChildInitialize(cls, supercls);
        }
    }
    
    else if (cls->isInitialized()) {
        // Set CLS_INITIALIZING failed because someone else already 
        //   initialized the class. Continue normally.
        // NOTE this check must come AFTER the ISINITIALIZING case.
        // Otherwise: Another thread is initializing this class. ISINITIALIZED 
        //   is false. Skip this clause. Then the other thread finishes 
        //   initialization and sets INITIALIZING=no and INITIALIZED=yes. 
        //   Skip the ISINITIALIZING clause. Die horribly.
        return;
    }
    
    else {
        // We shouldn't be here. 
        _objc_fatal("thread-safe class init in objc runtime is buggy!");
    }
}

void callInitialize(Class cls)
{
    ((void(*)(Class, SEL))objc_msgSend)(cls, @selector(initialize)); //走的是消息发送，因此类别的同名方法会覆盖类的。
    asm("");
}
```

由于调用的时候走的是消息发送，因此类别里的会覆盖类里的方法。

### 总结

1. 调用时机：第一次调用该类的类方法或实例方法前调用，可能永远不调用。
2. 调用顺序：先父类再子类。类别里的会覆盖类里的实现，多个类别同时实现时最后一个类别里的被执行。
3. 只执行一次。由于runtime会保证先调用父类的，所以最好不要在实现中调用super否则会造成多次执行，引发一些问题。

## 方法交换

问题：

1. 交换类的一个未实现的父类方法？
2. 方法交换时机？
3. 方法交换失效？
4. 动态控制方法的交换与不交换可不可行？P

方法交换的核心实现：

```c
void method_exchangeImplementations(Method m1, Method m2)
{
    if (!m1  ||  !m2) return;

    mutex_locker_t lock(runtimeLock);

    IMP m1_imp = m1->imp;
    m1->imp = m2->imp;
    m2->imp = m1_imp;


    // RR/AWZ updates are slow because class is unknown
    // Cache updates are slow because class is unknown
    // fixme build list of classes whose Methods are known externally?

    flushCaches(nil); //清除方法派发缓存，走正常的方法查找过程。

    adjustCustomFlagsForMethodChange(nil, m1);
    adjustCustomFlagsForMethodChange(nil, m2);
}
```

原理很简单就是交换了两个方法的IMP，于是当收到sel1消息时执行的将是m2的实现imp2。当收到sel2消息时执行的将是m1的实现imp1。



Q1：交换类的一个未实现的父类方法

UINavigationController内部是没有实现viewDidLoad方法的，可以通过以下方法佐证：

1. 通过打符号断点 `-[UINavigationController viewDidLoad]` 会发现并不会断住

2. 通过打印IMP

   ```objc
   - (void)viewDidAppear:(BOOL)animated
   {
       [super viewDidAppear:animated];
       
       IMP imp1 = class_getMethodImplementation(self.class, @selector(viewDidLoad));
       NSLog(@"ViewController viewDidLoad IMP = %p", imp1);
       
       IMP imp2 = class_getMethodImplementation(UIViewController.class, @selector(viewDidLoad));
       NSLog(@"UIViewController viewDidLoad IMP = %p", imp2);
       
       IMP imp3 = class_getMethodImplementation(UINavigationController.class, @selector(viewDidLoad));
       NSLog(@"UINavigationController viewDidLoad IMP = %p", imp3);
   }
   
   2020-07-26 11:22:26.414393+0800 方法交换Demo[8806:230552] ViewController viewDidLoad IMP = 0x10d5f9a40
   2020-07-26 11:22:26.414701+0800 方法交换Demo[8806:230552] UIViewController viewDidLoad IMP = 0x110d1ced7
   2020-07-26 11:22:26.415099+0800 方法交换Demo[8806:230552] UINavigationController viewDidLoad IMP = 0x110d1ced7
   ```

   可以发现UINavigationController和父类UIViewController viewDidLoad的IMP是相同的说明UINavigationController并没有重写该方法。

3. 通过class_copyMethodList遍历UINavigationController类的方法列表，查看是否有viewDidLoad在里面。

如果在UINavigationController的类别中交换viewDidLoad方法，将导致其他UIViewController子类在执行viewDidLoad时崩溃。

```
UINavigationController+Helper.m

+ (void)load
{
    SEL originalSEL = @selector(viewDidLoad);
    SEL swizzledSEL = @selector(fsn_viewDidLoad);

    Method originalMethod = class_getInstanceMethod(self, originalSEL); //这里其实是它的父类的实现
    Method swizzledMethod = class_getInstanceMethod(self, swizzledSEL);

    method_exchangeImplementations(originalMethod, swizzledMethod);
}

- (void)fsn_viewDidLoad
{
    NSLog(@"先执行我的实现");

    [self fsn_viewDidLoad]; //执行原来的实现

    NSLog(@"执行完原先的实现后,继续执行");
}
```

原因：

由于UINavigationController内部并没有实现viewDidLoad方法，所以在方法交换时，交换的其实是它的父类UIViewController的viewDidLoad方法。

当其他UIViewController子类执行-[XQViewController viewDidLoad]时：

```
- (void)viewDidLoad {
    [super viewDidLoad]; //调用父类的实现
    
    self.navigationItem.title = @"首页";
}
```

真正调用到的是-fsn_viewDidLoad的IMP实现。由于在fsn_viewDidLoad实现里有给自己发送fsn_viewDidLoad消息，此时根据消息分发机制XQViewController会先在自己的方法列表中查找fsn_viewDidLoad方法，当然是找不到的。于是再到它的父类UIViewController中查找，自然也是找不到的，因为fsn_viewDidLoad只存在于UINavigationController类里。经过一系列的查找，最终发生崩溃。提示`*** Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '-[ViewController fsn_viewDidLoad]: unrecognized selector sent to instance 0x7fca9040a0e0'` 。

正确交换：

```objc
+ (void)load
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{

        SEL originalSEL = @selector(viewDidLoad);
        SEL swizzledSEL = @selector(fsn_viewDidLoad);

        Method originalMethod = class_getInstanceMethod(self, originalSEL);
        Method swizzledMethod = class_getInstanceMethod(self, swizzledSEL);

        class_addMethod(self,
                        originalSEL,
                        class_getMethodImplementation(self, originalSEL),
                        method_getTypeEncoding(originalMethod)); //确保自身实现原始方法。
        class_addMethod(self,
                        swizzledSEL,
                        class_getMethodImplementation(self, swizzledSEL),
                        method_getTypeEncoding(swizzledMethod)); //确保自身实现目的方法。

        method_exchangeImplementations(class_getInstanceMethod(self, originalSEL), class_getInstanceMethod(self, swizzledSEL));
    });
}
```

总结：

在使用方法交换这个黑魔法时：

1. 确保自身类已经实现了待交换的原始方法和目的方法，否则交换的将是父类的实现。
2. 确保方法交换的代码只执行一次。
3. 以上两点如果没有遵守，可能会导致崩溃或者没效果或效果不符合预期。
4. 推荐使用GitHub上已经封装好的工具类。

一些类簇在使用方法交换时需要对它们的真身做交换，否则会没效果。



Q3：方法交换失效？

方法交换的原理其实很简单。而方法交换失效的本质原因是方法交换的代码被多次执行了。比如父类里在load方法里做了方法交换，子类也在load方法里做了方法交换且调用了super，这就会导致父类里的交换执行了两次。一种可能的现象就是没效果了。

没效果：

```
drinking
1.子类没有实现。
2.只有父类交换了drinking。
3.子类load里调用了super
子类没有实现也没有交换drinking，因此子类load里调用了super，相当于父类交换了两次因而又回到了初始状态。
```

不符合预期：

```
eat
1.子类有实现。
2.父类，子类都交换了eat。
3.子类load里调用了super。
由于子类load里调用了super，事情变的复杂。
父类初始状态
p_o_eat --> p_o_eat_imp
p_d_eat --> p_d_eat_imp

执行父类的load
p_o_eat --> p_d_eat_imp
p_d_eat --> p_o_eat_imp

子类初始状态
s_o_eat --> s_o_eat_imp
s_d_eat --> s_d_eat_imp
执行子类的load
调用super
s_o_eat --> p_o_eat_imp
p_d_eat --> s_o_eat_imp
执行自己的交换
s_o_eat --> s_d_eat_imp
s_d_eat --> p_o_eat_imp

因此当执行[p eat]时：p_d_eat_imp -- s_o_eat_imp //给父对象发送eat消息，结果执行的是子类的实现，刺激不
当执行[s eat]时：s_d_eat_imp -- p_o_eat_imp //执行的是父类的实现
```



## 消息机制

TODO



## 参考

[Objective-C Runtime Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008048)	官方文档

[GNUstep](http://www.gnustep.org/) 一套和苹果cocoa框架对等的框架。



[load 方法全程跟踪](https://www.desgard.com/iOS-Source-Probe/Objective-C/Runtime/load 方法全程跟踪.html)

[读 objc4 源码，深入理解 Objective-C Runtime](https://www.jianshu.com/p/e3d521b2cb43)  提出了一些问题可以看看

[RuntimePDF](https://github.com/DeveloperErenLiu/RuntimePDF)  runtime的系列文章

[iOS底层原理总结 - 探寻Runtime本质（二）](https://www.jianshu.com/p/27ee04f3ed7b)  runtime的系列文章。

[深入解析 ObjC 中方法的结构](https://draveness.me/method-struct/)  也是oc类的系列文章

[9. _read_images 从二进制文件中读取类信息](https://jianli2017.top/wiki/IOS/Runtime/objc/9__read_images/) 可以看看



tagged pointer

[Tagged Pointer小记](https://crmo.github.io/2018/07/04/Tagged Pointer小记/)  讲了tagged的内存布局

[Objective-C中的伪指针](https://jinxuebin.cn/2019/06/Objective-C中的伪指针/)

[【译】采用Tagged Pointer的字符串](http://www.cocoachina.com/ios/20150918/13449.html) 外国人写的，非常棒，深入探讨了tagged pointer的实现细节

[Let's Build Tagged Pointers](https://www.mikeash.com/pyblog/friday-qa-2012-07-27-lets-build-tagged-pointers.html)  原理性的一些讲解，深入的可以看看。

[Non-pointer isa](http://www.sealiesoftware.com/blog/archive/2013/09/24/objc_explain_Non-pointer_isa.html)  很详细的解释了OC对象的isa字段为什么不再是一个指针类型。主要是为了做优化。



引用计数

[探寻Objective-C引用计数本质](https://crmo.github.io/2018/05/26/探寻Objective-C引用计数本质/)

[Objective-C 引用计数原理](https://www.jianshu.com/p/2acbde294266)  主要是tagged部分



关于attribute操作__DATA段的

[\_\_attribute   used section iOS简单应用](https://www.jianshu.com/p/33663906f7a2)

[__attribute__](https://nshipster.com/__attribute__/)  mattt写的

[__attribute__ 总结](https://www.jianshu.com/p/29eb7b5c8b2d)



关于内存拷贝的：

[memmove 和 memcpy的区别以及处理内存重叠问题](https://blog.csdn.net/Li_Ning_/article/details/51418400)

