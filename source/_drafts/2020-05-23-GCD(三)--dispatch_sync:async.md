---
title: GCD(三)--dispatch_sync/async
tags:
  - 标签1
  - 标签2
  - 标签3
categories:
  - 分类1
  - 分类2
comments: true
date: 2020-05-23 15:23:33
---

GCD版本`libdispatch-1173.40.5`。

dispatch_sync/async问题：

1. dispatch_sync工作流程
5. dispatch_sync死锁检测是如何检测的，是否会有检测不到的情况？
6. dispatch_async工作流程
7. dispatch_async传入的任务是如何保存的，又是如何被执行的，派发到一个较高层级的queue时任务的执行过程又是怎样的
8. 底层的线程池是如何维护和管理的

dispatch_barrier_sync问题：(未完成)

1. dispatch_barrier_sync会等待追加到自定义并发队列的并行执行的处理全部结束之后，再将指定的处理追加到队列中的，那么它是如何监听到之前的任务都完成了

   猜测：因为dispatch_barrier_sync API会走尝试获取barrier逻辑，这种情况下应该会失败，从而线程进入等待状态，之前的任务完成后会唤醒该线程。但不清楚是怎么唤醒的，信号量？

2. 自己执行时又是怎么阻止其他任务执行的

3. 自己执行完后又是怎么恢复后面任务的执行的

dispatch_barrier_async问题：(未完成)

1. dispatch_barrier_async会等待追加到自定义并发队列的并行执行的处理全部结束之后，再将指定的处理追加到队列中的，那么它是如何监听到之前的任务都完成了

   可能跟out_with_no_width有关系，因为no_width它不会执行，因此它后面的也没法执行。

2. 自己执行时又是怎么阻止其他任务执行的

3. 自己执行完后又是怎么恢复后面任务的执行的

### dispatch_sync

一般来说同步任务是在当前线程中执行的（例外：丢到主队列的都是在主线程执行），同时它会阻塞当前线程直到任务执行完毕。

实现：

```c
DISPATCH_NOINLINE
void
dispatch_sync(dispatch_queue_t dq, dispatch_block_t work)
{
	uintptr_t dc_flags = DC_FLAG_BLOCK;
	if (unlikely(_dispatch_block_has_private_data(work))) {
		return _dispatch_sync_block_with_privdata(dq, work, dc_flags);
	}
	_dispatch_sync_f(dq, work, _dispatch_Block_invoke(work), dc_flags);
}
```

调用dispatch_sync函数时，dc_flags初始为 DC_FLAG_BLOCK。

主要调用了 `_dispatch_sync_f` --> `_dispatch_sync_f_inline`

#### _dispatch_sync_f

```c
DISPATCH_NOINLINE
static void
_dispatch_sync_f(dispatch_queue_t dq, void *ctxt, dispatch_function_t func,
		uintptr_t dc_flags)
{
	_dispatch_sync_f_inline(dq, ctxt, func, dc_flags);
}

DISPATCH_ALWAYS_INLINE
static inline void
  _dispatch_sync_f_inline(dispatch_queue_t dq, void *ctxt,
		dispatch_function_t func, uintptr_t dc_flags)
{
	if (likely(dq->dq_width == 1)) { //串行队列走下面_dispatch_barrier_sync_f逻辑
		return _dispatch_barrier_sync_f(dq, ctxt, func, dc_flags); //里面如果try_acquire_barrier失败也会进入_dispatch_sync_f_slow
	}

	if (unlikely(dx_metatype(dq) != _DISPATCH_LANE_TYPE)) {
		DISPATCH_CLIENT_CRASH(0, "Queue type doesn't support dispatch_sync");
	}

	dispatch_lane_t dl = upcast(dq)._dl;
	// Global concurrent queues and queues bound to non-dispatch threads
	// always fall into the slow case, see DISPATCH_ROOT_QUEUE_STATE_INIT_VALUE
	if (unlikely(!_dispatch_queue_try_reserve_sync_width(dl))) { //try_reserve_sync_width失败的队列走_dispatch_sync_f_slow逻辑。全局队列总是失败，自定义并发队列在一些情况下也会失败。
		return _dispatch_sync_f_slow(dl, ctxt, func, 0, dl, dc_flags);
	}

	if (unlikely(dq->do_targetq->do_targetq)) { //层级较深
		return _dispatch_sync_recurse(dl, ctxt, func, dc_flags);
	}
	_dispatch_introspection_sync_begin(dl);
	_dispatch_sync_invoke_and_complete(dl, ctxt, func DISPATCH_TRACE_ARG(
			_dispatch_trace_item_sync_push_pop(dq, ctxt, func, dc_flags))); //该函数内部主要是执行block
}

/* Used by _dispatch_sync on non-serial queues
 *
 * Initial state must be { sc:0, ib:0, pb:0, d:0 }
 * Final state: { w += 1 }
 */
DISPATCH_ALWAYS_INLINE DISPATCH_WARN_RESULT
static inline bool
_dispatch_queue_try_reserve_sync_width(dispatch_lane_t dq)
{
	uint64_t old_state, new_state;

	// <rdar://problem/24738102&24743140> reserving non barrier width
	// doesn't fail if only the ENQUEUED bit is set (unlike its barrier width
	// equivalent), so we have to check that this thread hasn't enqueued
	// anything ahead of this call or we can break ordering
	if (unlikely(dq->dq_items_tail)) { //如果队列不为空，则try失败
		return false;
	}

	return os_atomic_rmw_loop2o(dq, dq_state, old_state, new_state, relaxed, {
		if (unlikely(!_dq_state_is_sync_runnable(old_state)) ||
				_dq_state_is_dirty(old_state) ||
				_dq_state_has_pending_barrier(old_state)) {
			os_atomic_rmw_loop_give_up(return false); //当前dq的状态是不可同步执行状态或是dirty状态或是还有未开始的barrier任务，则返回false，其他就是返回true的情况。
		}
		new_state = old_state + DISPATCH_QUEUE_WIDTH_INTERVAL;
	}); //类似os_atomic_rmw_loop2o这样的宏，就是原子的比较交换功能，这里代表的就是原子的修改dq_state的值为新值new_state。
}
```

大致流程：

1. 判断队列的dq_width是否等于1，即是否是串行队列，如果是串行队列则走 `_dispatch_barrier_sync_f` 逻辑
2. 如果是全局并发队列或绑定在非派发线程的队列(?)或 `try_reserve_sync_width` try失败的自定义并发队列，走 `_dispatch_sync_f_slow` 逻辑
3.  `try_reserve_sync_width` try成功的自定义并发队列走正常的 `_dispatch_sync_invoke_and_complete` 逻辑，直接在当前线程执行block就完事了。

可以看到dispatch_sync并没有将block任务封装为一个dispatch_continuation_s对象添加到队列的任务链表上去，而是尽可能直接在当前线程执行的。

还可以看到`dispatch_sync` 的逻辑是根据派发队列类型分别处理的：

对于串行队列，由于串行队列的后一个任务必须等待前一个任务执行完后才能执行，为了保证这一特性，所以它会有一个 `dispatch_queue_try_acquire_barrier` “加锁”的操作，保证即使在多线程同时调用dispatch_sync时也不会出现同时执行两个任务的情况，因此走了`_dispatch_barrier_sync_f` 逻辑。

比如：

```objc
- (void)testSerialQueueFIFO {
    dispatch_queue_t serialQueue = dispatch_queue_create("mySerialQueue", DISPATCH_QUEUE_SERIAL);
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        sleep(1);
        NSLog(@"----");
        dispatch_sync(serialQueue, ^{
            NSLog(@"任务2开始");
            sleep(2);
            NSLog(@"任务2结束");
        });
    });
    dispatch_sync(serialQueue, ^{
        NSLog(@"任务1开始");
        sleep(3);
        NSLog(@"任务1结束");
    });
}
```

任务1开始后，任务2要等到任务1完成后才能开始。

而全局队列就是可以并发执行任务的并不需要等前一个任务是否完成，所以即使在多线程同时调用dispatch_sync时也没什么和它的定义并不冲突，所以可以不用 `dispatch_queue_try_acquire_barrier` “加锁”的操作而是直接进入 `_dispatch_sync_f_slow` 逻辑（该函数里有判断如果是全局队列则直接执行block）。

对于自定义的并发队列，也是直接在当前线程执行了。因此可以说dispatch_sync一个同步任务到并发队列，不管是全局并发队列还是自定义并发队列，怎么嵌套都不会导致死锁。

因此dispatch_sync逻辑主要看dq是串行队列时的 `_dispatch_barrier_sync_f` 逻辑，和 `dispatch_queue_try_reserve_sync_width` try失败时的 `_dispatch_sync_f_slow` 逻辑。

不过还是先来看`try_reserve_sync_width` try成功的自定义并发队列走的逻辑 `_dispatch_sync_invoke_and_complete` 。

##### _dispatch_sync_invoke_and_complete

该函数主要是执行block。

实现：

```c
DISPATCH_NOINLINE
static void
_dispatch_sync_invoke_and_complete(dispatch_lane_t dq, void *ctxt,
		dispatch_function_t func DISPATCH_TRACE_ARG(void *dc))
{
	_dispatch_sync_function_invoke_inline(dq, ctxt, func); //执行block
	_dispatch_trace_item_complete(dc);
	_dispatch_lane_non_barrier_complete(dq, 0); //里面有更新dq_state逻辑
}

DISPATCH_NOINLINE
static void
_dispatch_lane_non_barrier_complete(dispatch_lane_t dq,
		dispatch_wakeup_flags_t flags)
{
	uint64_t old_state, new_state, owner_self = _dispatch_lock_value_for_self();

	// see _dispatch_lane_resume()
	os_atomic_rmw_loop2o(dq, dq_state, old_state, new_state, relaxed, {
		new_state = old_state - DISPATCH_QUEUE_WIDTH_INTERVAL;
		if (unlikely(_dq_state_drain_locked(old_state))) {
			// make drain_try_unlock() fail and reconsider whether there's
			// enough width now for a new item
			new_state |= DISPATCH_QUEUE_DIRTY;
		} else if (likely(_dq_state_is_runnable(new_state))) {
			new_state = _dispatch_lane_non_barrier_complete_try_lock(dq,
					old_state, new_state, owner_self);
		}
	});

	_dispatch_lane_non_barrier_complete_finish(dq, flags, old_state, new_state);
}

DISPATCH_ALWAYS_INLINE
static void
_dispatch_lane_non_barrier_complete_finish(dispatch_lane_t dq,
		dispatch_wakeup_flags_t flags, uint64_t old_state, uint64_t new_state)
{
	if (_dq_state_received_override(old_state)) {
		// Ensure that the root queue sees that this thread was overridden.
		_dispatch_set_basepri_override_qos(_dq_state_max_qos(old_state));
	}

	if ((old_state ^ new_state) & DISPATCH_QUEUE_IN_BARRIER) {
		if (_dq_state_is_dirty(old_state)) {
			// <rdar://problem/14637483>
			// dependency ordering for dq state changes that were flushed
			// and not acted upon
			os_atomic_thread_fence(dependency);
			dq = os_atomic_force_dependency_on(dq, old_state);
		}
		return _dispatch_lane_barrier_complete(dq, 0, flags);
	}

	if ((old_state ^ new_state) & DISPATCH_QUEUE_ENQUEUED) {
		if (!(flags & DISPATCH_WAKEUP_CONSUME_2)) {
			_dispatch_retain_2(dq);
		}
		dispatch_assert(!_dq_state_is_base_wlh(new_state));
		_dispatch_trace_item_push(dq->do_targetq, dq);
		return dx_push(dq->do_targetq, dq, _dq_state_max_qos(new_state));
	}

	if (flags & DISPATCH_WAKEUP_CONSUME_2) {
		_dispatch_release_2_tailcall(dq);
	}
}
```

##### _dispatch_barrier_sync_f

直勾勾的调用了`_dispatch_barrier_sync_f_inline ` 。应该是所谓的接口隔离。

ps：`dispatch_barrier_sync` 内部也是调用 `_dispatch_barrier_sync_f` 完成工作的。分析完`_dispatch_barrier_sync_f_inline`，也就等于知道了`dispatch_barrier_sync` 的工作原理了。因此 `dispatch_sync` 派发到串行队列逻辑和 `dispatch_barrier_sync` 派发到串行队列逻辑是一样的。

