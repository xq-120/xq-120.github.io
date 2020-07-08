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

以下为了表述方便加1，减1，都表示加减DISPATCH_GROUP_VALUE_INTERVAL。

下面正式源码分析

关于dispatch_group的API其实并不多。

#### dispatch_group_create

实现：

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

创建一个dispatch_group_t对象。

dispatch_group_create函数传入的count为0。因此初始时dg_bits是等于0的。如果是大于0的数会变成`(uint32_t)-n * DISPATCH_GROUP_VALUE_INTERVAL`，(uint32_t)-1其实等于0xffffffff。传1的话dg_bits的高30位就全为1了。这里乘以DISPATCH_GROUP_VALUE_INTERVAL是为了保证操作的是高30位，不能修改了低两位。

#### dispatch_group_enter

实现：

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

主要作用：

将dg_bits减去“1”（DISPATCH_GROUP_VALUE_INTERVAL）。

os_atomic_sub_orig2o(dg, dg_bits, DISPATCH_GROUP_VALUE_INTERVAL, acquire);

展开：

```
__c11_atomic_fetch_sub(((__typeof__(*((&(dg)->dg_bits))) _Atomic *)((&(dg)->dg_bits))), ((DISPATCH_GROUP_VALUE_INTERVAL)), memory_order_acquire);
```

主要是__c11_atomic_fetch_sub函数的意思

```
T atomic_fetch_sub (volatile A* obj, M val) noexcept;
```

原子操作，将obj的值减去val，并返回obj之前的值。

这里就是原子操作对dg_bits变量的值减”1“（DISPATCH_GROUP_VALUE_INTERVAL 0x0100）并返回dg_bits之前的值。

如果old_value等于DISPATCH_GROUP_VALUE_MAX其实也就是（DISPATCH_GROUP_VALUE_INTERVAL），表示减的实在是太多了（从...000000 --> ...111100 --> ...111000 ---> ...000100）而且又没有WAITERS和NOTIFS，这种情况下直接崩溃"Too many nested calls to dispatch_group_enter()"

#### dispatch_group_leave

实现：

```c
void
dispatch_group_leave(dispatch_group_t dg)
{
	// The value is incremented on a 64bits wide atomic so that the carry for
	// the -1 -> 0 transition increments the generation atomically.
	uint64_t new_state, old_state = os_atomic_add_orig2o(dg, dg_state,
			DISPATCH_GROUP_VALUE_INTERVAL, release);
	uint32_t old_value = (uint32_t)(old_state & DISPATCH_GROUP_VALUE_MASK);

	if (unlikely(old_value == DISPATCH_GROUP_VALUE_1)) {
		old_state += DISPATCH_GROUP_VALUE_INTERVAL;
		do {
			new_state = old_state;
			if ((old_state & DISPATCH_GROUP_VALUE_MASK) == 0) {
				new_state &= ~DISPATCH_GROUP_HAS_WAITERS; //清HAS_WAITERS标志位，wait的时候会设置该标志位
				new_state &= ~DISPATCH_GROUP_HAS_NOTIFS; //清HAS_NOTIFS标志位，dispatch_group_notify会设置该标志位
			} else {
				// If the group was entered again since the atomic_add above,
				// we can't clear the waiters bit anymore as we don't know for
				// which generation the waiters are for
				new_state &= ~DISPATCH_GROUP_HAS_NOTIFS;
			}
			if (old_state == new_state) break;
		} while (unlikely(!os_atomic_cmpxchgv2o(dg, dg_state,
				old_state, new_state, &old_state, relaxed)));
		return _dispatch_group_wake(dg, old_state, true);
	}

	if (unlikely(old_value == 0)) {
		DISPATCH_CLIENT_CRASH((uintptr_t)old_value,
				"Unbalanced call to dispatch_group_leave()");
	}
}
```

该函数大致流程:

1. 原子操作，dg_state的值加上DISPATCH_GROUP_VALUE_INTERVAL，并返回dg_state之前的值。
2. 根据old_state计算出old_value
3. 如果old_value等于DISPATCH_GROUP_VALUE_1，最终将调用_dispatch_group_wake唤醒等待的线程。
4. 如果old_value等于0，则崩溃"Unbalanced call to dispatch_group_leave()"

