---
title: iOS GCD
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

GCD 501版本的还不算复杂，但我就是不想看旧版本的于是入坑最新版本`libdispatch-1173.40.5`。

GCD版本变化太大了，新版本里面的宏简直到了令人发指的地步了。

问题：

1. 队列的创建过程
2. GCD默认的队列
3. 自定义的串行队列、并发队列与主队列，全局队列之间的关系
4. dispatch_sync工作流程
5. dispatch_sync死锁检测是如何检测的，是否会有检测不到的情况？
6. dispatch_async工作流程
7. dispatch_async传入的任务是如何保存的，又是如何被执行的，特别是有targetQueue变更时任务的执行过程
8. 线程池的工作流程（线程池的管理及维护）

### dispatch_queue_t

#### dispatch_queue_create

队列的创建

实现：

```c
dispatch_queue_t
dispatch_queue_create(const char *label, dispatch_queue_attr_t attr)
{
	return _dispatch_lane_create_with_target(label, attr,
			DISPATCH_TARGET_QUEUE_DEFAULT, true);
  //#define DISPATCH_TARGET_QUEUE_DEFAULT NULL
}

DISPATCH_NOINLINE
static dispatch_queue_t
_dispatch_lane_create_with_target(const char *label, dispatch_queue_attr_t dqa,
		dispatch_queue_t tq, bool legacy)
{
	dispatch_queue_attr_info_t dqai = _dispatch_queue_attr_to_info(dqa);

	//
	// Step 1: Normalize arguments (qos, overcommit, tq)
	//

	dispatch_qos_t qos = dqai.dqai_qos;  //qos：quality of service
#if !HAVE_PTHREAD_WORKQUEUE_QOS
	if (qos == DISPATCH_QOS_USER_INTERACTIVE) {
		dqai.dqai_qos = qos = DISPATCH_QOS_USER_INITIATED;
	}
	if (qos == DISPATCH_QOS_MAINTENANCE) {
		dqai.dqai_qos = qos = DISPATCH_QOS_BACKGROUND;
	}
#endif // !HAVE_PTHREAD_WORKQUEUE_QOS

	_dispatch_queue_attr_overcommit_t overcommit = dqai.dqai_overcommit;
	if (overcommit != _dispatch_queue_attr_overcommit_unspecified && tq) {
		if (tq->do_targetq) {
			DISPATCH_CLIENT_CRASH(tq, "Cannot specify both overcommit and "
					"a non-global target queue");
		}
	}

	if (tq && dx_type(tq) == DISPATCH_QUEUE_GLOBAL_ROOT_TYPE) {
		// Handle discrepancies between attr and target queue, attributes win
		if (overcommit == _dispatch_queue_attr_overcommit_unspecified) {
			if (tq->dq_priority & DISPATCH_PRIORITY_FLAG_OVERCOMMIT) {
				overcommit = _dispatch_queue_attr_overcommit_enabled;
			} else {
				overcommit = _dispatch_queue_attr_overcommit_disabled;
			}
		}
		if (qos == DISPATCH_QOS_UNSPECIFIED) {
			qos = _dispatch_priority_qos(tq->dq_priority);
		}
		tq = NULL;
	} else if (tq && !tq->do_targetq) {
		// target is a pthread or runloop root queue, setting QoS or overcommit
		// is disallowed
		if (overcommit != _dispatch_queue_attr_overcommit_unspecified) {
			DISPATCH_CLIENT_CRASH(tq, "Cannot specify an overcommit attribute "
					"and use this kind of target queue");
		}
	} else {
		if (overcommit == _dispatch_queue_attr_overcommit_unspecified) {
			// Serial queues default to overcommit!
			overcommit = dqai.dqai_concurrent ?
					_dispatch_queue_attr_overcommit_disabled :
					_dispatch_queue_attr_overcommit_enabled;
		}
	}
	if (!tq) {
		tq = _dispatch_get_root_queue(
				qos == DISPATCH_QOS_UNSPECIFIED ? DISPATCH_QOS_DEFAULT : qos,
				overcommit == _dispatch_queue_attr_overcommit_enabled)->_as_dq; //qos未指定则取默认值DISPATCH_QOS_DEFAULT，因此一般我们创建的并发queue的tq其实是同一个。串行queue的tq是另外的同一个。
		if (unlikely(!tq)) {
			DISPATCH_CLIENT_CRASH(qos, "Invalid queue attribute");
		}
	}

	//
	// Step 2: Initialize the queue
	//

	if (legacy) {
		// if any of these attributes is specified, use non legacy classes
		if (dqai.dqai_inactive || dqai.dqai_autorelease_frequency) {
			legacy = false;
		}
	}

	const void *vtable;  
	dispatch_queue_flags_t dqf = legacy ? DQF_MUTABLE : 0;
	if (dqai.dqai_concurrent) { //是否是并发队列
		vtable = DISPATCH_VTABLE(queue_concurrent);
	} else {
		vtable = DISPATCH_VTABLE(queue_serial);
	}
	switch (dqai.dqai_autorelease_frequency) {
	case DISPATCH_AUTORELEASE_FREQUENCY_NEVER:
		dqf |= DQF_AUTORELEASE_NEVER;
		break;
	case DISPATCH_AUTORELEASE_FREQUENCY_WORK_ITEM:
		dqf |= DQF_AUTORELEASE_ALWAYS;
		break;
	}
	if (label) {
		const char *tmp = _dispatch_strdup_if_mutable(label);
		if (tmp != label) {
			dqf |= DQF_LABEL_NEEDS_FREE;
			label = tmp;
		}
	}

	dispatch_lane_t dq = _dispatch_object_alloc(vtable,
			sizeof(struct dispatch_lane_s));
	_dispatch_queue_init(dq, dqf, dqai.dqai_concurrent ?
			DISPATCH_QUEUE_WIDTH_MAX : 1, DISPATCH_QUEUE_ROLE_INNER |
			(dqai.dqai_inactive ? DISPATCH_QUEUE_INACTIVE : 0));

	dq->dq_label = label;
	dq->dq_priority = _dispatch_priority_make((dispatch_qos_t)dqai.dqai_qos,
			dqai.dqai_relpri);
	if (overcommit == _dispatch_queue_attr_overcommit_enabled) {
		dq->dq_priority |= DISPATCH_PRIORITY_FLAG_OVERCOMMIT;
	}
	if (!dqai.dqai_inactive) {
		_dispatch_queue_priority_inherit_from_target(dq, tq);
		_dispatch_lane_inherit_wlh_from_target(dq, tq);
	}
	_dispatch_retain(tq);
	dq->do_targetq = tq;
	_dispatch_object_debug(dq, "%s", __func__);
	return _dispatch_trace_queue_create(dq)._dq;
}
```

大致流程：

1. 根据传入的dispatch_queue_attr_t dqa获取到队列属性信息dispatch_queue_attr_info_t dqai
2. 根据dqai 规范化参数
3. 获取target_queue，调用了_dispatch_get_root_queue。我们自己创建的queue最终会target到gcd创建好的某个全局的队列上。
4. 根据dqai.dqai_concurrent的值设置vtable，应该是对象类型。如果是并发队列则vtable = (&OS_dispatch_queue_concurrent_class);否则为串行队列vtable = (&OS_dispatch_queue_serial_class);
5. 队列的label设置
6. 调用`_dispatch_object_alloc(vtable, sizeof(struct dispatch_lane_s))`创建队列对象dq。
7. 初始化dq及dq部分属性的更改。这里包括dq_width、dq_label、dq_priority还有do_targetq的设置等。如果队列是串行队列则dq_width=1，如果是并发队列则dq_width=DISPATCH_QUEUE_WIDTH_MAX（0xffe）。另外系统默认的全局队列的dq_width=DISPATCH_QUEUE_WIDTH_POOL（0xfff）
8. 返回创建的dq对象。

