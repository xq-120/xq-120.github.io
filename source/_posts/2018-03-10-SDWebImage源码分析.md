---
title: SDWebImage源码分析
date: 2018-03-10 21:08:37
description: 
categories: 第三方库使用
tags: SDWebImage
comments: true
---

### SDWebImage源码分析

> 本文主要分析解决以下几个疑问:
> 
 1. 同一图片的请求,SD如何从缓存中查找出来(key的命名规则)?或者说查找过程是怎样的?OK
 2. SD工作过程以及是如何从服务器下载的?
 3. SD是如何缓存图片的?OK
 4. 当空间不足时,它的删除策略是什么(按时间顺序,还是图片质量)?
 5. SD图片缓存机制与系统的NSURLCache的区别,为什么SD说自己的性能要略高于系统的性能?

#### SDWebImageOptions的枚举值解释
在使用SD提供的UIImageView类别方法: 
 
```
- (void)sd_setImageWithURL:(NSURL *)url placeholderImage:(UIImage *)placeholder options:(SDWebImageOptions)options progress:(SDWebImageDownloaderProgressBlock)progressBlock completed:(SDWebImageCompletionBlock)completedBlock;

```
下载图片时,有一个`SDWebImageOptions`参数,它可以用来配置SD的下载、缓存等行为.该参数是一个枚举,它有很多可选的值.  
下面将分析每个值的含义:  
`SDWebImageRetryFailed = 1 << 0`,SD的注释

```
/**
     * By default, when a URL fail to be downloaded, the URL is blacklisted so the library won't keep trying.
     * This flag disable this blacklisting.
*/
```
翻译:  
SD默认,当一个URL下载失败后(应该是网址无效导致的下载失败,而不是因为网络差原因导致的失败,**待确认**),该URL将会进入黑名单,SD将不会继续尝试下载.该选项将不使能黑名单.即即使该URL下载失败了,也不会进入黑名单,以后可以继续尝试下载.

`SDWebImageLowPriority = 1 << 1`,SD的注释

```
/**
     * By default, image downloads are started during UI interactions, this flags disable this feature,
     * leading to delayed download on UIScrollView deceleration for instance.
     */
```
翻译:  
SD默认,当用户点击下载时就开始下载了,但该枚举值将不使能该特性.比如UIScrollView还在减速的时候,即使UI的交互指示要开始下载了,但该枚举值会使SD延后下载.

`SDWebImageCacheMemoryOnly = 1 << 2`,下载的图片不缓存到磁盘,仅缓存到内存.

`SDWebImageProgressiveDownload = 1 << 3`,SD的注释

```
/**
     * This flag enables progressive download, the image is displayed progressively during download as a browser would do.
     * By default, the image is only displayed once completely downloaded.
     */
```
翻译:  
该枚举值使能了progressive download(显示进度的下载),图片将随着进度一点点显示.SD默认只有当图片完全下载完成后才一次性显示.

`SDWebImageRefreshCached = 1 << 4`,SD的注释

```
/**
     * Even if the image is cached, respect the HTTP response cache control, and refresh the image from remote location if needed.
     * The disk caching will be handled by NSURLCache instead of SDWebImage leading to slight performance degradation.
     * This option helps deal with images changing behind the same request URL, e.g. Facebook graph api profile pics.
     * If a cached image is refreshed, the completion block is called once with the cached image and again with the final image.
     *
     * Use this flag only if you can't make your URLs static with embedded cache busting parameter.
*/
```
翻译:  

* 即使图片已经被缓存了,但是根据HTTP响应缓存控制,如果必要的话还是会用远程的图片刷新本地的缓存的图片.
* 磁盘的缓存将使用系统的NSURLCache处理而不是SDWebImage,所以会导致一个轻微的性能下降.
* 该枚举值用于处理图片改变但其对应的URL没变的情况.
* 如果缓存的图片被刷新,那么the completion block会调用两次,第一次是缓存的图片,接着的第二次是最终的图片.
* 仅当你不能让你的URL为一个静态的地址时(旧图片是网址A,新图片还是网址A)才使用该枚举值.