```c
DISPATCH_NOINLINE
static void
_dispatch_barrier_sync_f(dispatch_queue_t dq, void *ctxt,
		dispatch_function_t func, uintptr_t dc_flags)
{
	_dispatch_barrier_sync_f_inline(dq, ctxt, func, dc_flags);
}

DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_barrier_sync_f_inline(dispatch_queue_t dq, void *ctxt,
		dispatch_function_t func, uintptr_t dc_flags)
{
	dispatch_tid tid = _dispatch_tid_self(); //获取当前线程id--tid标识

	if (unlikely(dx_metatype(dq) != _DISPATCH_LANE_TYPE)) {
		DISPATCH_CLIENT_CRASH(0, "Queue type doesn't support dispatch_sync");
	}

	dispatch_lane_t dl = upcast(dq)._dl;
	// The more correct thing to do would be to merge the qos of the thread
	// that just acquired the barrier lock into the queue state.
	//
	// However this is too expensive for the fast path, so skip doing it.
	// The chosen tradeoff is that if an enqueue on a lower priority thread
	// contends with this fast path, this thread may receive a useless override.
	//
	// Global concurrent queues and queues bound to non-dispatch threads
	// always fall into the slow case, see DISPATCH_ROOT_QUEUE_STATE_INIT_VALUE
	if (unlikely(!_dispatch_queue_try_acquire_barrier_sync(dl, tid))) { //尝试获得栅栏
		return _dispatch_sync_f_slow(dl, ctxt, func, DC_FLAG_BARRIER, dl,
				DC_FLAG_BARRIER | dc_flags); //dispatch_queue_try_acquire_barrier失败，线程进入_dispatch_sync_f_slow逻辑，此时dc_flag加入了DC_FLAG_BARRIER，slow表明不是那么快的执行block任务，而是可能会等待一会才会执行。
	}

	if (unlikely(dl->do_targetq->do_targetq)) {
		return _dispatch_sync_recurse(dl, ctxt, func,
				DC_FLAG_BARRIER | dc_flags);
	}
	_dispatch_introspection_sync_begin(dl);
	_dispatch_lane_barrier_sync_invoke_and_complete(dl, ctxt, func
			DISPATCH_TRACE_ARG(_dispatch_trace_item_sync_push_pop(
					dq, ctxt, func, dc_flags | DC_FLAG_BARRIER)));
}

void
dispatch_barrier_sync(dispatch_queue_t dq, dispatch_block_t work)
{
	uintptr_t dc_flags = DC_FLAG_BARRIER | DC_FLAG_BLOCK;
	if (unlikely(_dispatch_block_has_private_data(work))) {
		return _dispatch_sync_block_with_privdata(dq, work, dc_flags);
	}
	_dispatch_barrier_sync_f(dq, work, _dispatch_Block_invoke(work), dc_flags);
}
```

大致流程：

1. 获取当前线程id--tid标识。dispatch_queue_try_acquire_barrier会用到。另外队列死锁检测也会用到tid。
2. 调用 `_dispatch_queue_try_acquire_barrier_sync` 尝试获得屏障，该函数会更新队列的dq_state的值为new_state（与tid相关）。如果获取barrier失败则进入 `_dispatch_sync_f_slow`，线程进入等待状态直到可以执行时才执行。对于全局并发队列try_acquire_barrier一般都会失败。
3. `_dispatch_queue_try_acquire_barrier_sync` 获取barrier成功的逻辑
4. 如果dl->do_targetq->do_targetq有值则说明是比较高层次的队列，则走`_dispatch_sync_recurse`逻辑
5. 最后走常规逻辑`_dispatch_lane_barrier_sync_invoke_and_complete` 执行block。

这里先来看一下 `_dispatch_lane_barrier_sync_invoke_and_complete` 函数，后面再着重看一下 `_dispatch_queue_try_acquire_barrier_sync` 主要是了解一下dq_state的新值new_state是怎么计算得来的。

###### _dispatch_queue_try_acquire_barrier_sync

```c
DISPATCH_ALWAYS_INLINE DISPATCH_WARN_RESULT
static inline bool
_dispatch_queue_try_acquire_barrier_sync(dispatch_queue_class_t dq, uint32_t tid)
{
	return _dispatch_queue_try_acquire_barrier_sync_and_suspend(dq._dl, tid, 0);
}

/* Used by _dispatch_barrier_{try,}sync
 *
 * Note, this fails if any of e:1 or dl!=0, but that allows this code to be a
 * simple cmpxchg which is significantly faster on Intel, and makes a
 * significant difference on the uncontended codepath.
 *
 * See discussion for DISPATCH_QUEUE_DIRTY in queue_internal.h
 *
 * Initial state must be `completely idle`
 * Final state forces { ib:1, qf:1, w:0 }
 */
DISPATCH_ALWAYS_INLINE DISPATCH_WARN_RESULT
static inline bool
_dispatch_queue_try_acquire_barrier_sync_and_suspend(dispatch_lane_t dq,
		uint32_t tid, uint64_t suspend_count)
{
	uint64_t init  = DISPATCH_QUEUE_STATE_INIT_VALUE(dq->dq_width);
	uint64_t value = DISPATCH_QUEUE_WIDTH_FULL_BIT | DISPATCH_QUEUE_IN_BARRIER |
			_dispatch_lock_value_from_tid(tid) |
			(suspend_count * DISPATCH_QUEUE_SUSPEND_INTERVAL); //value值的计算，混合了WIDTH_FULL_BIT、IN_BARRIER、tid。
	uint64_t old_state, new_state;

	return os_atomic_rmw_loop2o(dq, dq_state, old_state, new_state, acquire, {
		uint64_t role = old_state & DISPATCH_QUEUE_ROLE_MASK;
		if (old_state != (init | role)) { 
			os_atomic_rmw_loop_give_up(break); 
		}
		new_state = value | role;  //更新队列的dq_state的值为new_state，这个值跟tid是有关系的。后面会根据该值与tid进行比较，用于判断是否是同一个线程重复获取栅栏
	});
}

DISPATCH_ALWAYS_INLINE
static inline dispatch_lock
_dispatch_lock_value_from_tid(dispatch_tid tid)
{
	return tid & DLOCK_OWNER_MASK;
}
```

展开下宏：

```c
return ({
    _Bool _result = 0;
    __typeof__(&(dq)->dq_state) _p = (&(dq)->dq_state);
    old_state = __c11_atomic_load(((__typeof__(*(_p)) _Atomic *)(_p)), memory_order_relaxed);
    do {
        {
            uint64_t role = old_state & DISPATCH_QUEUE_ROLE_MASK;
            if (old_state != (init | role)) {
                ({
                    __c11_atomic_thread_fence(memory_order_relaxed);
                    break; __builtin_unreachable(); 
                });
            }
            new_state = value | role;
        };
        _result = ({
            __typeof__(__c11_atomic_load(((__typeof__(*(_p)) _Atomic *)(_p)), memory_order_relaxed)) _r = (old_state);
            _Bool _b = __c11_atomic_compare_exchange_weak(((__typeof__(*(_p)) _Atomic *)(_p)), &_r, new_state, memory_order_acquire, memory_order_relaxed);
            *(&old_state) = _r;
            _b;
        });
    } while (unlikely(!_result));
    _result;
});
```

根据官方的注释：

```
/* Magic dq_state values for global queues: they have QUEUE_FULL and IN_BARRIER
 * set to force the slow path in dispatch_barrier_sync() and dispatch_sync()
 */
#define DISPATCH_ROOT_QUEUE_STATE_INIT_VALUE \
		(DISPATCH_QUEUE_WIDTH_FULL_BIT | DISPATCH_QUEUE_IN_BARRIER)
```

对于`if (old_state != (init | role))`条件判断：

如果是全局并发队列，dq_state的值默认是`DISPATCH_ROOT_QUEUE_STATE_INIT_VALUE`，可以用这些值去运行下就会发现这里会满足条件马上break，返回false，dq_state的值也不会发生改变。

如果是串行队列和自定义并发队列则会更新dq_state的值为new_state，new_state这个值跟tid是有关系的。后面`__DISPATCH_WAIT_FOR_QUEUE__` 函数会根据该值与tid进行比较，用于判断当前线程是否已经获得栅栏进行死锁检测。所以使用dispatch_barrier_sync派发到自定义并发队列如果嵌套调用的话是会死锁崩溃的，而派发到全局队列则不会。比如：

```c
- (void)test_dispatch_barrier_sync_custom_concurrent_deadlock {
    dispatch_queue_t queue = dispatch_queue_create("com.xq.queue", DISPATCH_QUEUE_CONCURRENT); //自定义的并发队列会死锁，并崩溃
//    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0); //全局队列不会死锁，顺序打印，但与barrier API本身的含义相违背了。
    dispatch_barrier_sync(queue, ^{ //不管是dispatch_barrier_async/sync
        NSLog(@"1current thread = %@", [NSThread currentThread]);
        dispatch_barrier_sync(queue, ^{
            NSLog(@"2current thread = %@", [NSThread currentThread]);
        });
        NSLog(@"---111---");
    });
    NSLog(@"---222---");
}
```

所以在使用dispatch_barrier API时传入的队列最好是自己定义的队列，不要使用全局队列，否则没有barrier的效果。

barrier技术有点类似给队列加锁，执行任务的线程给队列上锁后，执行该任务后面的任务的线程try_acquire_barrier_sync就会失败，从而进入等待状态，等到队列解锁后，等待的线程被唤醒并继续执行block任务。同样的，同一线程重复给队列加锁将导致死锁。

##### _dispatch_sync_f_slow

有如下两种情况会走slow逻辑：

1. 派发一个同步任务到全局并发队列，最开始的逻辑。

2. dispatch_queue_try_acquire_barrier失败的队列（串行或并发），也会进入_dispatch_sync_f_slow逻辑。

slow表明不是那么快的执行block任务，而是可能会等待一会才会执行。

```c
DISPATCH_NOINLINE
static void
_dispatch_sync_f_slow(dispatch_queue_class_t top_dqu, void *ctxt,
		dispatch_function_t func, uintptr_t top_dc_flags,
		dispatch_queue_class_t dqu, uintptr_t dc_flags)
{
	dispatch_queue_t top_dq = top_dqu._dq;
	dispatch_queue_t dq = dqu._dq;
	if (unlikely(!dq->do_targetq)) { //tq为null说明是全局队列，因此全局队列直接执行block，不会有死锁问题
		return _dispatch_sync_function_invoke(dq, ctxt, func);
	}

	pthread_priority_t pp = _dispatch_get_priority();
	struct dispatch_sync_context_s dsc = {
		.dc_flags    = DC_FLAG_SYNC_WAITER | dc_flags,
		.dc_func     = _dispatch_async_and_wait_invoke,
		.dc_ctxt     = &dsc,
		.dc_other    = top_dq,
		.dc_priority = pp | _PTHREAD_PRIORITY_ENFORCE_FLAG,
		.dc_voucher  = _voucher_get(),
		.dsc_func    = func,
		.dsc_ctxt    = ctxt,
		.dsc_waiter  = _dispatch_tid_self(), //保存将要等待的线程的ID
	};

	_dispatch_trace_item_push(top_dq, &dsc);
	__DISPATCH_WAIT_FOR_QUEUE__(&dsc, dq); //进入等待，设置队列的dq_state的SYNC_WAIT位为1，并且还会进行死锁检测

	if (dsc.dsc_func == NULL) {
		// dsc_func being cleared means that the block ran on another thread ie.
		// case (2) as listed in _dispatch_async_and_wait_f_slow.
		dispatch_queue_t stop_dq = dsc.dc_other;
		return _dispatch_sync_complete_recurse(top_dq, stop_dq, top_dc_flags);
	}

	_dispatch_introspection_sync_begin(top_dq);
	_dispatch_trace_item_pop(top_dq, &dsc);
	_dispatch_sync_invoke_and_complete_recurse(top_dq, ctxt, func,top_dc_flags
			DISPATCH_TRACE_ARG(&dsc));
}
```

该函数主要过程：

1. 如果队列是全局队列则直接执行block，因此派发一个同步任务到全局队列怎么嵌套都不会有死锁问题。
2. 调用 `__DISPATCH_WAIT_FOR_QUEUE__` 进行死锁检测，并进入等待。
3. 等待结束后，调用 `_dispatch_sync_invoke_and_complete_recurse` 执行block任务。

这里着重看`__DISPATCH_WAIT_FOR_QUEUE__`函数的实现。

###### \_\_DISPATCH_WAIT_FOR_QUEUE\_\_

```c
DISPATCH_NOINLINE
static void
__DISPATCH_WAIT_FOR_QUEUE__(dispatch_sync_context_t dsc, dispatch_queue_t dq)
{
	uint64_t dq_state = _dispatch_wait_prepare(dq);
	if (unlikely(_dq_state_drain_locked_by(dq_state, dsc->dsc_waiter))) { //死锁检测，条件为真时即发生死锁，将崩溃。
		DISPATCH_CLIENT_CRASH((uintptr_t)dq_state,
				"dispatch_sync called on queue "
				"already owned by current thread");
	}
	// Blocks submitted to the main thread MUST run on the main thread, and
	// dispatch_async_and_wait also executes on the remote context rather than
	// the current thread.
	//
	// For both these cases we need to save the frame linkage for the sake of
	// _dispatch_async_and_wait_invoke
	_dispatch_thread_frame_save_state(&dsc->dsc_dtf);
	
	...

}

DISPATCH_ALWAYS_INLINE
static inline uint64_t
_dispatch_wait_prepare(dispatch_queue_t dq)
{
	uint64_t old_state, new_state;

	os_atomic_rmw_loop2o(dq, dq_state, old_state, new_state, relaxed, {
		if (_dq_state_is_suspended(old_state) ||
				!_dq_state_is_base_wlh(old_state)) {
			os_atomic_rmw_loop_give_up(return old_state);
		}
		if (!_dq_state_drain_locked(old_state) ||
				_dq_state_in_sync_transfer(old_state)) {
			os_atomic_rmw_loop_give_up(return old_state);
		}
		new_state = old_state | DISPATCH_QUEUE_RECEIVED_SYNC_WAIT; //设置队列的dq_state的SYNC_WAIT位为1
	});
	return new_state;
}

DISPATCH_ALWAYS_INLINE
static inline bool
_dq_state_drain_locked_by(uint64_t dq_state, dispatch_tid tid)
{
	return _dispatch_lock_is_locked_by((dispatch_lock)dq_state, tid);
}

DISPATCH_ALWAYS_INLINE
static inline bool
_dispatch_lock_is_locked_by(dispatch_lock lock_value, dispatch_tid tid)
{
	// equivalent to _dispatch_lock_owner(lock_value) == tid
	return ((lock_value ^ tid) & DLOCK_OWNER_MASK) == 0;
}

#if TARGET_OS_MAC

typedef mach_port_t dispatch_tid;
typedef uint32_t dispatch_lock;

#define DLOCK_OWNER_MASK			((dispatch_lock)0xfffffffc)
#define DLOCK_WAITERS_BIT			((dispatch_lock)0x00000001)
#define DLOCK_FAILED_TRYLOCK_BIT	((dispatch_lock)0x00000002)

#define DLOCK_OWNER_NULL			((dispatch_tid)MACH_PORT_NULL)
#define _dispatch_tid_self()		((dispatch_tid)_dispatch_thread_port())

DISPATCH_ALWAYS_INLINE
static inline dispatch_tid
_dispatch_lock_owner(dispatch_lock lock_value)
{
	if (lock_value & DLOCK_OWNER_MASK) {
		return lock_value | DLOCK_WAITERS_BIT | DLOCK_FAILED_TRYLOCK_BIT;
	}
	return DLOCK_OWNER_NULL;
}
```

