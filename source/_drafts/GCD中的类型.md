---
title: GCD中的类型
tags:
  - 标签1
  - 标签2
  - 标签3
categories:
  - 分类1
  - 分类2
comments: true
date: 2020-07-04 14:00:24
---

版本`libdispatch-1173.40.5`。

### GCD中的类型声明

`libdispatch`库是纯C语言编写的，因此并不存在类似面向对象语言中的类、对象之类的概念。

常见的几个类型：dispatch_object_t，dispatch_queue_t，dispatch_group_t，dispatch_semaphore_t。在OC文件中：

```
- (void)viewDidLoad {
    [super viewDidLoad];
    
    dispatch_object_t obj = nil;
    dispatch_semaphore_t sm = nil;
    dispatch_queue_t queue = nil;
}
```

点进去后发现声明如下：

```
OS_OBJECT_DECL_CLASS(dispatch_object);

DISPATCH_DECL(dispatch_queue);

DISPATCH_DECL(dispatch_group);

DISPATCH_DECL(dispatch_semaphore);
```

首先是`OS_OBJECT_DECL_CLASS(name)`宏：

```c
#define OS_OBJECT_DECL_CLASS(name) \
		OS_OBJECT_DECL(name)
		
#define OS_OBJECT_DECL(name, ...) \
		OS_OBJECT_DECL_IMPL(name, <NSObject>)
		
#define OS_OBJECT_DECL_IMPL(name, ...) \
		OS_OBJECT_DECL_PROTOCOL(name, __VA_ARGS__) \
		typedef NSObject<OS_OBJECT_CLASS(name)> \
				* OS_OBJC_INDEPENDENT_CLASS name##_t

#define OS_OBJECT_DECL_PROTOCOL(name, ...) \
		@protocol OS_OBJECT_CLASS(name) __VA_ARGS__ \
		@end
		
#define OS_OBJECT_CLASS(name) OS_##name
```

以`OS_OBJECT_DECL_CLASS(dispatch_object);`为例

它展开后等价于：

```objc
@protocol OS_dispatch_object <NSObject>
@end
typedef NSObject<OS_dispatch_object> * __attribute__((objc_independent_class)) dispatch_object_t;
```

也是说`OS_OBJECT_DECL_CLASS(name)`宏：

1. 定义了一个OS_name的协议，协议遵守父协议NSObject协议。
2. 定义了一个类型别名name_t，原类型是`NSObject<OS_name> *`，即遵守了OS_name协议的NSObject类

name_t指向的对象就是一个遵守了OS_name协议的OC对象。虽说name_t指向的应该是一个OC对象，但是创建的时候并不是通过alloc init创建的还是调用C函数创建的，如`dispatch_semaphore_create`。这是由于`libdispatch`库是纯C语言编写的，它内部实际上指向的是一个C结构体实例。

还有一个宏`OS_OBJECT_DECL_SUBCLASS(name, super)`：

```
#define OS_OBJECT_DECL_SUBCLASS(name, super) \
		OS_OBJECT_DECL_IMPL(name, <OS_OBJECT_CLASS(super)>)
```

展开后：

```objc
@protocol OS_name <OS_super> 
@end 
typedef NSObject<OS_name> * __attribute__((objc_independent_class)) name_t;
```

也是说`OS_OBJECT_DECL_SUBCLASS(name, super)`宏：

1. 定义了一个OS_name的协议，协议**遵守父协议OS_super协议**。这也是语义里的SUBCLASS。
2. 定义了一个类型别名name_t，原类型是`NSObject<OS_name> *`，即遵守了OS_name协议的NSObject类

以上看懂之后，下面的宏就很简单了

```
#define DISPATCH_DECL(name) OS_OBJECT_DECL_SUBCLASS(name, dispatch_object)
#define DISPATCH_DECL_SUBCLASS(name, base) OS_OBJECT_DECL_SUBCLASS(name, base)
```

`DISPATCH_DECL(name)`宏定义了一个协议OS_name，该协议遵守父协议OS_dispatch_object，还定义了一个类型别名name_t，即遵守了OS_name协议的NSObject类。以后看到DISPATCH_DECL宏就表明这里声明的类型是“继承”自dispatch_object的。

