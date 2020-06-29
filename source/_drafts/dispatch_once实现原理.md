版本：`libdispatch-1173.40.5`

`typedef intptr_t dispatch_once_t;`

typedef语法：`typedef 原类型名 别名`。

```c
typedef __darwin_intptr_t       intptr_t;
typedef long                    __darwin_intptr_t;
```

dispatch_once_t就是一个整型类型。

`typedef void (^dispatch_block_t)(void);`

无参数无返回值的block类型。

dispatch_once()函数：

```c
void
dispatch_once(dispatch_once_t *val, dispatch_block_t block)
{
	dispatch_once_f(val, block, _dispatch_Block_invoke(block));
}
```

_dispatch_Block_invoke宏如下：

```c
#define _dispatch_Block_invoke(bb) \
		((dispatch_function_t)((struct Block_layout *)bb)->invoke)
```

dispatch_function_t定义如下：

`typedef void (*dispatch_function_t)(void *_Nullable);`

为一个函数指针类型（参数为任意指针类型，无返回值）。

因此`_dispatch_Block_invoke(block)`就是获取block实例的函数指针，为了后面执行block。

dispatch_once_f()函数：

```c
DISPATCH_NOINLINE
void
dispatch_once_f(dispatch_once_t *val, void *ctxt, dispatch_function_t func)
{
	dispatch_once_gate_t l = (dispatch_once_gate_t)val;

#if !DISPATCH_ONCE_INLINE_FASTPATH || DISPATCH_ONCE_USE_QUIESCENT_COUNTER
	uintptr_t v = os_atomic_load(&l->dgo_once, acquire);
	if (likely(v == DLOCK_ONCE_DONE)) {
		return;
	}
#if DISPATCH_ONCE_USE_QUIESCENT_COUNTER
	if (likely(DISPATCH_ONCE_IS_GEN(v))) {
		return _dispatch_once_mark_done_if_quiesced(l, v);
	}
#endif
#endif
	if (_dispatch_once_gate_tryenter(l)) {
		return _dispatch_once_callout(l, ctxt, func);
	}
	return _dispatch_once_wait(l);
}
```

dispatch_once_f函数里面调用了很多其他函数，但现在不要纠结那些函数的具体实现，根据函数名可以猜个大概。

该函数的工作流程大致如下：

1. 原子的加载dgo_once的值（保证不会出现多线程竞争），如果等于DLOCK_ONCE_DONE，说明block已经执行过了，直接return。
2. 调用`_dispatch_once_gate_tryenter`尝试进入`dispatch_once_gate_t`，可以肯定该函数也是有类似锁操作的，要不然会出现几个线程同时进入的情况。
3. `_dispatch_once_gate_tryenter`进入成功，则调用`_dispatch_once_callout`执行 block，执行完成后唤醒等待的线程。
4. `_dispatch_once_gate_tryenter`进入失败，则调用`_dispatch_once_wait`进入等待状态，等待 block 的执行完成，然后被唤醒。

从该函数的工作流程不难看出如果传入的 block 中又调用自身，则线程会再次`_dispatch_once_gate_tryenter`，但肯定会失败从而执行`_dispatch_once_wait`进入等待，于是便形成自己等待自己导致死锁。事实上此时`_dispatch_once_wait`函数会抛出异常。

如果想知道dispatch_once具体是怎样保证多线程下 block 也只执行一次，其他线程又是怎样进入等待，又是怎样被唤醒，那么就需要查看`_dispatch_once_gate_tryenter`，`_dispatch_once_callout` ， `_dispatch_once_wait`的具体实现了。不过不用担心，实际上你也看不到线程究竟是怎样进入等待的因为`_dispatch_once_wait`最终会调用到一个系统函数进入等待而该系统函数是未公开的。

**_dispatch_once_gate_tryenter函数**

具体实现：

```c
DISPATCH_ALWAYS_INLINE
static inline bool
_dispatch_once_gate_tryenter(dispatch_once_gate_t l)
{
	return os_atomic_cmpxchg(&l->dgo_once, DLOCK_ONCE_UNLOCKED,
			(uintptr_t)_dispatch_lock_value_for_self(), relaxed);
}
```



**_dispatch_once_wait函数**