dq_width的一些取值：

```c
#define DISPATCH_QUEUE_WIDTH_FULL			0x1000ull
#define DISPATCH_QUEUE_WIDTH_POOL (DISPATCH_QUEUE_WIDTH_FULL - 1) //0xfff
#define DISPATCH_QUEUE_WIDTH_MAX  (DISPATCH_QUEUE_WIDTH_FULL - 2) //0xffe
```

这里看下tq的获取：

```c
if (!tq) {
  tq = _dispatch_get_root_queue(
      qos == DISPATCH_QOS_UNSPECIFIED ? DISPATCH_QOS_DEFAULT : qos,
      overcommit == _dispatch_queue_attr_overcommit_enabled)->_as_dq;
  if (unlikely(!tq)) {
    DISPATCH_CLIENT_CRASH(qos, "Invalid queue attribute");
  }
}

DISPATCH_ALWAYS_INLINE DISPATCH_CONST
static inline dispatch_queue_global_t
_dispatch_get_root_queue(dispatch_qos_t qos, bool overcommit)
{
	if (unlikely(qos < DISPATCH_QOS_MIN || qos > DISPATCH_QOS_MAX)) { //1,6。这样计算出的index就刚好在0-11之间
		DISPATCH_CLIENT_CRASH(qos, "Corrupted priority");
	}
	return &_dispatch_root_queues[2 * (qos - 1) + overcommit]; //从全局数组中取出一个root_queue，qos越大，数组的index也越大。
}
```

_dispatch_root_queues其实是一个全局数组，里面有12个元素，数组的元素是struct dispatch_queue_global_s：

```c
// skip zero
// 1 - main_q
// 2 - mgr_q
// 3 - mgr_root_q
// 4,5,6,7,8,9,10,11,12,13,14,15 - global queues
// 17 - workloop_fallback_q
// we use 'xadd' on Intel, so the initial value == next assigned
#define DISPATCH_QUEUE_SERIAL_NUMBER_INIT 17
extern unsigned long volatile _dispatch_queue_serial_numbers;

// mark the workloop fallback queue to avoid finalizing objects on the base
// queue of custom outside-of-qos workloops
#define DISPATCH_QUEUE_SERIAL_NUMBER_WLF 16

extern struct dispatch_queue_static_s _dispatch_mgr_q; // serial 2
#if DISPATCH_USE_MGR_THREAD && DISPATCH_USE_PTHREAD_ROOT_QUEUES
extern struct dispatch_queue_global_s _dispatch_mgr_root_queue; // serial 3
#endif
extern struct dispatch_queue_global_s _dispatch_root_queues[]; // serials 4 - 15
```

它的初始化：每种优先级都有两种类型

```c
// 6618342 Contact the team that owns the Instrument DTrace probe before
//         renaming this symbol
struct dispatch_queue_global_s _dispatch_root_queues[] = {
#define _DISPATCH_ROOT_QUEUE_IDX(n, flags) \
		((flags & DISPATCH_PRIORITY_FLAG_OVERCOMMIT) ? \
		DISPATCH_ROOT_QUEUE_IDX_##n##_QOS_OVERCOMMIT : \
		DISPATCH_ROOT_QUEUE_IDX_##n##_QOS)
#define _DISPATCH_ROOT_QUEUE_ENTRY(n, flags, ...) \
	[_DISPATCH_ROOT_QUEUE_IDX(n, flags)] = { \
		DISPATCH_GLOBAL_OBJECT_HEADER(queue_global), \
		.dq_state = DISPATCH_ROOT_QUEUE_STATE_INIT_VALUE, \
		.do_ctxt = _dispatch_root_queue_ctxt(_DISPATCH_ROOT_QUEUE_IDX(n, flags)), \
		.dq_atomic_flags = DQF_WIDTH(DISPATCH_QUEUE_WIDTH_POOL), \  //全局队列width都是0xfff 
		.dq_priority = flags | ((flags & DISPATCH_PRIORITY_FLAG_FALLBACK) ? \
				_dispatch_priority_make_fallback(DISPATCH_QOS_##n) : \
				_dispatch_priority_make(DISPATCH_QOS_##n, 0)), \
		__VA_ARGS__ \
	}
	_DISPATCH_ROOT_QUEUE_ENTRY(MAINTENANCE, 0,
		.dq_label = "com.apple.root.maintenance-qos",
		.dq_serialnum = 4,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(MAINTENANCE, DISPATCH_PRIORITY_FLAG_OVERCOMMIT,
		.dq_label = "com.apple.root.maintenance-qos.overcommit",
		.dq_serialnum = 5,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(BACKGROUND, 0,
		.dq_label = "com.apple.root.background-qos",
		.dq_serialnum = 6,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(BACKGROUND, DISPATCH_PRIORITY_FLAG_OVERCOMMIT,
		.dq_label = "com.apple.root.background-qos.overcommit",
		.dq_serialnum = 7,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(UTILITY, 0,
		.dq_label = "com.apple.root.utility-qos",
		.dq_serialnum = 8,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(UTILITY, DISPATCH_PRIORITY_FLAG_OVERCOMMIT,
		.dq_label = "com.apple.root.utility-qos.overcommit",
		.dq_serialnum = 9,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(DEFAULT, DISPATCH_PRIORITY_FLAG_FALLBACK,
		.dq_label = "com.apple.root.default-qos",
		.dq_serialnum = 10,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(DEFAULT,
			DISPATCH_PRIORITY_FLAG_FALLBACK | DISPATCH_PRIORITY_FLAG_OVERCOMMIT,
		.dq_label = "com.apple.root.default-qos.overcommit",
		.dq_serialnum = 11,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(USER_INITIATED, 0,
		.dq_label = "com.apple.root.user-initiated-qos",
		.dq_serialnum = 12,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(USER_INITIATED, DISPATCH_PRIORITY_FLAG_OVERCOMMIT,
		.dq_label = "com.apple.root.user-initiated-qos.overcommit",
		.dq_serialnum = 13,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(USER_INTERACTIVE, 0,
		.dq_label = "com.apple.root.user-interactive-qos",
		.dq_serialnum = 14,
	),
	_DISPATCH_ROOT_QUEUE_ENTRY(USER_INTERACTIVE, DISPATCH_PRIORITY_FLAG_OVERCOMMIT,
		.dq_label = "com.apple.root.user-interactive-qos.overcommit",
		.dq_serialnum = 15,
	),
};
```

展开其中两个：

