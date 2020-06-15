#### @synchronized介绍

`@synchronized` 所做的事情跟锁（lock）一样：它防止不同的线程同时执行同一段代码。但相比于使用 `NSLock` 创建锁对象、加锁和解锁来说，`@synchronized` 用着更方便，可读性更高。

`@synchronized`是对`mutex`递归锁的封装， `@synchronized(obj)`内部会根据传入的同步对象obj得到一把递归锁，然后进行加锁、解锁操作。

使用示例：

```objc
- (void)increment
{
    @synchronized (self) {
        [_elements addObject:element];
    }
}
```

可以看到synchronized的使用是非常简单的。

关于synchronized有几个重要问题？

1.锁是如何与你传入 `@synchronized` 的对象关联上的？

2.`@synchronized`会保持（retain，增加引用计数）被锁住的对象么？

3.假如你传入 `@synchronized` 的对象在 `@synchronized` 的代码块里面被赋值为 `nil` 将会怎么样？

4.给`@synchronized`传入nil会怎样？

#### 原理探究

对：

```c
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSObject *obj = [NSObject new];
        @synchronized (obj) {
            NSLog(@"obj is @synchronized");
        }
    }
    return 0;
}
```

`clang -rewrite-objc main.m`  得到：

```c++
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */
    {
        __AtAutoreleasePool __autoreleasepool;
        NSObject *obj = ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("new"));
        {
            id _rethrow = 0; 
          	id _sync_obj = (id)obj; 
          	objc_sync_enter(_sync_obj);
            try {
                struct _SYNC_EXIT {
                    _SYNC_EXIT(id arg) : sync_exit(arg) {}
                    ~_SYNC_EXIT() {objc_sync_exit(sync_exit);}
                    id sync_exit;
                } _sync_exit(_sync_obj);

                NSLog((NSString *)&__NSConstantStringImpl__var_folders_5z_1pxqzfcn77s2n7z4gmr63sdr0000gn_T_main_4eec76_mi_0);
            } catch (id e) {
              _rethrow = e;
            }
            {
                struct _FIN {
                    _FIN(id reth) : rethrow(reth) {}
                    ~_FIN() { if (rethrow) objc_exception_throw(rethrow); }
                    id rethrow;
                } _fin_force_rethow(_rethrow);
            }
        }
    }
    return 0;
}
```

从上面的代码可以看出 `@synchronized (obj)` 的工作流程：

1.新建一个作用域将同步代码块包裹在内。

2.将传入的同步对象赋值给一个临时变量`id _sync_obj = (id)obj;`，后续的操作使用的都是该临时变量。 因此以前的变量的值发生更改对本次加解锁不会产生影响，但是当下一次其他线程执行到@synchronized时由于同步对象已经发生改变因此获取到的将不是同一把锁于是其他线程此时也能够进入临界区，这样就有可能临界区同一时刻有多个线程在访问，从而造成多线程问题。因此使用synchronized时需要保证指向同步对象的变量的值不被更改，同步对象自身的生命周期最好也不能太短。

3.调用objc_sync_enter函数加锁，objc_sync_enter(_sync_obj);	

4.对同步代码块进行try-catch，try作用域结束时，调用objc_sync_exit(sync_exit)解锁;

5.同步代码块中如果出现异常，则在作用域结束时重新抛出。

不知道大家注意到没有`objc_sync_exit(sync_exit);`并不是在什么@finally里面调用的，而是`_sync_exit`实例销毁时在自身析构函数里调用的。难道它就不怕try里面发生崩溃吗？如果try里面发生崩溃那么_sync_exit实例将不会销毁，析构函数也不会执行自然objc_sync_exit也不会调用，那么加解锁次数就不一致了，其他线程将永远获取不到锁了。这种情况如果在OC文件代码里默认情况下确实会这样，但是在C++里编译器会产生一些代码确保即使try里发生崩溃，try里的临时对象也会被销毁。所以不会出现上述情况。

这样try里面即使发生崩溃，`_sync_exit`实例也会执行析构函数，从而调用`objc_sync_exit(sync_exit);`进行解锁。另外需要注意的一点，由于异常会被重新抛出，所以当代码块里面真的会崩溃时，还是需要我们自己try-catch的。

