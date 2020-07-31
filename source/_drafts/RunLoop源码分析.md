版本 `CF-1151.16`

### RunLoop

1、RunLoop和线程之间的关系

（1）每条线程都有唯一的一个与之对应的RunLoop对象。

（2）主线程的RunLoop默认已经开启，子线程的RunLoop需要主动开启。

（3）RunLoop在第一次获取时创建，在线程结束时销毁。

线程和 RunLoop 之间是一一对应的，其关系是保存在一个全局的 Dictionary 里。线程刚创建时并没有 RunLoop，如果你不主动获取，那它一直都不会有。**RunLoop 的创建是发生在第一次获取时，RunLoop 的销毁是发生在线程结束时**。你只能在一个线程的内部获取其 RunLoop（主线程除外）.

#### RunLoop的伪代码实现

```
int runloop() {
    
    // 即将进入runloop
    doObserversRunLoopEntry();
    
    int retVal = 0;
    do {
        // 处理定时器
        
        // 处理输入源
        
        // 通知外部runloop即将休眠
        
        // 休眠
        
        // 唤醒. (因为某种输入源被唤醒)
        
        // 通知外部runloop已经被唤醒
        
        // 处理输入源
        
        // 某种情况下retVal被赋值为其他值.
    } while (retVal == 0);
    
    // 已经退出runloop
    doObserversRunLoopExit();
    
    return retVal;
}

void runloopRun() {
    int result = 0;
    do {
        result = runloop(); //跑圈
    } while(result != 1 && result != 2);
}
```

runloop相关的类型：

#### CFRunLoopActivity（RunLoop事件枚举）

```c
/* Run Loop Observer Activities */
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0),
    kCFRunLoopBeforeTimers = (1UL << 1),
    kCFRunLoopBeforeSources = (1UL << 2),
    kCFRunLoopBeforeWaiting = (1UL << 5),
    kCFRunLoopAfterWaiting = (1UL << 6),
    kCFRunLoopExit = (1UL << 7),
    kCFRunLoopAllActivities = 0x0FFFFFFFU
};
```

#### CFRunLoopRunResult（RunLoop退出原因枚举）

```c
/* Reasons for CFRunLoopRunInMode() to Return */
typedef CF_ENUM(SInt32, CFRunLoopRunResult) {
    kCFRunLoopRunFinished = 1,
    kCFRunLoopRunStopped = 2,
    kCFRunLoopRunTimedOut = 3,
    kCFRunLoopRunHandledSource = 4
};
```

#### struct __CFRunLoop

```c
typedef struct CF_BRIDGED_MUTABLE_TYPE(id) __CFRunLoop * CFRunLoopRef;

struct __CFRunLoop {
    CFRuntimeBase _base; //基类
    pthread_mutex_t _lock;			/* locked for accessing mode list */
    __CFPort _wakeUpPort;			// used for CFRunLoopWakeUp 
    Boolean _unused;
    volatile _per_run_data *_perRunData;              // reset for runs of the run loop
    pthread_t _pthread; //所属线程
    uint32_t _winthread;
    CFMutableSetRef _commonModes; //common modes集合
    CFMutableSetRef _commonModeItems;
    CFRunLoopModeRef _currentMode; //当前mode
    CFMutableSetRef _modes; //mode集合，说明可以有多个mode
    struct _block_item *_blocks_head; //这个链表用于保存提交到runloop上的block
    struct _block_item *_blocks_tail;
    CFAbsoluteTime _runTime;
    CFAbsoluteTime _sleepTime;
    CFTypeRef _counterpart;
};

struct _block_item {
    struct _block_item *_next;
    CFTypeRef _mode;	// CFString or CFSet
    void (^_block)(void); //保存我们的block
};
```

#### struct __CFRunLoopMode