`SDWebImageContinueInBackground = 1 << 5`,SD的注释

```
/**
     * In iOS 4+, continue the download of the image if the app goes to background. This is achieved by asking the system for
     * extra time in background to let the request finish. If the background task expires the operation will be cancelled.
     */
```
翻译:  
需要iOS4+,当APP进入后台时,继续图片的下载.该功能的实现是:进入后台时,通过向系统申请额外的时间来让请求完成.如果后台任务到期,操作将被取消.(xq注:有可能取消时,图片还没下载完成.这种情况应该比较少见,主要发生在下载高清图,但网速又不好的情况)

`SDWebImageHandleCookies = 1 << 6`,处理存储在NSHTTPCookieStore里的cookie.内部通过设置NSMutableURLRequest.HTTPShouldHandleCookies = YES;实现.

`SDWebImageAllowInvalidSSLCertificates = 1 << 7`,使能允许未信任的SSL证书.主要用于测试环境.生产环境谨慎使用.

`SDWebImageHighPriority = 1 << 8`,SD的注释

```
/**
     * By default, images are loaded in the order in which they were queued. This flag moves them to
     * the front of the queue.
     */

```
翻译:  
SD默认,图片是以它们入列时的顺序下载.该枚举值会将它们移到队列的前面.

`SDWebImageAvoidAutoSetImage = 1 << 11`,SD默认图片下载完成后自动设置imageView,但该枚举值可以让SD不自动设置,而是让程序员手动设置.比如下载完成后还要处理一下才设置到imageView上去.

#### SD的缓存路径
SD默认的缓存路径:

```
Library/Caches/default/com.hackemist.SDWebImageCache.default/cf00214b25574df24d89120a11424121.jpg
```
它是在`Library/Caches/`目录下创建了`default/com.hackemist.SDWebImageCache.default/`路径,以后缓存的图片将存储在该路径下.**SD保存图片的命名规则:**通过对URLString md5后,再拼接可能有的后缀名得来.如:cf00214b25574df24d89120a11424121.jpg

系统`NSURLCache`缓存的路径:
`Library/Caches/bundleIdXXX(APP bundle id)/`,然后在`bundleIdXXX`文件夹下存放缓存:
`Cache.db,Cache.db-shm,Cache.db-wal,fsCachedData/A755FD02-64B3-430C-899D-6AD44E4D3229(图片的名称)`.

```
SDWebImageDownloader *downloader = [SDWebImageDownloader sharedDownloader];
[downloader downloadImageWithURL:[NSURL URLWithString:@"http://b.zol-img.com.cn/sjbizhi/images/9/230x350/1469533909506.jpg"] options:SDWebImageDownloaderUseNSURLCache progress:nil completed:^(UIImage *image, NSData *data, NSError *error, BOOL finished) {
        NSLog(@"error:%@, finished:%d", error, finished);
        NSLog(@"%@", image);
}];
```
当`SDWebImageDownloaderOptions`选择`SDWebImageDownloaderUseNSURLCache`时,SD将采用系统的`NSURLCache`缓存图片,所以这里即使下载了,但使用SD的类别下载时[self.myImageView sd_setImageWithURL:]由于采用的是另一套查找机制,它没有去查找系统的缓存,所以还得重新下载.

#### 同一图片的请求,SD如何从缓存中查找出来(key的命名规则)?或者说查找过程是怎样的?
SD key的命名规则是怎样的?
`NSString *key = [self cacheKeyForURL:url];`

```
- (NSString *)cacheKeyForURL:(NSURL *)url {
    if (self.cacheKeyFilter) {
        return self.cacheKeyFilter(url);
    }
    else {
        return [url absoluteString];
    }
}
```
SD命名cache的key是通过返回URL的absoluteString属性的值.该key在后面的内存查找缓存和磁盘查找缓存都有用到的.