接下来就是objc_sync_enter和objc_sync_exit两个函数了。前面我们大概也知道了objc_sync_enter对应着加锁，objc_sync_exit对应着解锁，但是我们并不知道锁是怎么得来的和传入的同步对象又有什么关系？想知道这一切的话就得深入objc_sync_enter/exit的实现了。

#### objc_sync_enter

```c
// Begin synchronizing on 'obj'. 
// Allocates recursive mutex associated with 'obj' if needed.
// Returns OBJC_SYNC_SUCCESS once lock is acquired.  
int objc_sync_enter(id obj)
{
    int result = OBJC_SYNC_SUCCESS;

    if (obj) {
        SyncData* data = id2data(obj, ACQUIRE);
        ASSERT(data);
        data->mutex.lock();
    } else {
        // @synchronized(nil) does nothing
        if (DebugNilSync) {
            _objc_inform("NIL SYNC DEBUG: @synchronized(nil); set a breakpoint on objc_sync_nil to debug");
        }
        objc_sync_nil();
    }

    return result;
}
```

objc_sync_enter的作用：

如果obj不等于nil，则根据传入的obj找到一个SyncData，SyncData中有一把递归锁，然后加锁。

如果obj等于nil，则执行`objc_sync_nil；`。其实就是什么也不做直接返回OBJC_SYNC_SUCCESS，此时代码块就没有进行加锁保护。

objc_sync_nil实现：

```
BREAKPOINT_FUNCTION(
    void objc_sync_nil(void)
);
```

宏BREAKPOINT_FUNCTION的定义：

```
#   define BREAKPOINT_FUNCTION(prototype)                             \
    OBJC_EXTERN __attribute__((noinline, used, visibility("hidden"))) \
    prototype { asm(""); }
```

展开后等价于：


```c
extern __attribute__((noinline, used, visibility("hidden"))) 
void objc_sync_nil(void) { 		 		
  asm(""); 
};
```

宏BREAKPOINT_FUNCTION的作用就是定义了一个函数，函数只有一行实现`asm(""); `，从BREAKPOINT_FUNCTION语义上也可以明白定义的函数只是用来打断点的。没有什么实际用途。

具体是如何根据obj获取锁的，还得看id2data()函数的实现：

有几个数据结构

```c++
//
// Allocate a lock only when needed.  Since few locks are needed at any point
// in time, keep them on a single list.
//

typedef struct alignas(CacheLineSize) SyncData {
    struct SyncData* nextData;
    DisguisedPtr<objc_object> object;
    int32_t threadCount;  // number of THREADS using this block
    recursive_mutex_t mutex;
} SyncData;

typedef struct {
    SyncData *data;
    unsigned int lockCount;  // number of times THIS THREAD locked this block
} SyncCacheItem;

typedef struct SyncCache {
    unsigned int allocated;
    unsigned int used;
    SyncCacheItem list[0];
} SyncCache;
```

SyncData

nextData：指向下一个SyncData，SyncData应该是一个单链表。

object：传入的同步对象

threadCount：使用该block的线程数，该计数器作用：

1用于安全判断

```
if (result->threadCount <= 0  ||  item->lockCount <= 0) {
    _objc_fatal("id2data cache is buggy");
}
```

2 如果等于0，说明mutex处于空闲状态，那么SyncData就可以复用，也就避免了创建SyncData的开销。

mutex：分配的递归锁，用于加解锁。

为了能够更高效的根据同步对象获得一把锁，系统做了缓存。

SyncCacheItem

data：SyncData对象

lockCount：当前线程锁住该block（block应该指的是同步代码块）的次数。FastCache会使用到该计数器，加锁时计数器加1，解锁时计数器减1。减到0时，FastCache就会将保存的SyncData对象清除。

SyncCache

存储于TLS(线程本地存储)。

allocated：分配的SyncCacheItem内存空间个数，默认是4个。

used：已使用的SyncCacheItem内存空间个数

list：数组，保存已使用的SyncCacheItem

另外还有一个类型为StripedMap的全局变量sDataLists，StripedMap里面有一个数组总长度为8，数组的元素是SyncList。

```c
struct SyncList {
    SyncData *data;
    spinlock_t lock;

    constexpr SyncList() : data(nil), lock(fork_unsafe_lock) { }
};

// Use multiple parallel lists to decrease contention among unrelated objects.
#define LOCK_FOR_OBJ(obj) sDataLists[obj].lock
#define LIST_FOR_OBJ(obj) sDataLists[obj].data
static StripedMap<SyncList> sDataLists;
```

