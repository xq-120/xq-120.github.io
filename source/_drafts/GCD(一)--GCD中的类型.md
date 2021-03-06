---
title: GCD(一)--GCD中的类型
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

### GCD中的类在OC中的声明

常见的几个类型：dispatch_object_t，dispatch_queue_t，dispatch_group_t，dispatch_semaphore_t。

在OC文件中：

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

也就是说`OS_OBJECT_DECL_CLASS(name)`宏：

1. 定义了一个OS_name的协议，协议遵守父协议NSObject协议。
2. 定义了一个类型别名name_t，原类型是`NSObject<OS_name> *`，即遵守了OS_name协议的NSObject类

name_t指向的对象就是一个遵守了OS_name协议的OC对象。虽说name_t指向的应该是一个OC对象，但是创建的时候并不是通过alloc init创建的而是调用C函数创建的，如`dispatch_semaphore_create`。这是由于`libdispatch`库是纯C语言编写的，它内部实际上指向的是一个C结构体实例。

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
#if OS_OBJECT_USE_OBJC  //在OC文件中我们只需要看这个条件宏，如果是在C文件中那就是其他定义了。

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

可以看到它们其实都是一个遵守了某个协议的NSObject指针类型的别名。

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
Dispatch Public Headers/object.h头文件的定义，object.h里只暴露了internal.h的部分成员变量。
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
#define DISPATCH_DECL(name) typedef struct name##_s *name##_t   //在C中的定义
#define DISPATCH_DECL_SUBCLASS(name, base) typedef base##_t name##_t
#define DISPATCH_GLOBAL_OBJECT(type, object) ((type)&(object))
#define DISPATCH_RETURNS_RETAINED

Dispatch Project Headers/internal.h 头文件里的定义
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

	struct dispatch_continuation_s *_dc; //_dc已经不暴露了
	struct dispatch_sync_context_s *_dsc;
	struct dispatch_operation_s *_doperation;
	struct dispatch_disk_s *_ddisk;
	struct dispatch_workloop_s *_dwl;
	struct dispatch_lane_s *_dl;
	struct dispatch_queue_static_s *_dsq;
	struct dispatch_queue_global_s *_dgq;
	struct dispatch_queue_pthread_root_s *_dpq;
	dispatch_queue_class_t _dqu;
	dispatch_lane_class_t _dlu;
	uintptr_t _do_value;
} dispatch_object_t DISPATCH_TRANSPARENT_UNION;
```

纯C下`dispatch_object_t`其实是一个联合体类型。

dispatch_object在不同语言中的声明是不一样的。`dispatch_object_t`在OC中则被声明为遵守了某个协议的NSObject类型，在C中则是一个联合体类型。

另：introspection_private.h

```c
#ifndef __DISPATCH_INDIRECT__
/*
 * Typedefs of opaque types, for direct inclusion of header in lldb expressions
 */
typedef __typeof__(sizeof(int)) size_t;
typedef struct _opaque_pthread_t *pthread_t;
typedef void (*dispatch_function_t)(void *);
typedef struct Block_layout *dispatch_block_t;
typedef struct dispatch_continuation_s *dispatch_continuation_t;
typedef struct dispatch_queue_s *dispatch_queue_t;
typedef struct dispatch_source_s *dispatch_source_t;
typedef struct dispatch_group_s *dispatch_group_t;
typedef struct dispatch_object_s *dispatch_object_t;
#ifndef API_AVAILABLE
#define API_AVAILABLE(...)
#endif
#ifndef DISPATCH_EXPORT
#define DISPATCH_EXPORT extern
#endif
#endif // __DISPATCH_INDIRECT__
```

_s结尾的都是结构体类型，_t结尾的都是对应结构体指针类型。

### GCD中的类结构体

`libdispatch`库是纯C语言编写的，因此没有像OC中的类这样的东西，有的只是一个个结构体。

为了让C语言模拟出面向对象语言继承的特性，一种办法就是把父结构体里的成员变量拷贝到子结构体里。手动拷贝的话不仅麻烦而且容易出错，因此源码里封装了很多宏避免手动拷贝，比如宏`DISPATCH_OBJECT_HEADER(x)`、`DISPATCH_QUEUE_CLASS_HEADER(x, __pointer_sized_field__)`等 。

结构体的"继承关系图"：

![](https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/GCD%E7%B1%BB%E7%BB%A7%E6%89%BF%E5%9B%BE.png)

#### struct _os_object_s

定义：

使用了宏`_OS_OBJECT_HEADER(isa, ref_cnt, xref_cnt) `

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

定义：

使用了宏`_DISPATCH_OBJECT_HEADER(x)`

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

定义：

使用了宏`DISPATCH_QUEUE_CLASS_HEADER(x, __pointer_sized_field__)`

```c
DISPATCH_CLASS_DECL(queue, QUEUE);