`DISPATCH_DECL_SUBCLASS(name, base)`宏定义了一个协议OS_name，该协议遵守父协议OS_base，还定义了一个类型别名name_t，即遵守了OS_name协议的NSObject类。

举个例子：`DISPATCH_DECL_SUBCLASS(dispatch_queue_global, dispatch_queue);`

```objc
@protocol OS_dispatch_queue_global <OS_dispatch_queue> 
@end 
typedef NSObject<OS_dispatch_queue_global> * __attribute__((objc_independent_class)) dispatch_queue_global_t;
```

将上述dispatch_object_t，dispatch_queue_t，dispatch_group_t，dispatch_semaphore_t展开来看一下：

```objc
@protocol OS_dispatch_object <NSObject> 
@end 
typedef NSObject<OS_dispatch_object> * __attribute__((objc_independent_class)) dispatch_object_t;

@protocol OS_dispatch_queue <OS_dispatch_object> 
@end 
typedef NSObject<OS_dispatch_queue> * __attribute__((objc_independent_class)) dispatch_queue_t;

@protocol OS_dispatch_group <OS_dispatch_object> 
@end 
typedef NSObject<OS_dispatch_group> * __attribute__((objc_independent_class)) dispatch_group_t;

@protocol OS_dispatch_semaphore <OS_dispatch_object> 
@end 
typedef NSObject<OS_dispatch_semaphore> * __attribute__((objc_independent_class)) dispatch_semaphore_t;
```

可以看到它们其实都是一个遵守了某个协议的NSObject类型的别名。

```
/*
 * By default, dispatch objects are declared as Objective-C types when building
 * with an Objective-C compiler. This allows them to participate in ARC, in RR
 * management by the Blocks runtime and in leaks checking by the static
 * analyzer, and enables them to be added to Cocoa collections.
 * See <os/object.h> for details.
 */
 OS_OBJECT_DECL_CLASS(dispatch_object);
```

在OC编译器下dispatch objects都被声明为OC类型，主要是为了能够利用ARC的优势。

下面的是纯C编译器下dispatch_object_t的声明：

```c
#else /* Plain C */
typedef union {
	struct _os_object_s *_os_obj;
	struct dispatch_object_s *_do;
	struct dispatch_queue_s *_dq;
	struct dispatch_queue_attr_s *_dqa;
	struct dispatch_group_s *_dg;
	struct dispatch_source_s *_ds;
	struct dispatch_channel_s *_dch;
	struct dispatch_mach_s *_dm;
	struct dispatch_mach_msg_s *_dmsg;
	struct dispatch_semaphore_s *_dsema;
	struct dispatch_data_s *_ddata;
	struct dispatch_io_s *_dchannel;
} dispatch_object_t DISPATCH_TRANSPARENT_UNION;
#define DISPATCH_DECL(name) typedef struct name##_s *name##_t
#define DISPATCH_DECL_SUBCLASS(name, base) typedef base##_t name##_t
#define DISPATCH_GLOBAL_OBJECT(type, object) ((type)&(object))
#define DISPATCH_RETURNS_RETAINED
```

纯C下dispatch_object_t其实是一个联合体类型。

dispatch object在不同语言的源文件中的声明是不一样的。dispatch_object_t在OC中则被声明为遵守了某个协议的NSObject类型，在C中则是一个联合体类型。

### GCD中的类继承关系

宏`DISPATCH_OBJECT_HEADER(x)`是GCD结构体定义里面出现较多的宏。

这宏就是让C语言模拟出面向对象语言继承特性的宏，一些公共变量的类型会根据传入的参数进行替换拼接。这个宏的作用就是把一些公共变量和父类结构体成员变量拷贝到子类里。如果手动拷贝的话不仅麻烦而且容易出错，所以系统就封装了这个宏。

#### struct _os_object_s

```c
typedef struct _os_object_vtable_s {
	_OS_OBJECT_CLASS_HEADER();
} _os_object_vtable_s;

typedef struct _os_object_s {
	_OS_OBJECT_HEADER(
	const _os_object_vtable_s *os_obj_isa,
	os_obj_ref_cnt,
	os_obj_xref_cnt);
} _os_object_s;
```

展开：

```c
typedef struct _os_object_vtable_s {
    void *_os_obj_objc_class_t[5];
} _os_object_vtable_s;

typedef struct _os_object_s {
    const _os_object_vtable_s *os_obj_isa; //有一个isa指针
    int volatile os_obj_ref_cnt;
    int volatile os_obj_xref_cnt;
} _os_object_s;
```