```c
typedef struct __CFRunLoopMode *CFRunLoopModeRef;

struct __CFRunLoopMode {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;	/* must have the run loop locked before locking this */
    CFStringRef _name; //模式名称
    Boolean _stopped;
    char _padding[3];
    CFMutableSetRef _sources0;  ////输入源0集合
    CFMutableSetRef _sources1;  //输入源1集合
    CFMutableArrayRef _observers;  //观察者数组
    CFMutableArrayRef _timers;  //定时器数组
    CFMutableDictionaryRef _portToV1SourceMap;
    __CFPortSet _portSet; //端口集合
    CFIndex _observerMask;
#if USE_MK_TIMER_TOO
    mach_port_t _timerPort; //定时器端口
    Boolean _mkTimerArmed;
#endif

    uint64_t _timerSoftDeadline; /* TSR */
    uint64_t _timerHardDeadline; /* TSR */
};
```

#### struct __CFRunLoopSource

```c
typedef struct __CFRunLoopSource * CFRunLoopSourceRef;

struct __CFRunLoopSource {
    CFRuntimeBase _base;
    uint32_t _bits;
    pthread_mutex_t _lock;
    CFIndex _order;			/* immutable */
    CFMutableBagRef _runLoops;
    union {
				CFRunLoopSourceContext version0;	/* immutable, except invalidation */
        CFRunLoopSourceContext1 version1;	/* immutable, except invalidation */
    } _context;
};

typedef struct {
    CFIndex	version;
    void *	info;
    const void *(*retain)(const void *info);
    void	(*release)(const void *info);
    CFStringRef	(*copyDescription)(const void *info);
    Boolean	(*equal)(const void *info1, const void *info2);
    CFHashCode	(*hash)(const void *info);
    void	(*schedule)(void *info, CFRunLoopRef rl, CFStringRef mode);
    void	(*cancel)(void *info, CFRunLoopRef rl, CFStringRef mode);
    void	(*perform)(void *info);
} CFRunLoopSourceContext;

typedef struct {
    CFIndex	version;
    void *	info;
    const void *(*retain)(const void *info);
    void	(*release)(const void *info);
    CFStringRef	(*copyDescription)(const void *info);
    Boolean	(*equal)(const void *info1, const void *info2);
    CFHashCode	(*hash)(const void *info);
#if (TARGET_OS_MAC && !(TARGET_OS_EMBEDDED || TARGET_OS_IPHONE)) || (TARGET_OS_EMBEDDED || TARGET_OS_IPHONE)
    mach_port_t	(*getPort)(void *info);
    void *	(*perform)(void *msg, CFIndex size, CFAllocatorRef allocator, void *info);
#else
    void *	(*getPort)(void *info);
    void	(*perform)(void *info);
#endif
} CFRunLoopSourceContext1;
```

#### struct __CFRunLoopObserver

```c
typedef struct __CFRunLoopObserver * CFRunLoopObserverRef;

struct __CFRunLoopObserver {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;
    CFRunLoopRef _runLoop;
    CFIndex _rlCount;
    CFOptionFlags _activities;		/* immutable */
    CFIndex _order;			/* immutable */
    CFRunLoopObserverCallBack _callout;	/* immutable */
    CFRunLoopObserverContext _context;	/* immutable, except invalidation */
};
```

#### struct __CFRunLoopTimer

```c
typedef struct CF_BRIDGED_MUTABLE_TYPE(NSTimer) __CFRunLoopTimer * CFRunLoopTimerRef;

struct __CFRunLoopTimer {
    CFRuntimeBase _base;
    uint16_t _bits;
    pthread_mutex_t _lock;
    CFRunLoopRef _runLoop; //所在的runloop
    CFMutableSetRef _rlModes; //所属mode
    CFAbsoluteTime _nextFireDate;
    CFTimeInterval _interval;		/* immutable */  //定时器间隔时间
    CFTimeInterval _tolerance;          /* mutable */  //定时器误差
    uint64_t _fireTSR;			/* TSR units */
    CFIndex _order;			/* immutable */
    CFRunLoopTimerCallBack _callout;	/* immutable */ //定时器的回调
    CFRunLoopTimerContext _context;	/* immutable, except invalidation */
};
```