LOCK_FOR_OBJ(object)和LIST_FOR_OBJ(object)就是从全局StripedMap变量sDataLists的数组属性中找到一个SyncList，SyncList包含一个SyncData以及一把自旋锁。

StripedMap的部分定义：

```c++
// StripedMap<T> is a map of void* -> T, sized appropriately 
// for cache-friendly lock striping. 
// For example, this may be used as StripedMap<spinlock_t>
// or as StripedMap<SomeStruct> where SomeStruct stores a spin lock.
template<typename T>
class StripedMap {
#if TARGET_OS_IPHONE && !TARGET_OS_SIMULATOR
    enum { StripeCount = 8 };
#else
    enum { StripeCount = 64 };
#endif

    struct PaddedT {
        T value alignas(CacheLineSize);
    };

    PaddedT array[StripeCount];

    static unsigned int indexForPointer(const void *p) {
        uintptr_t addr = reinterpret_cast<uintptr_t>(p);
        return ((addr >> 4) ^ (addr >> 9)) % StripeCount;
    }

 public:
    T& operator[] (const void *p) { 
        return array[indexForPointer(p)].value; 
    }
    const T& operator[] (const void *p) const { 
        return const_cast<StripedMap<T>>(this)[p]; 
    }
    ...
```

StripedMap里定义了一个总长度为8的数组`PaddedT array[StripeCount];`

如何根据object查找到一个SyncList呢？

其实就是把object对象的地址映射为一个数组索引，然后从索引处获取一个SyncList。

```c
static unsigned int indexForPointer(const void *p) {
    uintptr_t addr = reinterpret_cast<uintptr_t>(p);
    return ((addr >> 4) ^ (addr >> 9)) % StripeCount;
}
```

地址右移四位的结果 按位异或 地址右移九位的结果 再模上数组总长度。这样处理后不同地址得到相同的结果可能性会小一些。

可以把sDataLists看做一个哈希表。根据传入的object找到一个SyncList，然后从SyncList单链表中查找SyncData的同步对象与传入的object相等的SyncData。

正式开始

```c
static SyncData* id2data(id object, enum usage why)
{
    spinlock_t *lockp = &LOCK_FOR_OBJ(object);
    SyncData **listp = &LIST_FOR_OBJ(object);
    SyncData* result = NULL;

		//从FastCache缓存里获取锁，如果找到则返回
  
  	//从SyncCache缓存中获取锁，如果找到则返回
  
    //从全局变量sDataLists列表（哈希表）中获取锁，并加入到FastCache缓存或SyncCache缓存
  
  	//新创建一把锁，并添加到sDataLists中的某个单链表中以及加入到FastCache缓存或SyncCache缓存
}
```

该函数的实现比较复杂，大体上分为四个部分：

1 从TLS（Thread Local Storage，线程本地存储）中的FastCache缓存中获取锁

2 从TLS中的SyncCache缓存中获取锁

3 从全局变量sDataLists列表（哈希表）中获取锁，并加入到FastCache缓存或SyncCache缓存

4 新创建一把锁，并添加到sDataLists中的某个单链表中以及加入到FastCache缓存或SyncCache缓存

**从TLS中的FastCache缓存里获取锁**

如果定义了SUPPORT_DIRECT_THREAD_KEYS则先从Fast cache里获取，Fast cache解释如下：

```
/*
  Fast cache: two fixed pthread keys store a single SyncCacheItem. 
  This avoids malloc of the SyncCache for threads that only synchronize 
  a single object at a time.
  SYNC_DATA_DIRECT_KEY  == SyncCacheItem.data
  SYNC_COUNT_DIRECT_KEY == SyncCacheItem.lockCount
 */
```

FastCache也是系统为了提高获取锁的效率做的一个优化。如果一个线程只同步一个对象就没必要为它再分配SyncCache内存，系统为这种情况在TLS中保留了一个存储SyncCacheItem的空间，通过SYNC_DATA_DIRECT_KEY和SYNC_COUNT_DIRECT_KEY来获取。