该函数的作用：

1. 设置队列的dq_state的SYNC_WAIT位为1。
2. 进行死锁检测。如果dq_state中有保存获得barrier成功的线程id，当该线程再次进入时必然和dq_state中保存的tid相等，于是判断为死锁，崩溃。
3. 进入等待

崩溃的提示：

![](https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/DDBEB112-9737-463C-B420-654ADFE083BC.png)

死锁的产生必须满足四个必要条件：

- 资源的互斥访问
- 请求与保持
- 不可剥夺
- 循环等待

libdispatch中创建并分配线程来执行任务块的过程中，线程/队列对资源的操作满足上述前三个条件，那么如果再满足第四个条件，必然会发生死锁。因此在GCD中满足下面两个条件即会形成循环等待的情形：

- 队列正在执行任务Task 1（无论是sync还是async方式提交的）
- Task 1未执行完成，又向队列中同步提交Task 2（**sync**方式提交）

在以前的版本中GCD是没有死锁检测的，当发生死锁时不会有任何反应（当然如果发生在主线程中那么会卡死界面，如果是在子线程中那就神不知鬼不觉了），后来苹果工程师可能觉得这样不好于是加入了死锁检测的功能，如果发生死锁则主动抛出异常。

源码中死锁检测的实现并不复杂：

如果队列对象中的dq_state中有保存获得barrier成功的线程id（一般"dispatch_sync+串行队列"或"dispatch_barrier_sync+串行队列/自定义并发队列"会有更新dq_state的值），当该线程再次进入时必然和dq_state中保存的tid相等，于是判断为死锁，崩溃。

但这种检测办法并不能检测出所有死锁的情况，它只能检测出同一线程--同一队列的情况，其他情况就检测不到了，比如下面的代码：会形成一个循环等待但并不是同一线程--同一队列的情况，所以检测不出来，也就不会崩溃。

```objc
- (void)test_dispatch_sync_serial_deadlock_but_not_checked {
    //这种死锁情况检测不到。
    dispatch_queue_t sQ1 = dispatch_queue_create("xxx", DISPATCH_QUEUE_SERIAL);
    dispatch_async(sQ1, ^{
        NSLog(@"Enter");
        dispatch_sync(dispatch_get_main_queue(), ^{ 
            NSLog(@"---");
            dispatch_sync(sQ1, ^{ 
                NSArray *a = [NSArray new];
                NSLog(@"Enter again %@", a);
            });
        });
        NSLog(@"Done");
    });
}
```

###### _dispatch_sync_invoke_and_complete_recurse

终于到了 `_dispatch_sync_f_slow` 最后调用的一个函数了，该函数就是执行block，以及让队列恢复正常执行任务

```c
DISPATCH_NOINLINE
static void
_dispatch_sync_invoke_and_complete_recurse(dispatch_queue_class_t dq,
		void *ctxt, dispatch_function_t func, uintptr_t dc_flags
		DISPATCH_TRACE_ARG(void *dc))
{
	_dispatch_sync_function_invoke_inline(dq, ctxt, func);
	_dispatch_trace_item_complete(dc);
	_dispatch_sync_complete_recurse(dq._dq, NULL, dc_flags);
}

DISPATCH_NOINLINE
static void
_dispatch_sync_complete_recurse(dispatch_queue_t dq, dispatch_queue_t stop_dq,
		uintptr_t dc_flags)
{
	bool barrier = (dc_flags & DC_FLAG_BARRIER);
	do {
		if (dq == stop_dq) return;
		if (barrier) {
			dx_wakeup(dq, 0, DISPATCH_WAKEUP_BARRIER_COMPLETE);
		} else {
			_dispatch_lane_non_barrier_complete(upcast(dq)._dl, 0);
		}
		dq = dq->do_targetq;
		barrier = (dq->dq_width == 1);
	} while (unlikely(dq->do_targetq));
}
```

#### _dispatch_lane_barrier_sync_invoke_and_complete

终于来到 `_dispatch_barrier_sync_f_inline` 函数最后调用的一个函数。

实现：

```c
/*
 * For queues we can cheat and inline the unlock code, which is invalid
 * for objects with a more complex state machine (sources or mach channels)
 */
DISPATCH_NOINLINE
static void
_dispatch_lane_barrier_sync_invoke_and_complete(dispatch_lane_t dq,
		void *ctxt, dispatch_function_t func DISPATCH_TRACE_ARG(void *dc))
{
	_dispatch_sync_function_invoke_inline(dq, ctxt, func); //执行block
	_dispatch_trace_item_complete(dc);
	if (unlikely(dq->dq_items_tail || dq->dq_width > 1)) { //dq_items_tail有值或队列是并发队列
		return _dispatch_lane_barrier_complete(dq, 0, 0);
	}

	// Presence of any of these bits requires more work that only
	// _dispatch_*_barrier_complete() handles properly
	//
	// Note: testing for RECEIVED_OVERRIDE or RECEIVED_SYNC_WAIT without
	// checking the role is sloppy, but is a super fast check, and neither of
	// these bits should be set if the lock was never contended/discovered.
	const uint64_t fail_unlock_mask = DISPATCH_QUEUE_SUSPEND_BITS_MASK |
			DISPATCH_QUEUE_ENQUEUED | DISPATCH_QUEUE_DIRTY |
			DISPATCH_QUEUE_RECEIVED_OVERRIDE | DISPATCH_QUEUE_SYNC_TRANSFER |
			DISPATCH_QUEUE_RECEIVED_SYNC_WAIT;
	uint64_t old_state, new_state;

	// similar to _dispatch_queue_drain_try_unlock
	os_atomic_rmw_loop2o(dq, dq_state, old_state, new_state, release, {
		new_state  = old_state - DISPATCH_QUEUE_SERIAL_DRAIN_OWNED;
		new_state &= ~DISPATCH_QUEUE_DRAIN_UNLOCK_MASK;
		new_state &= ~DISPATCH_QUEUE_MAX_QOS_MASK;
		if (unlikely(old_state & fail_unlock_mask)) {
			os_atomic_rmw_loop_give_up({
				return _dispatch_lane_barrier_complete(dq, 0, 0);
			});
		}
	});
	if (_dq_state_is_base_wlh(old_state)) {
		_dispatch_event_loop_assert_not_owned((dispatch_wlh_t)dq);
	}
}
```

大致流程：

1. 执行block
2. 如果dq_items_tail有值或队列是并发队列则调用 `_dispatch_lane_barrier_complete` 后返回。函数 `_dispatch_lane_barrier_complete` 里其实做了很多事，比如让队列继续正常drain倾倒。
3. 更新dq_state。因为前面dispatch_queue_try_acquire_barrier里有设置dq_state，所以这里应该是复位。

##### _dispatch_lane_barrier_complete

```c
DISPATCH_NOINLINE
static void
_dispatch_lane_barrier_complete(dispatch_lane_class_t dqu, dispatch_qos_t qos,
		dispatch_wakeup_flags_t flags)
{
	dispatch_queue_wakeup_target_t target = DISPATCH_QUEUE_WAKEUP_NONE;
	dispatch_lane_t dq = dqu._dl;

	if (dq->dq_items_tail && !DISPATCH_QUEUE_IS_SUSPENDED(dq)) { //如果任务列表还有任务，并且dq没有被暂停
		struct dispatch_object_s *dc = _dispatch_queue_get_head(dq); //取出任务列表的头任务
		if (likely(dq->dq_width == 1 || _dispatch_object_is_barrier(dc))) {
			if (_dispatch_object_is_waiter(dc)) {
				return _dispatch_lane_drain_barrier_waiter(dq, dc, flags, 0);
			}
		} else if (dq->dq_width > 1 && !_dispatch_object_is_barrier(dc)) {
			return _dispatch_lane_drain_non_barriers(dq, dc, flags); //并发队列恢复为正常操作。
		}

		if (!(flags & DISPATCH_WAKEUP_CONSUME_2)) {
			_dispatch_retain_2(dq);
			flags |= DISPATCH_WAKEUP_CONSUME_2;
		}
		target = DISPATCH_QUEUE_WAKEUP_TARGET;
	}

	uint64_t owned = DISPATCH_QUEUE_IN_BARRIER +
			dq->dq_width * DISPATCH_QUEUE_WIDTH_INTERVAL;
	return _dispatch_lane_class_barrier_complete(dq, qos, flags, target, owned);
}
```

### dispatch_async

往一个派发队列上提交一个**异步执行**的Block，**不阻塞当前线程** ，可能会开新线程。

该函数的实现特别复杂，调用层级特别深，分支判断也特别多。

串行队列：

<img src="https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20200715145015.png" style="zoom:50%;" />

自定义并发队列：

<img src="https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20200715145244.png" style="zoom:50%;" />

全局队列：

<img src="https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20200715145405.png" style="zoom:50%;" />

实现：

```c
void
dispatch_async(dispatch_queue_t dq, dispatch_block_t work)
{
	dispatch_continuation_t dc = _dispatch_continuation_alloc();
	uintptr_t dc_flags = DC_FLAG_CONSUME; //
	dispatch_qos_t qos;

	qos = _dispatch_continuation_init(dc, dq, work, 0, dc_flags); //传入了dq,work,dc_flags
	_dispatch_continuation_async(dq, dc, qos, dc->dc_flags);
}
```

大致流程：

1. 创建一个dispatch_continuation_t dc
2. 设置dc_flags为DC_FLAG_CONSUME，dc_flags还有很多其他类型。后面执行时可能会用来区分一些东西。
3. 初始化创建的dc，并返回qos。上述过程其实就是将我们传入的block封装为一个内部使用的dispatch_continuation_t对象。
4. 调用_dispatch_continuation_async。

DC_FLAG定义：

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

另外如果调用的是dispatch_group_async则`uintptr_t dc_flags = DC_FLAG_CONSUME | DC_FLAG_GROUP_ASYNC;`

#### _dispatch_continuation_async

实现：

```c
DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_continuation_async(dispatch_queue_class_t dqu,
		dispatch_continuation_t dc, dispatch_qos_t qos, uintptr_t dc_flags)
{
#if DISPATCH_INTROSPECTION
	if (!(dc_flags & DC_FLAG_NO_INTROSPECTION)) {
		_dispatch_trace_item_push(dqu, dc);
	}
#else
	(void)dc_flags;
#endif
	return dx_push(dqu._dq, dc, qos);
}
```

主要逻辑就是调用了`dx_push(dqu._dq, dc, qos)`

#### dx_push(dqu._dq, dc, qos)定义

dx_push是一个宏，展开：

```c
(&(dqu._dq)->do_vtable->_os_obj_vtable)->dq_push(dqu._dq, dc, qos);
```

这里是一个多态调用，对象是第一个参数，不同子类对象调用同一方法会有不同的响应，我们可以看一下init.c中各子类的实现：

```c
DISPATCH_VTABLE_INSTANCE(queue,
	// This is the base class for queues, no objects of this type are made
	.do_type        = _DISPATCH_QUEUE_CLUSTER, //类簇，抽象类
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
	.do_invoke      = _dispatch_lane_invoke, //执行添加的block

	.dq_activate    = _dispatch_lane_activate, //激活队列相关
	.dq_wakeup      = _dispatch_lane_wakeup, //唤醒它的tq全局队列将自己添加到全局队列的链表上
	.dq_push        = _dispatch_lane_push, //将任务添加到队列的链表中
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
```

这里我们先看一个最简单的例子：

调用dispatch_async将任务派发到一个全局队列，这样队列的层级最少，后面的分析会更清晰一些。

```c
dispatch_queue_t glQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)
dispatch_async(glQueue, ^{
    NSLog(@"--1--");
});
```

此时队列的层级为：全

那么上面的dx_push就是_dispatch_root_queue_push的调用。

#### dx_push多态调用--全局队列的_dispatch_root_queue_push