### CFRunLoopRun

实现：

```c
void CFRunLoopRun(void) {	/* DOES CALLOUT */
    int32_t result;
    do {
        result = CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);
        CHECK_FOR_FORK();
    } while (kCFRunLoopRunStopped != result && kCFRunLoopRunFinished != result);
}
```

外层是一个do-while循环用于驱动runloop退出时重新进入，只有当result为Stopped或Finished时才真正退出runloop。

循环体内调用了CFRunLoopRunSpecific，它有四个参数，分别为当前runloop、模式名称（字符串）、超时时间，处理源后是否退出runloop。

可以看到CFRunLoopRun的超时时间非常久基本上不会因为时间到了而退出，另外处理源后不退出runloop。

先来看一下是如何获取一个runloop的：

#### CFRunLoopGetCurrent

实现：

```c
CFRunLoopRef CFRunLoopGetCurrent(void) {
    CHECK_FOR_FORK();
    CFRunLoopRef rl = (CFRunLoopRef)_CFGetTSD(__CFTSDKeyRunLoop); //__CFTSDKeyRunLoop=10
    if (rl) return rl;
    return _CFRunLoopGet0(pthread_self());
}
```

大致逻辑：

1. 从线程的TSD中获取之前存的rl，如果有则直接返回。
2. 如果没有，则为该线程创建一个rl。

看一下具体的创建过程：

#### _CFRunLoopGet0

实现：

```c
static CFMutableDictionaryRef __CFRunLoops = NULL;

// should only be called by Foundation
// t==0 is a synonym for "main thread" that always works
CF_EXPORT CFRunLoopRef _CFRunLoopGet0(pthread_t t) {
    if (pthread_equal(t, kNilPthreadT)) {
        t = pthread_main_thread_np();
    }
    __CFLock(&loopsLock);
    if (!__CFRunLoops) {
        __CFUnlock(&loopsLock);
        CFMutableDictionaryRef dict = CFDictionaryCreateMutable(kCFAllocatorSystemDefault, 0, NULL, &kCFTypeDictionaryValueCallBacks);
        CFRunLoopRef mainLoop = __CFRunLoopCreate(pthread_main_thread_np());
        CFDictionarySetValue(dict, pthreadPointer(pthread_main_thread_np()), mainLoop);
        if (!OSAtomicCompareAndSwapPtrBarrier(NULL, dict, (void * volatile *)&__CFRunLoops)) {
            CFRelease(dict);
        }
        CFRelease(mainLoop);
        __CFLock(&loopsLock);
    }
    CFRunLoopRef loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
    __CFUnlock(&loopsLock);
    if (!loop) {
        CFRunLoopRef newLoop = __CFRunLoopCreate(t);
        __CFLock(&loopsLock);
        loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
        if (!loop) {
            CFDictionarySetValue(__CFRunLoops, pthreadPointer(t), newLoop);
            loop = newLoop;
        }
        // don't release run loops inside the loopsLock, because CFRunLoopDeallocate may end up taking it
        __CFUnlock(&loopsLock);
        CFRelease(newLoop);
    }
    if (pthread_equal(t, pthread_self())) {
        _CFSetTSD(__CFTSDKeyRunLoop, (void *)loop, NULL);
        if (0 == _CFGetTSD(__CFTSDKeyRunLoopCntr)) {
            _CFSetTSD(__CFTSDKeyRunLoopCntr, (void *)(PTHREAD_DESTRUCTOR_ITERATIONS-1), (void (*)(void *))__CFFinalizeRunLoop); //清除TSD时调用__CFFinalizeRunLoop销毁runloop
        }
    }
    return loop;
}
```

大致过程：

