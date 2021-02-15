---
title: NSOperation和GCD的区别
tags:
  - 标签1
  - 标签2
  - 标签3
categories:
  - 分类1
  - 分类2
comments: false
date: 2021-02-14 20:45:38
---


### NSOperation和GCD的区别

1.GCD底层使用C语言编写高效，而NSOperation是对GCD的面向对象的封装。

2.NSOperation可以方便的控制最大并发数，而GCD的线程是由系统管理的对上层是屏蔽的。

3.NSOperation可以方便的设置依赖关系，而GCD只能通过dispatch_barrier_async实现

4.NSOperation可以设置操作自身的优先级（通过queuePriority），而GCD只能设置队列优先级。

5.NSOperation可以通过KVO观察当前operation执行状态(就绪/执行/取消/完成)

6.NSOperation可以自定义operation，有更高的代码复用度。



### NSOperation取消任务

NSOperation的方法：

```objc
- (void)cancel;
```

调用cancel只是标记操作对象内部的状态为isCancelled。

而对于一个在正在运行中的操作，即使给这个操作发送了cancel消息，系统并不会真的终止运行，但我们可以查询操作的cancelled状态,然后手动取消(大多是return)



### GCD取消任务

挂起/恢复队列：

```c
API_AVAILABLE(macos(10.6), ios(4.0))
DISPATCH_EXPORT DISPATCH_NONNULL_ALL DISPATCH_NOTHROW
void
dispatch_suspend(dispatch_object_t object);

API_AVAILABLE(macos(10.6), ios(4.0))
DISPATCH_EXPORT DISPATCH_NONNULL_ALL DISPATCH_NOTHROW
void
dispatch_resume(dispatch_object_t object);
```

dispatch_suspend的次数需要和dispatch_resume的次数平衡。

dispatch_suspend对已经执行的处理没有影响，挂起后，追加到队列中但尚未执行的处理在此之后停止执行。调用dispatch_resume则恢复执行。

dispatch_cancel：

```c
DISPATCH_UNAVAILABLE
DISPATCH_EXPORT DISPATCH_NONNULL_ALL DISPATCH_NOTHROW
void
dispatch_cancel(void *object);
#if __has_extension(c_generic_selections)
#define dispatch_cancel(object) \
		_Generic((object), \
			dispatch_block_t:dispatch_block_cancel, \
			dispatch_source_t:dispatch_source_cancel \
		)((object))
#endif
```

dispatch_cancel函数取消指定的对象。类型泛型宏，根据第一个参数的类型映射到dispatch_block_cancel或dispatch_source_cancel。此函数对于任何其他对象类型都不可用。

同样，对于正在执行的处理没有影响，只对那些尚未执行才有效。

示例：

```
dispatch_cancel(self.timer);
```



### 自定义NSOperation

这里主要讨论自定义并发的NSOperation。可以参考SDWebImage的SDWebImageDownloaderOperation的实现。

对于并发的NSOperation有一些方法是需要重写的：

#### start

```
- (void)start;
```

start方法是必须重写的，并且不能调用super的实现。

eg:

```objc
- (void)start {
    @synchronized (self) {
        if (self.isCancelled) {
            if (!self.isFinished) self.finished = YES;
            // Operation cancelled by user before sending the request
            [self callCompletionBlocksWithError:[NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorCancelled userInfo:@{NSLocalizedDescriptionKey : @"Operation cancelled by user before sending the request"}]];
            [self reset];
            return;
        }

#if SD_UIKIT
        Class UIApplicationClass = NSClassFromString(@"UIApplication");
        BOOL hasApplication = UIApplicationClass && [UIApplicationClass respondsToSelector:@selector(sharedApplication)];
        if (hasApplication && [self shouldContinueWhenAppEntersBackground]) {
            __weak typeof(self) wself = self;
            UIApplication * app = [UIApplicationClass performSelector:@selector(sharedApplication)];
            self.backgroundTaskId = [app beginBackgroundTaskWithExpirationHandler:^{
                [wself cancel];
            }];
        }
#endif
        NSURLSession *session = self.unownedSession;
        if (!session) {
            NSURLSessionConfiguration *sessionConfig = [NSURLSessionConfiguration defaultSessionConfiguration];
            sessionConfig.timeoutIntervalForRequest = 15;
            
            /**
             *  Create the session for this task
             *  We send nil as delegate queue so that the session creates a serial operation queue for performing all delegate
             *  method calls and completion handler calls.
             */
            session = [NSURLSession sessionWithConfiguration:sessionConfig
                                                    delegate:self
                                               delegateQueue:nil];
            self.ownedSession = session;
        }
        
        if (self.options & SDWebImageDownloaderIgnoreCachedResponse) {
            // Grab the cached data for later check
            NSURLCache *URLCache = session.configuration.URLCache;
            if (!URLCache) {
                URLCache = [NSURLCache sharedURLCache];
            }
            NSCachedURLResponse *cachedResponse;
            // NSURLCache's `cachedResponseForRequest:` is not thread-safe, see https://developer.apple.com/documentation/foundation/nsurlcache#2317483
            @synchronized (URLCache) {
                cachedResponse = [URLCache cachedResponseForRequest:self.request];
            }
            if (cachedResponse) {
                self.cachedData = cachedResponse.data;
            }
        }
        
        self.dataTask = [session dataTaskWithRequest:self.request];
        self.executing = YES;
    }

    if (self.dataTask) {
        if (self.options & SDWebImageDownloaderHighPriority) {
            self.dataTask.priority = NSURLSessionTaskPriorityHigh;
            self.coderQueue.qualityOfService = NSQualityOfServiceUserInteractive;
        } else if (self.options & SDWebImageDownloaderLowPriority) {
            self.dataTask.priority = NSURLSessionTaskPriorityLow;
            self.coderQueue.qualityOfService = NSQualityOfServiceBackground;
        } else {
            self.dataTask.priority = NSURLSessionTaskPriorityDefault;
            self.coderQueue.qualityOfService = NSQualityOfServiceDefault;
        }
        [self.dataTask resume];
        for (SDWebImageDownloaderProgressBlock progressBlock in [self callbacksForKey:kProgressCallbackKey]) {
            progressBlock(0, NSURLResponseUnknownLength, self.request.URL);
        }
        __block typeof(self) strongSelf = self;
        dispatch_async(dispatch_get_main_queue(), ^{
            [[NSNotificationCenter defaultCenter] postNotificationName:SDWebImageDownloadStartNotification object:strongSelf];
        });
    } else {
        if (!self.isFinished) self.finished = YES;
        [self callCompletionBlocksWithError:[NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorInvalidDownloadOperation userInfo:@{NSLocalizedDescriptionKey : @"Task can't be initialized"}]];
        [self reset];
    }
}
```

#### main

main方法是可选重写的。

#### 状态方法isFinished，isExecuting，isAsynchronous

这几个状态是必须重写的。

```objc
- (void)setFinished:(BOOL)finished {
    [self willChangeValueForKey:@"isFinished"];
    _finished = finished;
    [self didChangeValueForKey:@"isFinished"];
}

- (void)setExecuting:(BOOL)executing {
    [self willChangeValueForKey:@"isExecuting"];
    _executing = executing;
    [self didChangeValueForKey:@"isExecuting"];
}

- (BOOL)isConcurrent {
    return YES;
}
```

再封装要执行的任务后，就完成了一个自定义的NSOperation。