```c
#if SUPPORT_DIRECT_THREAD_KEYS
    // Check per-thread single-entry fast cache for matching object
    bool fastCacheOccupied = NO;
    SyncData *data = (SyncData *)tls_get_direct(SYNC_DATA_DIRECT_KEY);
    if (data) {
        fastCacheOccupied = YES;

        if (data->object == object) {
            // Found a match in fast cache.
            uintptr_t lockCount;

            result = data;
            lockCount = (uintptr_t)tls_get_direct(SYNC_COUNT_DIRECT_KEY);
            if (result->threadCount <= 0  ||  lockCount <= 0) {
                _objc_fatal("id2data fastcache is buggy");
            }

            switch(why) {
            case ACQUIRE: {
                lockCount++;
                tls_set_direct(SYNC_COUNT_DIRECT_KEY, (void*)lockCount);
                break;
            }
            case RELEASE:
                lockCount--;
                tls_set_direct(SYNC_COUNT_DIRECT_KEY, (void*)lockCount);
                if (lockCount == 0) {
                    // remove from fast cache
                    tls_set_direct(SYNC_DATA_DIRECT_KEY, NULL);
                    // atomic because may collide with concurrent ACQUIRE
                    OSAtomicDecrement32Barrier(&result->threadCount);
                }
                break;
            case CHECK:
                // do nothing
                break;
            }

            return result;
        }
    }
#endif
```

如果在TLS的FastCache中找到一个SyncData，即`if (data->object == object)`为真。

则根据是加锁还是解锁操作lockCount计数器。当lockCount等于0时也就是当前线程即将释放该锁，将SyncData从FastCache中移除，FastCache重新变为空的，同时将SyncData的threadCount减1。

执行完switch后返回找到的SyncData。

如果`if (data->object == object)`为假，表明FastCache中虽然存在SyncData对象但是同步对象不一样，此时会执行后续代码。

这种情况就是synchronized嵌套但传入的同步对象不一样：

```
@synchronized(A) {
		......
		@synchronized(B) {
				....
		}
}
```

**从TLS中的SyncCache缓存中获取锁**

如果在FastCache中没有找到与同步对象关联的锁，则继续在SyncCache缓存中查找。

SyncCache也是存储于TLS中的，也就是每个线程都有自己的SyncCache。如果线程之前为同步对象分配过锁那么下一次再获取时直接从缓存中获取。

```c
// Check per-thread cache of already-owned locks for matching object
SyncCache *cache = fetch_cache(NO);
if (cache) {
    unsigned int i;
    for (i = 0; i < cache->used; i++) {
        SyncCacheItem *item = &cache->list[i];
        if (item->data->object != object) continue;

        // Found a match.
        result = item->data;
        if (result->threadCount <= 0  ||  item->lockCount <= 0) {
            _objc_fatal("id2data cache is buggy");
        }

        switch(why) {
        case ACQUIRE:
            item->lockCount++;
            break;
        case RELEASE:
            item->lockCount--;
            if (item->lockCount == 0) {
                // remove from per-thread cache
                cache->list[i] = cache->list[--cache->used];
                // atomic because may collide with concurrent ACQUIRE
                OSAtomicDecrement32Barrier(&result->threadCount);
            }
            break;
        case CHECK:
            // do nothing
            break;
        }

        return result;
    }
}
```

逻辑和FastCache差不多。当lockCount等于0时，将SyncCacheItem从SyncCache中移除，同时将SyncData的threadCount减1：

```
// remove from per-thread cache
cache->list[i] = cache->list[--cache->used];
// atomic because may collide with concurrent ACQUIRE
OSAtomicDecrement32Barrier(&result->threadCount);
```

上述移除的代码不是很懂，这不是把当前SyncCacheItem用最后一个SyncCacheItem替换吗？

**从全局变量sDataLists列表中获取**

如果没能在上述缓存中找到，则继续在sDataLists列表中查找。

```c
// Thread cache didn't find anything.
// Walk in-use list looking for matching object
// Spinlock prevents multiple threads from creating multiple 
// locks for the same new object.
// We could keep the nodes in some hash table if we find that there are
// more than 20 or so distinct locks active, but we don't do that now.

lockp->lock();

{
    SyncData* p;
    SyncData* firstUnused = NULL;
    for (p = *listp; p != NULL; p = p->nextData) {
        if ( p->object == object ) {
            result = p;
            // atomic because may collide with concurrent RELEASE
            OSAtomicIncrement32Barrier(&result->threadCount);
            goto done;
        }
        if ( (firstUnused == NULL) && (p->threadCount == 0) )
            firstUnused = p;
    }

    // no SyncData currently associated with object
    if ( (why == RELEASE) || (why == CHECK) )
        goto done;

    // an unused one was found, use it
    if ( firstUnused != NULL ) {
        result = firstUnused;
        result->object = (objc_object *)object;
        result->threadCount = 1;
        goto done;
    }
}
......
```