```c
DISPATCH_NOINLINE
void
_dispatch_root_queue_push(dispatch_queue_global_t rq, dispatch_object_t dou,
		dispatch_qos_t qos)
{
#if DISPATCH_USE_KEVENT_WORKQUEUE
	dispatch_deferred_items_t ddi = _dispatch_deferred_items_get();
	if (unlikely(ddi && ddi->ddi_can_stash)) {
		dispatch_object_t old_dou = ddi->ddi_stashed_dou;
		dispatch_priority_t rq_overcommit;
		rq_overcommit = rq->dq_priority & DISPATCH_PRIORITY_FLAG_OVERCOMMIT;

		if (likely(!old_dou._do || rq_overcommit)) {
			dispatch_queue_global_t old_rq = ddi->ddi_stashed_rq;
			dispatch_qos_t old_qos = ddi->ddi_stashed_qos;
			ddi->ddi_stashed_rq = rq;
			ddi->ddi_stashed_dou = dou;
			ddi->ddi_stashed_qos = qos;
			_dispatch_debug("deferring item %p, rq %p, qos %d",
					dou._do, rq, qos);
			if (rq_overcommit) {
				ddi->ddi_can_stash = false;
			}
			if (likely(!old_dou._do)) {
				return;
			}
			// push the previously stashed item
			qos = old_qos;
			rq = old_rq;
			dou = old_dou;
		}
	}
#endif
#if HAVE_PTHREAD_WORKQUEUE_QOS
	if (_dispatch_root_queue_push_needs_override(rq, qos)) {
		return _dispatch_root_queue_push_override(rq, dou, qos);
	}
#else
	(void)qos;
#endif
	_dispatch_root_queue_push_inline(rq, dou, dou, 1);
}
```

直接调用了_dispatch_root_queue_push_inline函数，作用就是将dou添加到队列的链表上。

##### _dispatch_root_queue_push_inline

```c
DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_root_queue_push_inline(dispatch_queue_global_t dq,
		dispatch_object_t _head, dispatch_object_t _tail, int n) //参数为1
{
	struct dispatch_object_s *hd = _head._do, *tl = _tail._do;
	if (unlikely(os_mpsc_push_list(os_mpsc(dq, dq_items), hd, tl, do_next))) {
		return _dispatch_root_queue_poke(dq, n, 0);
	}
}
```

os_mpsc_push_list(os_mpsc(dq, dq_items), hd, tl, do_next)

展开：

```c
({
    __typeof__(__c11_atomic_load(((__typeof__(*(&(dq)->dq_items_head)) _Atomic *)(&(dq)->dq_items_head)), memory_order_relaxed)) _token;
    _token = ({
        __typeof__(__c11_atomic_load(((__typeof__(*(&(dq)->dq_items_head)) _Atomic *)(&(dq)->dq_items_head)), memory_order_relaxed)) _tl = (tl);
        __c11_atomic_store(((__typeof__(*(&(_tl)->do_next)) _Atomic *)(&(_tl)->do_next)), (((void*)0)), memory_order_relaxed);
      //原子操作，使用_tl的值替换dq_items_tail的值，并返回dq_items_tail之前的值，因为dq_items_tail始终指向单链表的末尾元素
        __c11_atomic_exchange(((__typeof__(*(&(dq)->dq_items_tail)) _Atomic *)(&(dq)->dq_items_tail)), _tl, memory_order_release);
    }); //更新dq_items_tail的值为tl，并返回dq_items_tail的旧值
    ({
        __typeof__(__c11_atomic_load(((__typeof__(*(&(dq)->dq_items_head)) _Atomic *)(&(dq)->dq_items_head)), memory_order_relaxed)) _prev = (_token);
        if (likely(_prev)) {
            (void)__c11_atomic_store(((__typeof__(*(&(_prev)->do_next)) _Atomic *)(&(_prev)->do_next)), ((hd)), memory_order_relaxed);
        } else {
            (void)__c11_atomic_store(((__typeof__(*(&(dq)->dq_items_head)) _Atomic *)(&(dq)->dq_items_head)), (hd), memory_order_relaxed);
        }
    }); //将item添加到链表末尾
    ((_token) == ((void*)0)); //链表是否为空
})
```

这里的操作就是在链表尾部添加元素的操作，并返回之前链表是否是空。

大致流程：

1. 将任务添加到全局队列的dq_items链表尾部。队列上的链表保存的既可以是dq（串行或并发）也可以是dc。我们的例子这里自然就是dc。
2. 如果全局队列的链表之前是空的则执行_dispatch_root_queue_poke，准备执行任务了。如果不是空的那么就等着被pop执行就可以了。

##### _dispatch_root_queue_poke

实现：

`_dispatch_root_queue_poke` --> `_dispatch_root_queue_poke_slow`：

```c
DISPATCH_NOINLINE
void
_dispatch_root_queue_poke(dispatch_queue_global_t dq, int n, int floor) //串行 1，0
{
	if (!_dispatch_queue_class_probe(dq)) { //dq为空说明链表上的任务都派发完了并且也没有新添加的任务，因此return
		return;
	}
#if !DISPATCH_USE_INTERNAL_WORKQUEUE
#if DISPATCH_USE_PTHREAD_POOL
	if (likely(dx_type(dq) == DISPATCH_QUEUE_GLOBAL_ROOT_TYPE))
#endif
	{
		if (unlikely(!os_atomic_cmpxchg2o(dq, dgq_pending, 0, n, relaxed))) {
			_dispatch_root_queue_debug("worker thread request still pending "
					"for global queue: %p", dq);
			return;
		}
	}
#endif // !DISPATCH_USE_INTERNAL_WORKQUEUE
	return _dispatch_root_queue_poke_slow(dq, n, floor);
}

DISPATCH_NOINLINE
static void
_dispatch_root_queue_poke_slow(dispatch_queue_global_t dq, int n, int floor)
{
	int remaining = n; //remaining表示希望线程池里有这么多个可用线程。如果不够就只能创建了。
	int r = ENOSYS;

	_dispatch_root_queues_init();
	_dispatch_debug_root_queue(dq, __func__);
	_dispatch_trace_runtime_event(worker_request, dq, (uint64_t)n);

#if !DISPATCH_USE_INTERNAL_WORKQUEUE
#if DISPATCH_USE_PTHREAD_ROOT_QUEUES
	if (dx_type(dq) == DISPATCH_QUEUE_GLOBAL_ROOT_TYPE)
#endif
	{ //实际会执行这里的，因为gcd又优化了一下，没有使用内部的workqueue了，而是通过PTHREAD_ROOT_QUEUES来管理了。不过这里不打算分析DISPATCH_USE_PTHREAD_ROOT_QUEUES这一分支。因为_pthread_workqueue_addthreads是pthread库里的且最终调用了系统内部函数__workq_kernreturn，这就没得分析了。所以这里分析DISPATCH_USE_PTHREAD_POOL分支的。
		_dispatch_root_queue_debug("requesting new worker thread for global "
				"queue: %p", dq);
		r = _pthread_workqueue_addthreads(remaining,
				_dispatch_priority_to_pp_prefer_fallback(dq->dq_priority));
		(void)dispatch_assume_zero(r);
		return;
	}
#endif // !DISPATCH_USE_INTERNAL_WORKQUEUE
#if DISPATCH_USE_PTHREAD_POOL
	dispatch_pthread_root_queue_context_t pqc = dq->do_ctxt;
	if (likely(pqc->dpq_thread_mediator.do_vtable)) {
		while (dispatch_semaphore_signal(&pqc->dpq_thread_mediator)) { //先发送信号看看能不能唤醒一个等待的线程，信号量默认是0。如果有唤醒的就用唤醒的线程执行了
			_dispatch_root_queue_debug("signaled sleeping worker for "
					"global queue: %p", dq);
			if (!--remaining) { //有唤醒线程而且刚好够就不用创建了，直接在唤醒的线程上执行就可以了。所以这里直接返回
				return;
			}
		}
	}

	bool overcommit = dq->dq_priority & DISPATCH_PRIORITY_FLAG_OVERCOMMIT;
	if (overcommit) {
		os_atomic_add2o(dq, dgq_pending, remaining, relaxed);
	} else {
		if (!os_atomic_cmpxchg2o(dq, dgq_pending, 0, remaining, relaxed)) {
			_dispatch_root_queue_debug("worker thread request still pending for "
					"global queue: %p", dq);
			return;
		}
	}

	int can_request, t_count;
	// seq_cst with atomic store to tail <rdar://problem/16932833>
	t_count = os_atomic_load2o(dq, dgq_thread_pool_size, ordered);
	do {
		can_request = t_count < floor ? 0 : t_count - floor; //可用的线程数
		if (remaining > can_request) {
			_dispatch_root_queue_debug("pthread pool reducing request from %d to %d",
					remaining, can_request);
			os_atomic_sub2o(dq, dgq_pending, remaining - can_request, relaxed);
			remaining = can_request;
		}
		if (remaining == 0) { //线程池中没有可用的线程了，直接返回。
			_dispatch_root_queue_debug("pthread pool is full for root queue: "
					"%p", dq);
			return;
		}
	} while (!os_atomic_cmpxchgvw2o(dq, dgq_thread_pool_size, t_count,
			t_count - remaining, &t_count, acquire)); //检查线程池大小还有没有剩余

#if !defined(_WIN32)
	pthread_attr_t *attr = &pqc->dpq_thread_attr;
	pthread_t tid, *pthr = &tid;
#if DISPATCH_USE_MGR_THREAD && DISPATCH_USE_PTHREAD_ROOT_QUEUES
	if (unlikely(dq == &_dispatch_mgr_root_queue)) {
		pthr = _dispatch_mgr_root_queue_init();
	}
#endif
  //这里是重点，创建一条新的线程并执行_dispatch_worker_thread函数
	do {
		_dispatch_retain(dq); // released in _dispatch_worker_thread
		while ((r = pthread_create(pthr, attr, _dispatch_worker_thread, dq))) { //pthread_create第四个参数是第三个参数（一个函数）的参数，这里传入的是dq。
			if (r != EAGAIN) {
				(void)dispatch_assume_zero(r);
			}
			_dispatch_temporary_resource_shortage();
		}
	} while (--remaining);
#else // defined(_WIN32)
//..._WIN32的逻辑，先删除
#endif // defined(_WIN32)
#else
	(void)floor;
#endif // DISPATCH_USE_PTHREAD_POOL
}
```

该函数里会复用/创建线程执行_dispatch_worker_thread函数

##### _dispatch_worker_thread

```c
// 6618342 Contact the team that owns the Instrument DTrace probe before
//         renaming this symbol
static void *
_dispatch_worker_thread(void *context)
{
	dispatch_queue_global_t dq = context;
	dispatch_pthread_root_queue_context_t pqc = dq->do_ctxt;

	int pending = os_atomic_dec2o(dq, dgq_pending, relaxed);
	if (unlikely(pending < 0)) {
		DISPATCH_INTERNAL_CRASH(pending, "Pending thread request underflow");
	}

	if (pqc->dpq_observer_hooks.queue_will_execute) { //将要执行任务
		_dispatch_set_pthread_root_queue_observer_hooks(
				&pqc->dpq_observer_hooks);
	}
	if (pqc->dpq_thread_configure) {
		pqc->dpq_thread_configure();
	}

#if !defined(_WIN32)
	// workaround tweaks the kernel workqueue does for us
	_dispatch_sigmask();
#endif
	_dispatch_introspection_thread_add();

	const int64_t timeout = 5ull * NSEC_PER_SEC;
	pthread_priority_t pp = _dispatch_get_priority();
	dispatch_priority_t pri = dq->dq_priority;

	// If the queue is neither
	// - the manager
	// - with a fallback set
	// - with a requested QoS or QoS floor
	// then infer the basepri from the current priority.
	if ((pri & (DISPATCH_PRIORITY_FLAG_MANAGER |
			DISPATCH_PRIORITY_FLAG_FALLBACK |
			DISPATCH_PRIORITY_FLAG_FLOOR |
			DISPATCH_PRIORITY_REQUESTED_MASK)) == 0) {
		pri &= DISPATCH_PRIORITY_FLAG_OVERCOMMIT;
		if (pp & _PTHREAD_PRIORITY_QOS_CLASS_MASK) {
			pri |= _dispatch_priority_from_pp(pp);
		} else {
			pri |= _dispatch_priority_make_override(DISPATCH_QOS_SATURATED);
		}
	}

#if DISPATCH_USE_INTERNAL_WORKQUEUE
	bool monitored = ((pri & (DISPATCH_PRIORITY_FLAG_OVERCOMMIT |
			DISPATCH_PRIORITY_FLAG_MANAGER)) == 0);
	if (monitored) _dispatch_workq_worker_register(dq);
#endif
	//这里是重点，调用_dispatch_root_queue_drain开始执行任务了
	do {
		_dispatch_trace_runtime_event(worker_unpark, dq, 0);
		_dispatch_root_queue_drain(dq, pri, DISPATCH_INVOKE_REDIRECTING_DRAIN);
		_dispatch_reset_priority_and_voucher(pp, NULL); //重置线程的优先级
		_dispatch_trace_runtime_event(worker_park, NULL, 0);
	} while (dispatch_semaphore_wait(&pqc->dpq_thread_mediator,
			dispatch_time(0, timeout)) == 0); //执行完一个任务后，线程调用dispatch_semaphore_wait，信号量-1，如果信号量的结果>=0，则返回0线程不阻塞继续do-while,否则线程阻塞等待5s，超时后返回1，退出循环。

#if DISPATCH_USE_INTERNAL_WORKQUEUE
	if (monitored) _dispatch_workq_worker_unregister(dq);
#endif
	(void)os_atomic_inc2o(dq, dgq_thread_pool_size, release);
	_dispatch_root_queue_poke(dq, 1, 0); //退出循环后继续poke
	_dispatch_release(dq); // retained in _dispatch_root_queue_poke_slow
	return NULL;
}
```

##### _dispatch_root_queue_drain

实现：