#### struct dispatch_object_s

```
struct dispatch_object_s {
	_DISPATCH_OBJECT_HEADER(object);
};
```

展开：

```c
struct dispatch_object_s {
    struct _os_object_s _as_os_obj[0]; //父类
    const struct dispatch_object_vtable_s *do_vtable;
    int volatile do_ref_cnt;
    int volatile do_xref_cnt;
    struct dispatch_object_s *volatile do_next;
    struct dispatch_queue_s *do_targetq;
    void *do_ctxt;
    void *do_finalizer;
};
```

#### struct dispatch_queue_s

定义：使用宏DISPATCH_QUEUE_CLASS_HEADER

```c
struct dispatch_queue_s {
	DISPATCH_QUEUE_CLASS_HEADER(queue, void *__dq_opaque1);
	/* 32bit hole on LP64 */
} DISPATCH_ATOMIC64_ALIGN;
```

展开：

```c
struct dispatch_queue_s {
    struct dispatch_object_s _as_do[0]; //父类
    struct _os_object_s _as_os_obj[0];
    const struct dispatch_queue_vtable_s *do_vtable;
    int volatile do_ref_cnt;
    int volatile do_xref_cnt;
    struct dispatch_queue_s *volatile do_next;
    struct dispatch_queue_s *do_targetq;
    void *do_ctxt;
    void *do_finalizer;
    void *__dq_opaque1;
    _Static_assert(sizeof(struct { uint64_t volatile dq_state; }) == sizeof(struct { dispatch_lock dq_state_lock; uint32_t dq_state_bits; }), "bogus union");
    union {
        uint64_t volatile dq_state;
        struct {
            dispatch_lock dq_state_lock;
            uint32_t dq_state_bits;
        };
    };
    unsigned long dq_serialnum;
    const char *dq_label;
    _Static_assert(sizeof(struct { uint32_t volatile dq_atomic_flags; }) == sizeof(struct { const uint16_t dq_width; const uint16_t __dq_opaque2; }), "bogus union");
    union {
        uint32_t volatile dq_atomic_flags;
        struct {
            const uint16_t dq_width;
            const uint16_t __dq_opaque2;
        };
    };
    dispatch_priority_t dq_priority; //queue的优先级
    union {
        struct dispatch_queue_specific_head_s *dq_specific_head;
        struct dispatch_source_refs_s *ds_refs;
        struct dispatch_timer_source_refs_s *ds_timer_refs;
        struct dispatch_mach_recv_refs_s *dm_recv_refs;
        struct dispatch_channel_callbacks_s const *dch_callbacks;
    };
    int volatile dq_sref_cnt;
} __attribute__((aligned(8)));
```

dispatch_queue_s结构体是整个GCD中最重要的数据结构了。

#### struct dispatch_continuation_s

定义：

```c
typedef struct dispatch_continuation_s {
	DISPATCH_CONTINUATION_HEADER(continuation);
} *dispatch_continuation_t;
```

continuation使用的是另一个宏：DISPATCH_CONTINUATION_HEADER。

展开：

```c
typedef struct dispatch_continuation_s {
    union {
        const void *do_vtable; 
        uintptr_t dc_flags;
    };
    union {
        pthread_priority_t dc_priority;
        int dc_cache_cnt;
        uintptr_t dc_pad;
    };
    struct dispatch_continuation_s *volatile do_next;
    struct voucher_s *dc_voucher;
    dispatch_function_t dc_func;
    void *dc_ctxt;
    void *dc_data;
    void *dc_other;
} *dispatch_continuation_t;
```

之前版本里的还有一个`struct dispatch_object_s _as_do[0];`成员变量的，代表继承自dispatch_object_s，现在没了。可能觉得没必要有这种耦合。

另：`typedef void (*dispatch_function_t)(void *_Nullable);`，为一个函数指针。dc_func就是为了执行block的。

dispatch_continuation_s 应该是内部对block任务的一层封装。

#### struct dispatch_semaphore_s