看下old_value等于0的情况，等于0则表示在本次调用dispatch_group_leave之前，value 值就已经恢复初始状态，enter和leave已经是平衡的了，说明本次调用是没有配对的，因此崩溃"Unbalanced call to dispatch_group_leave()"。

主要是几个原子操作的函数的理解。

os_atomic_add_orig2o：

```
os_atomic_add_orig2o(dg, dg_state, DISPATCH_GROUP_VALUE_INTERVAL, release);
```

展开：

```
__c11_atomic_fetch_add(((__typeof__(*((&(dg)->dg_state))) _Atomic *)((&(dg)->dg_state))), ((DISPATCH_GROUP_VALUE_INTERVAL)), memory_order_release);
```

函数：

```
T atomic_fetch_add (volatile A* obj, M val) noexcept;
```

原子操作，obj的值加上val，并返回obj之前的值。

这里的话就是将dg_state加上DISPATCH_GROUP_VALUE_INTERVAL，并返回dg_state之前的值。

os_atomic_cmpxchgv2o：

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

#### dispatch_group_async

实现：

```c
void
dispatch_group_async(dispatch_group_t dg, dispatch_queue_t dq,
		dispatch_block_t db)
{
	dispatch_continuation_t dc = _dispatch_continuation_alloc();
	uintptr_t dc_flags = DC_FLAG_CONSUME | DC_FLAG_GROUP_ASYNC;
	dispatch_qos_t qos;

	qos = _dispatch_continuation_init(dc, dq, db, 0, dc_flags);
	_dispatch_continuation_group_async(dg, dq, dc, qos);
}

DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_continuation_group_async(dispatch_group_t dg, dispatch_queue_t dq,
		dispatch_continuation_t dc, dispatch_qos_t qos)
{
	dispatch_group_enter(dg);
	dc->dc_data = dg;
	_dispatch_continuation_async(dq, dc, qos, dc->dc_flags);
}
```

该函数大致流程：

1. 创建一个dispatch_continuation_t对象，并用传入的block初始化。其实传入的block都会封装为内部的dispatch_continuation_t对象。
2. 调用dispatch_group_enter
3. 调用_dispatch_continuation_async，将任务push到队列里。

可以看到该函数也会调用一次dispatch_group_enter。按照上面的分析有enter应该还有leave。

```c
DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_continuation_with_group_invoke(dispatch_continuation_t dc)
{
	struct dispatch_object_s *dou = dc->dc_data;
	unsigned long type = dx_type(dou);
	if (type == DISPATCH_GROUP_TYPE) {
		_dispatch_client_callout(dc->dc_ctxt, dc->dc_func);
		_dispatch_trace_item_complete(dc);
		dispatch_group_leave((dispatch_group_t)dou); //pop时会调用dispatch_group_leave
	} else {
		DISPATCH_INTERNAL_CRASH(dx_type(dou), "Unexpected object type");
	}
}
```

在_dispatch_continuation_with_group_invoke函数里可以找到dispatch_group_leave的调用，因此enter和leave也是配对的。

也就是说系统会在执行任务前调用enter，任务执行完后调用leave。这些都不需要我们自己去调用，既然这么优秀那其他地方都用dispatch_group_async不就完事了吗，也不会发生不配对的情况了，连dispatch_group_enter/leave这两个API似乎都没有必要公开了。

真实情况当然不是这样。如果你的block任务是会再发一个子线程去执行，那么这个任务在丢到队列里后其实马上就会执行完了（这里执行完，指得是block任务本身，而任务真正的完成是子线程里的完成才算），这时notify也会马上调用，这应该不是我们所期望的。

这种情况下仅仅使用dispatch_group_async就不能达到要求了。这个时候就只能我们自己手动enter和leave了。

比如：

