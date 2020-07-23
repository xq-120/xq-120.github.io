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

### 类的加载

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

runtime加载时会调用_objc_init函数。

该函数主要作用：

1. 各种初始化。比如初始化了unattachedCategories和allocatedClasses两张表。
2. 往dyld中注册了3个通知回调函数`map_images` , `load_images` , `unmap_image`。

unattachedCategories表：用于存储类别信息。

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

`map_images` 里面做了很多事情包括类的加载，类别的加载，协议的加载等等。

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

大致流程：

1. fix up selector references。将所有选择子注册到namedSelectors选择子表中。
2. discover classes
3. Fix up remapped classes
4. fix up objc_msgSend_fixup
5. discover protocols
6. Fix up @protocol references
7. Discover categories.
8. Realize non-lazy classes
9. realize future classes

**懒加载的类和非懒加载的类**

非懒加载的类指的是实现了+load方法的类，非懒加载的类在runtime启动时就会被实现。具体在`_read_images` 里实现。

懒加载的类就是没有实现+load方法的类，懒加载的类是在第一次收到消息时才会被实现。在`lookUpImpOrForward` 里调用`realizeClassMaybeSwiftAndLeaveLocked` 实现。

> 2. Discover classes. Fix up unresolved future classes. Mark bundle classes

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
```

大致流程：

1. 调用`_getObjc2ClassList` 从_DATA的"__objc_classlist" section获取类
2. 调用readClass，将类读取到runtime。主要是将类添加到gdb_objc_realized_classes 和 allocatedClasses表里去。一般情况下只有运行时创建的类才被添加到allocatedClasses表中，其他在编译时就已经确定的类runtime都能够知道所以就不会添加到allocatedClasses，运行时创建的类添加到表后，runtime再遇到时也就能够识别了。

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



### 参考

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



内存对齐：

[32位与64位系统基本数据类型的字节数](https://blog.csdn.net/u012611878/article/details/52455576)

[C/C++内存对齐详解](https://zhuanlan.zhihu.com/p/30007037)



引用计数

[探寻Objective-C引用计数本质](https://crmo.github.io/2018/05/26/探寻Objective-C引用计数本质/)

[Objective-C 引用计数原理](https://www.jianshu.com/p/2acbde294266)  主要是tagged部分



[GNUstep](http://www.gnustep.org/) 一套和苹果cocoa框架对等的框架。



关于attribute操作__DATA段的

[\_\_attribute   used section iOS简单应用](https://www.jianshu.com/p/33663906f7a2)

[__attribute__](https://nshipster.com/__attribute__/)  mattt写的

[__attribute__ 总结](https://www.jianshu.com/p/29eb7b5c8b2d)