```c
struct dispatch_queue_s;

DISPATCH_CLASS_DECL(semaphore, OBJECT);
struct dispatch_semaphore_s {
	DISPATCH_OBJECT_HEADER(semaphore);
	long volatile dsema_value;
	long dsema_orig;
	_dispatch_sema4_t dsema_sema;
};
```

展开：

```c
@protocol OS_dispatch_semaphore <OS_dispatch_object>
@end
@interface OS_dispatch_semaphore () <OS_dispatch_semaphore>
@end
struct dispatch_semaphore_s;
struct dispatch_semaphore_extra_vtable_s {
    unsigned long const do_type;
    void (*const do_dispose)(struct dispatch_semaphore_s *, _Bool *allow_free);
    size_t (*const do_debug)(struct dispatch_semaphore_s *, char *, size_t);
    void (*const do_invoke)(struct dispatch_semaphore_s *, dispatch_invoke_context_t, dispatch_invoke_flags_t);
};
struct dispatch_semaphore_vtable_s {
    void *_os_obj_objc_class_t[5];
    struct dispatch_semaphore_extra_vtable_s _os_obj_vtable;
};
extern const struct dispatch_semaphore_vtable_s _OS_dispatch_semaphore_vtable;
extern const struct dispatch_semaphore_vtable_s OS_dispatch_semaphore_class __asm__("_OBJC_CLASS_$_" "OS_dispatch_semaphore");

struct dispatch_semaphore_s {
    //这里的变量都是通过DISPATCH_OBJECT_HEADER(semaphore);宏展开得来的
    struct dispatch_object_s _as_do[0]; //直接父类
    struct _os_object_s _as_os_obj[0]; //dispatch_object_s的父类_os_object_s
    const struct dispatch_semaphore_vtable_s *do_vtable; //根据传入的名字替换
    int volatile do_ref_cnt;
    int volatile do_xref_cnt; 
    struct dispatch_semaphore_s *volatile do_next; //根据传入的名字替换
    struct dispatch_queue_s *do_targetq; 
    void *do_ctxt; 
    void *do_finalizer;
  
    long volatile dsema_value;
    long dsema_orig;
    _dispatch_sema4_t dsema_sema;
};
```

其中do_vtable和do_next会根据传入的名称替换。其他的就是父类里的变量，其实就是拷贝到自己里面。因为C语言没有继承。

#### struct dispatch_group_s

源码：

```c
DISPATCH_CLASS_DECL(group, OBJECT);
struct dispatch_group_s {
	DISPATCH_OBJECT_HEADER(group);
	DISPATCH_UNION_LE(uint64_t volatile dg_state,
			uint32_t dg_bits,
			uint32_t dg_gen
	) DISPATCH_ATOMIC64_ALIGN;
	struct dispatch_continuation_s *volatile dg_notify_head;
	struct dispatch_continuation_s *volatile dg_notify_tail;
};
```

展开：

```c
struct dispatch_group_s {
    struct dispatch_object_s _as_do[0];
    struct _os_object_s _as_os_obj[0];
    const struct dispatch_group_vtable_s *do_vtable;
    int volatile do_ref_cnt;
    int volatile do_xref_cnt;
    struct dispatch_group_s *volatile do_next;
    struct dispatch_queue_s *do_targetq;
    void *do_ctxt;
    void *do_finalizer;
  
    _Static_assert(sizeof(struct { uint64_t volatile dg_state; }) == sizeof(struct { uint32_t dg_bits; uint32_t dg_gen; }), "bogus union");
    union {
        uint64_t volatile dg_state;
        struct {
            uint32_t dg_bits;
            uint32_t dg_gen;
        };
    } __attribute__((aligned(8)));
    struct dispatch_continuation_s *volatile dg_notify_head;
    struct dispatch_continuation_s *volatile dg_notify_tail;
};
```

之前版本dispatch_group_s里面是有`_dispatch_sema4_t dsema_sema;`成员变量的现在已经没了。有可能实现方式已经更改了。

看它的注释：