```objc
- (void)test_dispatch_group_enter_and_leave1 {
    dispatch_group_t group = dispatch_group_create();
    
    __block NSInteger teacherListCnt = 0;
    dispatch_group_async(group, dispatch_get_global_queue(0, 0), ^{ 
        NSLog(@"111s thread:%@", [NSThread currentThread]);
        dispatch_group_enter(group); //计数器 -1
        [self requestTeacherListWithCompletion:^(NSInteger count) { //发起一个网络请求
            teacherListCnt = count;
            //请求完成之后,才会调用completionBlk,进而才会dispatch_group_leave.
            dispatch_group_leave(group); //计数器 +1
        }];
        NSLog(@"111e thread:%@", [NSThread currentThread]);
    });
    
    __block NSInteger liveListCnt = 0;
    dispatch_group_async(group, dispatch_get_global_queue(0, 0), ^{
        NSLog(@"222s thread:%@", [NSThread currentThread]);
        dispatch_group_enter(group); //计数器 -1
        [self requestLiveListWithCompletion:^(NSInteger count) {
            liveListCnt = count;
            dispatch_group_leave(group); //计数器 +1
        }];
        NSLog(@"222e thread:%@", [NSThread currentThread]);
    });
    
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        NSLog(@"dispatch_group_notify");
    });
}
```

这种情况下其实dispatch_group_async都可以去掉了。

#### dispatch_group_wait

实现：

```c
long
dispatch_group_wait(dispatch_group_t dg, dispatch_time_t timeout)
{
	uint64_t old_state, new_state;

	os_atomic_rmw_loop2o(dg, dg_state, old_state, new_state, relaxed, {
		if ((old_state & DISPATCH_GROUP_VALUE_MASK) == 0) {
			os_atomic_rmw_loop_give_up_with_fence(acquire, return 0);
		}
		if (unlikely(timeout == 0)) {
			os_atomic_rmw_loop_give_up(return _DSEMA4_TIMEOUT());
		}
		new_state = old_state | DISPATCH_GROUP_HAS_WAITERS;
		if (unlikely(old_state & DISPATCH_GROUP_HAS_WAITERS)) {
			os_atomic_rmw_loop_give_up(break);
		}
	});

	return _dispatch_group_wait_slow(dg, _dg_state_gen(new_state), timeout);
}
```

大致流程：

1. 简单判断下条件看能不能快速返回或者是快速进入常规逻辑。
2. 进入常规逻辑，调用_dispatch_group_wait_slow 进行等待。阻塞住线程。

os_atomic_rmw_loop2o宏展开：

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
            } 
            if (unlikely(timeout == 0)) {
                ({
                    __c11_atomic_thread_fence(memory_order_relaxed);
                    return _DSEMA4_TIMEOUT();
                    __builtin_unreachable();
                });
            }
            new_state = old_state | DISPATCH_GROUP_HAS_WAITERS标志位; //后续os_atomic_cmpxchgvw更新HAS_WAITERS标志位
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

os_atomic_rmw_loop2o宏是一个大都数情况下只循环一次的循环。

这段逻辑会更新dg_state的HAS_WAITERS标志位，表示有等待的线程。

函数:_dispatch_group_wait_slow

```c
DISPATCH_NOINLINE
static long
_dispatch_group_wait_slow(dispatch_group_t dg, uint32_t gen,
		dispatch_time_t timeout)
{
	for (;;) {
		int rc = _dispatch_wait_on_address(&dg->dg_gen, gen, timeout, 0);
		if (likely(gen != os_atomic_load2o(dg, dg_gen, acquire))) {
			return 0;
		}
		if (rc == ETIMEDOUT) {
			return _DSEMA4_TIMEOUT();
		}
	}
}
```

_dispatch_group_wait_slow(dg, _dg_state_gen(new_state), timeout);

其中的一个参数是`_dg_state_gen(new_state)`，表明线程在wait之前会先对当前dg_state的gen保存一份快照作为参数传入`_dispatch_group_wait_slow`，当`_dispatch_wait_on_address`函数返回时会再判断一下gen是否和刚才保存的gen相等，不相等则说明产生了一次generation。return 0，等待结束。时间到了也退出等待。其他情况则继续等待。

函数：_dispatch_wait_on_address

