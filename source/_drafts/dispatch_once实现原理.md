版本：`libdispatch-1173.40.5`

### dispatch_once

一些类型说明：

**dispatch_once_t**

`typedef intptr_t dispatch_once_t;`

typedef语法：`typedef 原类型名 别名`。

```c
typedef __darwin_intptr_t       intptr_t;
typedef long                    __darwin_intptr_t;
```

dispatch_once_t就是一个整型类型。

**dispatch_block_t**

`typedef void (^dispatch_block_t)(void);`

无参数无返回值的block类型。

#### dispatch_once()函数

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

因此`_dispatch_Block_invoke(block)`就是获取block实例的函数指针。

dispatch_once调用了dispatch_once_f函数。

#### dispatch_once_f()函数

```c
DISPATCH_NOINLINE
void
dispatch_once_f(dispatch_once_t *val, void *ctxt, dispatch_function_t func)
{
	dispatch_once_gate_t l = (dispatch_once_gate_t)val; //将传入的整型变量地址强转为dispatch_once_gate_t类型

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

1. 原子的加载dgo_once（onceToken）的值（保证不会出现多线程竞争），如果等于DLOCK_ONCE_DONE，说明block已经执行过了，直接return。
2. 调用`_dispatch_once_gate_tryenter`尝试进入`dispatch_once_gate_t`，可以肯定该函数也是有类似锁操作的，要不然会出现几个线程同时进入的情况。
3. `_dispatch_once_gate_tryenter`进入成功，则调用`_dispatch_once_callout`执行 block，执行完成后唤醒等待的线程。
4. `_dispatch_once_gate_tryenter`进入失败，则调用`_dispatch_once_wait`进入等待状态，等待 block 的执行完成，然后被唤醒。

从该函数的工作流程不难看出如果传入的 block 中又调用自身，则线程会再次`_dispatch_once_gate_tryenter`，但肯定会失败从而执行`_dispatch_once_wait`进入等待，于是便形成自己等待自己导致死锁。事实上此时`_dispatch_once_wait`函数会抛出异常。

如果想知道dispatch_once具体是怎样保证多线程下 block 也只执行一次，其他线程又是怎样进入等待，又是怎样被唤醒，传入的静态整型变量onceToken又是干什么的？

那么就需要查看`_dispatch_once_gate_tryenter`，`_dispatch_once_callout` ， `_dispatch_once_wait`的具体实现了。不过不用担心，实际上你也看不到线程究竟是怎样进入等待的因为`_dispatch_once_wait`最终会调用到一个系统函数进入等待而该系统函数是未公开的。

接下来一句句查看实现：

dispatch_once_gate_t定义：

```c
#pragma mark - gate lock

#define DLOCK_GATE_UNLOCKED	((dispatch_lock)0)

#define DLOCK_ONCE_UNLOCKED	((uintptr_t)0) //0
#define DLOCK_ONCE_DONE		(~(uintptr_t)0) //~0，一个很大的整数。

typedef struct dispatch_gate_s {
	dispatch_lock dgl_lock;
} dispatch_gate_s, *dispatch_gate_t;

typedef struct dispatch_once_gate_s {
	union {
		dispatch_gate_s dgo_gate;
		uintptr_t dgo_once;
	};
} dispatch_once_gate_s, *dispatch_once_gate_t;

#if DISPATCH_ONCE_USE_QUIESCENT_COUNTER
#define DISPATCH_ONCE_MAKE_GEN(gen)  (((gen) << 2) + DLOCK_FAILED_TRYLOCK_BIT)
#define DISPATCH_ONCE_IS_GEN(gen)    (((gen) & 3) == DLOCK_FAILED_TRYLOCK_BIT)
```

另：dispatch_lock其实就是uint32_t的别名。详情可以查看附录。

`uintptr_t v = os_atomic_load(&l->dgo_once, acquire);`

展开后为：

```
uintptr_t v = atomic_load_explicit(((__typeof__(*(&l->dgo_once)) _Atomic *)(&l->dgo_once)), memory_order_acquire);
```

这里主要是`atomic_load_explicit（const volatile A * obj，memory_order order）;`函数的作用：

以原子方式加载并返回指向的原子变量的当前值`obj`。该操作是原子读取操作。

简单的讲该代码的意思就是原子读取onceToken的值。另外DLOCK_ONCE_DONE就等于~0。

```c
uintptr_t v = os_atomic_load(&l->dgo_once, acquire);
if (likely(v == DLOCK_ONCE_DONE)) {
  return;
}
```

该条件表明，如果 onceToken 的值等于~0，则表明block 已经执行过。直接 return。

#### _dispatch_once_gate_tryenter函数

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

将宏展开：

```c
(
	{ 
		__typeof__(atomic_load_explicit(((__typeof__(*(&l->dgo_once)) _Atomic *)(&l->dgo_once)), memory_order_relaxed)) _r = (DLOCK_ONCE_UNLOCKED); 	  
    atomic_compare_exchange_strong_explicit(((__typeof__(*(&l->dgo_once)) _Atomic *)(&l->dgo_once)), &_r, (uintptr_t)_dispatch_lock_value_for_self(), memory_order_relaxed, memory_order_relaxed); 
	}
);