```c
[((DISPATCH_PRIORITY_FLAG_FALLBACK & DISPATCH_PRIORITY_FLAG_OVERCOMMIT) ? DISPATCH_ROOT_QUEUE_IDX_DEFAULT_QOS_OVERCOMMIT : DISPATCH_ROOT_QUEUE_IDX_DEFAULT_QOS)] = {
    .do_vtable = (&OS_dispatch_queue_global_class),
    .do_ref_cnt = DISPATCH_OBJECT_GLOBAL_REFCNT,
    .do_xref_cnt = DISPATCH_OBJECT_GLOBAL_REFCNT,
    .dq_state = DISPATCH_ROOT_QUEUE_STATE_INIT_VALUE,
    .do_ctxt = _dispatch_root_queue_ctxt(((DISPATCH_PRIORITY_FLAG_FALLBACK & DISPATCH_PRIORITY_FLAG_OVERCOMMIT) ? DISPATCH_ROOT_QUEUE_IDX_DEFAULT_QOS_OVERCOMMIT : DISPATCH_ROOT_QUEUE_IDX_DEFAULT_QOS)),
    .dq_atomic_flags = DQF_WIDTH(DISPATCH_QUEUE_WIDTH_POOL),
    .dq_priority = DISPATCH_PRIORITY_FLAG_FALLBACK | ((DISPATCH_PRIORITY_FLAG_FALLBACK & DISPATCH_PRIORITY_FLAG_FALLBACK) ? _dispatch_priority_make_fallback(DISPATCH_QOS_DEFAULT) : _dispatch_priority_make(DISPATCH_QOS_DEFAULT, 0)),
    .dq_label = "com.apple.root.default-qos",
    .dq_serialnum = 10,
},

[((DISPATCH_PRIORITY_FLAG_FALLBACK | DISPATCH_PRIORITY_FLAG_OVERCOMMIT & DISPATCH_PRIORITY_FLAG_OVERCOMMIT) ? DISPATCH_ROOT_QUEUE_IDX_DEFAULT_QOS_OVERCOMMIT : DISPATCH_ROOT_QUEUE_IDX_DEFAULT_QOS)] = {
    .do_vtable = (&OS_dispatch_queue_global_class),
    .do_ref_cnt = DISPATCH_OBJECT_GLOBAL_REFCNT,
    .do_xref_cnt = DISPATCH_OBJECT_GLOBAL_REFCNT,
    .dq_state = DISPATCH_ROOT_QUEUE_STATE_INIT_VALUE,
    .do_ctxt = _dispatch_root_queue_ctxt(((DISPATCH_PRIORITY_FLAG_FALLBACK | DISPATCH_PRIORITY_FLAG_OVERCOMMIT & DISPATCH_PRIORITY_FLAG_OVERCOMMIT) ? DISPATCH_ROOT_QUEUE_IDX_DEFAULT_QOS_OVERCOMMIT : DISPATCH_ROOT_QUEUE_IDX_DEFAULT_QOS)),
    .dq_atomic_flags = DQF_WIDTH(DISPATCH_QUEUE_WIDTH_POOL),
    .dq_priority = DISPATCH_PRIORITY_FLAG_FALLBACK | DISPATCH_PRIORITY_FLAG_OVERCOMMIT | ((DISPATCH_PRIORITY_FLAG_FALLBACK | DISPATCH_PRIORITY_FLAG_OVERCOMMIT & DISPATCH_PRIORITY_FLAG_FALLBACK) ? _dispatch_priority_make_fallback(DISPATCH_QOS_DEFAULT) : _dispatch_priority_make(DISPATCH_QOS_DEFAULT, 0)),
    .dq_label = "com.apple.root.default-qos.overcommit",
    .dq_serialnum = 11,
},

另：
#define DISPATCH_PRIORITY_FLAG_OVERCOMMIT    ((dispatch_priority_t)0x80000000) // _PTHREAD_PRIORITY_OVERCOMMIT_FLAG
#define DISPATCH_PRIORITY_FLAG_FALLBACK      ((dispatch_priority_t)0x04000000) // _PTHREAD_PRIORITY_FALLBACK_FLAG
#define DISPATCH_PRIORITY_FLAG_MANAGER       ((dispatch_priority_t)0x02000000) // _PTHREAD_PRIORITY_EVENT_MANAGER_FLAG
```

其实就是初始化[6]和[7]这两个元素的值。用宏的话省去了一些重复代码。另外rootQueue的do_targetq都是NULL。

一般通过dispatch_queue_create创建我们自己的队列：

```c
dispatch_queue_t
dispatch_queue_create(const char *_Nullable label,
		dispatch_queue_attr_t _Nullable attr);
```

可以传入两个参数label和attr（队列属性）

attr的创建：

```c
 * @param qos_class
 * A QOS class value:
 *  - QOS_CLASS_USER_INTERACTIVE
 *  - QOS_CLASS_USER_INITIATED
 *  - QOS_CLASS_DEFAULT
 *  - QOS_CLASS_UTILITY
 *  - QOS_CLASS_BACKGROUND
 * Passing any other value results in NULL being returned.
   
dispatch_queue_attr_t
dispatch_queue_attr_make_with_qos_class(dispatch_queue_attr_t _Nullable attr,
		dispatch_qos_class_t qos_class, int relative_priority);
```

有三个参数，可以参看官方详细注释

dispatch_qos_class_t qos_class：

```
__QOS_ENUM(qos_class, unsigned int,
	QOS_CLASS_USER_INTERACTIVE
			__QOS_CLASS_AVAILABLE(macos(10.10), ios(8.0)) = 0x21,
	QOS_CLASS_USER_INITIATED
			__QOS_CLASS_AVAILABLE(macos(10.10), ios(8.0)) = 0x19,
	QOS_CLASS_DEFAULT
			__QOS_CLASS_AVAILABLE(macos(10.10), ios(8.0)) = 0x15,
	QOS_CLASS_UTILITY
			__QOS_CLASS_AVAILABLE(macos(10.10), ios(8.0)) = 0x11,
	QOS_CLASS_BACKGROUND
			__QOS_CLASS_AVAILABLE(macos(10.10), ios(8.0)) = 0x09,
	QOS_CLASS_UNSPECIFIED
			__QOS_CLASS_AVAILABLE(macos(10.10), ios(8.0)) = 0x00,
);
```

relative_priority: [-15, 0]

```
/*!
 * @constant QOS_MIN_RELATIVE_PRIORITY
 * @abstract The minimum relative priority that may be specified within a
 * QOS class. These priorities are relative only within a given QOS class
 * and meaningful only for the current process.
 */
#define QOS_MIN_RELATIVE_PRIORITY (-15)
```

示例：

```c
//第一个参数用于设置串行队列/并发队列,第二个参数用于设置QOS会影响优先级，第三个参数用于设置相对优先级[QOS_MIN_RELATIVE_PRIORITY(-15), 0]
dispatch_queue_attr_t attr0 = dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_SERIAL, QOS_CLASS_USER_INITIATED, QOS_MIN_RELATIVE_PRIORITY); //串行队列属性
dispatch_queue_attr_t attr1 = dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_CONCURRENT, QOS_CLASS_DEFAULT, -3); //并发队列属性
dispatch_queue_t queue0 = dispatch_queue_create("xq.www.jk0", attr0);
dispatch_queue_t queue1 = dispatch_queue_create("xq.www.jk1", attr1);
dispatch_queue_t queue2 = dispatch_queue_create("xq.www.jk2", NULL);
dispatch_queue_t queue3 = dispatch_queue_create("xq.www.jk3", DISPATCH_QUEUE_SERIAL);
dispatch_queue_t queue4 = dispatch_queue_create("xq.www.jk4", DISPATCH_QUEUE_CONCURRENT);
dispatch_queue_t queue5 = dispatch_queue_create("xq.www.jk5", ((__bridge dispatch_queue_attr_t)&(_dispatch_queue_attr_concurrent)));
dispatch_queue_t mainQueue = dispatch_get_main_queue();
dispatch_queue_t glQueue1 = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_queue_t glQueue2 = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0);
dispatch_queue_t glQueue3 = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0);
dispatch_queue_t glQueue4 = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0);
```

DISPATCH_QUEUE_CONCURRENT宏展开：