```c
DISPATCH_NOT_TAIL_CALLED // prevent tailcall (for Instrument DTrace probe)
static void
_dispatch_root_queue_drain(dispatch_queue_global_t dq,
		dispatch_priority_t pri, dispatch_invoke_flags_t flags)
{
#if DISPATCH_DEBUG
	dispatch_queue_t cq;
	if (unlikely(cq = _dispatch_queue_get_current())) {
		DISPATCH_INTERNAL_CRASH(cq, "Premature thread recycling");
	}
#endif
	_dispatch_queue_set_current(dq);
	_dispatch_init_basepri(pri);
	_dispatch_adopt_wlh_anon();

	struct dispatch_object_s *item;
	bool reset = false;
	dispatch_invoke_context_s dic = { };
#if DISPATCH_COCOA_COMPAT
	_dispatch_last_resort_autorelease_pool_push(&dic); //push自动释放池
#endif // DISPATCH_COCOA_COMPAT
	_dispatch_queue_drain_init_narrowing_check_deadline(&dic, pri);
	_dispatch_perfmon_start();
  //这里是重点，从队列的链表里取出头元素（一个queue或一个dc）。循环操作。即这个线程如果执行完了一个任务，如果队列里还有，则这个线程可以继续取出来执行。
	while (likely(item = _dispatch_root_queue_drain_one(dq))) {
		if (reset) _dispatch_wqthread_override_reset();
		_dispatch_continuation_pop_inline(item, &dic, flags, dq); //执行任务
		reset = _dispatch_reset_basepri_override();
		if (unlikely(_dispatch_queue_drain_should_narrow(&dic))) {
			break;
		}
	}

	// overcommit or not. worker thread
	if (pri & DISPATCH_PRIORITY_FLAG_OVERCOMMIT) {
		_dispatch_perfmon_end(perfmon_thread_worker_oc);
	} else {
		_dispatch_perfmon_end(perfmon_thread_worker_non_oc);
	}

#if DISPATCH_COCOA_COMPAT
	_dispatch_last_resort_autorelease_pool_pop(&dic); //pop自动释放池
#endif // DISPATCH_COCOA_COMPAT
	_dispatch_reset_wlh();
	_dispatch_clear_basepri();
	_dispatch_queue_set_current(NULL);
}
```

该函数主要作用：

循环操作，从全局队列的链表里取出一个任务（一个queue或一个dc），调用_dispatch_continuation_pop_inline执行任务

另外：gcd在执行任务前也会push自动释放池，所有任务执行完后pop自动释放池。因此即使我们没有显式创建，也不会产生内存泄漏。

##### _dispatch_root_queue_drain_one

实现：

\#define DISPATCH_ROOT_QUEUE_MEDIATOR ((struct dispatch_object_s *)~0ul)

```c
DISPATCH_ALWAYS_INLINE_NDEBUG
static inline struct dispatch_object_s *
_dispatch_root_queue_drain_one(dispatch_queue_global_t dq)
{
	struct dispatch_object_s *head, *next;

start:
	// The MEDIATOR value acts both as a "lock" and a signal
	head = os_atomic_xchg2o(dq, dq_items_head,
			DISPATCH_ROOT_QUEUE_MEDIATOR, relaxed); //原子替换并返回dq_items_head之前的值。

	if (unlikely(head == NULL)) {
		// The first xchg on the tail will tell the enqueueing thread that it
		// is safe to blindly write out to the head pointer. A cmpxchg honors
		// the algorithm.
		if (unlikely(!os_atomic_cmpxchg2o(dq, dq_items_head,
				DISPATCH_ROOT_QUEUE_MEDIATOR, NULL, relaxed))) {
			goto start;
		}
		if (unlikely(dq->dq_items_tail)) { // <rdar://problem/14416349>
			if (__DISPATCH_ROOT_QUEUE_CONTENDED_WAIT__(dq,
					_dispatch_root_queue_head_tail_quiesced)) {
				goto start;
			}
		}
		_dispatch_root_queue_debug("no work on global queue: %p", dq);
		return NULL;
	}

	if (unlikely(head == DISPATCH_ROOT_QUEUE_MEDIATOR)) {
		// This thread lost the race for ownership of the queue.
		if (likely(__DISPATCH_ROOT_QUEUE_CONTENDED_WAIT__(dq,
				_dispatch_root_queue_mediator_is_gone))) { //线程失去了队列的拥有权，就开始等待，在__DISPATCH_ROOT_QUEUE_CONTENDED_WAIT__里忙等。
			goto start;
		}
		return NULL;
	}

	// Restore the head pointer to a sane value before returning.
	// If 'next' is NULL, then this item _might_ be the last item.
	next = head->do_next;

	if (unlikely(!next)) {
		os_atomic_store2o(dq, dq_items_head, NULL, relaxed);
		// 22708742: set tail to NULL with release, so that NULL write to head
		//           above doesn't clobber head from concurrent enqueuer
		if (os_atomic_cmpxchg2o(dq, dq_items_tail, head, NULL, release)) {
			// both head and tail are NULL now
			goto out; //队列里目前确实只有一个，那么就走out。
		}
		// There must be a next item now.
		next = os_mpsc_get_next(head, do_next);
	}

	os_atomic_store2o(dq, dq_items_head, next, relaxed); //更新头
	_dispatch_root_queue_poke(dq, 1, 0); //开始递归调用了，而poke会开线程于是任务并发执行了。
out:
	return head;
}
```

取出全局队列里链表上的头元素，并更新dq_items_head。

##### _dispatch_continuation_pop_inline

执行任务

```c
DISPATCH_ALWAYS_INLINE_NDEBUG
static inline void
_dispatch_continuation_pop_inline(dispatch_object_t dou,
		dispatch_invoke_context_t dic, dispatch_invoke_flags_t flags,
		dispatch_queue_class_t dqu)
{
	dispatch_pthread_root_queue_observer_hooks_t observer_hooks =
			_dispatch_get_pthread_root_queue_observer_hooks();
	if (observer_hooks) observer_hooks->queue_will_execute(dqu._dq); //通知将要执行
	flags &= _DISPATCH_INVOKE_PROPAGATE_MASK;
	if (_dispatch_object_has_vtable(dou)) { //如果dou是队列，则调用dx_invoke
		dx_invoke(dou._dq, dic, flags);
	} else { //如果是dc那么就直接执行了
		_dispatch_continuation_invoke_inline(dou, flags, dqu);
	}
	if (observer_hooks) observer_hooks->queue_did_execute(dqu._dq); //通知执行完成
}
```

大致流程：

1. 如果传入的dou是队列，则调用dx_invoke。因为全局队列里的任务既可以是dc也可以是dq。
2. 如果传入的dou是dc（真正我们提交的任务），则调用_dispatch_continuation_invoke_inline开始执行

在我们的例子中这里的dou显然是dc，所以先来看是dc的情况，会调用_dispatch_continuation_invoke_inline开始执行我们的block。

##### dx_invoke多态调用

多态调用，后面会分析到。

##### _dispatch_continuation_invoke_inline

```c
DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_continuation_invoke_inline(dispatch_object_t dou,
		dispatch_invoke_flags_t flags, dispatch_queue_class_t dqu)
{
	dispatch_continuation_t dc = dou._dc, dc1;
	dispatch_invoke_with_autoreleasepool(flags, {
		uintptr_t dc_flags = dc->dc_flags;
		// Add the item back to the cache before calling the function. This
		// allows the 'hot' continuation to be used for a quick callback.
		//
		// The ccache version is per-thread.
		// Therefore, the object has not been reused yet.
		// This generates better assembly.
		_dispatch_continuation_voucher_adopt(dc, dc_flags);
		if (!(dc_flags & DC_FLAG_NO_INTROSPECTION)) {
			_dispatch_trace_item_pop(dqu, dou);
		}
		if (dc_flags & DC_FLAG_CONSUME) {
			dc1 = _dispatch_continuation_free_cacheonly(dc);
		} else {
			dc1 = NULL;
		}
		if (unlikely(dc_flags & DC_FLAG_GROUP_ASYNC)) { //判断是不是任务组
			_dispatch_continuation_with_group_invoke(dc);
		} else {
			_dispatch_client_callout(dc->dc_ctxt, dc->dc_func);
			_dispatch_trace_item_complete(dc);
		}
		if (unlikely(dc1)) {
			_dispatch_continuation_free_to_cache_limit(dc1);
		}
	});
	_dispatch_perfmon_workitem_inc();
}

DISPATCH_NOINLINE
void
_dispatch_client_callout(void *ctxt, dispatch_function_t f)
{
	_dispatch_get_tsd_base();
	void *u = _dispatch_get_unwind_tsd();
	if (likely(!u)) return f(ctxt);
	_dispatch_set_unwind_tsd(NULL);
	f(ctxt);//终于执行block了
	_dispatch_free_unwind_tsd();
	_dispatch_set_unwind_tsd(u);
}

//任务组的情况
DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_continuation_with_group_invoke(dispatch_continuation_t dc)
{
	struct dispatch_object_s *dou = dc->dc_data;
	unsigned long type = dx_type(dou);
	if (type == DISPATCH_GROUP_TYPE) {
		_dispatch_client_callout(dc->dc_ctxt, dc->dc_func); //执行block
		_dispatch_trace_item_complete(dc);
		dispatch_group_leave((dispatch_group_t)dou); //调用dispatch_group_leave，匹配enter
	} else {
		DISPATCH_INTERNAL_CRASH(dx_type(dou), "Unexpected object type");
	}
}
```

可以看到该函数就是在工作线程中执行我们提交的任务。如果该dc是任务组相关的则执行完后还会调用dispatch_group_leave，匹配之前的enter。

到此为止dispatch_async一个任务到全局队列的过程就分析完了。

稍微总结一下：

dispatch_async一个block任务到全局队列，dispatch_async会创建一个dc，并将dc添加到全局队列的链表上，然后开始poke队列，poke操作就是唤醒一个线程或者创建一个线程执行drain操作，drain操作就是从全局队列上的任务列表上取出一个任务执行并更新任务列表，由于drain操作又会递归调用poke操作，因此会将当前任务列表上的所有任务并发执行。

#### dx_push多态调用--串行队列的_dispatch_lane_push

接下来我们再以dispatch_async将任务派发到一个串行队列为例，进行分析

```c
dispatch_queue_t sQueue = dispatch_queue_create("xq.www.jk4", DISPATCH_QUEUE_SERIAL);
dispatch_async(sQueue, ^{
    NSLog(@"--1--");
});
```

此时队列的层级为：串-全

回到我们最初分析的`dx_push(dqu._dq, dc, qos)` 那么这里的dx_push就是_dispatch_lane_push的调用

```c
DISPATCH_NOINLINE
void
_dispatch_lane_push(dispatch_lane_t dq, dispatch_object_t dou,
		dispatch_qos_t qos)
{
	dispatch_wakeup_flags_t flags = 0;
	struct dispatch_object_s *prev;

	if (unlikely(_dispatch_object_is_waiter(dou))) {
		return _dispatch_lane_push_waiter(dq, dou._dsc, qos);
	}

	dispatch_assert(!_dispatch_object_is_global(dq));
	qos = _dispatch_queue_push_qos(dq, qos);

	// If we are going to call dx_wakeup(), the queue must be retained before
	// the item we're pushing can be dequeued, which means:
	// - before we exchange the tail if we have to override
	// - before we set the head if we made the queue non empty.
	// Otherwise, if preempted between one of these and the call to dx_wakeup()
	// the blocks submitted to the queue may release the last reference to the
	// queue when invoked by _dispatch_lane_drain. <rdar://problem/6932776>

	prev = os_mpsc_push_update_tail(os_mpsc(dq, dq_items), dou._do, do_next);
	if (unlikely(os_mpsc_push_was_empty(prev))) { //空队列
		_dispatch_retain_2_unsafe(dq);
		flags = DISPATCH_WAKEUP_CONSUME_2 | DISPATCH_WAKEUP_MAKE_DIRTY;
	} else if (unlikely(_dispatch_queue_need_override(dq, qos))) {
		// There's a race here, _dispatch_queue_need_override may read a stale
		// dq_state value.
		//
		// If it's a stale load from the same drain streak, given that
		// the max qos is monotonic, too old a read can only cause an
		// unnecessary attempt at overriding which is harmless.
		//
		// We'll assume here that a stale load from an a previous drain streak
		// never happens in practice.
		_dispatch_retain_2_unsafe(dq);
		flags = DISPATCH_WAKEUP_CONSUME_2;
	}
	os_mpsc_push_update_prev(os_mpsc(dq, dq_items), prev, dou._do, do_next);
	if (flags) {
		return dx_wakeup(dq, qos, flags);
	}
}
```

大概流程：

1. 将任务添加到队列的任务链表末尾
2. 如果flags>0（当前队列为空等），则调用dx_wakeup唤醒队列，唤醒标志类型为DISPATCH_WAKEUP_CONSUME_2 | DISPATCH_WAKEUP_MAKE_DIRTY
3. 如果flags==0，那么任务添加到到队列的任务链表上后，就完了（？？？），这种情况说明队列正在drain，所以等着就完了。

看一下dx_wakeup，和上面dx_push一样是一个宏，主要也是多态调用，这里看一下串行队列的实现

##### dx_wakeup(dq, qos, flags)多态调用--串行队列的 _dispatch_lane_wakeup