```
/*
 * Dispatch Group State:
 *
 * Generation (32 - 63):
 *   32 bit counter that is incremented each time the group value reaches
 *   0 after a dispatch_group_leave. This 32bit word is used to block waiters
 *   (threads in dispatch_group_wait) in _dispatch_wait_on_address() until the
 *   generation changes.
 *
 * Value (2 - 31):
 *   30 bit value counter of the number of times the group was entered.
 *   dispatch_group_enter counts downward on 32bits, and dispatch_group_leave
 *   upward on 64bits, which causes the generation to bump each time the value
 *   reaches 0 again due to carry propagation.
 *
 * Has Notifs (1):
 *   This bit is set when the list of notifications on the group becomes non
 *   empty. It is also used as a lock as the thread that successfuly clears this
 *   bit is the thread responsible for firing the notifications.
 *
 * Has Waiters (0):
 *   This bit is set when there are waiters (threads in dispatch_group_wait)
 *   that need to be woken up the next time the value reaches 0. Waiters take
 *   a snapshot of the generation before waiting and will wait for the
 *   generation to change before they return.
 */
#define DISPATCH_GROUP_GEN_MASK         0xffffffff00000000ULL
#define DISPATCH_GROUP_VALUE_MASK       0x00000000fffffffcULL
#define DISPATCH_GROUP_VALUE_INTERVAL   0x0000000000000004ULL
#define DISPATCH_GROUP_VALUE_1          DISPATCH_GROUP_VALUE_MASK
#define DISPATCH_GROUP_VALUE_MAX        DISPATCH_GROUP_VALUE_INTERVAL
#define DISPATCH_GROUP_HAS_NOTIFS       0x0000000000000002ULL
#define DISPATCH_GROUP_HAS_WAITERS      0x0000000000000001ULL
```

现在都是操作dg_bits进行加减，dg_gen用来记录dg_bits变为0的次数。

1. 高32位dg_gen是用来干嘛的？
2. notify block是如何知道所有block都执行完成了的？



```
os_atomic_cmpxchgv2o(dg, dg_state, old_state, new_state, &old_state, relaxed)
```

展开：

```
({
    __typeof__(__c11_atomic_load(((__typeof__(*(&(dg)->dg_state)) _Atomic *)(&(dg)->dg_state)), memory_order_relaxed)) _r = ((old_state));
    _Bool _b = __c11_atomic_compare_exchange_strong(((__typeof__(*(&(dg)->dg_state)) _Atomic *)(&(dg)->dg_state)), &_r, (new_state), memory_order_relaxed, memory_order_relaxed);
    *((&old_state)) = _r;
    _b;
})
```

`_Bool atomic_compare_exchange_strong( volatile A* obj, C* expected, C desired );`

函数说明：

```
Atomically compares the contents of memory pointed to by obj with the contents of memory pointed to by expected, and if those are bitwise equal, replaces the former with desired (performs read-modify-write operation). Otherwise, loads the actual contents of memory pointed to by obj into *expected (performs load operation).

The result of the comparison: true if *obj was equal to *exp, false otherwise.
```

obj和expected相等则用desired更新obj的数据，否则用obj更新expected的数据。



```c
({
    _Bool _result = 0;
    __typeof__(&(dg)->dg_state) _p = (&(dg)->dg_state);
    old_state = os_atomic_load(_p, relaxed);
    do {
        {
            if ((old_state & DISPATCH_GROUP_VALUE_MASK) == 0) {
                ({
                    __c11_atomic_thread_fence(memory_order_acquire);
                    return 0;
                    __builtin_unreachable();
                });
            } if (unlikely(timeout == 0)) {
                ({
                    __c11_atomic_thread_fence(memory_order_relaxed);
                    return _DSEMA4_TIMEOUT();
                    __builtin_unreachable();
                });
            }
            new_state = old_state | DISPATCH_GROUP_HAS_WAITERS;
            if (unlikely(old_state & DISPATCH_GROUP_HAS_WAITERS)) {
                ({
                    __c11_atomic_thread_fence(memory_order_relaxed);
                    break;
                    __builtin_unreachable();
                });
            } };
        _result = os_atomic_cmpxchgvw(_p, old_state, new_state, &old_state, relaxed);
    } while (unlikely(!_result));
    _result;
});
```



### 参考

[GCD 中的类型](https://blog.csdn.net/u011374318/article/details/87870585)

[GCD源码分析（一）——“对象”和数据结构](https://chy305chy.github.io/2018/11/29/GCD%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%88%E4%B8%80%EF%BC%89%E2%80%94%E2%80%94%E2%80%9C%E5%AF%B9%E8%B1%A1%E2%80%9D%E5%92%8C%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/)  913版本的

