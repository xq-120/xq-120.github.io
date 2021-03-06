---
title: GCD(二)--dispatch_queue
tags:
  - 标签1
  - 标签2
  - 标签3
categories:
  - 分类1
  - 分类2
comments: true
date: 2020-07-18 11:51:49
---

GCD 501版本的还不算复杂，但我就是不想看旧版本的于是入坑当前最新版本`libdispatch-1173.40.5`。

GCD版本变化太大了，新版本里面的宏简直到了令人发指的地步。

问题：

1. 队列的创建过程
2. GCD默认的队列：主队列和全局队列
3. 自定义的串行队列、并发队列与主队列，全局队列之间的关系
4. 派发到不同优先级队列的任务，GCD是如何实现按优先级执行的？（未完成）


#### dispatch_queue_create

队列的创建

实现：

```c
dispatch_queue_t
dispatch_queue_create(const char *label, dispatch_queue_attr_t attr)
{
	return _dispatch_lane_create_with_target(label, attr,
			DISPATCH_TARGET_QUEUE_DEFAULT, true); //我们自己创建的队列第三个参数始终是NULL
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

	_dispatch_queue_attr_overcommit_t overcommit = dqai.dqai_overcommit; //枚举：0-未指定，1-开启，2-关闭
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

	const void *vtable;  //dq对象的类型
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
			sizeof(struct dispatch_lane_s)); //终于开始创建dq了。
	_dispatch_queue_init(dq, dqf, dqai.dqai_concurrent ?
			DISPATCH_QUEUE_WIDTH_MAX : 1, DISPATCH_QUEUE_ROLE_INNER |
			(dqai.dqai_inactive ? DISPATCH_QUEUE_INACTIVE : 0)); //初始化dq

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
	dq->do_targetq = tq; //关联到targetQueue上
	_dispatch_object_debug(dq, "%s", __func__);
	return _dispatch_trace_queue_create(dq)._dq;
}

void *
_dispatch_object_alloc(const void *vtable, size_t size)
{
#if OS_OBJECT_HAVE_OBJC1
	const struct dispatch_object_vtable_s *_vtable = vtable;
	dispatch_object_t dou;
	dou._os_obj = _os_object_alloc_realized(_vtable->_os_obj_objc_isa, size);
	dou._do->do_vtable = vtable;
	return dou._do;
#else
	return _os_object_alloc_realized(vtable, size);
#endif
}

inline _os_object_t
_os_object_alloc_realized(const void *cls, size_t size)
{
	_os_object_t obj;
	dispatch_assert(size >= sizeof(struct _os_object_s));
	while (unlikely(!(obj = calloc(1u, size)))) { //申请内存
		_dispatch_temporary_resource_shortage();
	}
	obj->os_obj_isa = cls; //设置isa，对象的类型
	return obj;
}
```

大致流程：

1. 根据传入的dispatch_queue_attr_t dqa（对于自定义的一般是NULL或者DISPATCH_QUEUE_CONCURRENT）获取到队列属性信息dispatch_queue_attr_info_t dqai
2. 根据dqai 规范化参数
3. 因为自己创建的queue，tq参数默认为NULL，所以这里会调用_dispatch_get_root_queue获取一个target_queue。因此自己创建的queue最终会target到gcd创建好的某个全局的队列上。
4. 根据dqai.dqai_concurrent的值设置vtable，应该是对象类型。如果是并发队列则vtable = (&OS_dispatch_queue_concurrent_class);否则为串行队列vtable = (&OS_dispatch_queue_serial_class);
5. 队列的label设置
6. 调用`_dispatch_object_alloc(vtable, sizeof(struct dispatch_lane_s))`创建队列对象dq。
7. 初始化dq及dq部分属性的更改。这里包括dq_width、dq_label、dq_priority还有do_targetq的设置等。如果队列是串行队列则dq_width=1，如果是并发队列则dq_width=DISPATCH_QUEUE_WIDTH_MAX（0xffe）。另外系统默认的全局队列的dq_width=DISPATCH_QUEUE_WIDTH_POOL（0xfff）
8. 返回创建的dq对象。

dq_width的一些取值：

```c
#define DISPATCH_QUEUE_WIDTH_FULL			0x1000ull
#define DISPATCH_QUEUE_WIDTH_POOL (DISPATCH_QUEUE_WIDTH_FULL - 1) //0xfff 4095
#define DISPATCH_QUEUE_WIDTH_MAX  (DISPATCH_QUEUE_WIDTH_FULL - 2) //0xffe 4094
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

可以看到`_dispatch_get_root_queue`就是根据qos和overcommit计算出index，然后从`_dispatch_root_queues` 数组的index处取出一个全局队列对象。

_dispatch_root_queues是一个全局数组，里面有12个元素，数组的元素是struct dispatch_queue_global_s：

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

它的初始化：每种优先级都有两种标识（带或不带overcommit）串行队列的最终tq都是带overcommit，并发队列的最终tq是不带overcommit。

overcommit 与能够创建的线程数量有关。目前来看

不带overcommit的全局队列最多同时可以开64个线程执行任务，

带overcommit的全局队列最多同时可以开512个线程执行任务。

_dispatch_root_queues 的定义如下：

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

可以传入两个参数label和attr（队列属性）。attr一般传NULL或DISPATCH_QUEUE_CONCURRENT，但其实是可以自己创建一个的。

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

```c
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