```c
int
_dispatch_wait_on_address(uint32_t volatile *_address, uint32_t value,
		dispatch_time_t timeout, dispatch_lock_options_t flags)
{
	uint32_t *address = (uint32_t *)_address;
	uint64_t nsecs = _dispatch_timeout(timeout);
	if (nsecs == 0) {
		return ETIMEDOUT;
	}
#if HAVE_UL_COMPARE_AND_WAIT
	uint64_t usecs = 0;
	int rc;
	if (nsecs == DISPATCH_TIME_FOREVER) {
		return _dispatch_ulock_wait(address, value, 0, flags);
	}
	do {
		usecs = howmany(nsecs, NSEC_PER_USEC);
		if (usecs > UINT32_MAX) usecs = UINT32_MAX;
		rc = _dispatch_ulock_wait(address, value, (uint32_t)usecs, flags);
	} while (usecs == UINT32_MAX && rc == ETIMEDOUT &&
			(nsecs = _dispatch_timeout(timeout)) != 0);
	return rc;
}
```

这里最终会调用到系统内部`__ulock_wait`函数进入休眠，这个函数在一些不当使用GCD API死锁崩溃的时候应该会经常看到。

这个函数领悟一下就可以了，作用应该就是等待_address地址的值和给定的value是否不一样（猜测）。

有wait就有wake

#### _dispatch_group_wake

有三个地方会调用到wake：dispatch_group_leave函数和dispatch_group_notify，dispatch_group_notify_f

实现：

```c
DISPATCH_NOINLINE
static void
_dispatch_group_wake(dispatch_group_t dg, uint64_t dg_state, bool needs_release)
{
	uint16_t refs = needs_release ? 1 : 0; // <rdar://problem/22318411>

	if (dg_state & DISPATCH_GROUP_HAS_NOTIFS) {
		dispatch_continuation_t dc, next_dc, tail;

		// Snapshot before anything is notified/woken <rdar://problem/8554546>
		dc = os_mpsc_capture_snapshot(os_mpsc(dg, dg_notify), &tail);
		do {
			dispatch_queue_t dsn_queue = (dispatch_queue_t)dc->dc_data;
			next_dc = os_mpsc_pop_snapshot_head(dc, tail, do_next);
			_dispatch_continuation_async(dsn_queue, dc,
					_dispatch_qos_from_pp(dc->dc_priority), dc->dc_flags);
			_dispatch_release(dsn_queue);
		} while ((dc = next_dc));

		refs++;
	}

	if (dg_state & DISPATCH_GROUP_HAS_WAITERS) {
		_dispatch_wake_by_address(&dg->dg_gen);
	}

	if (refs) _dispatch_release_n(dg, refs);
}
```

#### dispatch_group_notify

实现：

```c
void
dispatch_group_notify(dispatch_group_t dg, dispatch_queue_t dq,
		dispatch_block_t db)
{
	dispatch_continuation_t dsn = _dispatch_continuation_alloc();
	_dispatch_continuation_init(dsn, dq, db, 0, DC_FLAG_CONSUME);
	_dispatch_group_notify(dg, dq, dsn);
}

DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_group_notify(dispatch_group_t dg, dispatch_queue_t dq,
		dispatch_continuation_t dsn)
{
	uint64_t old_state, new_state;
	dispatch_continuation_t prev;

	dsn->dc_data = dq;
	_dispatch_retain(dq);

	prev = os_mpsc_push_update_tail(os_mpsc(dg, dg_notify), dsn, do_next);
	if (os_mpsc_push_was_empty(prev)) _dispatch_retain(dg);
	os_mpsc_push_update_prev(os_mpsc(dg, dg_notify), prev, dsn, do_next);
	if (os_mpsc_push_was_empty(prev)) {
		os_atomic_rmw_loop2o(dg, dg_state, old_state, new_state, release, {
			new_state = old_state | DISPATCH_GROUP_HAS_NOTIFS;
			if ((uint32_t)old_state == 0) {
				os_atomic_rmw_loop_give_up({
					return _dispatch_group_wake(dg, new_state, false);
				});
			}
		});
	}
}
```

os_atomic_rmw_loop2o宏逻辑里面会更新dg_state的HAS_NOTIFS标志位为DISPATCH_GROUP_HAS_NOTIFS。表示有通知者。