刚开始的几个变量定义：

```
spinlock_t *lockp = &LOCK_FOR_OBJ(object);
SyncData **listp = &LIST_FOR_OBJ(object);
SyncData* result = NULL;
```

从全局的sDataLists中根据传入的object找到一个SyncList。

不同于在TLS中的查找，这里查找和操作的是一个全局变量，所以代码首先使用SyncList里的自旋锁进行加锁，保证后续的查找与操作同时只有一个线程。

然后在SyncData这个单链表上开始查找：

如果找到一个与object相等的SyncData，则threadCount加1并执行后续done操作。

```
if ( p->object == object ) {
    result = p;
    // atomic because may collide with concurrent RELEASE
    OSAtomicIncrement32Barrier(&result->threadCount);
    goto done;
}
```

如果没找到与object相等的SyncData，但找到一个未使用的SyncData那就使用它。

```
// an unused one was found, use it
if ( firstUnused != NULL ) {
    result = firstUnused;
    result->object = (objc_object *)object;
    result->threadCount = 1;
    goto done;
}
```

SyncData中的object更新为传入object，threadCount更新为1。执行后续done操作。

从全局变量sDataLists列表中的查找就结束了，如果依然没有找到可用的SyncData则只能创建新的SyncData了。

**新创建锁**

最后是新创建一把锁，并添加到sDataLists中的某个单链表中以及加入到FastCache缓存或SyncCache缓存。

```c
// Thread cache didn't find anything.
// Walk in-use list looking for matching object
// Spinlock prevents multiple threads from creating multiple 
// locks for the same new object.
// We could keep the nodes in some hash table if we find that there are
// more than 20 or so distinct locks active, but we don't do that now.

lockp->lock();

{
    SyncData* p;
    SyncData* firstUnused = NULL;
    for (p = *listp; p != NULL; p = p->nextData) {
        if ( p->object == object ) {
            result = p;
            // atomic because may collide with concurrent RELEASE
            OSAtomicIncrement32Barrier(&result->threadCount);
            goto done;
        }
        if ( (firstUnused == NULL) && (p->threadCount == 0) )
            firstUnused = p;
    }

    // no SyncData currently associated with object
    if ( (why == RELEASE) || (why == CHECK) )
        goto done;

    // an unused one was found, use it
    if ( firstUnused != NULL ) {
        result = firstUnused;
        result->object = (objc_object *)object;
        result->threadCount = 1;
        goto done;
    }
}

// Allocate a new SyncData and add to list.
// XXX allocating memory with a global lock held is bad practice,
// might be worth releasing the lock, allocating, and searching again.
// But since we never free these guys we won't be stuck in allocation very often.
posix_memalign((void **)&result, alignof(SyncData), sizeof(SyncData));
result->object = (objc_object *)object;
result->threadCount = 1;
new (&result->mutex) recursive_mutex_t(fork_unsafe_lock);
result->nextData = *listp;
*listp = result;

done:
lockp->unlock();
if (result) {
    // Only new ACQUIRE should get here.
    // All RELEASE and CHECK and recursive ACQUIRE are 
    // handled by the per-thread caches above.
    if (why == RELEASE) {
        // Probably some thread is incorrectly exiting 
        // while the object is held by another thread.
        return nil;
    }
    if (why != ACQUIRE) _objc_fatal("id2data is buggy");
    if (result->object != object) _objc_fatal("id2data is buggy");

#if SUPPORT_DIRECT_THREAD_KEYS
    if (!fastCacheOccupied) {
        // Save in fast thread cache
        tls_set_direct(SYNC_DATA_DIRECT_KEY, result);
        tls_set_direct(SYNC_COUNT_DIRECT_KEY, (void*)1);
    } else 
#endif
    {
        // Save in thread cache
        if (!cache) cache = fetch_cache(YES);
        cache->list[cache->used].data = result;
        cache->list[cache->used].lockCount = 1;
        cache->used++;
    }
}

return result;
```