BOOL func(int a) {
    if (a > 0) {
        return true;
    }
    return false;
}

BOOL rs = ({int a = 9; func(a);}); //rs 的值就是 func 的返回值。
```

这里主要是atomic_compare_exchange_strong_explicit函数的作用：

```c
bool atomic_compare_exchange_strong_explicit (volatile A* obj,
        T* expected, T val, memory_order success, memory_order failure) noexcept;
```

Compares the contents of the value contained in obj with the value pointed by expected:
\- if true, it replaces the *contained value* with val.
\- if false, it replaces the value pointed by expected with the *contained value* .

原子比较替换操作，如果 obj 指向的值等于expected指向的值则用 val 更新 obj 指向的值。否则使用obj 指向的值更新expected指向的值。

返回值：

`true` if `*expected` compares equal to the *contained value*.
`false` otherwise.

tip：这些 C语言的原子操作函数作用最好不要看中文翻译的，网上大部分都是机器翻译的，看了后只能让你更迷糊。

因此`_dispatch_once_gate_tryenter`函数的作用如下：

原子比较替换操作保证多线程安全，第一次执行时l->dgo_once也就是onceToken的值默认是 0，必然等于DLOCK_ONCE_UNLOCKED=0，因此将onceToken的值更新为`_dispatch_lock_value_for_self()`，至于`_dispatch_lock_value_for_self()`到底是多少可以不必纠结，肯定不是一个等于 0 的数。这样当其他线程执行该函数时必然返回 false，tryenter 失败。其他线程将执行_dispatch_once_wait函数，等待 block 执行完成。

基本上到这里我们就能够知道onceToken静态整型变量的作用了，主要用于标记 block 执行的状态。0 表示 block 未执行，~0表示 block 已执行（block 执行完成后会更新为该值），`_dispatch_lock_value_for_self()`（可以用 1 代替理解）表示 block 正在执行中。操作onceToken都是使用原子操作函数完成的。通过该静态变量标记 block 的状态再加上原子操作就可以保证即使在多线程环境下 block 也只执行一次。这里并没有使用什么锁这种比较重的同步技术。另外该整型变量必须是 static 的，如果只是一个普通自动变量是起不到作用的。

#### _dispatch_once_callout()函数

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

该函数的主要作用：

1. 执行 block
2. 更新 onceToken 的值为DLOCK_ONCE_DONE即~0
3. 唤醒等待的线程

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

看一下：`_dispatch_once_mark_done`函数的实现，为啥不看`_dispatch_once_mark_quiescing`这个函数的实现呢？因为这个实在是看不懂了。

```c
DISPATCH_ALWAYS_INLINE
static inline uintptr_t
_dispatch_once_mark_done(dispatch_once_gate_t dgo)
{
	return os_atomic_xchg(&dgo->dgo_once, DLOCK_ONCE_DONE, release);
}
```

os_atomic_xchg是一个宏：

```
#define os_atomic_xchg(p, v, m) \
		atomic_exchange_explicit(_os_atomic_c11_atomic(p), v, memory_order_##m)
```

这里主要是atomic_exchange_explicit的作用：

```c
C atomic_exchange_explicit（volatile A * obj，C desired，memory_order order）;
```

原子替换操作，使用desired替换 obj 指向的值。返回之前obj 指向的值。

因此`_dispatch_once_mark_done`的作用就是更新onceToken 的值为DLOCK_ONCE_DONE。

然后根据onceToken之前的值与value_self比较查看是否有等待的线程（这里是一个小优化），有的话就继续_dispatch_gate_broadcast_slow的执行，唤醒等待的线程。

这里有一个细节onceToken的值在第一次 tryenter 的时候会被更改为`_dispatch_lock_value_for_self()`，后续如果其他线程执行到tryenter函数的话只会返回 false 而 onceToken的值是不会改变的，这说明onceToken的值还会在其他地方被改变，要不然会执行`if (likely((dispatch_lock)v == value_self)) return;`表明没有需要被唤醒的线程。究竟是在哪里被改变的呢？猜测只能是在_dispatch_once_wait函数了。

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

os_atomic_rmw_loop宏展开：这个宏简直丧心病狂。

```
({ _Bool _result = 0; __typeof__(&dgo->dgo_once) _p = (&dgo->dgo_once); old_v = atomic_load_explicit(((__typeof__(*(_p)) _Atomic *)(_p)), memory_order_relaxed); do { { if (likely(old_v == DLOCK_ONCE_DONE)) { ({ os_atomic_thread_fence(relaxed); return; __builtin_unreachable(); }); } new_v = old_v | (uintptr_t)DLOCK_WAITERS_BIT; if (new_v == old_v) ({ os_atomic_thread_fence(relaxed); break; __builtin_unreachable(); }); }; _result = ({ __typeof__(atomic_load_explicit(((__typeof__(*(_p)) _Atomic *)(_p)), memory_order_relaxed)) _r = (old_v); _Bool _b = atomic_compare_exchange_weak_explicit(((__typeof__(*(_p)) _Atomic *)(_p)), &_r, new_v, memory_order_relaxed, memory_order_relaxed); *(&old_v) = _r; _b; }); } while (unlikely(!_result)); _result; });
```

格式化：

```c
({
        _bool _result = 0;
        __typeof__(&dgo->dgo_once) _p = (&dgo->dgo_once);
        old_v = atomic_load_explicit(((__typeof__(*(_p)) _Atomic *)(_p)), memory_order_relaxed);
        do {
            {
                if (likely(old_v == DLOCK_ONCE_DONE))
                {
                    ({
                        os_atomic_thread_fence(relaxed);
                        return;
                        __builtin_unreachable();
                    });
                }
                new_v = old_v | (uintptr_t)DLOCK_WAITERS_BIT;
                if (new_v == old_v)
                ({
                    os_atomic_thread_fence(relaxed);
                    break;
                    __builtin_unreachable();
                });
            };
            _result = ({
                __typeof__(atomic_load_explicit(((__typeof__(*(_p)) _Atomic *)(_p)), memory_order_relaxed)) _r = (old_v);
                _bool _b = atomic_compare_exchange_weak_explicit(((__typeof__(*(_p)) _Atomic *)(_p)), &_r, new_v, memory_order_relaxed, memory_order_relaxed);
                *(&old_v) = _r;
                _b;
            });
    } while (unlikely(!_result));
    _result;
});
```

可以看到里面是有改变dgo_once（onceToken）的值的。表明有线程在等待 block 的执行。

退出os_atomic_rmw_loop循环后，接下来会判断

```c
if (unlikely(_dispatch_lock_is_locked_by((dispatch_lock)old_v, self))) {
  DISPATCH_CLIENT_CRASH(0, "trying to lock recursively");
}
```

这里如果是嵌套递归调用的话，这里条件就会满足于是抛出异常。

接下来就是调用 wait 方法进入等待。

到此为止 dispatch_once 的实现就分析完了。其实旧版本的 dispatch_once 源码挺简单的，越到后面代码越复杂特别是那些宏，后期的版本主要是代码优化及性能优化。如果你只想看一下大概的实现原理，那么看比较原始的版本就可以了，如果你想看一下代码的演化历史和一些性能优化那么就需要看新版本了。

#### 总结



### Question

Q1：`dispatch_once`如何保证在多线程环境下block也只执行一次？

Q2：static整型变量onceToken的作用是什么？能不能使用一个auto整型变量代替？

Q3：`dispatch_once`死锁崩溃？

`dispatch_once`嵌套调用发生崩溃：

<img src="https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20200629152738.png" style="zoom:50%;" />

进一步跟进断点：

<img src="https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20200629162804.png" style="zoom:50%;" />

发现是崩溃在libdispatch的_dispatch_once_wait函数里面。其实已经提示的很清楚了：尝试递归的加锁，而这正是因为我们嵌套的调用导致的。

### 附录

#### dispatch_lock类型

```c
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

其实就是一个无符号32 位整型。

#### DISPATCH_ONCE_USE_QUIESCENT_COUNTER 宏

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

#### os_atomic(type)宏

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

#### likely(x)

likely(x)的作用就是告诉编译器需要优化生成的汇编指令，likely(v == DLOCK_ONCE_DONE)表示v == DLOCK_ONCE_DONE极有可能发生，在生成汇编指令时注意优化。阅读代码时当做普通条件判断就可以了。

在一个条件判断语句中，当这个条件被认为是非常非常有可能满足时，则使用likely()宏，否则，条件非常非常不可能或很难满足时，则使用unlikely()宏。可以优化性能。



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