我们经常传入的DISPATCH_QUEUE_SERIAL其实就是NULL。

而另一个DISPATCH_QUEUE_CONCURRENT也是个宏：

```c
#define DISPATCH_QUEUE_CONCURRENT \
		DISPATCH_GLOBAL_OBJECT(dispatch_queue_attr_t, \
		_dispatch_queue_attr_concurrent)
```

展开：

`((__bridge dispatch_queue_attr_t)&(_dispatch_queue_attr_concurrent))`

_dispatch_queue_attr_concurrent是一个全局struct dispatch_queue_attr_s变量，定义如下：

```c
struct dispatch_queue_attr_s _dispatch_queue_attr_concurrent = {
    .do_vtable = (&OS_dispatch_queue_attr_class),
    .do_ref_cnt = DISPATCH_OBJECT_GLOBAL_REFCNT,
    .do_xref_cnt = DISPATCH_OBJECT_GLOBAL_REFCNT,
};
```

为了使用方便，系统使用宏封装了一下。

打印：

```
Printing description of queue2:
<OS_dispatch_queue_serial: xq.www.jk2[0x6000036a1780] = { xref = 1, ref = 1, sref = 1, target = com.apple.root.default-qos.overcommit[0x108316f80], width = 0x1, state = 0x001ffe2000000000, in-flight = 0}>
Printing description of queue3:
<OS_dispatch_queue_serial: xq.www.jk3[0x6000036a0b00] = { xref = 1, ref = 1, sref = 1, target = com.apple.root.default-qos.overcommit[0x108316f80], width = 0x1, state = 0x001ffe2000000000, in-flight = 0}>
Printing description of mainQueue:
<OS_dispatch_queue_main: com.apple.main-thread[0x108316b00] = { xref = -2147483648, ref = -2147483648, sref = 1, target = com.apple.root.default-qos.overcommit[0x108316f80], width = 0x1, state = 0x001ffe9000000300, dirty, in-flight = 0, thread = 0x303 }> //自定义的串行队列和主队列的tq也都是同一个com.apple.root.default-qos.overcommit[0x108316f80]

Printing description of glQueue1:
<OS_dispatch_queue_global: com.apple.root.default-qos[0x108316f00] = { xref = -2147483648, ref = -2147483648, sref = 1, target = [0x0], width = 0xfff, state = 0x0060000000000000, in-barrier}>
Printing description of glQueue2:
<OS_dispatch_queue_global: com.apple.root.user-initiated-qos[0x108317000] = { xref = -2147483648, ref = -2147483648, sref = 1, target = [0x0], width = 0xfff, state = 0x0060000000000000, in-barrier}>
Printing description of glQueue3:
<OS_dispatch_queue_global: com.apple.root.utility-qos[0x108316e00] = { xref = -2147483648, ref = -2147483648, sref = 1, target = [0x0], width = 0xfff, state = 0x0060000000000000, in-barrier}>
Printing description of glQueue4:
<OS_dispatch_queue_global: com.apple.root.background-qos[0x108316d00] = { xref = -2147483648, ref = -2147483648, sref = 1, target = [0x0], width = 0xfff, state = 0x0060000000000000, in-barrier}> //全局队列的tq都是NULL，width = 0xfff

Printing description of queue4:
<OS_dispatch_queue_concurrent: xq.www.jk4[0x6000036a1a80] = { xref = 1, ref = 1, sref = 1, target = com.apple.root.default-qos[0x108316f00], width = 0xffe, state = 0x0000041000000000, in-flight = 0}>
Printing description of queue5:
<OS_dispatch_queue_concurrent: xq.www.jk5[0x6000036a0d00] = { xref = 1, ref = 1, sref = 1, target = com.apple.root.default-qos[0x108316f00], width = 0xffe, state = 0x0000041000000000, in-flight = 0}> //自定义的并发队列q4和q5的tq是同一个，q4和q5的width = 0xffe
```

总结：

