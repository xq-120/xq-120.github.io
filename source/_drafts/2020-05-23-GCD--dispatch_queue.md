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

_dispatch_async_redirect_invoke

调用_dispatch_continuation_pop

调用_dispatch_continuation_invoke_inline

调用`_dispatch_continuation_with_group_invoke`或`_dispatch_client_callout`

1. 队列创建的过程
2. GCD默认的队列
3. 自定义串行队列与并发队列，主队列，全局队列等各队列之间的关系
4. 传入的任务是如何保存的，又是如何被执行的
5. 线程池

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
				overcommit == _dispatch_queue_attr_overcommit_enabled)->_as_dq;
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
7. 初始化dq及dq部分属性的更改。这里包括dq_width、dq_label、dq_priority还有do_targetq的设置等。
8. 返回创建的dq对象。

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
	if (unlikely(qos < DISPATCH_QOS_MIN || qos > DISPATCH_QOS_MAX)) {
		DISPATCH_CLIENT_CRASH(qos, "Corrupted priority");
	}
	return &_dispatch_root_queues[2 * (qos - 1) + overcommit]; //从全局数组中取出一个root_queue
}
```

_dispatch_root_queues其实是一个全局数组里面有12个元素，数组的元素是struct dispatch_queue_global_s：

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

它的初始化，部分代码：

```c
struct dispatch_queue_global_s _dispatch_root_queues[] = {
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
	...
}
```

创建的方式：

```c
dispatch_queue_attr_t attr = dispatch_queue_attr_make_with_qos_class(NULL, QOS_CLASS_DEFAULT, QOS_MIN_RELATIVE_PRIORITY);
dispatch_queue_t queue = dispatch_queue_create("xq.www.jk", attr);
queue = dispatch_queue_create("xq.www.jk", NULL);
queue = dispatch_queue_create("xq.www.jk", DISPATCH_QUEUE_SERIAL);
queue = dispatch_queue_create("xq.www.jk", DISPATCH_QUEUE_CONCURRENT); 
//_dispatch_queue_attr_concurrent是一个全局struct dispatch_queue_attr_s变量
queue = dispatch_queue_create("xq.www.jk", ((__bridge dispatch_queue_attr_t)&(_dispatch_queue_attr_concurrent)));
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

#### main_queue

dispatch_queue_t mainQueue = dispatch_get_main_queue();

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
```

#### global_queue

dispatch_queue_t glQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

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

其实就是从全局数组中取出的一个root_queue。

### dispatch_async





### 参考

[Concurrency Programming Guide](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html)  官方并发编程指南

[线程安全类的设计](https://objccn.io/issue-2-4/)  王巍译

[我所理解的 iOS 并发编程](https://blog.boolchow.com/2018/04/06/iOS-Concurrency-Programming/)  很全

[深入理解GCD](https://juejin.im/post/57cb8e8367f3560057bf4da1) 很不错

[Concurrent Programming: APIs and Challenges](https://www.objc.io/issues/2-concurrency/concurrency-apis-and-pitfalls/) 非常不错，尤其里面其他知识点文章也很不错

[Concurrent Programming](https://www.objc.io/issues/2-concurrency/)  2013/08 objc.io出品的系列文章

[GCD源码吐血分析(1)——GCD Queue](https://blog.csdn.net/u013378438/article/details/81031938)  GCD版本变化太大了，新版本里面的宏简直到了令人发指的地步了。作者的博客还可以，是成系列的。

[GCD源码解析(一)-dispatch_queue_create、dispatch_get_main_queue、dispatch_get_global_queue](https://www.jianshu.com/p/7702c06cda4c)  分析了队列的创建

### 附录

dispatch_qos_t

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