`((__bridge dispatch_queue_attr_t)&(_dispatch_queue_attr_concurrent))`

_dispatch_queue_attr_concurrent是一个全局struct dispatch_queue_attr_s变量，初始化：

```c
struct dispatch_queue_attr_s _dispatch_queue_attr_concurrent = {
    .do_vtable = (&OS_dispatch_queue_attr_class),
    .do_ref_cnt = DISPATCH_OBJECT_GLOBAL_REFCNT,
    .do_xref_cnt = DISPATCH_OBJECT_GLOBAL_REFCNT,
};
```

打印：

```
Printing description of queue2:
<OS_dispatch_queue_serial: xq.www.jk2[0x6000036a1780] = { xref = 1, ref = 1, sref = 1, target = com.apple.root.default-qos.overcommit[0x108316f80], width = 0x1, state = 0x001ffe2000000000, in-flight = 0}>
Printing description of queue3:
<OS_dispatch_queue_serial: xq.www.jk3[0x6000036a0b00] = { xref = 1, ref = 1, sref = 1, target = com.apple.root.default-qos.overcommit[0x108316f80], width = 0x1, state = 0x001ffe2000000000, in-flight = 0}>
Printing description of mainQueue:
<OS_dispatch_queue_main: com.apple.main-thread[0x108316b00] = { xref = -2147483648, ref = -2147483648, sref = 1, target = com.apple.root.default-qos.overcommit[0x108316f80], width = 0x1, state = 0x001ffe9000000300, dirty, in-flight = 0, thread = 0x303 }>
Printing description of glQueue1:
<OS_dispatch_queue_global: com.apple.root.default-qos[0x108316f00] = { xref = -2147483648, ref = -2147483648, sref = 1, target = [0x0], width = 0xfff, state = 0x0060000000000000, in-barrier}>
Printing description of glQueue2:
<OS_dispatch_queue_global: com.apple.root.user-initiated-qos[0x108317000] = { xref = -2147483648, ref = -2147483648, sref = 1, target = [0x0], width = 0xfff, state = 0x0060000000000000, in-barrier}>
Printing description of glQueue3:
<OS_dispatch_queue_global: com.apple.root.utility-qos[0x108316e00] = { xref = -2147483648, ref = -2147483648, sref = 1, target = [0x0], width = 0xfff, state = 0x0060000000000000, in-barrier}>
Printing description of glQueue4:
<OS_dispatch_queue_global: com.apple.root.background-qos[0x108316d00] = { xref = -2147483648, ref = -2147483648, sref = 1, target = [0x0], width = 0xfff, state = 0x0060000000000000, in-barrier}>
Printing description of queue4:
<OS_dispatch_queue_concurrent: xq.www.jk4[0x6000036a1a80] = { xref = 1, ref = 1, sref = 1, target = com.apple.root.default-qos[0x108316f00], width = 0xffe, state = 0x0000041000000000, in-flight = 0}>
Printing description of queue5:
<OS_dispatch_queue_concurrent: xq.www.jk5[0x6000036a0d00] = { xref = 1, ref = 1, sref = 1, target = com.apple.root.default-qos[0x108316f00], width = 0xffe, state = 0x0000041000000000, in-flight = 0}>
```

#### main_queue

`dispatch_queue_t mainQueue = dispatch_get_main_queue();`

```c
dispatch_queue_main_t
dispatch_get_main_queue(void)
{
	return DISPATCH_GLOBAL_OBJECT(dispatch_queue_main_t, _dispatch_main_q);
}

((__bridge dispatch_queue_main_t)&(_dispatch_main_q));
```

_dispatch_main_q其实是一个全局变量：

```c
API_AVAILABLE(macos(10.6), ios(4.0))
DISPATCH_EXPORT
struct dispatch_queue_s _dispatch_main_q;
```

它的初始化：

```c
// 6618342 Contact the team that owns the Instrument DTrace probe before
//         renaming this symbol
struct dispatch_queue_static_s _dispatch_main_q = {
	DISPATCH_GLOBAL_OBJECT_HEADER(queue_main),
#if !DISPATCH_USE_RESOLVERS
	.do_targetq = _dispatch_get_default_queue(true),
#endif
	.dq_state = DISPATCH_QUEUE_STATE_INIT_VALUE(1) |
			DISPATCH_QUEUE_ROLE_BASE_ANON,
	.dq_label = "com.apple.main-thread",
	.dq_atomic_flags = DQF_THREAD_BOUND | DQF_WIDTH(1),
	.dq_serialnum = 1,
};

#define _dispatch_get_default_queue(overcommit) \
		_dispatch_root_queues[DISPATCH_ROOT_QUEUE_IDX_DEFAULT_QOS + \
				!!(overcommit)]._as_dq

// must be in lowest to highest qos order (as encoded in dispatch_qos_t)
// overcommit qos index values need bit 1 set
enum {
	DISPATCH_ROOT_QUEUE_IDX_MAINTENANCE_QOS = 0,
	DISPATCH_ROOT_QUEUE_IDX_MAINTENANCE_QOS_OVERCOMMIT,
	DISPATCH_ROOT_QUEUE_IDX_BACKGROUND_QOS,
	DISPATCH_ROOT_QUEUE_IDX_BACKGROUND_QOS_OVERCOMMIT,
	DISPATCH_ROOT_QUEUE_IDX_UTILITY_QOS,
	DISPATCH_ROOT_QUEUE_IDX_UTILITY_QOS_OVERCOMMIT,
	DISPATCH_ROOT_QUEUE_IDX_DEFAULT_QOS, //6
	DISPATCH_ROOT_QUEUE_IDX_DEFAULT_QOS_OVERCOMMIT,  
	DISPATCH_ROOT_QUEUE_IDX_USER_INITIATED_QOS,
	DISPATCH_ROOT_QUEUE_IDX_USER_INITIATED_QOS_OVERCOMMIT,
	DISPATCH_ROOT_QUEUE_IDX_USER_INTERACTIVE_QOS,
	DISPATCH_ROOT_QUEUE_IDX_USER_INTERACTIVE_QOS_OVERCOMMIT, //11
	_DISPATCH_ROOT_QUEUE_IDX_COUNT, //12
};
```

main_queue的targetQueue是取的_dispatch_root_queues的第7个元素：

```
_DISPATCH_ROOT_QUEUE_ENTRY(DEFAULT,
    DISPATCH_PRIORITY_FLAG_FALLBACK | DISPATCH_PRIORITY_FLAG_OVERCOMMIT,
  .dq_label = "com.apple.root.default-qos.overcommit",
  .dq_serialnum = 11,
),
```

打印一下mainQueue：

```
Printing description of mainQueue:
<OS_dispatch_queue_main: com.apple.main-thread[0x10c5b6b00] = { xref = -2147483648, ref = -2147483648, sref = 1, target = com.apple.root.default-qos.overcommit[0x10c5b6f80], width = 0x1, state = 0x001ffe9000000300, dirty, in-flight = 0, thread = 0x303 }>
```

#### global_queue

`dispatch_queue_t glQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);`

实现：