struct dispatch_queue_s {
	DISPATCH_QUEUE_CLASS_HEADER(queue, void *__dq_opaque1);
	/* 32bit hole on LP64 */
} DISPATCH_ATOMIC64_ALIGN;
```

展开：

```c
@protocol OS_dispatch_queue <OS_dispatch_object>
@end
@interface OS_dispatch_queue () <OS_dispatch_queue>
@end
struct dispatch_queue_s;
struct dispatch_queue_extra_vtable_s {
    unsigned long const do_type;
    void (*const do_dispose)(struct dispatch_queue_s *, _Bool *allow_free);
    size_t (*const do_debug)(struct dispatch_queue_s *, char *, size_t);
    void (*const do_invoke)(struct dispatch_queue_s *, dispatch_invoke_context_t, dispatch_invoke_flags_t);
    void (*const dq_activate)(dispatch_queue_class_t);
    void (*const dq_wakeup)(dispatch_queue_class_t, dispatch_qos_t, dispatch_wakeup_flags_t);
    void (*const dq_push)(dispatch_queue_class_t, dispatch_object_t, dispatch_qos_t);
};
struct dispatch_queue_vtable_s {
    void *_os_obj_objc_class_t[5];
    struct dispatch_queue_extra_vtable_s _os_obj_vtable;
};
extern const struct dispatch_queue_vtable_s _OS_dispatch_queue_vtable;
extern const struct dispatch_queue_vtable_s OS_dispatch_queue_class __asm__("_OBJC_CLASS_$_" "OS_dispatch_queue");

struct dispatch_queue_s {
    struct dispatch_object_s _as_do[0]; //父类
    struct _os_object_s _as_os_obj[0];
    const struct dispatch_queue_vtable_s *do_vtable; //虚函数表，模拟多态调用。每个子类里都有一个自己的do_vtable，里面声名了一些函数指针，可以看做是一个函数派发表。
    int volatile do_ref_cnt;
    int volatile do_xref_cnt;
    struct dispatch_queue_s *volatile do_next;
    struct dispatch_queue_s *do_targetq;
    void *do_ctxt;
    void *do_finalizer;
  
    void *__dq_opaque1;
  
