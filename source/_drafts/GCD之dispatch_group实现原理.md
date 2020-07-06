---
title: GCD之dispatch_group实现原理
tags:
  - 标签1
  - 标签2
  - 标签3
categories:
  - 分类1
  - 分类2
comments: true
date: 2020-07-06 21:06:35
---

版本`libdispatch-1173.40.5`。

#### struct dispatch_group_s

struct dispatch_group_s结构体的定义：

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

展开宏：

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

之前版本dispatch_group_s里面是有`_dispatch_sema4_t dsema_sema;`成员变量的现在已经没了。这表明实现方式已经更改了。

源码对dg_state做了比较详细的注释：看明白了这里的注释，后面的源码会理解的更快一些（其实应该结合来看，光看其中一个也会比较迷糊）。

```
/*
 * Dispatch Group State:
 *
 * Generation (32 - 63):
 *   32 bit counter that is incremented each time the group value reaaches
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
#define DISPATCH_GROUP_GEN_MASK         0xffffffff00000000ULL   //Generation的mask，主要用于位运算
#define DISPATCH_GROUP_VALUE_MASK       0x00000000fffffffcULL  //...1100 VALUE的mask，也是用于位运算的
#define DISPATCH_GROUP_VALUE_INTERVAL   0x0000000000000004ULL //0100,加减的“1”
#define DISPATCH_GROUP_VALUE_1          DISPATCH_GROUP_VALUE_MASK
#define DISPATCH_GROUP_VALUE_MAX        DISPATCH_GROUP_VALUE_INTERVAL
#define DISPATCH_GROUP_HAS_NOTIFS       0x0000000000000002ULL //0010
#define DISPATCH_GROUP_HAS_WAITERS      0x0000000000000001ULL //0001
```

使用了一个64位的无符号整型变量dg_state来表示dispatch_group的状态。

**Generation**

占据dg_state的全部高32位（32-63），对应dg_gen字段。

每次group value的值变为0的时候，dg_gen就会+1。线程进入等待前会先保存当时的generation的快照，只有当dg_gen发生变化的时候那些wait的线程才会退出等待。

**Value**

占据dg_state的低32位中的第2-31位，对应dg_bits字段。

这30位是用于计数的。每次我们调用dispatch_group_enter时就会减“1”，每次调用dispatch_group_leave时就会加”1“。为啥加引号因为实际上是减`DISPATCH_GROUP_VALUE_INTERVAL`(0x0000000000000004ULL)，因为操作的是第2-31位。这个地方会有点难以理解，因为源码并没有单独用一个变量来计数。

调用dispatch_group_enter减1时是在一个uint32_t变量上减”1“，初始时这30位自然是0，减1将得到-1，但作为一个无符号的32位整型变量来说，此时这30位将全部变为1。当减为DISPATCH_GROUP_VALUE_MAX（其实就是0x0000000000000004ULL）时会发生崩溃提示`"Too many nested calls to dispatch_group_enter()"`嵌套太深。

调用dispatch_group_leave加1时是在一个uint64_t变量上加”1“，如果这30位全为1则加1后会变为0，并产生进位加在高32位上，这个过程表示发生了一次generation。

简单点讲，dg_bits用来进行加减，dg_gen用来记录dg_bits变为0的次数。

**Has Notifs**

占据dg_state的低32位中的第1位，对应dg_bits字段。

表示是否有通知。group上的通知列表不为空时被置为1。

**Has Waiters**

占据dg_state的低32位中的第0位，对应dg_bits字段。

表示是否有正在等待的线程，当有线程调用dispatch_group_wait进入等待时，该标志位会被置为1，这些线程等待Value重新变为0并产生一次generation，在等待之前它们会保存一份generation的快照，退出等待的条件就是比对generation是否发生改变。

其中第0位表示是否有正在等待的线程，当有线程调用dispatch_group_wait进入等待时，该标志位会被置为1，这些线程等待Value重新变为0，在等待之前它们会保存一份generation的快照，会比对generation是否发生改变。

这就是C语言，一个64位的uint64_t被玩出了花。这就是对性能的极致追求，源码里面很多这样的操作。不过对阅读代码来说是真滴不友好。

这里看不懂没有关系，后面结合源码分析的时候会好理解一些。

下面正式源码分析

关于dispatch_group的API其实并不多。

#### dispatch_group_create

源码：

```c
dispatch_group_t
dispatch_group_create(void)
{
	return _dispatch_group_create_with_count(0);
}

DISPATCH_ALWAYS_INLINE
static inline dispatch_group_t
_dispatch_group_create_with_count(uint32_t n)
{
	dispatch_group_t dg = _dispatch_object_alloc(DISPATCH_VTABLE(group),
			sizeof(struct dispatch_group_s));
	dg->do_next = DISPATCH_OBJECT_LISTLESS;
	dg->do_targetq = _dispatch_get_default_queue(false);
	if (n) {
		os_atomic_store2o(dg, dg_bits,
				(uint32_t)-n * DISPATCH_GROUP_VALUE_INTERVAL, relaxed);
		os_atomic_store2o(dg, do_ref_cnt, 1, relaxed); // <rdar://22318411>
	}
	return dg;
}
```

dispatch_group_create函数传入的count为0。因此初始时dg_bits是等于0的。如果是大于0的数会变成`(uint32_t)-n * DISPATCH_GROUP_VALUE_INTERVAL`，(uint32_t)-1其实等于0xffffffff。传1的话dg_bits的高30位就全为1了。这里乘以DISPATCH_GROUP_VALUE_INTERVAL是为了保证操作的是高30位，不能修改了低两位。

#### dispatch_group_enter

源码：

```c
void
dispatch_group_enter(dispatch_group_t dg)
{
	// The value is decremented on a 32bits wide atomic so that the carry
	// for the 0 -> -1 transition is not propagated to the upper 32bits.
	uint32_t old_bits = os_atomic_sub_orig2o(dg, dg_bits,
			DISPATCH_GROUP_VALUE_INTERVAL, acquire);
	uint32_t old_value = old_bits & DISPATCH_GROUP_VALUE_MASK;
	if (unlikely(old_value == 0)) {
		_dispatch_retain(dg); // <rdar://problem/22318411>
	}
	if (unlikely(old_value == DISPATCH_GROUP_VALUE_MAX)) {
		DISPATCH_CLIENT_CRASH(old_bits,
				"Too many nested calls to dispatch_group_enter()");
	}
}
```

原子操作对dg_bits变量的值减”1“（DISPATCH_GROUP_VALUE_INTERVAL 0x0100）。

如果old_value等于DISPATCH_GROUP_VALUE_MAX其实也就是（DISPATCH_GROUP_VALUE_INTERVAL），表示减的实在是太多了（从...000000 --> ...111100 --> ...111000 ---> ...000100）而且又没有WAITERS和NOTIFS，这种情况下直接崩溃"Too many nested calls to dispatch_group_enter()"