```c
dispatch_queue_global_t
dispatch_get_global_queue(long priority, unsigned long flags)
{
	dispatch_assert(countof(_dispatch_root_queues) ==
			DISPATCH_ROOT_QUEUE_COUNT);

	if (flags & ~(unsigned long)DISPATCH_QUEUE_OVERCOMMIT) {
		return DISPATCH_BAD_INPUT;
	}
	dispatch_qos_t qos = _dispatch_qos_from_queue_priority(priority);
#if !HAVE_PTHREAD_WORKQUEUE_QOS
	if (qos == QOS_CLASS_MAINTENANCE) {
		qos = DISPATCH_QOS_BACKGROUND;
	} else if (qos == QOS_CLASS_USER_INTERACTIVE) {
		qos = DISPATCH_QOS_USER_INITIATED;
	}
#endif
	if (qos == DISPATCH_QOS_UNSPECIFIED) {
		return DISPATCH_BAD_INPUT;
	}
	return _dispatch_get_root_queue(qos, flags & DISPATCH_QUEUE_OVERCOMMIT);
}
```

priority转qos:

```c
DISPATCH_ALWAYS_INLINE
static inline dispatch_qos_t
_dispatch_qos_from_queue_priority(long priority)
{
	switch (priority) {
	case DISPATCH_QUEUE_PRIORITY_BACKGROUND:      return DISPATCH_QOS_BACKGROUND; //2
	case DISPATCH_QUEUE_PRIORITY_NON_INTERACTIVE: return DISPATCH_QOS_UTILITY; //3
	case DISPATCH_QUEUE_PRIORITY_LOW:             return DISPATCH_QOS_UTILITY; //3
	case DISPATCH_QUEUE_PRIORITY_DEFAULT:         return DISPATCH_QOS_DEFAULT; //4
	case DISPATCH_QUEUE_PRIORITY_HIGH:            return DISPATCH_QOS_USER_INITIATED; //5
	default: return _dispatch_qos_from_qos_class((qos_class_t)priority);
	}
}

DISPATCH_ALWAYS_INLINE
static inline dispatch_qos_t
_dispatch_qos_from_qos_class(qos_class_t cls)
{
	switch ((unsigned int)cls) {
	case QOS_CLASS_USER_INTERACTIVE: return DISPATCH_QOS_USER_INTERACTIVE;
	case QOS_CLASS_USER_INITIATED:   return DISPATCH_QOS_USER_INITIATED;
	case QOS_CLASS_DEFAULT:          return DISPATCH_QOS_DEFAULT;
	case QOS_CLASS_UTILITY:          return DISPATCH_QOS_UTILITY;
	case QOS_CLASS_BACKGROUND:       return DISPATCH_QOS_BACKGROUND;
	case QOS_CLASS_MAINTENANCE:      return DISPATCH_QOS_MAINTENANCE;
	default: return DISPATCH_QOS_UNSPECIFIED;
	}
}
```

其实就是从全局数组_dispatch_root_queues中取出的一个root_queue。另外rootQueue的do_targetq是NULL。

**总结**

队列之间的关系：

### dispatch_sync

一般来说同步任务是在当前线程中执行（例外：丢到主队列的都是在主线程执行），同时它会阻塞当前线程直到任务执行完毕。

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

主要调用了`_dispatch_sync_f`-->`_dispatch_sync_f_inline`

```c
DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_sync_f_inline(dispatch_queue_t dq, void *ctxt,
		dispatch_function_t func, uintptr_t dc_flags)
{
	if (likely(dq->dq_width == 1)) { //串行队列走下面_dispatch_barrier_sync_f逻辑
		return _dispatch_barrier_sync_f(dq, ctxt, func, dc_flags);
	}

	if (unlikely(dx_metatype(dq) != _DISPATCH_LANE_TYPE)) {
		DISPATCH_CLIENT_CRASH(0, "Queue type doesn't support dispatch_sync");
	}

	dispatch_lane_t dl = upcast(dq)._dl;
	// Global concurrent queues and queues bound to non-dispatch threads
	// always fall into the slow case, see DISPATCH_ROOT_QUEUE_STATE_INIT_VALUE
	if (unlikely(!_dispatch_queue_try_reserve_sync_width(dl))) { //全局并发队列走_dispatch_sync_f_slow逻辑
		return _dispatch_sync_f_slow(dl, ctxt, func, 0, dl, dc_flags);
	}

	if (unlikely(dq->do_targetq->do_targetq)) {
		return _dispatch_sync_recurse(dl, ctxt, func, dc_flags);
	}
	_dispatch_introspection_sync_begin(dl);
	_dispatch_sync_invoke_and_complete(dl, ctxt, func DISPATCH_TRACE_ARG(
			_dispatch_trace_item_sync_push_pop(dq, ctxt, func, dc_flags)));
}
```

大致流程：

1. 判断队列的dq_width是否等于1，即是否是串行队列，如果是串行队列则走_dispatch_barrier_sync_f逻辑
2. 如果是全局并发队列，基本上走_dispatch_sync_f_slow逻辑
3. 其他情况比如自定义的并发队列则走正常的_dispatch_sync_invoke_and_complete逻辑，直接在当前线程执行block就完事了。不存在死锁不死锁的。

可以看到dispatch_sync并没有将block封装为一个dispatch_continuation_s对象然后添加到队列的链表上去，而是直接执行的。

为啥dispatch_sync逻辑要分串行队列还是并发队列呢？因为串行队列必须是前一个任务执行完后才能执行后一个任务，所以它内部会有一个`try_acquire_barrier`“加锁”的操作，保证即使在多线程同时调用dispatch_sync时也不会出现同时执行两个任务的情况。而并发队列就是可以同时执行任务的，所以可以不用`try_acquire_barrier`“加锁”的操作而是直接进入`_dispatch_sync_f_slow`逻辑。

因此dispatch_sync逻辑主要看第一个分支dq是串行队列时的_dispatch_barrier_sync_f逻辑。

#### _dispatch_barrier_sync_f

直勾勾的调用了`_dispatch_barrier_sync_f_inline`。  应该是所谓的接口隔离。