```c
DISPATCH_NOINLINE
void
_dispatch_lane_wakeup(dispatch_lane_class_t dqu, dispatch_qos_t qos,
		dispatch_wakeup_flags_t flags)
{
	dispatch_queue_wakeup_target_t target = DISPATCH_QUEUE_WAKEUP_NONE;

	if (unlikely(flags & DISPATCH_WAKEUP_BARRIER_COMPLETE)) { //队列因为barrier任务完成而唤醒
		return _dispatch_lane_barrier_complete(dqu, qos, flags);
	}
	if (_dispatch_queue_class_probe(dqu)) {
		target = DISPATCH_QUEUE_WAKEUP_TARGET;
	}
	return _dispatch_queue_wakeup(dqu, qos, flags, target); //正常分支
}

DISPATCH_ALWAYS_INLINE
static inline bool
_dispatch_queue_class_probe(dispatch_lane_class_t dqu)
{
	struct dispatch_object_s *tail;
	// seq_cst wrt atomic store to dq_state <rdar://problem/14637483>
	// seq_cst wrt atomic store to dq_flags <rdar://problem/22623242>
	tail = os_atomic_load2o(dqu._dl, dq_items_tail, ordered);
	return unlikely(tail != NULL);
}
```

在我们的例子中这里自然是走正常分支_dispatch_queue_wakeup

##### _dispatch_queue_wakeup

```c
DISPATCH_NOINLINE
void
_dispatch_queue_wakeup(dispatch_queue_class_t dqu, dispatch_qos_t qos,
		dispatch_wakeup_flags_t flags, dispatch_queue_wakeup_target_t target)
{
	dispatch_queue_t dq = dqu._dq;
	dispatch_assert(target != DISPATCH_QUEUE_WAKEUP_WAIT_FOR_EVENT);

	if (target && !(flags & DISPATCH_WAKEUP_CONSUME_2)) {
		_dispatch_retain_2(dq);
		flags |= DISPATCH_WAKEUP_CONSUME_2;
	}

	if (unlikely(flags & DISPATCH_WAKEUP_BARRIER_COMPLETE)) {
		//
		// _dispatch_lane_class_barrier_complete() is about what both regular
		// queues and sources needs to evaluate, but the former can have sync
		// handoffs to perform which _dispatch_lane_class_barrier_complete()
		// doesn't handle, only _dispatch_lane_barrier_complete() does.
		//
		// _dispatch_lane_wakeup() is the one for plain queues that calls
		// _dispatch_lane_barrier_complete(), and this is only taken for non
		// queue types.
		//
		dispatch_assert(dx_metatype(dq) == _DISPATCH_SOURCE_TYPE);
		qos = _dispatch_queue_wakeup_qos(dq, qos);
		return _dispatch_lane_class_barrier_complete(upcast(dq)._dl, qos,
				flags, target, DISPATCH_QUEUE_SERIAL_DRAIN_OWNED);
	}

	if (target) {
		uint64_t old_state, new_state, enqueue = DISPATCH_QUEUE_ENQUEUED;
		if (target == DISPATCH_QUEUE_WAKEUP_MGR) {
			enqueue = DISPATCH_QUEUE_ENQUEUED_ON_MGR;
		}
		qos = _dispatch_queue_wakeup_qos(dq, qos);
		os_atomic_rmw_loop2o(dq, dq_state, old_state, new_state, release, {
			new_state = _dq_state_merge_qos(old_state, qos);
			if (flags & DISPATCH_WAKEUP_CLEAR_ACTIVATING) {
				// When an event is being delivered to a source because its
				// unote was being registered before the ACTIVATING state
				// had a chance to be cleared, we don't want to fail the wakeup
				// which could lead to a priority inversion.
				//
				// Instead, these wakeups are allowed to finish the pending
				// activation.
				if (_dq_state_is_activating(old_state)) {
					new_state &= ~DISPATCH_QUEUE_ACTIVATING;
				}
			}
			if (likely(!_dq_state_is_suspended(new_state) &&
					!_dq_state_is_enqueued(old_state) &&
					(!_dq_state_drain_locked(old_state) ||
					enqueue != DISPATCH_QUEUE_ENQUEUED_ON_MGR))) {
				// Always set the enqueued bit for async enqueues on all queues
				// in the hierachy
				new_state |= enqueue;
			}
			if (flags & DISPATCH_WAKEUP_MAKE_DIRTY) {
				new_state |= DISPATCH_QUEUE_DIRTY;
			} else if (new_state == old_state) {
				os_atomic_rmw_loop_give_up(goto done);
			}
		});
	  //重点看这里
		if (likely((old_state ^ new_state) & enqueue)) {
			dispatch_queue_t tq;
			if (target == DISPATCH_QUEUE_WAKEUP_TARGET) {
				// the rmw_loop above has no acquire barrier, as the last block
				// of a queue asyncing to that queue is not an uncommon pattern
				// and in that case the acquire would be completely useless
				//
				// so instead use depdendency ordering to read
				// the targetq pointer.
				os_atomic_thread_fence(dependency);
				tq = os_atomic_load_with_dependency_on2o(dq, do_targetq,
						(long)new_state); //取出dq的do_targetq
			} else {
				tq = target;
			}
			dispatch_assert(_dq_state_is_enqueued(new_state));
			return _dispatch_queue_push_queue(tq, dq, new_state); //push_queue
		}
#if HAVE_PTHREAD_WORKQUEUE_QOS
		if (unlikely((old_state ^ new_state) & DISPATCH_QUEUE_MAX_QOS_MASK)) {
			if (_dq_state_should_override(new_state)) {
				return _dispatch_queue_wakeup_with_override(dq, new_state,
						flags);
			}
		}
	} else if (qos) {
		//
		// Someone is trying to override the last work item of the queue.
		//
		uint64_t old_state, new_state;
		os_atomic_rmw_loop2o(dq, dq_state, old_state, new_state, relaxed, {
			// Avoid spurious override if the item was drained before we could
			// apply an override
			if (!_dq_state_drain_locked(old_state) &&
				!_dq_state_is_enqueued(old_state)) {
				os_atomic_rmw_loop_give_up(goto done);
			}
			new_state = _dq_state_merge_qos(old_state, qos);
			if (new_state == old_state) {
				os_atomic_rmw_loop_give_up(goto done);
			}
		});
		if (_dq_state_should_override(new_state)) {
			return _dispatch_queue_wakeup_with_override(dq, new_state, flags);
		}
#endif // HAVE_PTHREAD_WORKQUEUE_QOS
	}
done:
	if (likely(flags & DISPATCH_WAKEUP_CONSUME_2)) {
		return _dispatch_release_2_tailcall(dq);
	}
}
```

1. 取出dq的do_targetq赋值给tq。
2. 调用`_dispatch_queue_push_queue(tq, dq, new_state);` 看名称可以知道这里是将一个队列作为任务添加到另一个队列上。

##### _dispatch_queue_push_queue

```c
DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_queue_push_queue(dispatch_queue_t tq, dispatch_queue_class_t dq,
		uint64_t dq_state)
{
#if DISPATCH_USE_KEVENT_WORKLOOP
	if (likely(_dq_state_is_base_wlh(dq_state))) {
		_dispatch_trace_runtime_event(worker_request, dq._dq, 1);
		return _dispatch_event_loop_poke((dispatch_wlh_t)dq._dq, dq_state,
				DISPATCH_EVENT_LOOP_CONSUME_2);
	}
#endif // DISPATCH_USE_KEVENT_WORKLOOP
	_dispatch_trace_item_push(tq, dq);
	return dx_push(tq, dq, _dq_state_max_qos(dq_state));//调用tq的push
}
```

串行队列的tq是全局队列，所以这里的dx_push调用的是_dispatch_root_queue_push（没错就是上面分析过的）。如果参数tq还不是全局队列那么又会绕一下会将queue自己作为一个任务添加到它的tq上，但最终会是全局队列，因为所有自定义的queue的tq最终都是全局队列。

##### _dispatch_root_queue_push

这里的例子就是将串行队列dq作为一个任务添加到全局队列的任务链表末尾。后面的逻辑跟前面全局队列一样的了。

沿着上面的分析`_dispatch_continuation_pop_inline` 里面会有一个dx_invoke的多态调用由于这里我们是串行队列，因此调用`_dispatch_lane_invoke`

##### _dispatch_lane_invoke

```c
DISPATCH_NOINLINE
void
_dispatch_lane_invoke(dispatch_lane_t dq, dispatch_invoke_context_t dic,
		dispatch_invoke_flags_t flags)
{
	_dispatch_queue_class_invoke(dq, dic, flags, 0, _dispatch_lane_invoke2);
}

DISPATCH_ALWAYS_INLINE
static inline dispatch_queue_wakeup_target_t
_dispatch_lane_invoke2(dispatch_lane_t dq, dispatch_invoke_context_t dic,
		dispatch_invoke_flags_t flags, uint64_t *owned)
{
	dispatch_queue_t otq = dq->do_targetq;
	dispatch_queue_t cq = _dispatch_queue_get_current();

	if (unlikely(cq != otq)) {
		return otq;
	}
	if (dq->dq_width == 1) { //串行走_dispatch_lane_serial_drain
		return _dispatch_lane_serial_drain(dq, dic, flags, owned);
	}
  //并发走_dispatch_lane_concurrent_drain
	return _dispatch_lane_concurrent_drain(dq, dic, flags, owned);
}

DISPATCH_NOINLINE
dispatch_queue_wakeup_target_t
_dispatch_lane_serial_drain(dispatch_lane_class_t dqu,
		dispatch_invoke_context_t dic, dispatch_invoke_flags_t flags,
		uint64_t *owned)
{
	flags &= ~(dispatch_invoke_flags_t)DISPATCH_INVOKE_REDIRECTING_DRAIN; //serial_drain会更改flags标志。
	return _dispatch_lane_drain(dqu._dl, dic, flags, owned, true);
}

DISPATCH_NOINLINE
static dispatch_queue_wakeup_target_t
_dispatch_lane_concurrent_drain(dispatch_lane_class_t dqu,
		dispatch_invoke_context_t dic, dispatch_invoke_flags_t flags,
		uint64_t *owned)
{
	return _dispatch_lane_drain(dqu._dl, dic, flags, owned, false);
}
```

大致流程：

1. dq为串行队列走_dispatch_lane_serial_drain逻辑
2. dq为并发队列走_dispatch_lane_concurrent_drain逻辑

区别就是serial_drain会更改flags标志位为非DISPATCH_INVOKE_REDIRECTING_DRAIN，而concurrent_drain不会，而这会影响 `_dispatch_lane_drain` 的模式。二者最终调用了_dispatch_lane_drain。

##### _dispatch_lane_drain