同样创建新的SyncData时也是在自旋锁保护区里的，主要的创建代码为：

```
// Allocate a new SyncData and add to list.
// XXX allocating memory with a global lock held is bad practice,
// might be worth releasing the lock, allocating, and searching again.
// But since we never free these guys we won't be stuck in allocation very often.
posix_memalign((void **)&result, alignof(SyncData), sizeof(SyncData));
result->object = (objc_object *)object;
result->threadCount = 1;
new (&result->mutex) recursive_mutex_t(fork_unsafe_lock);
result->nextData = *listp;
*listp = result;
```

新创建的SyncData会作为头结点添加到当前SyncList中：

```
result->nextData = *listp;
*listp = result;
```

最后是done操作了：

前面在TLS的缓存中如果查找到可用的SyncData是不会走到done操作的，只有在哈希表里找到的或者新创建的SyncData才会走到done。

done的作用就是把从哈希表里或者新创建的SyncData加入到TLS中的FastCache或SyncCache。

到此为止id2data函数的工作流程就结束了。

总结一下：

id2data函数根据传入的同步对象获取锁的过程大体上分为四个部分：

1 从TLS（Thread Local Storage，线程本地存储）中的FastCache缓存中获取锁

2 从TLS中的SyncCache（普通数组）缓存中获取锁

3 从全局变量sDataLists列表（哈希表）中获取锁，并加入到FastCache缓存或SyncCache缓存

4 新创建一把锁，并添加到sDataLists中的某个单链表中以及加入到FastCache缓存或SyncCache缓存

id2data函数中为了能够更高效的根据同步对象获得一把锁，系统做了缓存。1和2是TLS中的缓存，3是哈希表中的缓存。

1和2对synchronized嵌套使用情况优化提升较大，但如果临界区不需要递归锁的特性，那么1和2的缓存对性能提升不是那么大。

即使系统做了如此多的优化，但是synchronized锁在效率上依然是所有锁中最低的，毕竟里面缓存操作也是要消耗时间的。

#### objc_sync_exit

最后是objc_sync_exit()的实现：

```c
// End synchronizing on 'obj'. 
// Returns OBJC_SYNC_SUCCESS or OBJC_SYNC_NOT_OWNING_THREAD_ERROR
int objc_sync_exit(id obj)
{
    int result = OBJC_SYNC_SUCCESS;
    
    if (obj) {
        SyncData* data = id2data(obj, RELEASE); 
        if (!data) {
            result = OBJC_SYNC_NOT_OWNING_THREAD_ERROR;
        } else {
            bool okay = data->mutex.tryUnlock();
            if (!okay) {
                result = OBJC_SYNC_NOT_OWNING_THREAD_ERROR;
            }
        }
    } else {
        // @synchronized(nil) does nothing
    }
	

    return result;
}
```

经过上述的id2data函数的分析，这里就很简单了。

就是根据传入的obj获取到SyncData后，进行解锁，传入nil则什么也不做。使用`@synchronized (obj){...}`时，objc_sync_enter和objc_sync_exit的参数obj必然是同一个obj，因此获取到的自然也是同一把锁。



查看DisguisedPtr模板类的实现，其内部并没有一个T类型的指针指向传入的object，在给DisguisedPtr变量赋值时，其实是将传入的对象的地址转化为一个无符号整数

```c
DisguisedPtr<T>& operator = (T* rhs) {
    value = disguise(rhs);
    return *this;
}

static uintptr_t disguise(T* ptr) {
    return -(uintptr_t)ptr;
}

static T* undisguise(uintptr_t val) {
    return (T*)-val;
}
```

举例：

```objc
typedef struct SyncDisguise {
    DisguisedPtr<Person> object;
    int32_t threadCount;
} SyncDisguise;
    
SyncDisguise myStruct;
    
@implementation XQContainer

- (void)testDisguise {
    Person *li = [Person new];
    li.name = @"li";
    
    myStruct.threadCount = 1;
    myStruct.object = li;
    
    NSLog(@"disguise1: %p", &myStruct);
    
    Person *pox = (Person *)myStruct.object; //需要在MRC下，ARC下编译通不过
    
    [li release];
}

@end
```