ps：`dispatch_barrier_sync` 内部也是调用`_dispatch_barrier_sync_f`完成工作的。分析完`_dispatch_barrier_sync_f_inline`，也就等于知道了`dispatch_barrier_sync` 的工作原理了。因此`dispatch_sync`派发到串行队列逻辑和`dispatch_barrier_sync` 派发到串行队列逻辑是一样的。

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
	dispatch_tid tid = _dispatch_tid_self(); //获取线程id--tid标识

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
	if (unlikely(!_dispatch_queue_try_acquire_barrier_sync(dl, tid))) { //尝试获得栅栏,Global concurrent queues always fall into the slow case
		return _dispatch_sync_f_slow(dl, ctxt, func, DC_FLAG_BARRIER, dl,
				DC_FLAG_BARRIER | dc_flags); //try_acquire_barrier失败，线程进入_dispatch_sync_f_slow逻辑，slow表明不是那么快的执行block任务，而是可能会等待一会才会执行。
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

1. 获取线程id--tid标识。try_acquire_barrier会用到。另外队列死锁检测也会用到tid。
2. 调用`_dispatch_queue_try_acquire_barrier_sync`尝试获得屏障，该函数会更新队列的dq_state的值为new_state（与tid相关）。如果失败则进入`_dispatch_sync_f_slow`，线程进入等待状态直到可以执行时才执行。对于全局并发队列try_acquire_barrier一般都会失败。
3. `_dispatch_queue_try_acquire_barrier_sync` 获取barrier成功的逻辑
4. 如果dl->do_targetq->do_targetq有值则说明是比较高层次的队列，则走`_dispatch_sync_recurse`逻辑
5. 最后走常规逻辑`_dispatch_lane_barrier_sync_invoke_and_complete`。

这里着重看一下`_dispatch_queue_try_acquire_barrier_sync` ，主要是了解一下dq_state的新值new_state是怎么计算得来的。

#### _dispatch_queue_try_acquire_barrier_sync

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

如果是串行队列和自定义并发队列则会更新dq_state的值为new_state，new_state这个值跟tid是有关系的。后面`__DISPATCH_WAIT_FOR_QUEUE__` 函数会根据该值与tid进行比较，用于判断线程是否已经获得栅栏进行死锁检测。所以使用dispatch_barrier_sync派发到自定义并发队列如果嵌套调用的话是会死锁崩溃的，而派发到全局队列则不会。比如：

```c
- (void)test_dispatch_barrier_sync_custom_concurrent_deadlock {
//    dispatch_queue_t queue = dispatch_queue_create("com.xq.queue", DISPATCH_QUEUE_CONCURRENT); //自定义的并发队列会死锁
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0); //全局队列不会死锁
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

#### _dispatch_sync_f_slow

try_acquire_barrier失败，线程进入_dispatch_sync_f_slow逻辑，slow表明不是那么快的执行block任务，而是可能会等待一会才会执行。

```c
DISPATCH_NOINLINE
static void
_dispatch_sync_f_slow(dispatch_queue_class_t top_dqu, void *ctxt,
		dispatch_function_t func, uintptr_t top_dc_flags,
		dispatch_queue_class_t dqu, uintptr_t dc_flags)
{
	dispatch_queue_t top_dq = top_dqu._dq;
	dispatch_queue_t dq = dqu._dq;
	if (unlikely(!dq->do_targetq)) { //全局队列满足条件走下面的逻辑，不会有死锁
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
		.dsc_waiter  = _dispatch_tid_self(), //获取线程ID
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

1. 调用`__DISPATCH_WAIT_FOR_QUEUE__`进入等待
2. 等待结束后，调用_dispatch_sync_invoke_and_complete_recurse执行block任务。

这里着重看`__DISPATCH_WAIT_FOR_QUEUE__`函数的实现。

#### \_\_DISPATCH_WAIT_FOR_QUEUE\_\_

```c
DISPATCH_NOINLINE
static void
__DISPATCH_WAIT_FOR_QUEUE__(dispatch_sync_context_t dsc, dispatch_queue_t dq)
{
	uint64_t dq_state = _dispatch_wait_prepare(dq);
	if (unlikely(_dq_state_drain_locked_by(dq_state, dsc->dsc_waiter))) {
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
2. 进行死锁检测。如果dq_state中有保存获得barrier成功的线程id（一般串行队列会有更新dq_state的值），当该线程再次进入时必然和dq_state中保存的tid相等，于是判断为死锁，崩溃。
3. 进入等待

崩溃的提示：

![](https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/DDBEB112-9737-463C-B420-654ADFE083BC.png)

死锁的产生必须满足四个必要条件：

- 资源互斥访问
- 请求与保持
- 不剥夺
- 循环等待

libdispatch中创建并分配线程来执行任务块的过程中，线程/队列对资源的操作满足上述前三个条件，那么如果再满足第四个条件，必然会发生死锁。因此在GCD中满足两个条件即会形成循环等待的情形：

- **串行队列**正在执行任务Task 1（无论是sync还是async方式提交的）
- Task 1未执行完成，又向队列中同步提交Task 2（**dispatch_sync**方式提交）

在以前的版本中GCD是没有死锁检测的，当发生死锁时不会有任何反应（当然如果发生在主线程中那么会卡死界面，如果是在子线程中那就神不知鬼不觉了），后来苹果工程师可能觉得这样不好于是加入了死锁检测的功能，如果发生死锁则主动抛出异常。

源码中死锁检测的实现并不复杂：

如果队列对象中的dq_state中有保存获得barrier成功的线程id（一般"dispatch_sync+串行队列"或"dispatch_barrier_sync+串行队列/自定义并发队列"会有更新dq_state的值），当该线程再次进入时必然和dq_state中保存的tid相等，于是判断为死锁，崩溃。

但这种检测办法并不能检测出所有死锁的情况，它只能检测出同一线程--同一队列的情况，其他情况就检测不到了，比如下面的代码：会形成一个循环等待但并不是同一线程--同一队列的情况，所以检测不出来，也就不会崩溃。

```
- (void)test_dispatch_sync_serial_deadlock_but_not_checked {
    //这里的死锁检测不到。
    dispatch_queue_t sQ1 = dispatch_queue_create("st01", DISPATCH_QUEUE_SERIAL);
    dispatch_async(sQ1, ^{
        NSLog(@"Enter");
        dispatch_sync(dispatch_get_main_queue(), ^{ //子线程拥有main_queue
            NSLog(@"---");
            dispatch_sync(sQ1, ^{ //主线程拥有sQ1
                NSArray *a = [NSArray new];
                NSLog(@"Enter again %@", a);
            });
        });
        NSLog(@"Done");
    });
}
```

### dispatch_async

往一个派发队列上提交一个**异步执行**的Block,**不阻塞当前线程**.可能会开新线程.

该函数的实现特别复杂，调用层级特别深，分支判断也特别多。

串行队列：

![](https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20200715145015.png)

自定义并发队列：

![](https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20200715145244.png)

全局队列：

![](https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20200715145405.png)

实现：

```c
void
dispatch_async(dispatch_queue_t dq, dispatch_block_t work)
{
	dispatch_continuation_t dc = _dispatch_continuation_alloc();
	uintptr_t dc_flags = DC_FLAG_CONSUME;
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

DC_FLAG_CONSUME：

```c
// continuation resources are freed on run
// this is set on async or for non event_handler source handlers
#define DC_FLAG_CONSUME					0x004ul
```

另外如果调用的是dispatch_group_async则uintptr_t dc_flags = DC_FLAG_CONSUME | DC_FLAG_GROUP_ASYNC;

**_dispatch_continuation_async**

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

主要是dx_push宏，展开：

```c
(&(dqu._dq)->do_vtable->_os_obj_vtable)->dq_push(dqu._dq, dc, qos);
```

这里是一个多态调用，不同子类对象调用同一方法会有不同的响应，我们可以看一下init.c中各子类的实现：

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
	.dq_wakeup      = _dispatch_lane_wakeup, //唤醒
	.dq_push        = _dispatch_lane_push, //push到链表中
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

以串行队列_dispatch_lane_push为例：

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
	if (unlikely(os_mpsc_push_was_empty(prev))) {
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
2. 如果有需要的话调用dx_wakeup唤醒

看一下dx_wakeup，和上面dx_push一样是一个宏，主要也是多态调用，这里看一下串行队列的实现_dispatch_lane_wakeup：

```c
DISPATCH_NOINLINE
void
_dispatch_lane_wakeup(dispatch_lane_class_t dqu, dispatch_qos_t qos,
		dispatch_wakeup_flags_t flags)
{
	dispatch_queue_wakeup_target_t target = DISPATCH_QUEUE_WAKEUP_NONE;

	if (unlikely(flags & DISPATCH_WAKEUP_BARRIER_COMPLETE)) {
		return _dispatch_lane_barrier_complete(dqu, qos, flags);
	}
	if (_dispatch_queue_class_probe(dqu)) {
		target = DISPATCH_QUEUE_WAKEUP_TARGET;
	}
	return _dispatch_queue_wakeup(dqu, qos, flags, target);
}
```

会调用：_dispatch_queue_wakeup

```
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
	
	...
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
						(long)new_state);
			} else {
				tq = target;
			}
			dispatch_assert(_dq_state_is_enqueued(new_state));
			return _dispatch_queue_push_queue(tq, dq, new_state);
		}
		...
}
```

会调用到_dispatch_queue_push_queue

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
	return dx_push(tq, dq, _dq_state_max_qos(dq_state));
}
```

因为串行队列的tq是全局队列，所以这里的dx_push调用的是_dispatch_root_queue_push

```c
DISPATCH_NOINLINE
void
_dispatch_root_queue_push(dispatch_queue_global_t rq, dispatch_object_t dou,
		dispatch_qos_t qos)
{
	...
	_dispatch_root_queue_push_inline(rq, dou, dou, 1);
}

DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_root_queue_push_inline(dispatch_queue_global_t dq,
		dispatch_object_t _head, dispatch_object_t _tail, int n)
{
	struct dispatch_object_s *hd = _head._do, *tl = _tail._do;
	if (unlikely(os_mpsc_push_list(os_mpsc(dq, dq_items), hd, tl, do_next))) {
		return _dispatch_root_queue_poke(dq, n, 0);
	}
}
```

大致流程：

1. 将任务push到最终的全局队列上
2. 如果可能则执行_dispatch_root_queue_poke

xxx

`_dispatch_root_queue_poke` --> `_dispatch_root_queue_poke_slow`：

```c
DISPATCH_NOINLINE
static void
_dispatch_root_queue_poke_slow(dispatch_queue_global_t dq, int n, int floor)
{
	int remaining = n;
	int r = ENOSYS;

	_dispatch_root_queues_init();
	_dispatch_debug_root_queue(dq, __func__);
	_dispatch_trace_runtime_event(worker_request, dq, (uint64_t)n);
	
	...
	
	dispatch_pthread_root_queue_context_t pqc = dq->do_ctxt;
	if (likely(pqc->dpq_thread_mediator.do_vtable)) {
		while (dispatch_semaphore_signal(&pqc->dpq_thread_mediator)) {
			_dispatch_root_queue_debug("signaled sleeping worker for "
					"global queue: %p", dq);
			if (!--remaining) {
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
		can_request = t_count < floor ? 0 : t_count - floor;
		if (remaining > can_request) {
			_dispatch_root_queue_debug("pthread pool reducing request from %d to %d",
					remaining, can_request);
			os_atomic_sub2o(dq, dgq_pending, remaining - can_request, relaxed);
			remaining = can_request;
		}
		if (remaining == 0) {
			_dispatch_root_queue_debug("pthread pool is full for root queue: "
					"%p", dq);
			return;
		}
	} while (!os_atomic_cmpxchgvw2o(dq, dgq_thread_pool_size, t_count,
			t_count - remaining, &t_count, acquire));

#if !defined(_WIN32)
	pthread_attr_t *attr = &pqc->dpq_thread_attr;
	pthread_t tid, *pthr = &tid;
#if DISPATCH_USE_MGR_THREAD && DISPATCH_USE_PTHREAD_ROOT_QUEUES
	if (unlikely(dq == &_dispatch_mgr_root_queue)) {
		pthr = _dispatch_mgr_root_queue_init();
	}
#endif
  //这里是重点
	do {
		_dispatch_retain(dq); // released in _dispatch_worker_thread
		while ((r = pthread_create(pthr, attr, _dispatch_worker_thread, dq))) {
			if (r != EAGAIN) {
				(void)dispatch_assume_zero(r);
			}
			_dispatch_temporary_resource_shortage();
		}
	} while (--remaining);
}
```

该函数里会创建线程执行_dispatch_worker_thread函数：

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
	//这里是重点调用_dispatch_root_queue_drain开始执行了
	do {
		_dispatch_trace_runtime_event(worker_unpark, dq, 0);
		_dispatch_root_queue_drain(dq, pri, DISPATCH_INVOKE_REDIRECTING_DRAIN);
		_dispatch_reset_priority_and_voucher(pp, NULL);
		_dispatch_trace_runtime_event(worker_park, NULL, 0);
	} while (dispatch_semaphore_wait(&pqc->dpq_thread_mediator,
			dispatch_time(0, timeout)) == 0);

#if DISPATCH_USE_INTERNAL_WORKQUEUE
	if (monitored) _dispatch_workq_worker_unregister(dq);
#endif
	(void)os_atomic_inc2o(dq, dgq_thread_pool_size, release);
	_dispatch_root_queue_poke(dq, 1, 0);
	_dispatch_release(dq); // retained in _dispatch_root_queue_poke_slow
	return NULL;
}
```













执行block：

```c
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
	if (dq->dq_width == 1) {
		return _dispatch_lane_serial_drain(dq, dic, flags, owned);
	}
	return _dispatch_lane_concurrent_drain(dq, dic, flags, owned);
}

DISPATCH_NOINLINE
void
_dispatch_lane_invoke(dispatch_lane_t dq, dispatch_invoke_context_t dic,
		dispatch_invoke_flags_t flags)
{
	_dispatch_queue_class_invoke(dq, dic, flags, 0, _dispatch_lane_invoke2);
}

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

	dc = _dispatch_queue_get_head(dq);
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
		if (unlikely(_dq_state_is_suspended(dq_state))) {
			break;
		}
		if (unlikely(orig_tq != dq->do_targetq)) {
			break;
		}

		if (serial_drain || _dispatch_object_is_barrier(dc)) {
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
			if (owned == DISPATCH_QUEUE_IN_BARRIER) {
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

			if (flags & DISPATCH_INVOKE_REDIRECTING_DRAIN) {
				owned -= DISPATCH_QUEUE_WIDTH_INTERVAL;
				// This is a re-redirect, overrides have already been applied by
				// _dispatch_continuation_async*
				// However we want to end up on the root queue matching `dc`
				// qos, so pick up the current override of `dq` which includes
				// dc's override (and maybe more)
				_dispatch_continuation_redirect_push(dq, dc,
						_dispatch_queue_max_qos(dq));
				continue;
			}
		}

		_dispatch_continuation_pop_inline(dc, dic, flags, dq);
	}

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

DISPATCH_NOINLINE
static dispatch_queue_wakeup_target_t
_dispatch_lane_concurrent_drain(dispatch_lane_class_t dqu,
		dispatch_invoke_context_t dic, dispatch_invoke_flags_t flags,
		uint64_t *owned)
{
	return _dispatch_lane_drain(dqu._dl, dic, flags, owned, false);
}

DISPATCH_NOINLINE
dispatch_queue_wakeup_target_t
_dispatch_lane_serial_drain(dispatch_lane_class_t dqu,
		dispatch_invoke_context_t dic, dispatch_invoke_flags_t flags,
		uint64_t *owned)
{
	flags &= ~(dispatch_invoke_flags_t)DISPATCH_INVOKE_REDIRECTING_DRAIN;
	return _dispatch_lane_drain(dqu._dl, dic, flags, owned, true);
}
```

重要函数：

_dispatch_worker_thread

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

	if (pqc->dpq_observer_hooks.queue_will_execute) {
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

	do {
		_dispatch_trace_runtime_event(worker_unpark, dq, 0);
		_dispatch_root_queue_drain(dq, pri, DISPATCH_INVOKE_REDIRECTING_DRAIN); //调用_dispatch_root_queue_drain
		_dispatch_reset_priority_and_voucher(pp, NULL);
		_dispatch_trace_runtime_event(worker_park, NULL, 0);
	} while (dispatch_semaphore_wait(&pqc->dpq_thread_mediator,
			dispatch_time(0, timeout)) == 0);

#if DISPATCH_USE_INTERNAL_WORKQUEUE
	if (monitored) _dispatch_workq_worker_unregister(dq);
#endif
	(void)os_atomic_inc2o(dq, dgq_thread_pool_size, release);
	_dispatch_root_queue_poke(dq, 1, 0); //调用poke
	_dispatch_release(dq); // retained in _dispatch_root_queue_poke_slow
	return NULL;
}
```

_dispatch_root_queue_poke:

```
DISPATCH_NOINLINE
void
_dispatch_root_queue_poke(dispatch_queue_global_t dq, int n, int floor)
{
	if (!_dispatch_queue_class_probe(dq)) {
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
	return _dispatch_root_queue_poke_slow(dq, n, floor); //调用poke_slow
}
```

_dispatch_root_queue_poke_slow:

```
do {
		_dispatch_retain(dq); // released in _dispatch_worker_thread
		while ((r = pthread_create(pthr, attr, _dispatch_worker_thread, dq))) { //创建线程执行_dispatch_worker_thread函数
			if (r != EAGAIN) {
				(void)dispatch_assume_zero(r);
			}
			_dispatch_temporary_resource_shortage();
		}
	} while (--remaining);
```

`_dispatch_root_queue_drain_one` -->`_dispatch_root_queue_poke`

```c
DISPATCH_ALWAYS_INLINE_NDEBUG
static inline struct dispatch_object_s *
_dispatch_root_queue_drain_one(dispatch_queue_global_t dq)
{
	struct dispatch_object_s *head, *next;

start:
	// The MEDIATOR value acts both as a "lock" and a signal
	head = os_atomic_xchg2o(dq, dq_items_head,
			DISPATCH_ROOT_QUEUE_MEDIATOR, relaxed);

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
				_dispatch_root_queue_mediator_is_gone))) {
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
			goto out;
		}
		// There must be a next item now.
		next = os_mpsc_get_next(head, do_next);
	}

	os_atomic_store2o(dq, dq_items_head, next, relaxed);
	_dispatch_root_queue_poke(dq, 1, 0); //调用poke
out:
	return head;
}
```

`_dispatch_root_queue_drain`：调用`_dispatch_root_queue_drain_one`

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
	_dispatch_last_resort_autorelease_pool_push(&dic);
#endif // DISPATCH_COCOA_COMPAT
	_dispatch_queue_drain_init_narrowing_check_deadline(&dic, pri);
	_dispatch_perfmon_start();
	while (likely(item = _dispatch_root_queue_drain_one(dq))) {
		if (reset) _dispatch_wqthread_override_reset();
		_dispatch_continuation_pop_inline(item, &dic, flags, dq);
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
	_dispatch_last_resort_autorelease_pool_pop(&dic);
#endif // DISPATCH_COCOA_COMPAT
	_dispatch_reset_wlh();
	_dispatch_clear_basepri();
	_dispatch_queue_set_current(NULL);
}
```

### dispatch_barrier_sync

`dispatch_barrier_sync` 内部是调用`_dispatch_barrier_sync_f`完成工作的。

```c
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

_dispatch_barrier_sync_f 的分析移步dispatch_sync的分析。

### dispatch_barrier_async







### 参考

[Concurrency Programming Guide](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html)  官方并发编程指南

[深入理解GCD](https://juejin.im/post/57cb8e8367f3560057bf4da1) 很不错

[GCD源码吐血分析(1)——GCD Queue](https://blog.csdn.net/u013378438/article/details/81031938)  作者的博客还可以，是成系列的。

[GCD源码吐血分析(2)——dispatch_async/dispatch_sync/dispatch_once/dispatch group](https://blog.csdn.net/u013378438/article/details/81076116?utm_medium=distribute.pc_relevant_right.none-task-blog-BlogCommendFromMachineLearnPai2-3.nonecase&depth_1-utm_source=distribute.pc_relevant_right.none-task-blog-BlogCommendFromMachineLearnPai2-3.nonecase)  作者对死锁问题有比较深入的分析

[GCD源码解析(一)-dispatch_queue_create、dispatch_get_main_queue、dispatch_get_global_queue](https://www.jianshu.com/p/7702c06cda4c)  分析了队列的创建过程

[GCD源码分析（二）——Dispatch Queue和Thread Pool](https://chy305chy.github.io/2018/11/29/GCD%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%88%E4%BA%8C%EF%BC%89%E2%80%94%E2%80%94Dispatch-Queue%E5%92%8CThread-Pool/) 主要是里面死锁的分析

[iOS GCD源码浅析](https://juejin.im/post/5e50e193f265da576f5303ca)  分析的还行

[我所理解的 iOS 并发编程](https://blog.boolchow.com/2018/04/06/iOS-Concurrency-Programming/)  很全

[Concurrent Programming: APIs and Challenges](https://www.objc.io/issues/2-concurrency/concurrency-apis-and-pitfalls/) 不错，尤其里面其他知识点文章也很不错

主要讲了实现并发的几种方法，以及列举了一些并发编程可能会带来的问题。

[Concurrent Programming](https://www.objc.io/issues/2-concurrency/)  2013/08 objc.io出品的系列文章，都是些理论和建议，感觉不是很深入。

[线程安全类的设计](https://objccn.io/issue-2-4/)  王巍译

### 附录

dispatch_qos_t  优先级，会影响index的计算，因为targetQueue是从全局队列数组中取的。

定义：

```c
typedef uint32_t dispatch_qos_t;
typedef uint32_t dispatch_priority_t;

#define DISPATCH_QOS_UNSPECIFIED        ((dispatch_qos_t)0)
#define DISPATCH_QOS_MAINTENANCE        ((dispatch_qos_t)1)
#define DISPATCH_QOS_BACKGROUND         ((dispatch_qos_t)2)
#define DISPATCH_QOS_UTILITY            ((dispatch_qos_t)3)
#define DISPATCH_QOS_DEFAULT            ((dispatch_qos_t)4)
#define DISPATCH_QOS_USER_INITIATED     ((dispatch_qos_t)5)
#define DISPATCH_QOS_USER_INTERACTIVE   ((dispatch_qos_t)6)
#define DISPATCH_QOS_MIN                DISPATCH_QOS_MAINTENANCE
#define DISPATCH_QOS_MAX                DISPATCH_QOS_USER_INTERACTIVE
#define DISPATCH_QOS_SATURATED          ((dispatch_qos_t)15)

#define DISPATCH_QOS_NBUCKETS           (DISPATCH_QOS_MAX - DISPATCH_QOS_MIN + 1)
#define DISPATCH_QOS_BUCKET(qos)        ((qos) - DISPATCH_QOS_MIN)
```

dispatch_qos_class_t

```c
/*!
 * @typedef dispatch_qos_class_t
 * Alias for qos_class_t type.
 */
#if __has_include(<sys/qos.h>)
typedef qos_class_t dispatch_qos_class_t;
#else
typedef unsigned int dispatch_qos_class_t;
#endif
```

struct dispatch_introspection_queue_s

队列自省结构体类型

```c
typedef struct dispatch_introspection_queue_s {
	dispatch_queue_t queue;
	dispatch_queue_t target_queue; //NULL indicates queue is a root queue.	
	const char *label;
	unsigned long serialnum;
	unsigned int width;
	unsigned int suspend_count;
	unsigned long enqueued:1,
			barrier:1,
			draining:1,
			global:1,
			main:1;
} dispatch_introspection_queue_s;
typedef dispatch_introspection_queue_s *dispatch_introspection_queue_t;
```



已知的

1. gcd创建了一个全局数组变量里面共12个全局队列，一个全局变量mainQueue。
2. gcd底层维护了一个线程池
3. 传入的block任务会添加到队列的单链表中