1. 懒加载一个全局的字典，该字典用于保存所有的runloop对象。将mainLoop添加到字典
2. 先从runloop字典中尝试获取线程的runloop对象
3. 如果没获取到则调用 `__CFRunLoopCreate` 新创建一个runloop并保存到runloop字典中
4. 得到runloop后设置到线程的TSD中，下次就直接从线程的TSD中取得了，加快速度。当线程销毁时会清除TSD，清除TSD就会调用__CFFinalizeRunLoop销毁runloop

终于到创建runloop过程了：

#### __CFRunLoopCreate

```c
static CFRunLoopRef __CFRunLoopCreate(pthread_t t) {
    CFRunLoopRef loop = NULL;
    CFRunLoopModeRef rlm;
    uint32_t size = sizeof(struct __CFRunLoop) - sizeof(CFRuntimeBase);
    loop = (CFRunLoopRef)_CFRuntimeCreateInstance(kCFAllocatorSystemDefault, CFRunLoopGetTypeID(), size, NULL); //分配内存
    if (NULL == loop) {
				return NULL;
    }
    (void)__CFRunLoopPushPerRunData(loop);
    __CFRunLoopLockInit(&loop->_lock); //初始化loop的锁
    loop->_wakeUpPort = __CFPortAllocate();
    if (CFPORT_NULL == loop->_wakeUpPort) HALT;
    __CFRunLoopSetIgnoreWakeUps(loop);
    loop->_commonModes = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
    CFSetAddValue(loop->_commonModes, kCFRunLoopDefaultMode); //给_commonModes集合中添加了一个DefaultMode
    loop->_commonModeItems = NULL;
    loop->_currentMode = NULL;
    loop->_modes = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
    loop->_blocks_head = NULL;
    loop->_blocks_tail = NULL;
    loop->_counterpart = NULL;
    loop->_pthread = t; //关联到该线程上
#if DEPLOYMENT_TARGET_WINDOWS
    loop->_winthread = GetCurrentThreadId();
#else
    loop->_winthread = 0;
#endif
    rlm = __CFRunLoopFindMode(loop, kCFRunLoopDefaultMode, true);
    if (NULL != rlm) __CFRunLoopModeUnlock(rlm);
    return loop;
}
```

大致逻辑：

1. 分配内存
2. 初始化loop，主要是给runloop的属性设置一些默认值。

跟我们平时创建对象一样，先计算实例的大小，再在堆上申请那么多的内存，再初始化实例的属性。

#### __CFFinalizeRunLoop

销毁runloop实现：

```c
// Called for each thread as it exits
CF_PRIVATE void __CFFinalizeRunLoop(uintptr_t data) {
    CFRunLoopRef rl = NULL;
    if (data <= 1) {
        __CFLock(&loopsLock);
        if (__CFRunLoops) {
            rl = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(pthread_self()));
            if (rl) CFRetain(rl);
            CFDictionaryRemoveValue(__CFRunLoops, pthreadPointer(pthread_self()));
        }
        __CFUnlock(&loopsLock);
    } else {
        _CFSetTSD(__CFTSDKeyRunLoopCntr, (void *)(data - 1), (void (*)(void *))__CFFinalizeRunLoop);
    }
    if (rl && CFRunLoopGetMain() != rl) { // protect against cooperative threads
        if (NULL != rl->_counterpart) {
            CFRelease(rl->_counterpart);
            rl->_counterpart = NULL;
        }
        // purge all sources before deallocation
        CFArrayRef array = CFRunLoopCopyAllModes(rl);
        for (CFIndex idx = CFArrayGetCount(array); idx--;) {
            CFStringRef modeName = (CFStringRef)CFArrayGetValueAtIndex(array, idx);
            __CFRunLoopRemoveAllSources(rl, modeName);
        }
        __CFRunLoopRemoveAllSources(rl, kCFRunLoopCommonModes);
        CFRelease(array);
    }
    if (rl) CFRelease(rl);
}
```

大致流程：

1. 从全局字典中移除
2. 移除所有mode的所有source
3. 释放runloop