```c
void
_dispatch_once_wait(dispatch_once_gate_t dgo)
{
	dispatch_lock self = _dispatch_lock_value_for_self();
	uintptr_t old_v, new_v;
	dispatch_lock *lock = &dgo->dgo_gate.dgl_lock;
	uint32_t timeout = 1;

	for (;;) {
		os_atomic_rmw_loop(&dgo->dgo_once, old_v, new_v, relaxed, {
			if (likely(old_v == DLOCK_ONCE_DONE)) {
				os_atomic_rmw_loop_give_up(return);
			}
#if DISPATCH_ONCE_USE_QUIESCENT_COUNTER
			if (DISPATCH_ONCE_IS_GEN(old_v)) {
				os_atomic_rmw_loop_give_up({
					os_atomic_thread_fence(acquire);
					return _dispatch_once_mark_done_if_quiesced(dgo, old_v);
				});
			}
#endif
			new_v = old_v | (uintptr_t)DLOCK_WAITERS_BIT;
			if (new_v == old_v) os_atomic_rmw_loop_give_up(break);
		});
		if (unlikely(_dispatch_lock_is_locked_by((dispatch_lock)old_v, self))) {
			DISPATCH_CLIENT_CRASH(0, "trying to lock recursively");
		}
#if HAVE_UL_UNFAIR_LOCK
		_dispatch_unfair_lock_wait(lock, (dispatch_lock)new_v, 0,
				DLOCK_LOCK_NONE);
#elif HAVE_FUTEX
		_dispatch_futex_wait(lock, (dispatch_lock)new_v, NULL,
				FUTEX_PRIVATE_FLAG);
#else
		_dispatch_thread_switch(new_v, flags, timeout++);
#endif
		(void)timeout;
	}
}
```







likely(x)的作用就是告诉编译器需要优化生成的汇编指令，v == DLOCK_ONCE_DONE极有可能发生，在生成汇编指令时注意优化。

在一个条件判断语句中，当这个条件被认为是非常非常有可能满足时，则使用likely()宏，否则，条件非常非常不可能或很难满足时，则使用unlikely()宏。可以优化性能。

DISPATCH_ONCE_USE_QUIESCENT_COUNTER 宏

```c
#if defined(__x86_64__) || defined(__i386__) || defined(__s390x__)
#define DISPATCH_ONCE_USE_QUIESCENT_COUNTER 0
#elif __APPLE__
#define DISPATCH_ONCE_USE_QUIESCENT_COUNTER 1
#else
#define DISPATCH_ONCE_USE_QUIESCENT_COUNTER 0
#endif
```

所有的apple系统都会定义` __APPLE__`，包括MacOSX和iOS

真机的时候自然DISPATCH_ONCE_USE_QUIESCENT_COUNTER就为1了。

_dispatch_once_mark_done_if_quiesced函数：

```c
DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_once_mark_done_if_quiesced(dispatch_once_gate_t dgo, uintptr_t gen)
{
	if (_dispatch_once_generation() - gen >= DISPATCH_ONCE_GEN_SAFE_DELTA) {
		/*
		 * See explanation above, when the quiescing counter approach is taken
		 * then this store needs only to be relaxed as it is used as a witness
		 * that the required barriers have happened.
		 */
		os_atomic_store(&dgo->dgo_once, DLOCK_ONCE_DONE, relaxed);
	}
}
```

另

```c
#define _os_atomic_c11_atomic(p) \
		((__typeof__(*(p)) _Atomic *)(p))

#define os_atomic_load(p, m) \
		atomic_load_explicit(_os_atomic_c11_atomic(p), memory_order_##m)
#define os_atomic_store(p, v, m) \
		atomic_store_explicit(_os_atomic_c11_atomic(p), v, memory_order_##m)
```

在\#import <stdatomic.h>下有宏如下：

`#define atomic_load_explicit __c11_atomic_load`

```c
int _Atomic a = 1;
atomic_load_explicit(&a, memory_order_acquire);
```

__c11_atomic_load应该是某个函数名，暂时没找到。





lock相关：

```c
#define DLOCK_GATE_UNLOCKED	((dispatch_lock)0)

#define DLOCK_ONCE_UNLOCKED	((uintptr_t)0)
#define DLOCK_ONCE_DONE		(~(uintptr_t)0)   //~0，一个很大的整数。

typedef struct dispatch_gate_s {
	dispatch_lock dgl_lock;
} dispatch_gate_s, *dispatch_gate_t;

typedef struct dispatch_once_gate_s {
	union {
		dispatch_gate_s dgo_gate;
		uintptr_t dgo_once;
	};
} dispatch_once_gate_s, *dispatch_once_gate_t;
```

_dispatch_once_gate_tryenter()函数：