1. 系统内部定义了12个全局队列，分为6种优先级：maintenance-qos，background-qos，utility-qos，default-qos，user-initiated-qos，user-interactive-qos。每种优先级又分带或不带overcommit。
2. 我们自己创建的自定义串行队列或并发队列都会target到一个全局队列。为什么要这么做？因为gcd底层有一个线程池，线程池管理着线程的创建与回收，并不是所有队列都直接和线程池交互，事实上只有全局队列才能和线程池交互，而我们的自定义队列和线程池是隔离的，派发到我们队列上的任务最终也是在某个全局队列上执行的。这样设计才能统一管理线程池，要不然随便一个队列都可以创建线程，那线程的数量就没法控制了。

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

dispatch_get_global_queue函数的作用就是从全局数组_dispatch_root_queues中根据priority计算出index后取出一个全局的root_queue。全局队列的tq是NULL。

#### QoS

priority其实就是一个整数，系统定义的有两组：

宏

```c
#define DISPATCH_QUEUE_PRIORITY_HIGH 2
#define DISPATCH_QUEUE_PRIORITY_DEFAULT 0
#define DISPATCH_QUEUE_PRIORITY_LOW (-2)
#define DISPATCH_QUEUE_PRIORITY_BACKGROUND INT16_MIN
```

和qos_class_t

```c
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

使用其他整数值将返回DISPATCH_QOS_UNSPECIFIED

对应的priority转qos函数为：

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

qos的内部定义

dispatch_qos_t  优先级，会影响index的计算，进而影响targetQueue的取值，因为targetQueue是根据index从全局队列数组中取的。

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



#### 队列之间的关系

如下图：

<img src="https://www.objc.io/images/issue-2/gcd-queues@2x-82965db9.png" style="zoom:50%;" />

注：上述箭头代表当前queue的targetQueue，当然全局队列没有tq。可以看到串行队列的tq可以设置为串行队列也可以设置为并发队列，tq的层级最好不要太深，要不然用起来容易把自己绕晕。

eg：

并1--并2--串1--全，队列层级，派发到并1或并2的任务其实是串行执行的

并1--并2--串1--并3--全，队列层级，派发到并1或并2的任务也是串行执行的，串1和并3上的任务是并发执行的，但串1自己上的任务是串行执行的。

串1--并1--并2--全，队列层级，派发到串1，并1，并2上的任务是并发执行的，但串1自己上的任务是串行执行的。

简单来说并发队列target到串行队列，则任务将串行执行。串行队列target到并发队列则不受影响串行队列上的任务依然串行执行，这个很好理解毕竟默认情况下串行队列本身就是target在全局队列上的。

#### 队列与线程的关系

1.主线程和主队列的关系

结论：主线程和主队列不是同一个东西，在主线程不一定意味着你就在主队列，在主队列也不一定意味着你就在主线程，不过对于UI类apps比如iOS应用和mac应用在主队列就代表着在主线程，因为Cocoa应用系统已经将主队列绑定在主线程了。

在主线程不一定意味着你就在主队列。这个很好理解，我可以在主线程执行一个派发到自定义串行队列的同步任务，这个block任务显然是在主线程执行的，但这个block所在的队列并不是主队列。

eg:

```swift
override func viewDidLoad() {
	super.viewDidLoad()
	let myQueue = DispatchQueue.init(label: "xxx")
  myQueue.sync {
      if Thread.isMainThread {
					...
      }
  }
}
```

还有一个比较诡异的系统bug：MKMapView的addOverlay必须在主队列执行，在主线程执行都不行(至少在iOS10.3.3是崩溃的，后面的版本不知道系统有没有修复)。详细讨论可以参考下面的 [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa) issue:

[24025596: MapKit: It's not safe to call MKMapView's addOverlay on a non-main-queue, even if it is executed on the main thread](https://github.com/lionheart/openradar-mirror/issues/7053)

[UIScheduler not working as expected](https://github.com/ReactiveCocoa/ReactiveCocoa/issues/2635)

示例代码:

```swift
class ViewController: UIViewController, MKMapViewDelegate {

    override func viewDidLoad() {
        super.viewDidLoad()
        
        view.backgroundColor = UIColor.white
        
        let mapView = MKMapView(frame: CGRect(x: 0, y: 88, width: view.frame.size.width, height: 400))
        view.addSubview(mapView)
        mapView.delegate = self

        let visibleRect = MKMapRect(origin: MKMapPoint(x: 146405966, y: 93063354),
            size: MKMapSize(width: 18634, height: 28375))
        mapView.setVisibleMapRect(visibleRect, animated: false)
        var coords = [CLLocationCoordinate2D(latitude: 48.211151, longitude: 16.365379),
            CLLocationCoordinate2D(latitude: 48.199167, longitude: 16.351016)]
        let polyline = MKPolyline(coordinates: &coords, count: coords.count)
        
        //崩溃
        let myQueue = DispatchQueue.init(label: "xxx")
        myQueue.sync {
            if Thread.isMainThread { //在主线程，但没在主队列，结果崩溃。
                mapView.addOverlay(polyline, level: .aboveRoads)
            }
        }
        //正常
//        DispatchQueue.main.async {
//            mapView.addOverlay(polyline, level: .aboveRoads)
//        }
    }