#### CFRunLoopRunSpecific

实现：

```c
SInt32 CFRunLoopRunSpecific(CFRunLoopRef rl, CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {     /* DOES CALLOUT */
    CHECK_FOR_FORK();
    if (__CFRunLoopIsDeallocating(rl)) return kCFRunLoopRunFinished;
    __CFRunLoopLock(rl);
    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(rl, modeName, false);
    if (NULL == currentMode || __CFRunLoopModeIsEmpty(rl, currentMode, rl->_currentMode)) {
        Boolean did = false;
        if (currentMode) __CFRunLoopModeUnlock(currentMode);
        __CFRunLoopUnlock(rl);
        return did ? kCFRunLoopRunHandledSource : kCFRunLoopRunFinished;
    }
    volatile _per_run_data *previousPerRun = __CFRunLoopPushPerRunData(rl);
    CFRunLoopModeRef previousMode = rl->_currentMode;
    rl->_currentMode = currentMode;
    int32_t result = kCFRunLoopRunFinished;
    
    if (currentMode->_observerMask & kCFRunLoopEntry ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
    result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);
    if (currentMode->_observerMask & kCFRunLoopExit ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);
    
    __CFRunLoopModeUnlock(currentMode);
    __CFRunLoopPopPerRunData(rl, previousPerRun);
    rl->_currentMode = previousMode;
    __CFRunLoopUnlock(rl);
    return result;
}
```

大致逻辑：

1. 根据modeName从runloop的mode集合中查找到currentMode对象，可能找不到
2. 判断currentMode是否等于NULL或者currentMode是一个空mode（没有任何输入源，定时器等），如果为真则返回kCFRunLoopRunFinished。因此在启动runloop前需要保证我们的mode是有源的。
3. 将runloop的_currentMode替换为currentMode
4. 通知观察者runloop即将以当前mode进入
5. 调用 `__CFRunLoopRun` 正式”跑圈“，`__CFRunLoopRun` 在某些情况下会退出跑圈并返回退出原因result。
6. 通知观察者runloop已经退出。
7. 又将runloop的_currentMode替换为之前的previousMode。
8. 返回runloop退出原因result。

核心实现是 `__CFRunLoopRun` 该函数很长因为要处理很多事件。

#### __CFRunLoopRun

实现：