DisguisedPtr重载了赋值操作符，因此myStruct.object = li;会调用`DisguisedPtr<T>& operator = (T* rhs)`，而该函数内部是将传入的对象地址转化为了一个无符号整数，这样就把一个对象的地址给隐藏起来了。与之相反的操作是解伪装：将一个无符号整数转化为一个对象地址

```c
static T* undisguise(uintptr_t val) {
    return (T*)-val;
}
```

使用`Person *pox = (Person *)myStruct.object;`会调用`undisguise`函数进行解伪装，于是又可以重新获取到对象的地址。

`DisguisedPtr<Person> object; `类似于`Person *`但并不是真正的`Person *`，因此myStruct并不会持有传入的对象，在ARC下testDisguise方法执行完后Person对象就释放销毁了。

总结：SyncData对象虽然会被缓存而一直存活，但SyncData对象并没有持有传入的object对象，因此object对象会正常的释放。

#### 总结

`@synchronized (obj){...}`就是根据传入的obj获取到一把递归锁，在代码块开始处先加锁，代码块执行完后再解锁。传nil，则不加锁。

到此为止我们可以回答一下开头的几个问题

1.锁是如何与你传入 `@synchronized` 的对象关联上的？

这个就是id2data函数的工作流程了：

* 从TLS（Thread Local Storage，线程本地存储）中的FastCache缓存中获取锁

* 从TLS中的SyncCache缓存中获取锁
* 从全局变量sDataLists列表（哈希表）中获取锁，并加入到FastCache缓存或SyncCache缓存

* 新创建一把锁，并添加到sDataLists中的某个单链表中以及加入到FastCache缓存或SyncCache缓存

2.`@synchronized`会保持（retain，增加引用计数）被锁住的对象么？

MRC下不会，ARC下会。

因为@synchronized{...}等价于

```c
{
    id _rethrow = 0; 
    id _sync_obj = (id)obj; 
    objc_sync_enter(_sync_obj);
    try {
        struct _SYNC_EXIT {
            _SYNC_EXIT(id arg) : sync_exit(arg) {}
            ~_SYNC_EXIT() {objc_sync_exit(sync_exit);}
            id sync_exit;
        } _sync_exit(_sync_obj);

        NSLog((NSString *)&__NSConstantStringImpl__var_folders_5z_1pxqzfcn77s2n7z4gmr63sdr0000gn_T_main_4eec76_mi_0);
    } catch (id e) {
      _rethrow = e;
    }
    {
        struct _FIN {
            _FIN(id reth) : rethrow(reth) {}
            ~_FIN() { if (rethrow) objc_exception_throw(rethrow); }
            id rethrow;
        } _fin_force_rethow(_rethrow);
    }
}
```

而id _sync_obj = (id)obj;在MRC下这仅仅是一个赋值语句因此obj的引用计数不变，但在ARC下编译后会插入一条retain语句。当@synchronized代码块执行完后，临时变量销毁时又会释放对象。因此ARC下在@synchronized代码块内同步对象是被强引用的，代码块没执行完同步对象是不会被销毁的。

3.假如你传入 `@synchronized` 的对象在 `@synchronized` 的代码块里面被赋值为 `nil` 将会怎么样？

对本次加解锁不会造成影响，但是当被置为nil后，如果此时恰好有一个线程执行到@synchronized，那么@synchronized(obj) 相当于@synchronized(nil)即无锁状态，于是会导致多个线程同时访问临界区代码，造成线程安全问题。

4.给@synchronized()传入nil会怎样？

@synchronized(nil)等于没加锁，代码块内的代码可以被多个线程同时访问，会造成线程安全问题。

#### 参考

[@synchronized 内幕揭秘](https://jianli2017.top/wiki/IOS/Runtime/objc/18_syncsize/)   synchronized锁的源码分析

[关于 @synchronized，这儿比你想知道的还要多 ](http://yulingtianxia.com/blog/2015/11/01/More-than-you-want-to-know-about-synchronized/)   很不错，国外的程序员写的，尤其是提出的几个问题

关于try-catch内存管理的

[iOS try-catch memory leak详解](https://blog.csdn.net/lvmaker/article/details/88723115)

[异常安全代码的内存管理需谨慎!](https://yingkong1987.github.io/blog/2014/01/11/bewareofmemorymanagementwithexception-safe.html)

关于TLS的

[线程局部存储](https://yq.aliyun.com/articles/691215)