SD自定义了一个缓存类SDImageCache,该类维护着一个内存缓存和一个可选的磁盘缓存.Disk cache的写操作是异步的因此不用担心影响到UI.

维护的内存缓存是一个AutoPurgeCache对象.`@interface AutoPurgeCache : NSCache`.AutoPurageCache主要是在初始化的时候注册了一个UIApplicationDidReceiveMemoryWarningNotification内存警告的通知.如果收到内存警告通知,则调用`-removeAllObjects`清空缓存.

```
@interface SDImageCache ()

@property (strong, nonatomic) NSCache *memCache;
@property (strong, nonatomic) NSString *diskCachePath;
@property (strong, nonatomic) NSMutableArray *customPaths;
@property (SDDispatchQueueSetterSementics, nonatomic) dispatch_queue_t ioQueue;

@end
```

```
// Init the memory cache
_memCache = [[AutoPurgeCache alloc] init];
_memCache.name = fullNamespace;
```

SD下载图片时是将下载封装到一个操作里面的.

SD的主类`SDWebImageManager`有两个重要的属性

```
@property (strong, nonatomic, readonly) SDImageCache *imageCache;
@property (strong, nonatomic, readonly) SDWebImageDownloader *imageDownloader;
```
`SDImageCache`是用来操作缓存的,`SDWebImageDownloader `是用来下载图片的.

SD通过上面的Key参数和方法`- (NSOperation *)queryDiskCacheForKey:(NSString *)key done:(SDWebImageQueryCompletedBlock)doneBlock`来查找缓存.

#### 同步查找内存缓存

```
- (UIImage *)imageFromMemoryCacheForKey:(NSString *)key {
    return [self.memCache objectForKey:key];
}

```
内存查找比较简单,就是使用NSCache的方法`-objectForKey:`,如果在内存里找到那么就直接返回图片了.如果没查到,就**异步的从磁盘里查找**.也就是说**SD的`- (NSOperation *)queryDiskCacheForKey:(NSString *)key done:(SDWebImageQueryCompletedBlock)doneBlock`方法是同步的查找内存中缓存,异步的查找磁盘中的缓存.得好好看一下这种设计的实现.**  
PS:NSCache类似于NSDictionary.它里面存放的对象都是位于内存的.

#### 异步查找磁盘缓存
```
dispatch_async(self.ioQueue, ^{
	if (operation.isCancelled) {
	    return;
	}
	
	@autoreleasepool {
	    UIImage *diskImage = [self diskImageForKey:key];
	    if (diskImage && self.shouldCacheImagesInMemory) {
	        NSUInteger cost = SDCacheCostForImage(diskImage);
	        [self.memCache setObject:diskImage forKey:key cost:cost];
	    }
	
	    dispatch_async(dispatch_get_main_queue(), ^{
	        doneBlock(diskImage, SDImageCacheTypeDisk);
	    });
	}
});
```

#### 根据Key在磁盘中找到对应的缓存图片
主要方法:`UIImage *diskImage = [self diskImageForKey:key];`
  
SD是通过Key得到文件(图片)名的,`NSString *filename = [self cachedFileNameForKey:key];`方法内部是对key进行md5签名,得到一个签名串,然后再拼接后缀名(如果key里面有后缀的话).这个时候就得到了文件名,再拼接SD自己的存储路径self.diskCachePath(`/Library/Caches/default/com.hackemist.SDWebImageCache.default`事先指定好的路径名).到这一步就获得了指定key的缓存图片的路径(`/Library/Caches/default/com.hackemist.SDWebImageCache.default/cf00214b25574df24d89120a11424121.jpg`).通过`NSData *data = [NSData dataWithContentsOfFile:defaultPath];`读取出文件的数据.最后将NSData转化为UIImage.至此从磁盘查找指定key的缓存图片就到此结束了.  
PS:`[result appendFormat:@"%02x", digest[i]];`  
x/X:表示以十六进制形式输出.   
02:表示不足两位,前面补0输出;超过两位,则直接输出.