```c
/* rl, rlm are locked on entrance and exit */
static int32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode) {
    uint64_t startTSR = mach_absolute_time();

    if (__CFRunLoopIsStopped(rl)) {
        __CFRunLoopUnsetStopped(rl);
        return kCFRunLoopRunStopped;
    } else if (rlm->_stopped) {
        rlm->_stopped = false;
        return kCFRunLoopRunStopped;
    }
    
    mach_port_name_t dispatchPort = MACH_PORT_NULL;
    Boolean libdispatchQSafe = pthread_main_np() && ((HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && NULL == previousMode) || (!HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && 0 == _CFGetTSD(__CFTSDKeyIsInGCDMainQ)));
    if (libdispatchQSafe && (CFRunLoopGetMain() == rl) && CFSetContainsValue(rl->_commonModes, rlm->_name)) dispatchPort = _dispatch_get_main_queue_port_4CF();
    
    
    dispatch_source_t timeout_timer = NULL;
    struct __timeout_context *timeout_context = (struct __timeout_context *)malloc(sizeof(*timeout_context));
    if (seconds <= 0.0) { // instant timeout
        seconds = 0.0;
        timeout_context->termTSR = 0ULL;
    } else if (seconds <= TIMER_INTERVAL_LIMIT) { //指定了有效超时，就初始化一个gcd timeout_timer,这样就不受制于runloop 
        dispatch_queue_t queue = pthread_main_np() ? __CFDispatchQueueGetGenericMatchingMain() : __CFDispatchQueueGetGenericBackground();
        timeout_timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
        dispatch_retain(timeout_timer);
        timeout_context->ds = timeout_timer;
        timeout_context->rl = (CFRunLoopRef)CFRetain(rl);
        timeout_context->termTSR = startTSR + __CFTimeIntervalToTSR(seconds);
        dispatch_set_context(timeout_timer, timeout_context); // source gets ownership of context
        dispatch_source_set_event_handler_f(timeout_timer, __CFRunLoopTimeout);
        dispatch_source_set_cancel_handler_f(timeout_timer, __CFRunLoopTimeoutCancel);
        uint64_t ns_at = (uint64_t)((__CFTSRToTimeInterval(startTSR) + seconds) * 1000000000ULL);
        dispatch_source_set_timer(timeout_timer, dispatch_time(1, ns_at), DISPATCH_TIME_FOREVER, 1000ULL);
        dispatch_resume(timeout_timer);
    } else { // infinite timeout
        seconds = 9999999999.0;
        timeout_context->termTSR = UINT64_MAX;
    }

    Boolean didDispatchPortLastTime = true;
    int32_t retVal = 0;
    do {
			voucher_mach_msg_state_t voucherState = VOUCHER_MACH_MSG_STATE_UNCHANGED;
      voucher_t voucherCopy = NULL;
      uint8_t msg_buffer[3 * 1024];
			mach_msg_header_t *msg = NULL;
      mach_port_t livePort = MACH_PORT_NULL;
      
      __CFPortSet waitSet = rlm->_portSet;

      __CFRunLoopUnsetIgnoreWakeUps(rl);

			//通知即将处理timers
      if (rlm->_observerMask & kCFRunLoopBeforeTimers) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);
      //通知即将处理Sources
      if (rlm->_observerMask & kCFRunLoopBeforeSources) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);

      __CFRunLoopDoBlocks(rl, rlm); //执行block
			
      //处理Sources0
      Boolean sourceHandledThisLoop = __CFRunLoopDoSources0(rl, rlm, stopAfterHandle);
      if (sourceHandledThisLoop) {
          __CFRunLoopDoBlocks(rl, rlm);
      }

      Boolean poll = sourceHandledThisLoop || (0ULL == timeout_context->termTSR);

      if (MACH_PORT_NULL != dispatchPort && !didDispatchPortLastTime) {
					msg = (mach_msg_header_t *)msg_buffer;
          if (__CFRunLoopServiceMachPort(dispatchPort, &msg, sizeof(msg_buffer), &livePort, 0, &voucherState, NULL)) {
              goto handle_msg;
          }
      }

      didDispatchPortLastTime = false;

      if (!poll && (rlm->_observerMask & kCFRunLoopBeforeWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting); //通知即将休眠
      __CFRunLoopSetSleeping(rl); //设置休眠标志位
      // do not do any user callouts after this point (after notifying of sleeping)

      // Must push the local-to-this-activation ports in on every loop
      // iteration, as this mode could be run re-entrantly and we don't
      // want these ports to get serviced.

      __CFPortSetInsert(dispatchPort, waitSet);

      __CFRunLoopModeUnlock(rlm);
      __CFRunLoopUnlock(rl);

      CFAbsoluteTime sleepStart = poll ? 0.0 : CFAbsoluteTimeGetCurrent();

      if (kCFUseCollectableAllocator) {
          // objc_clear_stack(0);
          // <rdar://problem/16393959>
          memset(msg_buffer, 0, sizeof(msg_buffer));
       }
      msg = (mach_msg_header_t *)msg_buffer;
      __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY, &voucherState, &voucherCopy); //进入休眠，poll为真其实不会休眠
      
      __CFRunLoopLock(rl);
      __CFRunLoopModeLock(rlm);

      rl->_sleepTime += (poll ? 0.0 : (CFAbsoluteTimeGetCurrent() - sleepStart));

      // Must remove the local-to-this-activation ports in on every loop
      // iteration, as this mode could be run re-entrantly and we don't
      // want these ports to get serviced. Also, we don't want them left
      // in there if this function returns.

      __CFPortSetRemove(dispatchPort, waitSet);

      __CFRunLoopSetIgnoreWakeUps(rl);

      // user callouts now OK again
      __CFRunLoopUnsetSleeping(rl);
      // 通知已经唤醒
      if (!poll && (rlm->_observerMask & kCFRunLoopAfterWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopAfterWaiting);

      handle_msg:;
      __CFRunLoopSetIgnoreWakeUps(rl);


      if (MACH_PORT_NULL == livePort) {
          CFRUNLOOP_WAKEUP_FOR_NOTHING();
          // handle nothing
      } else if (livePort == rl->_wakeUpPort) { //因为_wakeUpPort唤醒
          CFRUNLOOP_WAKEUP_FOR_WAKEUP();
          // do nothing on Mac OS
      }
#if USE_MK_TIMER_TOO
      else if (rlm->_timerPort != MACH_PORT_NULL && livePort == rlm->_timerPort) { //因为_timerPort唤醒
          CFRUNLOOP_WAKEUP_FOR_TIMER();
          // On Windows, we have observed an issue where the timer port is set before the time which we requested it to be set. For example, we set the fire time to be TSR 167646765860, but it is actually observed firing at TSR 167646764145, which is 1715 ticks early. The result is that, when __CFRunLoopDoTimers checks to see if any of the run loop timers should be firing, it appears to be 'too early' for the next timer, and no timers are handled.
          // In this case, the timer port has been automatically reset (since it was returned from MsgWaitForMultipleObjectsEx), and if we do not re-arm it, then no timers will ever be serviced again unless something adjusts the timer list (e.g. adding or removing timers). The fix for the issue is to reset the timer here if CFRunLoopDoTimers did not handle a timer itself. 9308754
          if (!__CFRunLoopDoTimers(rl, rlm, mach_absolute_time())) { //处理定时器
              // Re-arm the next timer
              __CFArmNextTimerInMode(rlm, rl);
          }
      }
#endif
      else if (livePort == dispatchPort) { //因为dispatchPort唤醒
          CFRUNLOOP_WAKEUP_FOR_DISPATCH();
          __CFRunLoopModeUnlock(rlm);
          __CFRunLoopUnlock(rl);
          _CFSetTSD(__CFTSDKeyIsInGCDMainQ, (void *)6, NULL);

          __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
          _CFSetTSD(__CFTSDKeyIsInGCDMainQ, (void *)0, NULL);
          __CFRunLoopLock(rl);
          __CFRunLoopModeLock(rlm);
          sourceHandledThisLoop = true;
          didDispatchPortLastTime = true;
      } else {
          CFRUNLOOP_WAKEUP_FOR_SOURCE();

          // If we received a voucher from this mach_msg, then put a copy of the new voucher into TSD. CFMachPortBoost will look in the TSD for the voucher. By using the value in the TSD we tie the CFMachPortBoost to this received mach_msg explicitly without a chance for anything in between the two pieces of code to set the voucher again.
          voucher_t previousVoucher = _CFSetTSD(__CFTSDKeyMachMessageHasVoucher, (void *)voucherCopy, os_release);

          // Despite the name, this works for windows handles as well
          CFRunLoopSourceRef rls = __CFRunLoopModeFindSourceForMachPort(rl, rlm, livePort);
          if (rls) {
							mach_msg_header_t *reply = NULL;
							sourceHandledThisLoop = __CFRunLoopDoSource1(rl, rlm, rls, msg, msg->msgh_size, &reply) || sourceHandledThisLoop;  //处理source1
							if (NULL != reply) {
		    					(void)mach_msg(reply, MACH_SEND_MSG, reply->msgh_size, 0, MACH_PORT_NULL, 0, MACH_PORT_NULL);
		    					CFAllocatorDeallocate(kCFAllocatorSystemDefault, reply);
							}
          }

          // Restore the previous voucher
          _CFSetTSD(__CFTSDKeyMachMessageHasVoucher, previousVoucher, os_release);

      }

      __CFRunLoopDoBlocks(rl, rlm);


      if (sourceHandledThisLoop && stopAfterHandle) { //判断是否处理了source
          retVal = kCFRunLoopRunHandledSource;
      } else if (timeout_context->termTSR < mach_absolute_time()) { //判断是否超时
          retVal = kCFRunLoopRunTimedOut;
      } else if (__CFRunLoopIsStopped(rl)) { //判断是否调用了CFRunLoopStop
          __CFRunLoopUnsetStopped(rl);
          retVal = kCFRunLoopRunStopped;
      } else if (rlm->_stopped) {
          rlm->_stopped = false;
          retVal = kCFRunLoopRunStopped;
      } else if (__CFRunLoopModeIsEmpty(rl, rlm, previousMode)) {//判断是否mode变成空的了
          retVal = kCFRunLoopRunFinished;
      }

    } while (0 == retVal);

    if (timeout_timer) {
        dispatch_source_cancel(timeout_timer);
        dispatch_release(timeout_timer);
    } else {
        free(timeout_context);
    }

    return retVal;
}
```

 `__CFRunLoopRun` 内部也有一个do-while循环