    func mapView(_ mapView: MKMapView, rendererFor overlay: MKOverlay) -> MKOverlayRenderer {
        let renderer = MKPolylineRenderer(overlay: overlay)
        renderer.lineWidth = 3
        renderer.strokeColor = UIColor.blue
        return renderer
    }
}
```

因此如果你仅仅以 `Thread.isMainThread` 判断在主线程了，就执行一些操作，可能还不够，还必须得保证在主队列。

那么怎么判断正在执行的block属于哪个队列呢？系统提供了一个 `dispatch_get_current_queue` ，不过该方法已经被废弃了。目前来看只能通过 dispatch_queue_set_specific/dispatch_get_specific 给队列添加一个key-value 来判断：

```objc
- (void)btnDidClicked:(UIButton *)sender {
    NSLog(@"btnDidClicked");
   
    dispatch_queue_set_specific(dispatch_get_main_queue(), "key", @"value", NULL);
    dispatch_async(dispatch_get_main_queue(), ^{
        NSString *value = (__bridge NSString *)(dispatch_get_specific("key"));
        NSLog(@"1 value:%@  current_queue:%@", value, dispatch_get_current_queue());
    });
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSString *value = (__bridge NSString *)(dispatch_get_specific("key"));
        NSLog(@"2 value:%@  current_queue:%@", value, dispatch_get_current_queue());
    });
}
```

打印：

```
2020-10-05 11:49:08.634407+0800 multithreadDemo[2518:7725112] btnDidClicked
2020-10-05 11:49:08.635368+0800 multithreadDemo[2518:7725201] 2 value:(null)  current_queue:<OS_dispatch_queue_global: com.apple.root.default-qos>
2020-10-05 11:49:08.636136+0800 multithreadDemo[2518:7725112] 1 value:value  current_queue:<OS_dispatch_queue_main: com.apple.main-thread>
```

前面也已经说了对于Cocoa应用系统已经将主队列绑定在主线程了，因此对于上述问题最好的解决办法就是使用 `DispatchQueue.main.async` 确保即在主线程又在主队列。

那么，怎么理解在主队列也不一定意味着你就在主线程？

示例代码：

我们新建一个非UI apps的Demo工程，使用dispatch_main，执行提交到主队列的block。

```objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        dispatch_async(dispatch_get_main_queue(), ^{
            NSLog(@"current_queue: %@", dispatch_get_current_queue());
            NSLog(@"currentThread: %@", [NSThread currentThread]);
            NSLog(@"is main thread? %i", (int)[NSThread isMainThread]);
        });
        dispatch_main();
        NSLog(@"----");
    }
    return 0;
}
```

打印：

```
2020-10-05 12:38:06.496003+0800 MainThreadDemo[3614:7773084] current_queue: <OS_dispatch_queue_main: com.apple.main-thread>
2020-10-05 12:38:06.497073+0800 MainThreadDemo[3614:7773084] currentThread: <NSThread: 0x10050f650>{number = 2, name = (null)}
2020-10-05 12:38:06.497202+0800 MainThreadDemo[3614:7773084] is main thread? 0
```

可以看到block并不是在主线程执行的。此时的主队列并没有绑定在主线程，就像普通的自定义串行队列，新开了一条线程执行block任务。

2.全局队列，自定义并发队列，自定义串行队列与线程的关系

没有关系，执行提交到队列的任务的线程是由系统管理的线程池分配的，线程执行完任务后会被回收。因此下一个提交到队列的任务可能还是上一次的线程执行，也可能是别的线程执行它。

[DispatchQueue](https://developer.apple.com/documentation/dispatch/dispatchqueue) 的说明：

```
Dispatch queues are FIFO queues to which your application can submit tasks in the form of block objects. Dispatch queues execute tasks either serially or concurrently. Work submitted to dispatch queues executes on a pool of threads managed by the system. Except for the dispatch queue representing your app's main thread, the system makes no guarantees about which thread it uses to execute a task.
```

[Queues are not bound to any specific thread](https://blog.krzyzanowskim.com/2016/06/03/queues-are-not-bound-to-any-specific-thread/)

总结：队列与线程没有对应或绑定关系，除了cocoa应用的主队列总是在主线程执行。

#### 参考

[GCD源码吐血分析(1)——GCD Queue](https://blog.csdn.net/u013378438/article/details/81031938)  作者的博客还可以，是成系列的。

[GCD源码解析(一)-dispatch_queue_create、dispatch_get_main_queue、dispatch_get_global_queue](https://www.jianshu.com/p/7702c06cda4c)  分析了队列的创建过程