```c
DISPATCH_ALWAYS_INLINE
static inline bool
_dispatch_once_gate_tryenter(dispatch_once_gate_t l)
{
	return os_atomic_cmpxchg(&l->dgo_once, DLOCK_ONCE_UNLOCKED,
			(uintptr_t)_dispatch_lock_value_for_self(), relaxed);
}
```

_dispatch_once_callout()函数：

```c
DISPATCH_NOINLINE
static void
_dispatch_once_callout(dispatch_once_gate_t l, void *ctxt,
		dispatch_function_t func)
{
	_dispatch_client_callout(ctxt, func);
	_dispatch_once_gate_broadcast(l);
}
```

_dispatch_client_callout函数如下：

```c
DISPATCH_NOINLINE
void
_dispatch_client_callout(void *ctxt, dispatch_function_t f)
{
	_dispatch_get_tsd_base();
	void *u = _dispatch_get_unwind_tsd();
	if (likely(!u)) return f(ctxt);
	_dispatch_set_unwind_tsd(NULL);
	f(ctxt);
	_dispatch_free_unwind_tsd();
	_dispatch_set_unwind_tsd(u);
}
```

该函数的作用就是执行block。

_dispatch_once_gate_broadcast函数如下：

```c
DISPATCH_ALWAYS_INLINE
static inline void
_dispatch_once_gate_broadcast(dispatch_once_gate_t l)
{
	dispatch_lock value_self = _dispatch_lock_value_for_self();
	uintptr_t v;
#if DISPATCH_ONCE_USE_QUIESCENT_COUNTER
	v = _dispatch_once_mark_quiescing(l);
#else
	v = _dispatch_once_mark_done(l);
#endif
	if (likely((dispatch_lock)v == value_self)) return;
	_dispatch_gate_broadcast_slow(&l->dgo_gate, (dispatch_lock)v);
}
```



**question**

如何保证只执行一次，和传入的那个static整型变量有什么关系？

如果保证在多线程环境下也只执行一次。

多线程环境下的执行过程是怎样的。

Q4：`dispatch_once`死锁崩溃？

`dispatch_once`嵌套调用发生崩溃：

<img src="https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20200629152738.png" style="zoom:50%;" />

进一步跟进断点：

<img src="https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20200629162804.png" style="zoom:50%;" />

发现是崩溃在libdispatch的_dispatch_once_wait函数里面。其实已经提示的很清楚了：尝试递归的加锁，而这正是因为我们嵌套的调用导致的。







`#define os_atomic(type) type _Atomic`

使用：

`os_atomic(int) a = 1;`，预处理后就是：`int _Atomic a = 1;`。即定义一个原子操作的int类型变量a。

宏定义形如`#define A B`，使用A来代表B，通常A是一个含义更加清晰的名称。预处理过程会将A又替换成B，A空格后的内容都属于B。因此上述宏预处理后即为`int _Atomic`。

_Atomic

_Atomic是C语言中的原子操作修饰符。

```
int _Atomic c = 0;
- (void)viewDidLoad {
    [super viewDidLoad];

    os_atomic(int) a = 1;
    
    for (int i = 0; i < 1000; i++) {
        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
            c++;
        });
    }
    
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(3 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        NSLog(@"c = %d", c);
    });
}
```

此时c才会等于1000。如果不加_Atomic，则c的值将很有可能不等于1000而是小于1000。



结构体

```c
typedef struct MyStruct {
    int x;
    int y;
}MyStruct, *MyStructPtr;

MyStruct aStruct = {2, 3};
aStruct.x += 1;
NSLog(@"%d", aStruct.x);

struct MyStruct bStruct = {3, 5};
bStruct = aStruct; //只是aStruct的各属性的值赋值给bStruct，此后二者没啥关系。和普通类型赋值后是一样的。
bStruct.y = 10;

MyStructPtr ptr = &bStruct;
ptr->y = 14;
NSLog(@"%d %d", ptr->x, ptr->y); //指针操作时只能用->操作符。

NSLog(@"%p %p %p", &aStruct, &bStruct, ptr);
```



~0

```
unsigned int b = ~0;
NSLog(@"%u %x", b, b); //4294967295 ffffffff
NSLog(@"%d %x", b, b); //-1 ffffffff

int c = ~0;
NSLog(@"%d %x", c, c); //-1 ffffffff
NSLog(@"%u %x", c, c); //4294967295 ffffffff
```

%u或%d,%x只是告诉系统应该以何种格式化方式打印变量，对变量自身的值无影响。%u表示将变量的值格式化为无符号整型打印，%d表示将变量的值格式化为有符号整型打印。

参考：

https://www.jianshu.com/p/b25535a7403e