```c
/*
 * Drain comes in 2 flavours (serial/concurrent) and 2 modes
 * (redirecting or not).
 *
 * Serial
 * ~~~~~~
 * Serial drain is about serial queues (width == 1). It doesn't support
 * the redirecting mode, which doesn't make sense, and treats all continuations
 * as barriers. Bookkeeping is minimal in serial flavour, most of the loop
 * is optimized away.
 *
 * Serial drain stops if the width of the queue grows to larger than 1.
 * Going through a serial drain prevents any recursive drain from being
 * redirecting.
 *
 * Concurrent
 * ~~~~~~~~~~
 * When in non-redirecting mode (meaning one of the target queues is serial),
 * non-barriers and barriers alike run in the context of the drain thread.
 * Slow non-barrier items are still all signaled so that they can make progress
 * toward the dispatch_sync() that will serialize them all .
 *
 * In redirecting mode, non-barrier work items are redirected downward.
 *
 * Concurrent drain stops if the width of the queue becomes 1, so that the
 * queue drain moves to the more efficient serial mode.
 */
DISPATCH_ALWAYS_INLINE
static dispatch_queue_wakeup_target_t
_dispatch_lane_drain(dispatch_lane_t dq, dispatch_invoke_context_t dic,
		dispatch_invoke_flags_t flags, uint64_t *owned_ptr, bool serial_drain)
{
	dispatch_queue_t orig_tq = dq->do_targetq;
	dispatch_thread_frame_s dtf;
	struct dispatch_object_s *dc = NULL, *next_dc;
	uint64_t dq_state, owned = *owned_ptr;

	if (unlikely(!dq->dq_items_tail)) return NULL;

	_dispatch_thread_frame_push(&dtf, dq);
	if (serial_drain || _dq_state_is_in_barrier(owned)) {
		// we really own `IN_BARRIER + dq->dq_width * WIDTH_INTERVAL`
		// but width can change while draining barrier work items, so we only
		// convert to `dq->dq_width * WIDTH_INTERVAL` when we drop `IN_BARRIER`
		owned = DISPATCH_QUEUE_IN_BARRIER;
	} else {
		owned &= DISPATCH_QUEUE_WIDTH_MASK;
	}

	dc = _dispatch_queue_get_head(dq); //从队列（串行或并发）的任务链表中取出头元素
	goto first_iteration;

	for (;;) {
		dispatch_assert(dic->dic_barrier_waiter == NULL);
		dc = next_dc;
		if (unlikely(!dc)) {
			if (!dq->dq_items_tail) {
				break;
			}
			dc = _dispatch_queue_get_head(dq);
		}
		if (unlikely(_dispatch_needs_to_return_to_kernel())) {
			_dispatch_return_to_kernel();
		}
		if (unlikely(serial_drain != (dq->dq_width == 1))) {
			break;
		}
		if (unlikely(_dispatch_queue_drain_should_narrow(dic))) {
			break;
		}
		if (likely(flags & DISPATCH_INVOKE_WORKLOOP_DRAIN)) {
			dispatch_workloop_t dwl = (dispatch_workloop_t)_dispatch_get_wlh();
			if (unlikely(_dispatch_queue_max_qos(dwl) > dwl->dwl_drained_qos)) {
				break;
			}
		}

first_iteration:
		dq_state = os_atomic_load(&dq->dq_state, relaxed);
		if (unlikely(_dq_state_is_suspended(dq_state))) { //队列挂起了则跳出循环暂停执行任务
			break;
		}
		if (unlikely(orig_tq != dq->do_targetq)) {
			break;
		}

		if (serial_drain || _dispatch_object_is_barrier(dc)) { //串行队列或barrier dc/dq
			if (!serial_drain && owned != DISPATCH_QUEUE_IN_BARRIER) {
				if (!_dispatch_queue_try_upgrade_full_width(dq, owned)) {
					goto out_with_no_width;
				}
				owned = DISPATCH_QUEUE_IN_BARRIER;
			}
			if (_dispatch_object_is_sync_waiter(dc) &&
					!(flags & DISPATCH_INVOKE_THREAD_BOUND)) {
				dic->dic_barrier_waiter = dc;
				goto out_with_barrier_waiter;
			}
			next_dc = _dispatch_queue_pop_head(dq, dc);
		} else {
			if (owned == DISPATCH_QUEUE_IN_BARRIER) { //barrier相关的
				// we just ran barrier work items, we have to make their
				// effect visible to other sync work items on other threads
				// that may start coming in after this point, hence the
				// release barrier
				os_atomic_xor2o(dq, dq_state, owned, release);
				owned = dq->dq_width * DISPATCH_QUEUE_WIDTH_INTERVAL;
			} else if (unlikely(owned == 0)) {
				if (_dispatch_object_is_waiter(dc)) {
					// sync "readers" don't observe the limit
					_dispatch_queue_reserve_sync_width(dq);
				} else if (!_dispatch_queue_try_acquire_async(dq)) {
					goto out_with_no_width;
				}
				owned = DISPATCH_QUEUE_WIDTH_INTERVAL;
			}

			next_dc = _dispatch_queue_pop_head(dq, dc);
			if (_dispatch_object_is_waiter(dc)) {
				owned -= DISPATCH_QUEUE_WIDTH_INTERVAL;
				_dispatch_non_barrier_waiter_redirect_or_wake(dq, dc);
				continue;
			}

			if (flags & DISPATCH_INVOKE_REDIRECTING_DRAIN) { //
				owned -= DISPATCH_QUEUE_WIDTH_INTERVAL;
				// This is a re-redirect, overrides have already been applied by
				// _dispatch_continuation_async*
				// However we want to end up on the root queue matching `dc`
				// qos, so pick up the current override of `dq` which includes
				// dc's override (and maybe more)
				_dispatch_continuation_redirect_push(dq, dc,
						_dispatch_queue_max_qos(dq)); //重定向push。重定向到dq.tq
				continue;
			}
		} //else的
		//循环从队列的链表中取出dc,执行_dispatch_continuation_pop_inline
		_dispatch_continuation_pop_inline(dc, dic, flags, dq);
	}//for的

	if (owned == DISPATCH_QUEUE_IN_BARRIER) {
		// if we're IN_BARRIER we really own the full width too
		owned += dq->dq_width * DISPATCH_QUEUE_WIDTH_INTERVAL;
	}
	if (dc) {
		owned = _dispatch_queue_adjust_owned(dq, owned, dc);
	}
	*owned_ptr &= DISPATCH_QUEUE_ENQUEUED | DISPATCH_QUEUE_ENQUEUED_ON_MGR;
	*owned_ptr |= owned;
	_dispatch_thread_frame_pop(&dtf);
	return dc ? dq->do_targetq : NULL;

out_with_no_width:
	*owned_ptr &= DISPATCH_QUEUE_ENQUEUED | DISPATCH_QUEUE_ENQUEUED_ON_MGR;
	_dispatch_thread_frame_pop(&dtf);
	return DISPATCH_QUEUE_WAKEUP_WAIT_FOR_EVENT;

out_with_barrier_waiter:
	if (unlikely(flags & DISPATCH_INVOKE_DISALLOW_SYNC_WAITERS)) {
		DISPATCH_INTERNAL_CRASH(0,
				"Deferred continuation on source, mach channel or mgr");
	}
	_dispatch_thread_frame_pop(&dtf);
	return dq->do_targetq;
}
```

该函数的作用就是循环的从队列的链表中取出dc（也可能是dq），执行_dispatch_continuation_pop_inline，又回到了之前的那个调用。

drain分为两种flavour (serial/concurrent)，两种mode(redirecting or not)。如果dq是串行队列则只支持non-redirecting mode。如果dq是并发队列则两种mode都支持，当处于non-redirecting mode时（比如自定义并发队列的tq为串行队列），block任务都将在这一条线程上执行，其实已经相当于串行执行了。当处于redirecting mode时，自定义并发队列上的任务将进行重定向到它的tq上去。

到此为止dispatch_async一个任务到串行队列的工作流程就完了。

稍微总结一下：

dispatch_async到串行队列，dispatch_async会创建一个dc，并将dc添加到串行队列的链表上，如果串行队列添加前为空，则调用 `_dispatch_lane_wakeup` 唤醒串行队列，并执行 `_dispatch_queue_push_queue` 将串行队列作为一个任务添加到它的tq（这里是全局队列）的任务链表上，最后全局队列申请一个线程，串行队列通过调用`_dispatch_lane_drain` 倾倒队列，所有任务依次在该线程上执行。

#### dx_push多态调用--自定义并发队列的_dispatch_lane_concurrent_push

接下来我们再以dispatch_async将任务派发到一个并发队列为例，进行分析

```c
dispatch_queue_t cQueue = dispatch_queue_create("xq", DISPATCH_QUEUE_CONCURRENT);
dispatch_async(cQueue, ^{
    NSLog(@"--1--");
});
```

此时队列的层级为：并-全

回到我们最初分析的`dx_push(dqu._dq, dc, qos)` 那么这里的dx_push就是_dispatch_lane_concurrent_push的调用

```c
DISPATCH_NOINLINE
void
_dispatch_lane_concurrent_push(dispatch_lane_t dq, dispatch_object_t dou,
		dispatch_qos_t qos)
{
	// <rdar://problem/24738102&24743140> reserving non barrier width
	// doesn't fail if only the ENQUEUED bit is set (unlike its barrier
	// width equivalent), so we have to check that this thread hasn't
	// enqueued anything ahead of this call or we can break ordering
	if (dq->dq_items_tail == NULL &&
			!_dispatch_object_is_waiter(dou) &&
			!_dispatch_object_is_barrier(dou) &&
			_dispatch_queue_try_acquire_async(dq)) {
		return _dispatch_continuation_redirect_push(dq, dou, qos); //一来就重定向非常严格，首先队列必须为空，因为如果push了一个barrier的任务，那么没等它执行完，后面的任务是不能执行的，只能先入队排好队。
	}

	_dispatch_lane_push(dq, dou, qos);
}

/* Used by target-queue recursing code
 *
 * Initial state must be { sc:0, ib:0, qf:0, pb:0, d:0 }
 * Final state: { w += 1 }
 */
DISPATCH_ALWAYS_INLINE DISPATCH_WARN_RESULT
static inline bool
_dispatch_queue_try_acquire_async(dispatch_lane_t dq)
{
	uint64_t old_state, new_state;

	return os_atomic_rmw_loop2o(dq, dq_state, old_state, new_state, acquire, {
		if (unlikely(!_dq_state_is_runnable(old_state) ||
				_dq_state_is_dirty(old_state) ||
				_dq_state_has_pending_barrier(old_state))) {
			os_atomic_rmw_loop_give_up(return false);
		}
		new_state = old_state + DISPATCH_QUEUE_WIDTH_INTERVAL;
	});
}

DISPATCH_ALWAYS_INLINE
static inline bool
_dispatch_object_is_barrier(dispatch_object_t dou) //是否是barrier类型的dou，通过dispatch_barrier API调用
{
	dispatch_queue_flags_t dq_flags;

	if (!_dispatch_object_has_vtable(dou)) { //dou实际是dc
		return (dou._dc->dc_flags & DC_FLAG_BARRIER);
	}
	if (dx_cluster(dou._do) != _DISPATCH_QUEUE_CLUSTER) {
		return false;
	}
	dq_flags = os_atomic_load2o(dou._dq, dq_atomic_flags, relaxed); //dou实际是dq，则检查dq的dq_atomic_flags值
	return dq_flags & DQF_BARRIER_BIT;
}

DISPATCH_ALWAYS_INLINE
static inline bool
_dispatch_object_is_waiter(dispatch_object_t dou) //是否是waiter类型的dou,通过dispatch_async_and_wait类似API调用的
{
	if (_dispatch_object_has_vtable(dou)) {
		return false;
	}
	return (dou._dc->dc_flags & (DC_FLAG_SYNC_WAITER | DC_FLAG_ASYNC_AND_WAIT));
}

DISPATCH_NOINLINE
static void
_dispatch_continuation_redirect_push(dispatch_lane_t dl,
		dispatch_object_t dou, dispatch_qos_t qos)
{
	if (likely(!_dispatch_object_is_redirection(dou))) {
		dou._dc = _dispatch_async_redirect_wrap(dl, dou);
	} else if (!dou._dc->dc_ctxt) {
		// find first queue in descending target queue order that has
		// an autorelease frequency set, and use that as the frequency for
		// this continuation.
		dou._dc->dc_ctxt = (void *)
		(uintptr_t)_dispatch_queue_autorelease_frequency(dl);
	}

	dispatch_queue_t dq = dl->do_targetq; //这里假设没有太深queue层级，那么此时dq就是全局队列，下面的dx_push就是root_queue_push
	if (!qos) qos = _dispatch_priority_qos(dq->dq_priority);
	dx_push(dq, dou, qos);
}

DISPATCH_ALWAYS_INLINE
static inline dispatch_continuation_t
_dispatch_async_redirect_wrap(dispatch_lane_t dq, dispatch_object_t dou)
{
	dispatch_continuation_t dc = _dispatch_continuation_alloc();

	dou._do->do_next = NULL;
	dc->do_vtable = DC_VTABLE(ASYNC_REDIRECT);
	dc->dc_func = NULL;
	dc->dc_ctxt = (void *)(uintptr_t)_dispatch_queue_autorelease_frequency(dq);
	dc->dc_data = dq;
	dc->dc_other = dou._do;
	dc->dc_voucher = DISPATCH_NO_VOUCHER;
	dc->dc_priority = DISPATCH_NO_PRIORITY;
	_dispatch_retain_2(dq); // released in _dispatch_async_redirect_invoke
	return dc;
}
```

大致逻辑：

1. 当前dq是空并且满足一定条件后，走_dispatch_continuation_redirect_push逻辑，如果没有太深的queue层级关系的话，这里最终会将任务重定向的添加到全局队列的链表上。
2. 如果dq不为空，或dc是一个waiter，barrier等走 `_dispatch_lane_push` 逻辑，将任务添加队列自身的任务链表上，不用担心，在执行 `_dispatch_lane_drain` 时会继续重定向到它的tq上去的。

在我们这个例子中，刚开始队列肯定是空的，所以走的就是_dispatch_continuation_redirect_push逻辑。

##### _dispatch_continuation_redirect_push

```c
DISPATCH_NOINLINE
static void
_dispatch_continuation_redirect_push(dispatch_lane_t dl,
		dispatch_object_t dou, dispatch_qos_t qos)
{
	if (likely(!_dispatch_object_is_redirection(dou))) {
		dou._dc = _dispatch_async_redirect_wrap(dl, dou);
	} else if (!dou._dc->dc_ctxt) {
		// find first queue in descending target queue order that has
		// an autorelease frequency set, and use that as the frequency for
		// this continuation.
		dou._dc->dc_ctxt = (void *)
		(uintptr_t)_dispatch_queue_autorelease_frequency(dl);
	}

	dispatch_queue_t dq = dl->do_targetq; 
	if (!qos) qos = _dispatch_priority_qos(dq->dq_priority);
	dx_push(dq, dou, qos); 
}
```

该函数就是将并发队列的任务重定向到它的tq上去，直白点就是将dc添加到它的tq的任务列表上去而不是自身上。该例子中dq就是全局队列，于是又回到`_dispatch_root_queue_push`逻辑。如果dq是串行队列则回到`_dispatch_lan_push` 逻辑。如果dq是并发队列则回到自身逻辑。这里面是有点绕的。

##### _dispatch_lane_push

这里的逻辑同先前分析的。

综合上面几种情况总结一下：

调用dispatch_async派发一个任务到一个队列过程：

1. 将我们的block封装为一个dc
2. 如果队列是一个串行队列，则将dc添加到队列的链表上，并唤醒队列的tq继续push操作，push的元素为串行队列自身。
3. 如果队列是一个并发队列，则将dc重定向到它的tq上去。
4. 如此循环往复2，3直到tq为全局队列，因此全局队列上的元素可能是dc也可能是队列dq（串行或并发）。
5. 全局队列为链表上的每个任务从线程池中申请/创建一个线程执行，各任务是并发执行的。

可以看到我们自定义的队列是不创建线程的，统一由全局队列从线程池里获取线程，如果没有的话就创建一个线程，执行队列上的任务。这样就可以避免创建过多的线程。

并1--并2--串1--全，队列层级，派发到并1或并2的任务其实是串行执行的

并1--并2--串1--并3--全，队列层级，派发到并1或并2的任务也是串行执行的，串1和并3上的任务是并发执行的，但串1自己上的任务是串行执行的。

串1--并1--并2--全，队列层级，派发到串1，并1，并2上的任务是并发执行的，但串1自己上的任务是串行执行的。

### 线程池

在queue.c中有如下代码：