### SD是如何缓存图片的?
主要是通过`SDImageCache`类完成缓存功能.`SDImageCache`提供的接口有:

```

@interface SDImageCache ()

#pragma mark - Properties
@property (strong, nonatomic, nonnull) NSCache *memCache; //里面是关联了一个系统的NSCache对象.
@property (strong, nonatomic, nonnull) NSString *diskCachePath;
@property (strong, nonatomic, nullable) NSMutableArray<NSString *> *customPaths;
@property (SDDispatchQueueSetterSementics, nonatomic, nullable) dispatch_queue_t ioQueue;

@end
```

* 初始化方法:
`+ (SDImageCache *)sharedImageCache;`、`- (id)initWithNamespace:(NSString *)ns diskCacheDirectory:(NSString *)directory;`提供缓存路径的设置.

* 存储缓存方法:
`- (void)storeImage:(UIImage *)image forKey:(NSString *)key toDisk:(BOOL)toDisk;`

* 获取缓存方法:
	1. `- (NSOperation *)queryDiskCacheForKey:(NSString *)key done:(SDWebImageQueryCompletedBlock)doneBlock;`
	2. `- (UIImage *)imageFromMemoryCacheForKey:(NSString *)key;`
	3. `- (UIImage *)imageFromDiskCacheForKey:(NSString *)key;`

* 移除缓存的方法:
	1. `- (void)removeImageForKey:(NSString *)key fromDisk:(BOOL)fromDisk withCompletion:(SDWebImageNoParamsBlock)completion;`
	2. `- (void)clearDisk;`
	3. `- (void)cleanDiskWithCompletionBlock:(SDWebImageNoParamsBlock)completionBlock;`

* 查询信息方法(查询某个key的缓存是否存在,查询缓存占用了多少空间):
	1. `- (NSUInteger)getSize;`
	2. `- (BOOL)diskImageExistsWithKey:(NSString *)key;`
	3. `- (void)calculateSizeWithCompletionBlock:(SDWebImageCalculateSizeBlock)completionBlock;`

可以看到这个类的接口功能是比较完善的,有初始化方法、存、取、删、查询信息等方法.并且基本都提供了一个同步的,一个异步的.**可以好好学习一下.**

当图片下载完成后,就进行缓存:

```
if (downloadedImage && finished) {
	[self.imageCache storeImage:downloadedImage recalculateFromImage:NO imageData:data forKey:key toDisk:cacheOnDisk];
}

dispatch_main_sync_safe(^{
	if (strongOperation && !strongOperation.isCancelled) {
	    completedBlock(downloadedImage, nil, SDImageCacheTypeNone, finished, url);
	}
});

//最终是将图片的二进制数据写入文件
// Make sure to call form io queue by caller
- (void)_storeImageDataToDisk:(nullable NSData *)imageData forKey:(nullable NSString *)key {
    if (!imageData || !key) {
        return;
    }
    
    if (![self.fileManager fileExistsAtPath:_diskCachePath]) {
        [self.fileManager createDirectoryAtPath:_diskCachePath withIntermediateDirectories:YES attributes:nil error:NULL];
    }
    
    // get cache Path for image key
    NSString *cachePathForKey = [self defaultCachePathForKey:key];
    // transform to NSUrl
    NSURL *fileURL = [NSURL fileURLWithPath:cachePathForKey];
    
    [imageData writeToURL:fileURL options:self.config.diskCacheWritingOptions error:nil];
    
    // disable iCloud backup
    if (self.config.shouldDisableiCloud) {
        [fileURL setResourceValue:@YES forKey:NSURLIsExcludedFromBackupKey error:nil];
    }
}
```

#### SD清除缓存规则

SD的缓存清理涉及到两个关键的配置属性:

```
//最大缓存时间
/**
 * The maximum length of time to keep an image in the cache, in seconds.
 */
@property (assign, nonatomic) NSInteger maxCacheAge;

//最大缓存空间
/**
 * The maximum size of the cache, in bytes.
 */
@property (assign, nonatomic) NSUInteger maxCacheSize;

- (void)setMaxMemoryCost:(NSUInteger)maxMemoryCost {
    self.memCache.totalCostLimit = maxMemoryCost;
}

- (NSUInteger)maxMemoryCost {
    return self.memCache.totalCostLimit;
}

```
SD根据这两个属性的值,先删除最旧的文件.删除后若还超出了maxCacheSize,则进一步删除旧文件.不难看出SD清除缓存的策略就是删除旧文件.而其他框架可能会有所不同.比如采用LFU(Least Frequently Used)算法.它是根据数据的历史访问频率来淘汰数据，其核心思想是“如果数据过去被访问多次，那么将来被访问的频率也更高”。

缓存清除时机:app即将干掉,或进入后台时.

```
// Subscribe to app events
        [[NSNotificationCenter defaultCenter] addObserver:self
                                                 selector:@selector(deleteOldFiles)
                                                     name:UIApplicationWillTerminateNotification
                                                   object:nil];

        [[NSNotificationCenter defaultCenter] addObserver:self
                                                 selector:@selector(backgroundDeleteOldFiles)
                                                     name:UIApplicationDidEnterBackgroundNotification
                                                   object:nil];
                                                   
// Start the long-running task and return immediately.
    [self deleteOldFilesWithCompletionBlock:^{
        [application endBackgroundTask:bgTask];
        bgTask = UIBackgroundTaskInvalid;
    }];
```

### SD图片下载

```
@interface SDWebImageCombinedOperation : NSObject <SDWebImageOperation>

@property (assign, nonatomic, getter = isCancelled) BOOL cancelled;
@property (copy, nonatomic) SDWebImageNoParamsBlock cancelBlock;
@property (strong, nonatomic) NSOperation *cacheOperation;

@end
```

`__block SDWebImageCombinedOperation *operation = [SDWebImageCombinedOperation new];`
`operation.cacheOperation = [self.imageCache queryDiskCacheForKey:key done:^(UIImage *image, SDImageCacheType cacheType) {...}];`
**不太明白`SDWebImageCombinedOperation`的cacheOperation赋值以后有什么用**,好像没有哪个地方将它添加到队列里.仅仅在`SDWebImageCombinedOperation `的cancel方法里看到用了一下.

```
- (void)cancel {
    self.cancelled = YES;
    if (self.cacheOperation) {
        [self.cacheOperation cancel];
        self.cacheOperation = nil;
    }
    if (self.cancelBlock) {
        self.cancelBlock();
        
        // TODO: this is a temporary fix to #809.
        // Until we can figure the exact cause of the crash, going with the ivar instead of the setter
//        self.cancelBlock = nil;
        _cancelBlock = nil;
    }
}
```

```
- (id <SDWebImageOperation>)loadImageWithURL:(nullable NSURL *)url
                                     options:(SDWebImageOptions)options
                                    progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                   completed:(nullable SDInternalCompletionBlock)completedBlock;
                                   
- (nullable SDWebImageDownloadToken *)addProgressCallback:(SDWebImageDownloaderProgressBlock)progressBlock
                                           completedBlock:(SDWebImageDownloaderCompletedBlock)completedBlock
                                                   forURL:(nullable NSURL *)url
                                           createCallback:(SDWebImageDownloaderOperation *(^)(void))createCallback;
```
下载的操作就是在该方法中加入到队列里的,代码片段:

```
// Add operation to operation queue only after all configuration done according to Apple's doc.
// `addOperation:` does not synchronously execute the `operation.completionBlock` so this will not cause deadlock.
[self.downloadQueue addOperation:operation];
```