    _Static_assert(sizeof(struct { uint64_t volatile dq_state; }) == sizeof(struct { dispatch_lock dq_state_lock; uint32_t dq_state_bits; }), "bogus union");
    union {
        uint64_t volatile dq_state; //队列状态,非常重要。queue_internal.h有比较详细的解释,读懂了dq_state对整个任务的执行过程有很大帮助
        struct {
            dispatch_lock dq_state_lock; //一个32位的整型数
            uint32_t dq_state_bits;
        };
    };
    unsigned long dq_serialnum;
    const char *dq_label;
    _Static_assert(sizeof(struct { uint32_t volatile dq_atomic_flags; }) == sizeof(struct { const uint16_t dq_width; const uint16_t __dq_opaque2; }), "bogus union");
    union {
        uint32_t volatile dq_atomic_flags;
        struct {
            const uint16_t dq_width; //串行队列为1，全局队列为0xfff,自定义并发队列为0xffe
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

#### struct dispatch_queue_attr_s

队列的属性queue_attr

定义：

```c
struct dispatch_queue_attr_s {
    OS_OBJECT_STRUCT_HEADER(dispatch_queue_attr);
};
```

展开：

```c
struct dispatch_queue_attr_s {
    const struct dispatch_queue_attr_vtable_s *do_vtable;
    int volatile do_ref_cnt;
    int volatile do_xref_cnt;
};
```

跟它相关的queue_attr_info：

```c
typedef struct dispatch_queue_attr_info_s {
	dispatch_qos_t dqai_qos : 8;
	int      dqai_relpri : 8;
	uint16_t dqai_overcommit:2;
	uint16_t dqai_autorelease_frequency:2;
	uint16_t dqai_concurrent:1;  
	uint16_t dqai_inactive:1;
} dispatch_queue_attr_info_t;

typedef enum {
	_dispatch_queue_attr_overcommit_unspecified = 0,
	_dispatch_queue_attr_overcommit_enabled,
	_dispatch_queue_attr_overcommit_disabled,
} _dispatch_queue_attr_overcommit_t;
```

#### dispatch_lane_class_t，dispatch_queue_class_t

```c
// Lane cluster class: type for all the queues that have a single head/tail pair
typedef union {
	struct dispatch_lane_s *_dl;
	struct dispatch_queue_static_s *_dsq; 
	struct dispatch_queue_global_s *_dgq;
	struct dispatch_queue_pthread_root_s *_dpq;
	struct dispatch_source_s *_ds;
	struct dispatch_channel_s *_dch;
	struct dispatch_mach_s *_dm;
#ifdef __OBJC__
	id<OS_dispatch_queue> _objc_dq; // unsafe cast for the sake of object.m
#endif
} dispatch_lane_class_t DISPATCH_TRANSPARENT_UNION;

// Dispatch queue cluster class: type for any dispatch_queue_t
typedef union {
	struct dispatch_queue_s *_dq;  //
	struct dispatch_workloop_s *_dwl; //
	struct dispatch_lane_s *_dl;
	struct dispatch_queue_static_s *_dsq;
	struct dispatch_queue_global_s *_dgq;
	struct dispatch_queue_pthread_root_s *_dpq;
	struct dispatch_source_s *_ds;
	struct dispatch_channel_s *_dch;
	struct dispatch_mach_s *_dm;
	dispatch_lane_class_t _dlu;
#ifdef __OBJC__
	id<OS_dispatch_queue> _objc_dq;
#endif
} dispatch_queue_class_t DISPATCH_TRANSPARENT_UNION;
```

最好参考`queue_internal.h`头文件里的注释。

#### struct dispatch_lane_s

定义：

使用了宏`DISPATCH_LANE_CLASS_HEADER(x)`

```c
typedef struct dispatch_lane_s {
    DISPATCH_LANE_CLASS_HEADER(lane);
    /* 32bit hole on LP64 */
} DISPATCH_ATOMIC64_ALIGN *dispatch_lane_t;
```

展开：

```c
typedef struct dispatch_lane_s {
    struct dispatch_queue_s _as_dq[0]; //父类
    struct dispatch_object_s _as_do[0];
    struct _os_object_s _as_os_obj[0];
    const struct dispatch_lane_vtable_s *do_vtable;
    int volatile do_ref_cnt;
    int volatile do_xref_cnt;
    struct dispatch_lane_s *volatile do_next; //队列也是一个单链表
    struct dispatch_queue_s *do_targetq;
    void *do_ctxt;
    void *do_finalizer;
  
    struct dispatch_object_s *volatile dq_items_tail; //指向任务链表的尾部
  
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
            const uint16_t dq_width; //队列宽度，串行队列为1，并发队列为一个较大的数
            const uint16_t __dq_opaque2;
        };
    };
    dispatch_priority_t dq_priority;
    union {
        struct dispatch_queue_specific_head_s *dq_specific_head;
        struct dispatch_source_refs_s *ds_refs;
        struct dispatch_timer_source_refs_s *ds_timer_refs;
        struct dispatch_mach_recv_refs_s *dm_recv_refs;
        struct dispatch_channel_callbacks_s const *dch_callbacks;
    };
    int volatile dq_sref_cnt;
  
    dispatch_unfair_lock_s dq_sidelock;
    struct dispatch_object_s *volatile dq_items_head; //指向任务链表的头部，里面的元素可以是dc也可以是dq
    uint32_t dq_side_suspend_cnt;
} __attribute__((aligned(8))) *dispatch_lane_t;
```

我们派发到队列的block作为一个个的Item添加到队列的单链表中

#### struct dispatch_queue_static_s

定义：

使用了宏`DISPATCH_LANE_CLASS_HEADER(x)`

```c
// Cache aligned type for static queues (main queue, manager)
struct dispatch_queue_static_s {
	struct dispatch_lane_s _as_dl[0]; \
	DISPATCH_LANE_CLASS_HEADER(lane);
} DISPATCH_CACHELINE_ALIGN;
```

展开：

```c
struct dispatch_queue_static_s {
    struct dispatch_lane_s _as_dl[0];
    struct dispatch_queue_s _as_dq[0];
    struct dispatch_object_s _as_do[0];
    struct _os_object_s _as_os_obj[0];
    const struct dispatch_lane_vtable_s *do_vtable;
    int volatile do_ref_cnt;
    int volatile do_xref_cnt;
    struct dispatch_lane_s *volatile do_next;
    struct dispatch_queue_s *do_targetq;
    void *do_ctxt;
    void *do_finalizer;
    struct dispatch_object_s *volatile dq_items_tail;
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
    dispatch_priority_t dq_priority;
    union {
        struct dispatch_queue_specific_head_s *dq_specific_head;
        struct dispatch_source_refs_s *ds_refs;
        struct dispatch_timer_source_refs_s *ds_timer_refs;
        struct dispatch_mach_recv_refs_s *dm_recv_refs;
        struct dispatch_channel_callbacks_s const *dch_callbacks;
    };
    int volatile dq_sref_cnt;
    dispatch_unfair_lock_s dq_sidelock;
    struct dispatch_object_s *volatile dq_items_head;
    uint32_t dq_side_suspend_cnt;
} DISPATCH_CACHELINE_ALIGN;
```

#### struct dispatch_queue_global_s

定义：

使用了宏`DISPATCH_QUEUE_ROOT_CLASS_HEADER(x)`

```c
struct dispatch_queue_global_s {
    DISPATCH_QUEUE_ROOT_CLASS_HEADER(lane);
} DISPATCH_CACHELINE_ALIGN;
```

展开：

```c
struct dispatch_queue_global_s {
    struct dispatch_queue_s _as_dq[0]; //父类
    struct dispatch_object_s _as_do[0];
    struct _os_object_s _as_os_obj[0];
    const struct dispatch_lane_vtable_s *do_vtable;
    int volatile do_ref_cnt;
    int volatile do_xref_cnt;
    struct dispatch_lane_s *volatile do_next;
    struct dispatch_queue_s *do_targetq;
    void *do_ctxt; //dispatch_pthread_root_queue_context_t
    void *do_finalizer;
  
    struct dispatch_object_s *volatile dq_items_tail;
  
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
    dispatch_priority_t dq_priority;
    union {
        struct dispatch_queue_specific_head_s *dq_specific_head;
        struct dispatch_source_refs_s *ds_refs;
        struct dispatch_timer_source_refs_s *ds_timer_refs;
        struct dispatch_mach_recv_refs_s *dm_recv_refs;
        struct dispatch_channel_callbacks_s
        const *dch_callbacks;
    };
    int volatile dq_sref_cnt;
  
    int volatile dgq_thread_pool_size; //线程池大小，全局队列才有该属性
    struct dispatch_object_s *volatile dq_items_head;
    int volatile dgq_pending;
} DISPATCH_CACHELINE_ALIGN;
```

#### struct dispatch_queue_pthread_root_s

```c
typedef struct dispatch_queue_pthread_root_s {
    struct dispatch_queue_global_s _as_dgq[0];
    DISPATCH_QUEUE_ROOT_CLASS_HEADER(lane);
    struct dispatch_pthread_root_queue_context_s dpq_ctxt;
} *dispatch_queue_pthread_root_t;
```

展开：

```c
typedef struct dispatch_queue_pthread_root_s {
    struct dispatch_queue_global_s _as_dgq[0]; //继承自struct dispatch_queue_global_s
    struct dispatch_queue_s _as_dq[0];
    struct dispatch_object_s _as_do[0];
    struct _os_object_s _as_os_obj[0];
    const struct dispatch_lane_vtable_s *do_vtable;
    int volatile do_ref_cnt;
    int volatile do_xref_cnt;
    struct dispatch_lane_s *volatile do_next;
    struct dispatch_queue_s *do_targetq;
    void *do_ctxt;
    void *do_finalizer;
    struct dispatch_object_s *volatile dq_items_tail;
    _Static_assert(sizeof(struct { uint64_t volatile dq_state; }) == sizeof(struct { dispatch_lock dq_state_lock; uint32_t dq_state_bits; }), "bogus union");
    union {
        uint64_t volatile dq_state;
        struct {
            dispatch_lock dq_state_lock; uint32_t dq_state_bits;
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
    dispatch_priority_t dq_priority;
    union {
        struct dispatch_queue_specific_head_s *dq_specific_head;
        struct dispatch_source_refs_s *ds_refs;
        struct dispatch_timer_source_refs_s *ds_timer_refs;
        struct dispatch_mach_recv_refs_s *dm_recv_refs;
        struct dispatch_channel_callbacks_s const *dch_callbacks;
    };
    int volatile dq_sref_cnt;
    int volatile dgq_thread_pool_size;
    struct dispatch_object_s *volatile dq_items_head;
    int volatile dgq_pending;
    struct dispatch_pthread_root_queue_context_s dpq_ctxt;
} *dispatch_queue_pthread_root_t;
```

#### struct dispatch_pthread_root_queue_context_s

```c
#if DISPATCH_USE_PTHREAD_POOL
typedef struct dispatch_pthread_root_queue_context_s {
#if !defined(_WIN32)
	pthread_attr_t dpq_thread_attr;
#endif
	dispatch_block_t dpq_thread_configure;
	struct dispatch_semaphore_s dpq_thread_mediator; //线程调度器，是一个信号量。
	dispatch_pthread_root_queue_observer_hooks_s dpq_observer_hooks;
} *dispatch_pthread_root_queue_context_t;
#endif // DISPATCH_USE_PTHREAD_POOL
```

#### struct dispatch_continuation_s

定义：

```c
typedef struct dispatch_continuation_s {
	DISPATCH_CONTINUATION_HEADER(continuation);
} *dispatch_continuation_t;
```

展开：

```c
typedef struct dispatch_continuation_s {
    union {
        const void *do_vtable; 
        uintptr_t dc_flags; //dc的类型标志
    };
    union {
        pthread_priority_t dc_priority; //任务优先级
        int dc_cache_cnt;
        uintptr_t dc_pad;
    };
    struct dispatch_continuation_s *volatile do_next; //下一个任务，dispatch_continuation_s是一个单链表。
    struct voucher_s *dc_voucher;
    dispatch_function_t dc_func;  //block所在的函数
    void *dc_ctxt;
    void *dc_data; //可能会保存当前队列
    void *dc_other;
} *dispatch_continuation_t;
```

之前版本里的还有一个`struct dispatch_object_s _as_do[0];`成员变量的，代表继承自dispatch_object_s，现在没了，感觉有问题啊，是不是漏了，要不然队列还怎么添加任务啊类型都不一样了（ps:共用体dispatch_object_t里面有 `struct dispatch_continuation_s *_dc;` 所以没问题）。

另：`typedef void (*dispatch_function_t)(void *_Nullable);`，为一个函数指针。dc_func就是为了执行block的。

dispatch_continuation_s 是内部对block任务的一层封装。

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
    struct dispatch_object_s _as_do[0]; //父类
    struct _os_object_s _as_os_obj[0]; 
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

定义：

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

之前版本dispatch_group_s里面是有`_dispatch_sema4_t dsema_sema;`成员变量的现在已经没了。说明实现方式已经更改了。

#### vtable

queue_internal.h中声名了很多queue相关的do_vtable成员变量的类型

比如`const struct dispatch_queue_vtable_s *do_vtable;`中的struct dispatch_queue_vtable_s结构体的定义。

do_vtable相当于结构体里的一个函数派发表。

```
DISPATCH_CLASS_DECL(queue, QUEUE);
DISPATCH_CLASS_DECL_BARE(lane, QUEUE);
DISPATCH_CLASS_DECL(workloop, QUEUE);
DISPATCH_SUBCLASS_DECL(queue_serial, queue, lane);
DISPATCH_SUBCLASS_DECL(queue_main, queue_serial, lane);
DISPATCH_SUBCLASS_DECL(queue_concurrent, queue, lane);
DISPATCH_SUBCLASS_DECL(queue_global, queue, lane);
#if DISPATCH_USE_PTHREAD_ROOT_QUEUES
DISPATCH_INTERNAL_SUBCLASS_DECL(queue_pthread_root, queue, lane);
#endif
DISPATCH_INTERNAL_SUBCLASS_DECL(queue_runloop, queue_serial, lane);
DISPATCH_INTERNAL_SUBCLASS_DECL(queue_mgr, queue_serial, lane);
```

这里的宏就不一一展开了。

初始化在init.c：

```c
/*
 * Dispatch queue cluster
 */

DISPATCH_NOINLINE
static void
_dispatch_queue_no_activate(dispatch_queue_class_t dqu)
{
	DISPATCH_INTERNAL_CRASH(dx_type(dqu._dq), "dq_activate called");
}

DISPATCH_VTABLE_INSTANCE(queue,
	// This is the base class for queues, no objects of this type are made
	.do_type        = _DISPATCH_QUEUE_CLUSTER,
	.do_dispose     = _dispatch_object_no_dispose,
	.do_debug       = _dispatch_queue_debug,
	.do_invoke      = _dispatch_object_no_invoke,

	.dq_activate    = _dispatch_queue_no_activate,
);

DISPATCH_VTABLE_INSTANCE(workloop,
	.do_type        = DISPATCH_WORKLOOP_TYPE,
	.do_dispose     = _dispatch_workloop_dispose,
	.do_debug       = _dispatch_queue_debug,
	.do_invoke      = _dispatch_workloop_invoke,

	.dq_activate    = _dispatch_queue_no_activate,
	.dq_wakeup      = _dispatch_workloop_wakeup,
	.dq_push        = _dispatch_workloop_push,
);

DISPATCH_VTABLE_SUBCLASS_INSTANCE(queue_serial, lane,
	.do_type        = DISPATCH_QUEUE_SERIAL_TYPE,
	.do_dispose     = _dispatch_lane_dispose,
	.do_debug       = _dispatch_queue_debug,
	.do_invoke      = _dispatch_lane_invoke,

	.dq_activate    = _dispatch_lane_activate,
	.dq_wakeup      = _dispatch_lane_wakeup,
	.dq_push        = _dispatch_lane_push,
);

DISPATCH_VTABLE_SUBCLASS_INSTANCE(queue_concurrent, lane,
	.do_type        = DISPATCH_QUEUE_CONCURRENT_TYPE,
	.do_dispose     = _dispatch_lane_dispose,
	.do_debug       = _dispatch_queue_debug,
	.do_invoke      = _dispatch_lane_invoke,

	.dq_activate    = _dispatch_lane_activate,
	.dq_wakeup      = _dispatch_lane_wakeup,
	.dq_push        = _dispatch_lane_concurrent_push,
);

DISPATCH_VTABLE_SUBCLASS_INSTANCE(queue_global, lane,
	.do_type        = DISPATCH_QUEUE_GLOBAL_ROOT_TYPE,
	.do_dispose     = _dispatch_object_no_dispose,
	.do_debug       = _dispatch_queue_debug,
	.do_invoke      = _dispatch_object_no_invoke,

	.dq_activate    = _dispatch_queue_no_activate,
	.dq_wakeup      = _dispatch_root_queue_wakeup,
	.dq_push        = _dispatch_root_queue_push,
);

#if DISPATCH_USE_PTHREAD_ROOT_QUEUES
DISPATCH_VTABLE_SUBCLASS_INSTANCE(queue_pthread_root, lane,
	.do_type        = DISPATCH_QUEUE_PTHREAD_ROOT_TYPE,
	.do_dispose     = _dispatch_pthread_root_queue_dispose,
	.do_debug       = _dispatch_queue_debug,
	.do_invoke      = _dispatch_object_no_invoke,

	.dq_activate    = _dispatch_queue_no_activate,
	.dq_wakeup      = _dispatch_root_queue_wakeup,
	.dq_push        = _dispatch_root_queue_push,
);
#endif // DISPATCH_USE_PTHREAD_ROOT_QUEUES

DISPATCH_VTABLE_SUBCLASS_INSTANCE(queue_mgr, lane,
	.do_type        = DISPATCH_QUEUE_MGR_TYPE,
	.do_dispose     = _dispatch_object_no_dispose,
	.do_debug       = _dispatch_queue_debug,
#if DISPATCH_USE_MGR_THREAD
	.do_invoke      = _dispatch_mgr_thread,
#else
	.do_invoke      = _dispatch_object_no_invoke,
#endif

	.dq_activate    = _dispatch_queue_no_activate,
	.dq_wakeup      = _dispatch_mgr_queue_wakeup,
	.dq_push        = _dispatch_mgr_queue_push,
);

DISPATCH_VTABLE_SUBCLASS_INSTANCE(queue_main, lane,
	.do_type        = DISPATCH_QUEUE_MAIN_TYPE,
	.do_dispose     = _dispatch_lane_dispose,
	.do_debug       = _dispatch_queue_debug,
	.do_invoke      = _dispatch_lane_invoke,

	.dq_activate    = _dispatch_queue_no_activate,
	.dq_wakeup      = _dispatch_main_queue_wakeup,
	.dq_push        = _dispatch_main_queue_push,
);

#if DISPATCH_COCOA_COMPAT
DISPATCH_VTABLE_SUBCLASS_INSTANCE(queue_runloop, lane,
	.do_type        = DISPATCH_QUEUE_RUNLOOP_TYPE,
	.do_dispose     = _dispatch_runloop_queue_dispose,
	.do_debug       = _dispatch_queue_debug,
	.do_invoke      = _dispatch_lane_invoke,

	.dq_activate    = _dispatch_queue_no_activate,
	.dq_wakeup      = _dispatch_runloop_queue_wakeup,
	.dq_push        = _dispatch_lane_push,
);
#endif

DISPATCH_VTABLE_INSTANCE(source,
	.do_type        = DISPATCH_SOURCE_KEVENT_TYPE,
	.do_dispose     = _dispatch_source_dispose,
	.do_debug       = _dispatch_source_debug,
	.do_invoke      = _dispatch_source_invoke,

	.dq_activate    = _dispatch_source_activate,
	.dq_wakeup      = _dispatch_source_wakeup,
	.dq_push        = _dispatch_lane_push,
);

DISPATCH_VTABLE_INSTANCE(channel,
	.do_type        = DISPATCH_CHANNEL_TYPE,
	.do_dispose     = _dispatch_channel_dispose,
	.do_debug       = _dispatch_channel_debug,
	.do_invoke      = _dispatch_channel_invoke,

	.dq_activate    = _dispatch_lane_activate,
	.dq_wakeup      = _dispatch_channel_wakeup,
	.dq_push        = _dispatch_lane_push,
);

#if HAVE_MACH
DISPATCH_VTABLE_INSTANCE(mach,
	.do_type        = DISPATCH_MACH_CHANNEL_TYPE,
	.do_dispose     = _dispatch_mach_dispose,
	.do_debug       = _dispatch_mach_debug,
	.do_invoke      = _dispatch_mach_invoke,

	.dq_activate    = _dispatch_mach_activate,
	.dq_wakeup      = _dispatch_mach_wakeup,
	.dq_push        = _dispatch_lane_push,
);
#endif // HAVE_MACH
```

#### dispatch_wakeup_flags_t

队列唤醒标志类型

```c
DISPATCH_OPTIONS(dispatch_wakeup_flags, uint32_t,
	// The caller of dx_wakeup owns two internal refcounts on the object being
	// woken up. Two are needed for WLH wakeups where two threads need
	// the object to remain valid in a non-coordinated way
	// - the thread doing the poke for the duration of the poke
	// - drainers for the duration of their drain
	DISPATCH_WAKEUP_CONSUME_2               = 0x00000001,

	// Some change to the object needs to be published to drainers.
	// If the drainer isn't the same thread, some scheme such as the dispatch
	// queue DIRTY bit must be used and a release barrier likely has to be
	// involved before dx_wakeup returns
	DISPATCH_WAKEUP_MAKE_DIRTY              = 0x00000002,

	// This wakeup is made by a sync owner that still holds the drain lock
	DISPATCH_WAKEUP_BARRIER_COMPLETE        = 0x00000004,

	// This wakeup is caused by a dispatch_block_wait()
	DISPATCH_WAKEUP_BLOCK_WAIT              = 0x00000008,

	// This wakeup may cause the source to leave its DSF_NEEDS_EVENT state
	DISPATCH_WAKEUP_EVENT                   = 0x00000010,

	// This wakeup is allowed to clear the ACTIVATING state of the object
	DISPATCH_WAKEUP_CLEAR_ACTIVATING        = 0x00000020,
);
```

#### dispatch_invoke_flags_t

队列调用标志类型

```c
DISPATCH_OPTIONS(dispatch_invoke_flags, uint32_t,
	DISPATCH_INVOKE_NONE					= 0x00000000,

	// Invoke modes
	//
	// @const DISPATCH_INVOKE_STEALING
	// This invoke is a stealer, meaning that it doesn't own the
	// enqueue lock at drain lock time.
	//
	// @const DISPATCH_INVOKE_WLH
	// This invoke is for a bottom WLH
	//
	DISPATCH_INVOKE_STEALING				= 0x00000001,
	DISPATCH_INVOKE_WLH						= 0x00000002,

	// Misc flags
	//
	// @const DISPATCH_INVOKE_ASYNC_REPLY
	// An asynchronous reply to a message is being handled.
	//
	// @const DISPATCH_INVOKE_DISALLOW_SYNC_WAITERS
	// The next serial drain should not allow sync waiters.
	//
	DISPATCH_INVOKE_ASYNC_REPLY				= 0x00000004,
	DISPATCH_INVOKE_DISALLOW_SYNC_WAITERS	= 0x00000008,

	// Below this point flags are propagated to recursive calls to drain(),
	// continuation pop() or dx_invoke().
#define _DISPATCH_INVOKE_PROPAGATE_MASK		  0xffff0000u

	// Drain modes
	//
	// @const DISPATCH_INVOKE_WORKER_DRAIN
	// Invoke has been issued by a worker thread (work queue thread, or
	// pthread root queue) drain. This flag is NOT set when the main queue,
	// manager queue or runloop queues are drained
	//
	// @const DISPATCH_INVOKE_REDIRECTING_DRAIN
	// Has only been draining concurrent queues so far
	// Implies DISPATCH_INVOKE_WORKER_DRAIN
	//
	// @const DISPATCH_INVOKE_MANAGER_DRAIN
	// We're draining from a manager context
	//
	// @const DISPATCH_INVOKE_THREAD_BOUND
	// We're draining from the context of a thread-bound queue (main thread)
	//
	// @const DISPATCH_INVOKE_WORKLOOP_DRAIN
	// The queue at the bottom of this drain is a workloop that supports
	// reordering.
	//
	DISPATCH_INVOKE_WORKER_DRAIN			= 0x00010000,
	DISPATCH_INVOKE_REDIRECTING_DRAIN		= 0x00020000,
	DISPATCH_INVOKE_MANAGER_DRAIN			= 0x00040000,
	DISPATCH_INVOKE_THREAD_BOUND			= 0x00080000,
	DISPATCH_INVOKE_WORKLOOP_DRAIN			= 0x00100000,
#define _DISPATCH_INVOKE_DRAIN_MODE_MASK	  0x00ff0000u

	// Autoreleasing modes
	//
	// @const DISPATCH_INVOKE_AUTORELEASE_ALWAYS
	// Always use autoreleasepools around callouts
	//
	// @const DISPATCH_INVOKE_AUTORELEASE_NEVER
	// Never use autoreleasepools around callouts
	//
	DISPATCH_INVOKE_AUTORELEASE_ALWAYS		= 0x01000000,
	DISPATCH_INVOKE_AUTORELEASE_NEVER		= 0x02000000,
#define _DISPATCH_INVOKE_AUTORELEASE_MASK	  0x03000000u
);
```

#### DC_FLAG

continuation对象的种类标志

```c
// continuation is a dispatch_sync or dispatch_barrier_sync
#define DC_FLAG_SYNC_WAITER				0x001ul
// continuation acts as a barrier
#define DC_FLAG_BARRIER					0x002ul
// continuation resources are freed on run
// this is set on async or for non event_handler source handlers
#define DC_FLAG_CONSUME					0x004ul
// continuation has a group in dc_data
#define DC_FLAG_GROUP_ASYNC				0x008ul
// continuation function is a block (copied in dc_ctxt)
#define DC_FLAG_BLOCK					0x010ul
// continuation function is a block with private data, implies BLOCK_BIT
#define DC_FLAG_BLOCK_WITH_PRIVATE_DATA	0x020ul
// source handler requires fetching context from source
#define DC_FLAG_FETCH_CONTEXT			0x040ul
// continuation is a dispatch_async_and_wait
#define DC_FLAG_ASYNC_AND_WAIT			0x080ul
// bit used to make sure dc_flags is never 0 for allocated continuations
#define DC_FLAG_ALLOCATED				0x100ul
// continuation is an internal implementation detail that should not be
// introspected
#define DC_FLAG_NO_INTROSPECTION		0x200ul
// The item is a channel item, not a continuation
#define DC_FLAG_CHANNEL_ITEM			0x400ul
```

#### dispatch_queue_flags

队列标志：dq_atomic_flags

```c
#pragma mark -
#pragma mark dispatch_queue_flags, dq_state

DISPATCH_OPTIONS(dispatch_queue_flags, uint32_t,
	DQF_NONE                = 0x00000000,
	DQF_AUTORELEASE_ALWAYS  = 0x00010000,
	DQF_AUTORELEASE_NEVER   = 0x00020000,
#define _DQF_AUTORELEASE_MASK 0x00030000
	DQF_THREAD_BOUND        = 0x00040000, // queue is bound to a thread
	DQF_BARRIER_BIT         = 0x00080000, // queue is a barrier on its target
	DQF_TARGETED            = 0x00100000, // queue is targeted by another object
	DQF_LABEL_NEEDS_FREE    = 0x00200000, // queue label was strdup()ed
	DQF_MUTABLE             = 0x00400000,
	DQF_RELEASED            = 0x00800000, // xref_cnt == -1

	//
	// Only applies to sources
	//
	// @const DSF_STRICT
	// Semantics of the source are strict (implies DQF_MUTABLE being unset):
	// - handlers can't be changed past activation
	// - EV_VANISHED causes a hard failure
	// - source can't change WLH
	//
	// @const DSF_WLH_CHANGED
	// The wlh for the source changed (due to retarget past activation).
	// Only used for debugging and diagnostics purposes.
	//
	// @const DSF_CANCELED
	// Explicit cancelation has been requested.
	//
	// @const DSF_CANCEL_WAITER
	// At least one caller of dispatch_source_cancel_and_wait() is waiting on
	// the cancelation to finish. DSF_CANCELED must be set if this bit is set.
	//
	// @const DSF_NEEDS_EVENT
	// The source has started to delete its unotes due to cancelation, but
	// couldn't finish its unregistration and is waiting for some asynchronous
	// events to fire to be able to.
	//
	// This flag prevents spurious wakeups when the source state machine
	// requires specific events to make progress. Events that are likely
	// to unblock a source state machine pass DISPATCH_WAKEUP_EVENT
	// which neuters the effect of DSF_NEEDS_EVENT.
	//
	// @const DSF_DELETED
	// The source can now only be used as a queue and is not allowed to register
	// any new unote anymore. All the previously registered unotes are inactive
	// and their knote is gone. However, these previously registered unotes may
	// still be in the process of delivering their last event.
	//
	// Sources have an internal refcount taken always while they use eventing
	// subsystems which is consumed when this bit is set.
	//
	DSF_STRICT              = 0x04000000,
	DSF_WLH_CHANGED         = 0x08000000,
	DSF_CANCELED            = 0x10000000,
	DSF_CANCEL_WAITER       = 0x20000000,
	DSF_NEEDS_EVENT         = 0x40000000,
	DSF_DELETED             = 0x80000000,

#define DQF_FLAGS_MASK        ((dispatch_queue_flags_t)0xffff0000)
#define DQF_WIDTH_MASK        ((dispatch_queue_flags_t)0x0000ffff)
#define DQF_WIDTH(n)          ((dispatch_queue_flags_t)(uint16_t)(n))
);
```



#### TSD和TLS

`线程特有数据`（Thread-Specific Data 或 TSD）

`线程局部存储`（Thread-Local Storage 或 TLS）

### 参考

[GCD 中的类型](https://blog.csdn.net/u011374318/article/details/87870585)

[GCD源码分析（一）——“对象”和数据结构](https://chy305chy.github.io/2018/11/29/GCD%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%88%E4%B8%80%EF%BC%89%E2%80%94%E2%80%94%E2%80%9C%E5%AF%B9%E8%B1%A1%E2%80%9D%E5%92%8C%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/)  913版本的