#### RunLoopMode

runloop和mode

![](https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/runloop%E4%B8%8Emode.png)

一个 RunLoop 可以包含若干个 Mode，每个 Mode 又包含若干个 Source/Timer/Observer。每次调用 RunLoop 的主函数时，只能指定其中一个 Mode，这个Mode被称作 CurrentMode。如果需要切换 Mode，只能退出 Loop，再重新指定一个 Mode 进入。

**为什么要有这么多mode?**

个人感觉应该是为了有个事件处理分类或者有个事件处理的优先级.比如ScrollView在滚动时,肯定处理追踪滚动的优先级要高,如果这个时候还去处理一些定时器事件,可能会导致追踪滚动处理不过来.

**如何手动切换mode?**

指定mode调用CFRunLoopRunInMode就可以了。

系统默认注册了5个Mode，其中常见的有1.2两种：

1. kCFRunLoopDefaultMode：App的默认Mode，通常主线程是在这个Mode下运行
2. UITrackingRunLoopMode：界面跟踪 Mode，用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他 Mode 影响
3. UIInitializationRunLoopMode: 在刚启动 App 时进入的第一个 Mode，启动完成后就不再使用
4. GSEventReceiveRunLoopMode: 接受系统事件的内部 Mode，通常用不到
5. kCFRunLoopCommonModes: 这是一个占位用的Mode，作为标记kCFRunLoopDefaultMode和UITrackingRunLoopMode用，并不是一种真正的Mode

上面的 Source/Timer/Observer 被统称为 mode item，一个 item 可以被同时加入多个 mode。但一个 item 被重复加入同一个 mode 时是不会有效果的。如果一个 mode 中一个 item 都没有，则 RunLoop 会直接退出，不进入循环。

#### RunLoop的输入源,定时器,观察者


#### RunLoop的生命周期

![](https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/runloop%E5%86%85%E9%83%A8%E9%80%BB%E8%BE%91.png)


#### 使用RunLoop进行卡顿检测
NSRunLoop调用方法主要就是在kCFRunLoopBeforeSources和kCFRunLoopBeforeWaiting之间,还有kCFRunLoopAfterWaiting之后,也就是如果我们发现这两个时间内耗时太长,那么就可以判定出此时主线程卡顿.

再加上信号量等待超时特性.

[iOS实时卡顿检测-RunLoop(附实例)](https://www.jianshu.com/p/d0aab0eb8ce4)



### 参考

[CF源码](https://opensource.apple.com/tarballs/CF/)

[iOS RunLoop 详解](https://imlifengfeng.github.io/article/487/) 作者有点东西.

[深入理解RunLoop](https://blog.ibireme.com/2015/05/18/runloop/)    ibireme写的