#### _dispatch_root_queues_init

```c
DISPATCH_STATIC_GLOBAL(dispatch_once_t _dispatch_root_queues_pred);
DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_root_queues_init(void)
{
	dispatch_once_f(&_dispatch_root_queues_pred, NULL,
			_dispatch_root_queues_init_once);
}

static void
_dispatch_root_queues_init_once(void *context DISPATCH_UNUSED)
{
	_dispatch_fork_becomes_unsafe();
#if DISPATCH_USE_INTERNAL_WORKQUEUE  //现在的GCD已经不使用内部的workqueue了，但这里为分析方便还是采用这里的
	size_t i;
	for (i = 0; i < DISPATCH_ROOT_QUEUE_COUNT; i++) {
		_dispatch_root_queue_init_pthread_pool(&_dispatch_root_queues[i], 0,
				_dispatch_root_queues[i].dq_priority);
	}
#else
  //现在使用的是pthread_workqueue
	int wq_supported = _pthread_workqueue_supported();
	int r = ENOTSUP;

	if (!(wq_supported & WORKQ_FEATURE_MAINTENANCE)) {
		DISPATCH_INTERNAL_CRASH(wq_supported,
				"QoS Maintenance support required");
	}

#if DISPATCH_USE_KEVENT_SETUP //1
	struct pthread_workqueue_config cfg = {
		.version = PTHREAD_WORKQUEUE_CONFIG_VERSION,
		.flags = 0,
		.workq_cb = 0,
		.kevent_cb = 0,
		.workloop_cb = 0,
		.queue_serialno_offs = dispatch_queue_offsets.dqo_serialnum,
#if PTHREAD_WORKQUEUE_CONFIG_VERSION >= 2
		.queue_label_offs = dispatch_queue_offsets.dqo_label,
#endif
	};
#endif

#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wunreachable-code"
	if (unlikely(!_dispatch_kevent_workqueue_enabled)) {
#if DISPATCH_USE_KEVENT_SETUP
		cfg.workq_cb = _dispatch_worker_thread2;
		r = pthread_workqueue_setup(&cfg, sizeof(cfg));
#else
		r = _pthread_workqueue_init(_dispatch_worker_thread2,
				offsetof(struct dispatch_queue_s, dq_serialnum), 0);
#endif // DISPATCH_USE_KEVENT_SETUP
#if DISPATCH_USE_KEVENT_WORKLOOP //1
	} else if (wq_supported & WORKQ_FEATURE_WORKLOOP) {
#if DISPATCH_USE_KEVENT_SETUP //1
		cfg.workq_cb = _dispatch_worker_thread2; //开的线程调用的也是_dispatch_worker_thread2入口
		cfg.kevent_cb = (pthread_workqueue_function_kevent_t) _dispatch_kevent_worker_thread;
		cfg.workloop_cb = (pthread_workqueue_function_workloop_t) _dispatch_workloop_worker_thread;
		r = pthread_workqueue_setup(&cfg, sizeof(cfg));
#else
		r = _pthread_workqueue_init_with_workloop(_dispatch_worker_thread2,
				(pthread_workqueue_function_kevent_t)
				_dispatch_kevent_worker_thread,
				(pthread_workqueue_function_workloop_t)
				_dispatch_workloop_worker_thread,
				offsetof(struct dispatch_queue_s, dq_serialnum), 0);
#endif // DISPATCH_USE_KEVENT_SETUP
#endif // DISPATCH_USE_KEVENT_WORKLOOP
#if DISPATCH_USE_KEVENT_WORKQUEUE
	} else if (wq_supported & WORKQ_FEATURE_KEVENT) {
#if DISPATCH_USE_KEVENT_SETUP
		cfg.workq_cb = _dispatch_worker_thread2;
		cfg.kevent_cb = (pthread_workqueue_function_kevent_t) _dispatch_kevent_worker_thread;
		r = pthread_workqueue_setup(&cfg, sizeof(cfg));
#else
		r = _pthread_workqueue_init_with_kevent(_dispatch_worker_thread2,
				(pthread_workqueue_function_kevent_t)
				_dispatch_kevent_worker_thread,
				offsetof(struct dispatch_queue_s, dq_serialnum), 0);
#endif // DISPATCH_USE_KEVENT_SETUP
#endif
	} else {
		DISPATCH_INTERNAL_CRASH(wq_supported, "Missing Kevent WORKQ support");
	}
#pragma clang diagnostic pop

	if (r != 0) {
		DISPATCH_INTERNAL_CRASH((r << 16) | wq_supported,
				"Root queue initialization failed");
	}
#endif // DISPATCH_USE_INTERNAL_WORKQUEUE
}
```

如果只看DISPATCH_USE_INTERNAL_WORKQUEUE分支，那么该函数的作用：

遍历设置全局队列数组中的全局队列的如下属性：

* do_ctxt属性的dpq_thread_mediator.dsema_sema，线程调度器，信号量实现。

* dgq_thread_pool_size属性值的设置

* do_ctxt属性的dpq_thread_attr

#### _dispatch_root_queue_init_pthread_pool

实现：

```c
static inline void
_dispatch_root_queue_init_pthread_pool(dispatch_queue_global_t dq,
		int pool_size, dispatch_priority_t pri)
{
	dispatch_pthread_root_queue_context_t pqc = dq->do_ctxt;
	int thread_pool_size = DISPATCH_WORKQ_MAX_PTHREAD_COUNT; //带overcommit的全局队列最多可开255条线程
	if (!(pri & DISPATCH_PRIORITY_FLAG_OVERCOMMIT)) { //普通全局队列开的线程数和内核数相等
		thread_pool_size = (int32_t)dispatch_hw_config(active_cpus);
	}
	if (pool_size && pool_size < thread_pool_size) thread_pool_size = pool_size;
	dq->dgq_thread_pool_size = thread_pool_size; 
	qos_class_t cls = _dispatch_qos_to_qos_class(_dispatch_priority_qos(pri) ?:
			_dispatch_priority_fallback_qos(pri));
	if (cls) {
#if !defined(_WIN32)
		pthread_attr_t *attr = &pqc->dpq_thread_attr;
		int r = pthread_attr_init(attr);
		dispatch_assume_zero(r);
		r = pthread_attr_setdetachstate(attr, PTHREAD_CREATE_DETACHED);
		dispatch_assume_zero(r);
#endif // !defined(_WIN32)
#if HAVE_PTHREAD_WORKQUEUE_QOS
		r = pthread_attr_set_qos_class_np(attr, cls, 0);
		dispatch_assume_zero(r);
#endif // HAVE_PTHREAD_WORKQUEUE_QOS
	}
	_dispatch_sema4_t *sema = &pqc->dpq_thread_mediator.dsema_sema;
	pqc->dpq_thread_mediator.do_vtable = DISPATCH_VTABLE(semaphore);
	_dispatch_sema4_init(sema, _DSEMA4_POLICY_LIFO);
	_dispatch_sema4_create(sema, _DSEMA4_POLICY_LIFO);
}
```

DISPATCH_WORKQ_MAX_PTHREAD_COUNT的定义：

```
#ifndef DISPATCH_WORKQ_MAX_PTHREAD_COUNT
#define DISPATCH_WORKQ_MAX_PTHREAD_COUNT 255
#endif
```

这里需要注意的是目前的GCD线程池的大小为：

如果是不带overcommit的全局队列，线程池最多开64个线程。64个线程都在用的话，后面再丢任务到并发队列也不会新开线程了这个时候就得等待其中某个线程变为空闲。需要注意的是64是线程池的最大线程数，也就是说即使两个全局队列同时执行任务，最多也只会开64个线程。

如果是带overcommit的全局队列，线程池最多开512个线程。一个串行队列一个线程，最多可以同时运行512个串行队列。

### dispatch_barrier_sync

`dispatch_barrier_sync` 内部是调用`_dispatch_barrier_sync_f`完成工作的。

```c
void
dispatch_barrier_sync(dispatch_queue_t dq, dispatch_block_t work)
{
	uintptr_t dc_flags = DC_FLAG_BARRIER | DC_FLAG_BLOCK; //虽然sync类API不会创建dc对象，但DC_FLAG还是要的，BARRIER类型的dc
	if (unlikely(_dispatch_block_has_private_data(work))) {
		return _dispatch_sync_block_with_privdata(dq, work, dc_flags);
	}
	_dispatch_barrier_sync_f(dq, work, _dispatch_Block_invoke(work), dc_flags);
}
```

注意这里传入的 `dc_flags = DC_FLAG_BARRIER | DC_FLAG_BLOCK;` 。

_dispatch_barrier_sync_f 的详细分析移步dispatch_sync的分析。

稍微总结一下dispatch_barrier_sync的流程：

1. 传入dc_flags为 DC_FLAG_BARRIER | DC_FLAG_BLOCK，代表这是一个barrier block任务。
2. 线程会走 `_dispatch_queue_try_acquire_barrier_sync` 尝试获取barrier，如果前面还有在执行的任务，会获取失败，线程进入等待状态。当前面的任务执行完后，线程被唤醒执行block，block执行完后，调用 `_dispatch_sync_invoke_and_complete_recurse` 恢复队列正常操作。
3. 如果尝试获取barrier成功，则会执行block，block执行完成后会调用 `_dispatch_lane_barrier_complete` 恢复队列正常操作。

dispatch_barrier_sync：会等待之前的任务全部执行完成，并使当前线程会进入等待状态，当之前的任务全部执行完成后，当前线程才会被唤醒.于是开始同步执行自身的block任务并阻塞当前线程直到任务完成才返回。和dispatch_barrier_async、dispatch_sync是不同的

dispatch_barrier_async：会等待之前的任务全部执行完成才执行自身的block任务。但它并不会让当前线程进入等待状态而是立刻返回，这一点与dispatch_barrier_sync有很大的不同。

dispatch_sync：同步执行自身的block任务并阻塞当前线程直到任务完成才返回。跟之前的任务没有任何关系。

### dispatch_barrier_async

```c
void
dispatch_barrier_async(dispatch_queue_t dq, dispatch_block_t work)
{
	dispatch_continuation_t dc = _dispatch_continuation_alloc();
	uintptr_t dc_flags = DC_FLAG_CONSUME | DC_FLAG_BARRIER;
	dispatch_qos_t qos;

	qos = _dispatch_continuation_init(dc, dq, work, 0, dc_flags);
	_dispatch_continuation_async(dq, dc, qos, dc_flags);
}
```

1. 创建并初始化dc，dc_flags。dc_flags是DC_FLAG_CONSUME | DC_FLAG_BARRIER。跟普通async多了DC_FLAG_BARRIER标记。可以肯定后面会根据dc_flags对不同dc进行分开处理。
2. 调用_dispatch_continuation_async。

可以看到和dispatch_async类似，只不过调用_dispatch_continuation_async时传入的参数不同。

注意：如果dq传的是全局队列，那么dispatch_barrier_async等价于dispatch_async，即dispatch_barrier_async失去了barrier的作用。事实上dispatch_barrier_sync/dispatch_barrier_async都应该传自定义并发队列。

### 问题

1.串行队列后一个任务需要等待前一个任务完成后才开始执行，底层是如何实现的？

如果是使用dispatch_sync派发一个同步任务A到串行队列，dispatch_sync内部会调用 `_dispatch_queue_try_acquire_barrier_sync` 函数队列会尝试获得barrier，获取成功后队列的状态会更新，如果此时另一个线程又派发了一个同步任务B到该串行队列，则队列继续获取barrier肯定会失败，线程进入等待状态，直到任务A完成。

如果是使用dispatch_async派发一个异步任务到串行队列，由于此时会新开一个线程，并且所有的任务都在这一个线程执行，因此任务自然是按顺序执行的。

### 参考

[Concurrency Programming Guide](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html)  官方并发编程指南

[深入理解GCD](https://juejin.im/post/57cb8e8367f3560057bf4da1) 很不错

[GCD源码吐血分析(2)——dispatch_async/dispatch_sync/dispatch_once/dispatch group](https://blog.csdn.net/u013378438/article/details/81076116?utm_medium=distribute.pc_relevant_right.none-task-blog-BlogCommendFromMachineLearnPai2-3.nonecase&depth_1-utm_source=distribute.pc_relevant_right.none-task-blog-BlogCommendFromMachineLearnPai2-3.nonecase)  作者对死锁问题有比较深入的分析

[GCD源码分析（二）——Dispatch Queue和Thread Pool](https://chy305chy.github.io/2018/11/29/GCD%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%88%E4%BA%8C%EF%BC%89%E2%80%94%E2%80%94Dispatch-Queue%E5%92%8CThread-Pool/) 主要是里面死锁的分析

[我所理解的 iOS 并发编程](https://blog.boolchow.com/2018/04/06/iOS-Concurrency-Programming/)  很全

[Concurrent Programming: APIs and Challenges](https://www.objc.io/issues/2-concurrency/concurrency-apis-and-pitfalls/) 不错，尤其里面其他知识点文章也很不错

主要讲了实现并发的几种方法，以及列举了一些并发编程可能会带来的问题。

[Concurrent Programming](https://www.objc.io/issues/2-concurrency/)  2013/08 objc.io出品的系列文章，不过都是理论和建议，感觉不是很深入。

[线程安全类的设计](https://objccn.io/issue-2-4/)  王巍译

[iOS面试题17-多线程](https://honkersk.github.io/2018/09/12/iOS面试题17-多线程/)

[Analysis of GCD source code principle](https://www.programmersought.com/article/78131683865/) 国外的，但广告有点多



### 附录



