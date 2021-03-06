AVPlayer边下边播缓存实现

```
/**
 1.边缓存,边播放?
 2.拖动快进快退?
 3.播放/暂停
 4.进度条显示,剩余时间显示
 5.播放完成后又快退到某个点播放.
 6.播放完成后要不要把player置为nil.
 7.播放中拖动进度条有点飘
 8.偶现播放完成时,进度时间显示错误超出了.快进快退导致
 9.AVPlayerItemDidPlayToEndTimeNotification 网速不好的时候,缓冲了差不多10s后,居然回调了AVPlayerItemDidPlayToEndTimeNotification通知.但缓冲还在继续.
 这是由于[_player setAutomaticallyWaitsToMinimizeStalling:NO];导致
 10.[_player setAutomaticallyWaitsToMinimizeStalling:YES]就会导致需要缓冲差不多完成才开始播放.
 11.如何在有了一些缓冲后立即播放?
 12.极限优化,seekTime时,缓冲只从附近开始缓冲,缓冲一些后就开始播放,而不是从头开始缓冲.
 */
```

什么是边下边播？

边下边播指的是从服务器获取到的数据一边供给播放器播放一边缓存到本地，即使用一遍的流量完成播放和缓存。下次播放时有缓存的地方则播放缓存数据，没有缓存的地方则从服务器获取再播放。

边下边播缓存实现的思路其实很简单：

1. 代理播放器的请求

2. 向服务器发起请求，接收到数据后塞给播放器播放并缓存到本地。
3. 缓存清除策略。

思路虽然简单，但每一步的实现着实不容易。

### 方案选择

#### 方案1：前台播放，后台下载

该方案其实不算边下边播，因为下载的数据并没有提供给播放器播放，二者是各自独立的，总共消耗了两遍的流量。不过该方案实现简单，在早期没有充足的时间下可以考虑使用。

#### 方案2：使用本地代理服务器

该方案是在播放器与视频源服务器之间加一层代理服务器，截取视频播放器发送的请求，根据截取的请求，向网络服务器请求数据，然后写到本地。本地代理服务器从文件中读取数据并发送给播放器进行播放。

![](http://www.samirchen.com/images/video-playback-bandwidth-cost/player-cache-proxy.png)

对于 iOS 端代理服务器的实现，可以参考和使用 CocoaHTTPServer。对于 iOS 端的视频缓存管理，可以参考和使用 KTVHTTPCache。

HTTPServer 不管我们有没有使用缓存功能，都要在应用打开的时候默默开启，对APP性能是一大损耗。并且引入 HTTPServer 库也会增加一些包体积。

TODO：这种方案没有试过，感觉技术难度比较高。

#### 方案3：使用系统原生API--AVAssetResourceLoaderDelegate

方案三跟方案二原理差不多，但是不需要我们自己开启本地服务器，实现相对简单。默认情况下使用AVPlayer播放音视频，系统是不会把数据缓存到本地的。但是我们可以通过设置AVAssetResourceLoader的delegate来代理系统的请求。代理系统请求后我们就可以对数据进行缓存。

本文采用方案三来实现边下边播缓存实现。接下来将围绕如何代理播放器请求，如何缓存数据，缓存清除策略，遇到的问题这几个方面来详细讲解。

### 整体架构设计



### 1. AVAssetResourceLoaderDelegate的使用

一个resource对应一个loader，一个loader管理多个loadingRequest，每个loadingRequest对应一个真正的requestOperation。一个loader虽然管理多个loadingRequest，但同一时刻只能有一个requestOperation下载数据，所以当接收到一个loadingRequest时必须先取消掉之前所有的loadingRequest，确保同一时刻只有一个dataTask在下载数据（这里思路错误，导致后面测试发现很多问题）。正确做法是限制最大请求数，不能全部取消。

#### 1.1 代理系统的请求

在代理系统的请求前，可以通过抓包，大致了解下AVPlayer播放一个URL时的请求机制：  
先请求0-1的数据段，成功后再请求一大段数据，但只接收一小段数据就cancel掉请求。播放一小段后又请求一大段数据，但接收一小段数据后又cancel掉请求，循环这个操作直到接收所有数据。而当用户seek时会马上取消当前的下载请求，然后从seek处发起一个新请求。

了解这一过程对后面的实现非常有帮助，下面正式代理系统的请求。

代理系统的请求非常简单，只需要将我们的对象设置为 `AVAssetResourceLoader` 的代理，并实现 `AVAssetResourceLoaderDelegate` 协议。

```objc
//1.将正常的scheme替换为自定义的scheme，并将self设置为resourceLoader的代理
- (AVURLAsset *)getCustomSchemeAssetWithItemUrl:(NSURL *)url {
    NSURL *csUrl = [ZAEPlayerUtils customSchemeUrlWithUrl:url]; 
    AVURLAsset *urlAsset = [[AVURLAsset alloc] initWithURL:csUrl options:nil];
    [urlAsset.resourceLoader setDelegate:self queue:dispatch_get_main_queue()]; 
    return urlAsset;
}

//2.实现`AVAssetResourceLoaderDelegate`协议中的如下两个，如果有其他的需求可以实现其他几个协议。
//接收到一个新的loadingRequest。该方法仅在系统不知道如何处理URLAsset资源时才会回调，所以我们需要自定义scheme。如果scheme是我们自定义的则返回YES表示我们将接管资源的请求，否则返回NO。
- (BOOL)resourceLoader:(AVAssetResourceLoader *)resourceLoader shouldWaitForLoadingOfRequestedResource:(AVAssetResourceLoadingRequest *)loadingRequest {
    NSURLRequest *req = loadingRequest.request;
    if ([ZAEPlayerUtils isCustomSchemeUrl:req.URL]) {
        ZAEResourceLoader *loader = [self resourceLoaderWithKey:req.URL.absoluteString];
        if (loader == nil) {
            NSURL *originUrl = [ZAEPlayerUtils originalUrlWithUrl:req.URL];
            loader = [[ZAEResourceLoader alloc] initWithURL:originUrl];
            [self setResourceLoader:loader withKey:req.URL.absoluteString];
        }
        [loader addLoadingRequest:loadingRequest];
        return YES;
    } else {
        return NO;
    }
}

//系统取消加载资源后回调。在该方法里我们需要取消掉我们之前发的请求。
- (void)resourceLoader:(AVAssetResourceLoader *)resourceLoader didCancelLoadingRequest:(AVAssetResourceLoadingRequest *)loadingRequest {
    NSURLRequest *req = loadingRequest.request;
    ZAEResourceLoader *loader = [self resourceLoaderWithKey:req.URL.absoluteString];
    [loader cancelLoadingRequest:loadingRequest];
}
```

这里唯一需要注意的是 URL 必须是自定义的 URLScheme，我们需要把原始 URL 的 `http://` 或 `https://` 替换成 `xxx://`，协议方法才会生效。这里我们在原scheme后面拼接“-mine”作为自定义的scheme。选择在原scheme后面拼接的好处就是当我们自己去服务器请求的时候能够很方便的解析出原scheme。到此为止我们就成功拦截了播放器的请求，拦截请求之后我们就需要获取数据并提供给播放器。数据可以从本地缓存获取也可以从服务器获取，这是我们后面要做的事。

然而非常坑人的事情来了，经过测试发现在iOS10.3.3中

```objc
- (void)resourceLoader:(AVAssetResourceLoader *)resourceLoader didCancelLoadingRequest:(AVAssetResourceLoadingRequest *)loadingRequest API_AVAILABLE(macos(10.9), ios(7.0), tvos(9.0)) API_UNAVAILABLE(watchos);
```

didCancelLoadingRequest:方法除了手机重启后的第一次运行会被调用，其他时候都不会被调用，非常诡异。而在iOS14上则正常，AVAssetResourceLoader会时不时调用didCancelLoadingRequest:让你有机会cancel掉之前的请求。

参考：[AVAssetResourceLoaderDelegate -resourceLoader: didCancelLoadingRequest: naver called (in the Device only)](https://developer.apple.com/forums/thread/29039)

#### 1.2 发起请求

代理系统的请求后，我们需要根据请求去获取相应的数据，而在shouldWaitForLoadingOfRequestedResource代理方法里，系统只提供了一个AVAssetResourceLoadingRequest对象，因此我们需要根据AVAssetResourceLoadingRequest相关的API来获取本次range请求的范围。

AVAssetResourceLoadingRequest：

```objc
@interface AVAssetResourceLoadingRequest : NSObject {
@private
	AVAssetResourceLoadingRequestInternal *_loadingRequest;
}
AV_INIT_UNAVAILABLE

@property (nonatomic, readonly, nullable) AVAssetResourceLoadingContentInformationRequest *contentInformationRequest API_AVAILABLE(macos(10.9), ios(7.0), tvos(9.0)) API_UNAVAILABLE(watchos);

@property (nonatomic, readonly, nullable) AVAssetResourceLoadingDataRequest *dataRequest API_AVAILABLE(macos(10.9), ios(7.0), tvos(9.0)) API_UNAVAILABLE(watchos);

- (void)finishLoading API_AVAILABLE(macos(10.9), ios(7.0), tvos(9.0)) API_UNAVAILABLE(watchos); //本次LoadingRequest请求完成时调用finishLoading通知系统。

- (void)finishLoadingWithError:(nullable NSError *)error; //本次LoadingRequest请求出错时调用finishLoadingWithError:通知系统。

@end
```

contentInformationRequest：视频元数据信息。

dataRequest：数据请求，里面包含本次range请求的范围。

AVAssetResourceLoadingContentInformationRequest：

```objc
@interface AVAssetResourceLoadingContentInformationRequest : NSObject {
@private
	AVAssetResourceLoadingContentInformationRequestInternal *_contentInformationRequest;
}
AV_INIT_UNAVAILABLE

@property (nonatomic, copy, nullable) NSString *contentType; //响应头的content-type，但要转为UTI才能赋值给contentType
@property (nonatomic) long long contentLength; //视频长度（byte）
@property (nonatomic, getter=isByteRangeAccessSupported) BOOL byteRangeAccessSupported; //是否支持range请求。

@end
```

AVAssetResourceLoadingDataRequest：

```objc
@interface AVAssetResourceLoadingDataRequest : NSObject {
@private
	AVAssetResourceLoadingDataRequestInternal *_dataRequest;
}
AV_INIT_UNAVAILABLE
  
@property (nonatomic, readonly) long long requestedOffset;
@property (nonatomic, readonly) NSInteger requestedLength;
@property (nonatomic, readonly) BOOL requestsAllDataToEndOfResource API_AVAILABLE(macos(10.11), ios(9.0), tvos(9.0)) API_UNAVAILABLE(watchos);
@property (nonatomic, readonly) long long currentOffset;
- (void)respondWithData:(NSData *)data; //将请求到的数据塞给播放器。

@end
```

requestedOffset：本次range请求的起始字节位置。

requestedLength：本次range请求的长度。

currentOffset：当前已下载的数据的偏移量。requestedOffset + data.length = currentOffset。每次调用respondWithData:方法后，currentOffset会改变。

示意图：



#### 1.3 塞数据给播放器



#### 1.4 取消请求



#### 1.5 其他

1.有没有必要先发一次获取contentInfo信息的请求比如HEAD请求，再请求数据？

没必要，因为系统每次播放前都会先发一个0-1的range请求，服务器响应后，我们可以从响应head里获取到contentInfo信息，同时保存到本地。

2.loaderDict什么时候移除loader

a.ZAEResourceLoaderManager 调用cancel时清空loaderDict。

3.loadingRequests什么时候移除loadingRequest

a.在addLoadingRequest准备发起一个下载请求时，取消并移除最旧的、较小range请求的loadingRequest。

b.下载完成后移除当前loadingRequest。

c.系统调用didCancelLoadingRequest时移除。

TODO：由于在iOS 10.3.3上didCancelLoadingRequest:方法不会被调用，目前设置了最大请求数3，但是抓包发现其实有一些请求请求的长度完全一样，或者处于包含关系，这里应该可以优化一下。

### 2.缓存数据，并建立区间记录表

1.根据系统的请求范围，查找区间记录表，将系统请求拆分为一个请求列表。[本地请求，远程请求，本地请求...]

2.依次执行请求列表中的每个请求，并将请求到的数据供给播放器。

3.远程请求接收到数据后缓存到音频文件，并合并式更新区间记录表。

音频元数据的保存，音频文件的保存，区间记录表的保存？读写同步问题？

### 3.缓存清除策略

> 1.计算文件大小

[How can I calculate the size of a folder?](https://stackoverflow.com/questions/2188469/how-can-i-calculate-the-size-of-a-folder/28660040#28660040)

[“文件大小”和“占用空间”的区别](https://blog.csdn.net/duyusean/article/details/78643475)

系统提供了一些key用于获取文件信息

```objc
/* Resource keys applicable only to regular files
 */
FOUNDATION_EXPORT NSURLResourceKey const NSURLFileSizeKey                    API_AVAILABLE(macos(10.6), ios(4.0), watchos(2.0), tvos(9.0)); // Total file size in bytes (Read-only, value type NSNumber)
FOUNDATION_EXPORT NSURLResourceKey const NSURLFileAllocatedSizeKey           API_AVAILABLE(macos(10.6), ios(4.0), watchos(2.0), tvos(9.0)); // Total size allocated on disk for the file in bytes (number of blocks times block size) (Read-only, value type NSNumber)
FOUNDATION_EXPORT NSURLResourceKey const NSURLTotalFileSizeKey               API_AVAILABLE(macos(10.7), ios(5.0), watchos(2.0), tvos(9.0)); // Total displayable size of the file in bytes (this may include space used by metadata), or nil if not available. (Read-only, value type NSNumber)
FOUNDATION_EXPORT NSURLResourceKey const NSURLTotalFileAllocatedSizeKey      API_AVAILABLE(macos(10.7), ios(5.0), watchos(2.0), tvos(9.0)); // Total allocated size of the file in bytes (this may include space used by metadata), or nil if not available. This can be less than the value returned by NSURLTotalFileSizeKey if the resource is compressed. (Read-only, value type NSNumber)
FOUNDATION_EXPORT NSURLResourceKey const NSURLIsAliasFileKey                 API_AVAILABLE(macos(10.6), ios(4.0), watchos(2.0), tvos(9.0)); // true if the resource is a Finder alias file or a symlink, false otherwise ( Read-only, value type boolean NSNumber)
FOUNDATION_EXPORT NSURLResourceKey const NSURLFileProtectionKey              API_AVAILABLE(macos(10.11), ios(9.0), watchos(2.0), tvos(9.0)); // The protection level for this file
```

注意NSURLTotalFileAllocatedSizeKey的说明：This can be less than the value returned by NSURLTotalFileSizeKey if the resource is compressed. 也就是说文件大小和文件实际占用的磁盘大小有可能不一致，系统在存储文件时有可能会进行压缩。

eg: 

```
http://1251661065.vod2.myqcloud.com/98deaa00vodgzp1251661065/c6200b325285890794913485240/saGb5NcsJREA.mp4
```

上面视频有553M，在缓存时会先初始化一个553M的占位盒子，然后边下载边往里面填充实际数据。如果没有缓存完就停止缓存，此时用NSURLFileSizeKey 和 NSURLTotalFileAllocatedSizeKey去获取文件大小就会发现二者的值不一样。

NSURLFileSizeKey 返回的是文件自身的大小。

而NSURLTotalFileAllocatedSizeKey 返回的是该文件占用的磁盘大小，因为实际只缓存了一点点数据，系统会对他进行压缩。所以该值会小于NSURLFileSizeKey 返回的值。当完全缓存时，该值就会约等于NSURLFileSizeKey 返回的值（实际会稍大于，因为NSURLTotalFileAllocatedSizeKey 还会计算元数据占用的空间）。

```
2021-01-26 11:01:24.590838+0800 AudioDemo[6292:586528] attrs:{
    NSFileCreationDate = "2021-01-25 09:31:02 +0000";
    NSFileExtensionHidden = 0;
    NSFileGroupOwnerAccountID = 501;
    NSFileGroupOwnerAccountName = mobile;
    NSFileModificationDate = "2021-01-26 02:43:28 +0000";
    NSFileOwnerAccountID = 501;
    NSFileOwnerAccountName = mobile;
    NSFilePosixPermissions = 420;
    NSFileProtectionKey = NSFileProtectionCompleteUntilFirstUserAuthentication;
    NSFileReferenceCount = 1;
    NSFileSize = 553363384;
    NSFileSystemFileNumber = 4470737601;
    NSFileSystemNumber = 16777223;
    NSFileType = NSFileTypeRegular;
}

2021-01-26 11:03:29.875546+0800 AudioDemo[6292:586700] 文件：file:///private/var/mobile/Containers/Data/Application/407F45FC-F6D8-411F-89FA-F014ADC5F4EA/Library/Caches/ZAEAudioCache/com.xq.ZAEAudioCache.ZAEAudioCache/ddcada74ee08d06dda5cd13b4117ad30.mp4，totalAllocatedSize：553365504
```

iOS 14.3上“ iPhone存储空间”里的App“文稿与数据”只显示App沙盒Documents目录下文件占用的空间，并且展示的是文件实际占用的磁盘大小即NSURLTotalFileAllocatedSizeKey的值，而Cache目录下的文件并不会计算在内。主要是因为Cache目录下的文件在磁盘空间紧张时系统会自动清理。

iOS 10.3.3上“ iPhone存储空间”里的App“文稿与数据”显示的是所有沙盒目录下的文件占用的空间。

> 2.OC 二级指针的内存管理

在传出error时，error对象过早释放导致野指针崩溃。

```objc
- (NSData *)mediaDataWithStartOffset:(unsigned long long)startOffset length:(unsigned long long)numberOfBytesToRespondWith readingFileHandle:(NSFileHandle *)readingFileHandle error:(NSError **)error {
    @try {
        @autoreleasepool {
            [readingFileHandle seekToFileOffset:startOffset];
            NSData *retData = [readingFileHandle readDataOfLength:numberOfBytesToRespondWith];
            if (retData == nil || retData.length != numberOfBytesToRespondWith) {
								*error = [[NSError alloc] initWithDomain:@"com.zhenai.emotioncounsel" code:-1001 userInfo:@{NSLocalizedDescriptionKey: @"read cached data length error"}]; //方法退出后，调用者野指针崩溃，因为访问了已经释放了的error对象地址。
                
                return nil;
            }
            return retData;
        }
    }
    @catch (NSException *exception) {
        DDLogError(@"read cached data error %@",exception);
        *error = [NSError errorWithCode:-1000 msg:@"read cached data error"];
        return nil;
    }
}

想修改：
- (NSData *)mediaDataWithStartOffset:(unsigned long long)startOffset length:(unsigned long long)numberOfBytesToRespondWith readingFileHandle:(NSFileHandle *)readingFileHandle error:(NSError **)error {
    @try {
//        NSError *myErr = nil;
//        NSError *myErr = [[NSError alloc] initWithDomain:@"com.zhenai.emotioncounsel" code:-1001 userInfo:@{NSLocalizedDescriptionKey: @"read cached data length error"}];
        NSError *myErr = [NSError errorWithCode:-1001 msg:@"read cached data length error"];
        @autoreleasepool {
            [readingFileHandle seekToFileOffset:startOffset];
            NSData *retData = [readingFileHandle readDataOfLength:numberOfBytesToRespondWith];
            if (retData == nil || retData.length != numberOfBytesToRespondWith) {
//                *error = [NSError errorWithCode:-1001 msg:@"read cached data length error"];
//                *error = [[NSError alloc] initWithDomain:@"com.zhenai.emotioncounsel" code:-1001 userInfo:@{NSLocalizedDescriptionKey: @"read cached data length error"}];
//                NSError *myErr = [[NSError alloc] initWithDomain:@"com.zhenai.emotioncounsel" code:-1001 userInfo:@{NSLocalizedDescriptionKey: @"read cached data length error"}];
//                *error = myErr;
                
//                myErr = [[NSError alloc] initWithDomain:@"com.zhenai.emotioncounsel" code:-1001 userInfo:@{NSLocalizedDescriptionKey: @"read cached data length error"}];
//                *error = myErr;
                
                *error = myErr;
                
                return nil;
            }
            return retData;
        }
    }
    @catch (NSException *exception) {
        DDLogError(@"read cached data error %@",exception);
        *error = [NSError errorWithCode:-1000 msg:@"read cached data error"];
        return nil;
    }
}
```

本质上是因为*error指向的是一个注册到自动释放池的对象，而刚好外面包了一层autoreleasepool，导致超出autoreleasepool作用域后对象就释放销毁了。

参考：[二级指针与ARC不为人知的特性](http://www.cocoachina.com/articles/18861)

> 播放缓存中，进入后台，发现磁盘空间满了，然后清除掉缓存，由于有做本地请求失败转为远程请求，所以不会出现播放失败的现象。

### 4.缓冲估算

> 正在播放10s，然后seek快退到未下载的5s，loading不及时。

在seek前，想通过loadedTimeRanges判断，但发现没效果。

```
- (BOOL)loadedTimeRangesContainTime:(NSTimeInterval)time {
    CFAbsoluteTime startTime = CFAbsoluteTimeGetCurrent();
    BOOL ret = NO;
    NSArray *loadedTimeRanges = [[self.player currentItem] loadedTimeRanges];
    for (NSValue *val in loadedTimeRanges) {
        CMTimeRange timeRange = [val CMTimeRangeValue];// 获取缓冲区域
        float startSeconds = CMTimeGetSeconds(timeRange.start);
        float durationSeconds = CMTimeGetSeconds(timeRange.duration);
        if (startSeconds <= time && time < startSeconds + durationSeconds) {
            ret = YES;
            break;
        }
    }
    CFAbsoluteTime linkTime = (CFAbsoluteTimeGetCurrent() - startTime);
    NSLog(@"判断是否包含time:%f, 耗时：%fms", time, linkTime * 1000);
    return ret;
}
```

TODO另一种办法：[keyPath isEqualToString:@"loadedTimeRanges"]，当KVO回调时，合并区间。当用户seek时，就从区间里判断是否包含。

### 5.优化

> 1.如果本地请求失败了，则转为远程请求继续请求。OK

这里要注意剔除因为系统取消而导致的失败。

> 2.需要增加对缓存新鲜度的检测。目前只要缓存了就直接拿本地的数据播放，而没有先去服务器验证缓存新鲜度。（TODO）

思路：当第一次发起0-1的range请求时，需要保存好ETag，last modified，max-age等信息，当下一次播放时先发起0-1的range请求并带上保存好的ETag，last modified等信息，若服务器返回304则表示未修改，可以使用本地的缓存。否则删除本地缓存。

### 问题

##### 问题0：在缓存到80%的时候，出现range的left>right的现象。

```
[headers setValue:[NSString stringWithFormat:@"bytes=%lld-%ld", loadingRequest.dataRequest.requestedOffset, loadingRequest.dataRequest.requestedLength-1] forKey:@"range"];

<AVAssetResourceLoadingDataRequest: 0x281423340, requested offset = 32178176, requested length = 5897261, requests all data to end of resource = YES, current offset = 32178176>
2020-12-02 22:08:14.738839+0800 AudioDemo[19856:1539854] headers:{
    range = "bytes=32178176-5897260";
}
```

后台返回了 `416 Range Not Satisfiable` 错误。	

dataRequest的几个偏移量属性没搞清楚导致。

另外，塞数据给播放器及保存到本地文件注意点currentOffset的含义。

```objc
- (void)dataTaskDidReceiveDataWithSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask data:(NSData *)data {
//    DDLogError(@"接收数据LoadingRequest：%ld，Operation：%@", self.loadingRequest.hash, self);
    [self.mediaCache storeMediaDataWithContentInfo:self.contentInfo
                                     currentOffset:self.loadingRequest.dataRequest.currentOffset
                                              data:data
                                            forKey:self.resourceURL.absoluteString
                                        completion:nil];
    
    if (self.delgate && [self.delgate respondsToSelector:@selector(requestOperation:didReceiveData:)]) {
        [self.delgate requestOperation:self didReceiveData:data];
    }
}
```

上面两个方法不能交换位置，因为只要给播放器塞了数据，self.loadingRequest.dataRequest.currentOffset的值就变了，导致保存到本地时offset与实际应该的位置就错了。所以只能先使用loadingRequest.dataRequest.currentOffset的值保存数据到本地，再将data塞给播放器。或者自己记录currentOffset。

##### 问题1：下载请求超时失败了，但是播放器没有收到任何回调，并且也没有后续请求发起。（偶现）

频繁seek导致。

一般情况，某次loader下载超时后，系统会再次发起一个新请求的。但有时候不会，待观察。

```
2020-12-03 18:58:47.770897+0800 AudioDemo[23569:1843035] headers:{
    range = "bytes=28704768-";
}
2020-12-03 18:59:48.765805+0800 AudioDemo[23569:1846557] Task <02047503-697B-4DF0-A6DE-780421F142D7>.<8> finished with error [-1001] Error Domain=NSURLErrorDomain Code=-1001 "The request timed out." UserInfo={_kCFStreamErrorCodeKey=-2102, NSUnderlyingError=0x281a90570 {Error Domain=kCFErrorDomainCFNetwork Code=-1001 "(null)" UserInfo={_kCFStreamErrorCodeKey=-2102, _kCFStreamErrorDomainKey=4}}, _NSURLErrorFailingURLSessionTaskErrorKey=LocalDataTask <02047503-697B-4DF0-A6DE-780421F142D7>.<8>, _NSURLErrorRelatedURLSessionTaskErrorKey=(
    "LocalDataTask <02047503-697B-4DF0-A6DE-780421F142D7>.<8>"
), NSLocalizedDescription=The request timed out., NSErrorFailingURLStringKey=https://zhenai4saylove-1251661065.file.myqcloud.com/psychology/2019/08/3628348681546230.mp3, NSErrorFailingURLKey=https://zhenai4saylove-1251661065.file.myqcloud.com/psychology/2019/08/3628348681546230.mp3, _kCFStreamErrorDomainKey=4}
2020-12-03 18:59:48.766937+0800 AudioDemo[23569:1846557] error:Error Domain=NSURLErrorDomain Code=-1001 "The request timed out." UserInfo={_kCFStreamErrorCodeKey=-2102, NSUnderlyingError=0x281a90570 {Error Domain=kCFErrorDomainCFNetwork Code=-1001 "(null)" UserInfo={_kCFStreamErrorCodeKey=-2102, _kCFStreamErrorDomainKey=4}}, _NSURLErrorFailingURLSessionTaskErrorKey=LocalDataTask <02047503-697B-4DF0-A6DE-780421F142D7>.<8>, _NSURLErrorRelatedURLSessionTaskErrorKey=(
    "LocalDataTask <02047503-697B-4DF0-A6DE-780421F142D7>.<8>"
), NSLocalizedDescription=The request timed out., NSErrorFailingURLStringKey=https://zhenai4saylove-1251661065.file.myqcloud.com/psychology/2019/08/3628348681546230.mp3, NSErrorFailingURLKey=https://zhenai4saylove-1251661065.file.myqcloud.com/psychology/2019/08/3628348681546230.mp3, _kCFStreamErrorDomainKey=4}
```

##### 问题2：自定义scheme URL <---> 原URL互推。

在原scheme上拼接字符串，这样就可以方便互推。

##### 问题3：代理系统请求后，iOS14上播放正常，但iOS10上却无法播放。

明明请求的是0-9587589，但返回的却是bytes 0-1。

```
2020-12-04 17:04:40.618715+0800 AudioDemo[401:47404] request:{
    Accept = "*/*";
    "Accept-Encoding" = "gzip, deflate";
    "Accept-Language" = "zh-Hans-CN;q=1";
    Range = "bytes=0-9587589";
    "User-Agent" = "AudioDemo/1.0 (iPhone; iOS 10.3.3; Scale/3.00)";
}
response:<NSHTTPURLResponse: 0x174223da0> { URL: http://1251661065.vod2.myqcloud.com/98deaa00vodgzp1251661065/bd17bc125285890786674261612/mVsqUwWaIywA.mp3 } { status code: 206, headers {
    "Access-Control-Allow-Credentials" = true;
    "Access-Control-Allow-Headers" = "Origin,No-Cache,X-Requested-With,If-Modified-Since,Pragma,Last-Modified,Cache-Control,Expires,Content-Type,X_Requested_With,Range";
    "Access-Control-Allow-Methods" = "GET,POST,OPTIONS";
    "Access-Control-Allow-Origin" = "*";
    "Cache-Control" = "max-age=600";
    "Content-Length" = 2;
    "Content-Range" = "bytes 0-1/9587590";
    "Content-Type" = "audio/mp3";
    Date = "Fri, 04 Dec 2020 09:00:41 GMT";
    Expires = "Fri, 04 Dec 2020 09:10:41 GMT";
    "Last-Modified" = "Mon, 11 Mar 2019 17:11:12 GMT";
    "Proxy-Connection" = "keep-alive";
    Server = "NWS_VP";
    "X-Cache-Lookup" = "Hit From Disktank3, Hit From Inner Cluster, Hit From Inner Cluster";
    "X-Daa-Tunnel" = "hop_count=1";
    "X-NWS-LOG-UUID" = "2c325051-4888-4af4-9a71-6d7f96616d17 73052cfbf9529a51ed878f7d1692e466";
} }
```

看抓包都没有，应该是请求没有发出去。

如果直接使用NSURLSession简单实现的，

```
- (void)test_range_request {
    NSString *rUrlStr = @"http://1251661065.vod2.myqcloud.com/98deaa00vodgzp1251661065/bd17bc125285890786674261612/mVsqUwWaIywA.mp3";
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:[NSURL URLWithString:rUrlStr]];
    [request addValue:@"bytes=1244-958700" forHTTPHeaderField:@"range"];
    NSURLSessionDataTask *dataTask = [[NSURLSession sharedSession] dataTaskWithRequest:request completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
        NSLog(@"error-->%@  data:%d", error, data.length);
    }];
    [dataTask resume];
}
```

发现请求不会带有 `If-Modified-Since: Mon, 11 Mar 2019 17:11:12 GMT` 头部，这时每次都会有请求发出：

```
GET /98deaa00vodgzp1251661065/bd17bc125285890786674261612/mVsqUwWaIywA.mp3 HTTP/1.1
Host: 1251661065.vod2.myqcloud.com
Range: bytes=1244-958700
Accept: */*
User-Agent: AudioDemo/1 CFNetwork/811.5.4 Darwin/16.7.0
Accept-Language: zh-cn
Accept-Encoding: gzip, deflate
Connection: keep-alive


HTTP/1.1 206 Partial Content
Server: NWS_VP
Date: Fri, 04 Dec 2020 09:37:52 GMT
Cache-Control: max-age=600
Expires: Fri, 04 Dec 2020 09:47:52 GMT
Last-Modified: Mon, 11 Mar 2019 17:11:12 GMT
Content-Range: bytes 1244-958700/9587590
Content-Type: audio/mp3
Content-Length: 957457
X-NWS-LOG-UUID: 5b23e80d-ff4b-4848-b7c8-2dabe74e24fd 73052cfbf9529a51b556a601f57ece1b
Access-Control-Allow-Credentials: true
Access-Control-Allow-Headers: Origin,No-Cache,X-Requested-With,If-Modified-Since,Pragma,Last-Modified,Cache-Control,Expires,Content-Type,X_Requested_With,Range
Access-Control-Allow-Methods: GET,POST,OPTIONS
Access-Control-Allow-Origin: *
X-Cache-Lookup: Hit From Disktank3
X-Daa-Tunnel: hop_count=1
X-Cache-Lookup: Hit From Inner Cluster
Proxy-Connection: keep-alive

="16Int"
```

原因：

AF 的requestSerializer的cachePolicy默认是NSURLRequestUseProtocolCachePolicy。导致后续请求除了第一个请求正常，后面的请求都没发出去。系统一直把第一次请求的2byte一直返回给当前请求。

改为：

```
_httpSessionManager.requestSerializer.cachePolicy = NSURLRequestReloadIgnoringLocalCacheData;
```

##### 问题4：在iOS10.3.3上，播放结束后，马上seek到0会崩溃

```
*** Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: 'AVPlayerItem cannot service a seek request with a completion handler until its status is AVPlayerItemStatusReadyToPlay.'
terminating with uncaught exception of type NSException
```

原来是：自身代码逻辑问题。播放结束会播放下一首导致会resetAudio，从而上一个AVPlayerItem被销毁。因此seek崩溃。把seekToTime提前就可以了。

```
- (void)playerDidPlayToEnd:(id<ZAEPlayerAudioPlayback>)playerManager
{
    //播放完成seek到0s处.
    [self seekToTime:0 completionHandler:nil];
    
    [self.multicastDelegate playerDidPlayToEnd:self]; //这里逻辑会播放下一首导致会resetAudio，从而AVPlayerItem被销毁。因此seek崩溃。把seekToTime提前就可以了。
}
```

##### 问题5：拿缓存的文件播放时，在某一秒滋了一声。但原URL播放则没有，难搞。

原因：缓存的文件二进制数据和原文件二进制数据有几个地方不相同导致。不知道什么原因导致。dataRequest的几个数据偏移量参数没搞明白导致存入的数据有些地方有问题导致。

##### 问题6：串行队列缓存，有时候读取本地缓存特别是0-1区间的2个字节耗时非常严重

```
执行本地请求：0-1，请求总长度：2，实际返回：2, 耗时：4830.241084ms
执行本地请求：0-1，请求总长度：2，实际返回：2, 耗时：13428.519011ms
```

这里不知道为啥读取本地文件耗时这么久（难道是串行IO队列阻塞了？），导致阻塞主线程。

原因是串行IO队列过载了。复现步骤：音频A没有缓存过，音频B已经缓存完成。播放音频A，音频A会请求数据，并往磁盘写入数据。此时切换到音频B，音频A的请求会取消，但是之前的数据还没有写完，音频B的读任务只能排在后面一直等待，直到之前A的数据都写完。这个有点难搞。

临时解决办法：将原逻辑的同步主线程读取数据改为异步子线程读取数据。

最终解决办法：采用读写锁。

这个问题在旧设备特别突出：

```
2020-12-14 16:19:52.889086+0800 AudioDemo[2449:1316027] 执行本地请求：21823488-22397262，请求总长度：573775，实际返回：573775, 耗时：20084.769011ms
2020-12-14 16:19:09.954618+0800 AudioDemo[2449:1316027] 执行本地请求：225722-4470625，请求总长度：4244904，实际返回：4244904, 耗时：34085.933089ms

2020-12-14 18:19:41.068640+0800 AudioDemo[2466:1328746] 执行本地请求：391905280-464203163，请求总长度：72297884，本地分片子请求0返回数据长度：16777216, 耗时：37972.537994ms
2020-12-14 18:19:41.231954+0800 AudioDemo[2466:1328746] 执行本地请求：391905280-464203163，请求总长度：72297884，本地分片子请求1返回数据长度：16777216, 耗时：38135.836959ms
2020-12-14 18:19:41.382457+0800 AudioDemo[2466:1328746] 执行本地请求：391905280-464203163，请求总长度：72297884，本地分片子请求2返回数据长度：16777216, 耗时：38286.347032ms
2020-12-14 18:19:41.518178+0800 AudioDemo[2466:1328746] 执行本地请求：391905280-464203163，请求总长度：72297884，本地分片子请求3返回数据长度：16777216, 耗时：38422.080040ms
2020-12-14 18:19:41.565657+0800 AudioDemo[2466:1328746] 执行本地请求：391905280-464203163，请求总长度：72297884，本地分片子请求4返回数据长度：5189020, 耗时：38469.568014ms
```

##### 问题7：代理系统请求，iOS10.3.3系统。开始播放后，直接seek到结尾处，会播放失败，而系统的则会正常播放完成。与此同时在请求完成之后会卡UI，直到提示播放失败才不卡UI。在此期间还不能切歌，即使切到下一首了，还会导致下一首也失败。难搞。

临时解决办法：当直接seek到末尾时，取消代理系统请求。这个时候走的就是系统自己的，于是正常播放完成。感觉很trick。（已真正解决）

还是代理系统请求模块有问题，如果不取消之前的请求，就不会有这个问题。但如果不取消之前的请求，就会造成请求数量过多的问题。

最终解决方案：限制最大请求数量为2个，超过则取消较早的请求，问题得到解决。根本原因还是iOS10系统didCancelLoadingRequest不被调用。

```
2020-12-10 12:12:01.581702+0800 AudioDemo[1320:781576] 执行远程请求：38074245-38075436，请求总长度：1192
2020-12-10 12:12:01.582358+0800 AudioDemo[1320:781576] weakSelf：(null) error:cancelled
2020-12-10 12:12:01.580476+0800 AudioDemo[1320:781600] [] __nwlog_err_simulate_crash simulate crash already simulated "tcp_connection_write called with null connection"
2020-12-10 12:12:01.583532+0800 AudioDemo[1320:781600] [] tcp_connection_write called with null connection, dumping backtrace:
        [arm64] libnetcore-856.60.1
    0   libsystem_network.dylib             0x000000018c26d2ac __nw_create_backtrace_string + 116
    1   libnetwork.dylib                    0x0000000199e4d49c tcp_connection_write + 264
    2   CFNetwork                           0x000000018d9b9a3c <redacted> + 248
    3   CFNetwork                           0x000000018da26228 <redacted> + 1424
    4   libdispatch.dylib                   0x00000001015e5a50 _dispatch_call_block_and_release + 24
    5   libdispatch.dylib                   0x00000001015e5a10 _dispatch_client_callout + 16
    6   libdispatch.dylib                   0x00000001015f32e8 _dispatch_queue_serial_drain + 1140
    7   libdispatch.dylib                   0x00000001015e9634 _dispatch_queue_invoke + 852
    8   libdispatch.dylib                   0x00000001015f5630 _dispatch_root_queue_drain + 552
    9   libdispatch.dylib                   0x00000001015f539c _dispatch_worker_thread3 + 140
    10  libsystem_pthread.dylib             0x000000018c2c7100 _pthread_wqthread + 1096
    11  libsystem_pthread.dylib             0x000000018c2c6cac start_wqthread + 4
2020-12-10 12:12:02.700802+0800 AudioDemo[1320:781576] weakSelf：<ZAEResourceRequestOperation: 0x17409a810> error:(null)
2020-12-10 12:12:02.700887+0800 AudioDemo[1320:781576] 本次请求序列以远程请求结束
2020-12-10 12:12:02.701170+0800 AudioDemo[1320:781576] 下载完成,error:(null),移除LoadingRequest：0x170008580,当前：
loadingRequests：(
    "<AVAssetResourceLoadingRequest: 0x170008580, URL request = <NSMutableURLRequest: 0x170008550> { URL: https-mine://zhenai4saylove-1251661065.file.myqcloud.com/psychology/2019/08/3628348681546230.mp3 }, request ID = 30, content information request = (null), data request = <AVAssetResourceLoadingDataRequest: 0x170008720, requested offset = 38074245, requested length = 1192, requests all data to end of resource = YES, current offset = 38075437>>"
)
dataOperationDict：{
    0x170008580 = "<ZAEResourceRequestOperation: 0x17409a810>";
}
2020-12-10 12:12:02.701235+0800 AudioDemo[1320:781576] <ZAEResourceRequestOperation: 0x17409a810>销毁
2020-12-10 12:13:05.106830+0800 AudioDemo[1320:781576] 播放失败,error:Error Domain=AVFoundationErrorDomain Code=-11819 "Cannot Complete Action" UserInfo={NSLocalizedDescription=Cannot Complete Action, NSLocalizedRecoverySuggestion=Try again later.}
```

##### 问题8：代理系统请求，有时候系统会一直发起请求--取消请求，持续10多秒，然后才完成。

因为iOS10 didCancelLoadingRequest不被调用，自己的逻辑在addLoadingRequest时会先cancel掉之前所有的请求，这一处理导致在iOS10上出现上述问题。改为不取消所有请求而是限制最大请求数后，该问题也得以解决。

##### 问题9：代理系统请求，对于长音频（1.5h），弱网频繁seek会播放失败，并且在快播放失败前会卡死UI直到真的失败。

```
2020-12-12 11:41:36.428330+0800 AudioDemo[1251:456747] 执行远程请求：8448-523239423，请求总长度：523230976，op:<ZAEResourceRequestOperation: 0x17408ffa0, remoteDataTask:0x102142ed0>
2020-12-12 11:41:37.180955+0800 AudioDemo[1251:456747] 响应头 code:206, Operation：<ZAEResourceRequestOperation: 0x17408ffa0, remoteDataTask:0x102142ed0>
2020-12-12 11:41:49.944471+0800 AudioDemo[1251:456747] TouchUpOutside:0.185000,  seekTime:1008.450553
2020-12-12 11:41:49.945338+0800 AudioDemo[1251:456747] seek cmTime
{48405626/48000 = 1008.451, rounded}
2020-12-12 11:41:50.261768+0800 AudioDemo[1251:456747] TouchUpOutside:0.392500,  seekTime:2139.550541   //开始卡
2020-12-12 11:41:50.262010+0800 AudioDemo[1251:456747] seekToTime 1008.450553,是否完成:0
2020-12-12 11:41:59.297397+0800 AudioDemo[1251:456747] seek cmTime
{1283730/600 = 2139.550, rounded} //结束卡，卡了9s
2020-12-12 11:41:59.318434+0800 AudioDemo[1251:456747] TouchUpOutside:0.582500,  seekTime:3175.256326
2020-12-12 11:41:59.318567+0800 AudioDemo[1251:456747] seekToTime 2139.550541,是否完成:0
2020-12-12 11:41:59.318817+0800 AudioDemo[1251:456747] seek cmTime
{1905153/600 = 3175.255, rounded}
2020-12-12 11:42:00.268563+0800 AudioDemo[1251:456747] 播放失败,error:Error Domain=AVFoundationErrorDomain Code=-11819 "Cannot Complete Action" UserInfo={NSLocalizedDescription=Cannot Complete Action, NSLocalizedRecoverySuggestion=Try again later.}
2020-12-12 11:42:00.277536+0800 AudioDemo[1251:456747] 取消all LoadingRequest，当前：
loadingRequests：(
    "<AVAssetResourceLoadingRequest: 0x17401e4a0, URL request = <NSMutableURLRequest: 0x17401f490> { URL: http-mine://1251661065.vod2.myqcloud.com/98deaa00vodgzp1251661065/023401157447398156618029358/MfWImeetaJgA.wav }, request ID = 5, content information request = (null), data request = <AVAssetResourceLoadingDataRequest: 0x17401eab0, requested offset = 8448, requested length = 523230976, requests all data to end of resource = NO, current offset = 1854360>>"
)
dataOperationDict：{
    0x17401e4a0 = "<ZAEResourceRequestOperation: 0x17408ffa0, remoteDataTask:0x102142ed0>";
}
2020-12-12 11:42:00.277707+0800 AudioDemo[1251:456747] 取消请求序列，当前op：<ZAEResourceRequestOperation: 0x17408ffa0, remoteDataTask:0x102142ed0>
2020-12-12 11:42:00.293349+0800 AudioDemo[1251:456747] <ZAEResourceRequestOperation: 0x17408ffa0, remoteDataTask:0x102142ed0>销毁
2020-12-12 11:42:00.293988+0800 AudioDemo[1251:456747] seekToTime 3175.256326,是否完成:0
2020-12-12 11:42:00.294096+0800 AudioDemo[1251:456747] <ZAEResourceLoader: 0x17024fb10>销毁
```

##### 问题10：播放一个527M，8分54秒的视频，当完全缓存后，下一次播放时会从本地读取，但是由于请求的range是0-end，导致一次性读取整个文件大小到内存，5s直接OOM了。非常难搞。

解决步骤：

1.渐进式读取本地数据，每次读取1M（最初是16M-->10M-->1M）（必须）

2.加@autoreleasepool （必须）

3.本地读取操作取消功能 （必须）

必须实现本地读取操作的取消功能，要不然如果有多个范围很大的本地读取请求s1--end，如果不能取消很快就会OOM。

串行队列下，看下面的一段日志：后面堆积的读取任务结束时已经耗时到131662ms了，该过程中还会导致视频无法播放的问题（确定问题，频繁seek一段完全缓存的视频，就会导致播放不了，要等一会才能播放）。

```
2020-12-15 23:19:31:198 AudioDemo:-[ZAEResourceLoader requestOperation:didCompleteWithError:],[Line 91]:
下载完成,error:(null),移除Loading Request：0x174012300,op:<ZAEResourceRequestOperation: 0x17008fe10, loadingRequest:0x174012300, remoteDataTask:0x0>
2020-12-15 23:19:31:198 AudioDemo:-[ZAEResourceRequestOperation dealloc],[Line 33]:
<ZAEResourceRequestOperation: 0x17008fe10, loadingRequest:0x174012300, remoteDataTask:0x0>销毁
2020-12-15 23:19:31:199 AudioDemo:-[ZAEResourceRequestOperation dequeueRequestRanges:]_block_invoke,[Line 149]:
执行本地请求完成:<ZAEResourceRequestOperation: 0x174091580, loadingRequest:0x174012390, remoteDataTask:0x0>：427687936-427753471，请求总长度：65536，本地请求返回数据长度：65536, 耗时：131662.903070ms
2020-12-15 23:19:31:199 AudioDemo:-[ZAEResourceRequestOperation dequeueRequestRanges:]_block_invoke,[Line 153]:
本次请求序列：<ZAEResourceRequestOperation: 0x174091580, loadingRequest:0x174012390, remoteDataTask:0x0>以本地请求结束
2020-12-15 23:19:31.199296+0800 AudioDemo[3343:713854] *** -[AVAssetResourceLoadingRequest finishLoading] was sent to an instance of AVAssetResourceLoadingRequest that was already finished. Ignoring.
2020-12-15 23:19:31:201 AudioDemo:-[ZAEResourceLoader requestOperation:didCompleteWithError:],[Line 91]:
下载完成,error:(null),移除Loading Request：0x174012390,op:<ZAEResourceRequestOperation: 0x174091580, loadingRequest:0x174012390, remoteDataTask:0x0>
2020-12-15 23:19:31:201 AudioDemo:-[ZAEResourceRequestOperation dealloc],[Line 33]:
<ZAEResourceRequestOperation: 0x174091580, loadingRequest:0x174012390, remoteDataTask:0x0>销毁
2020-12-15 23:19:31.390683+0800 AudioDemo[3343:713942] 读取427884544--553363383数据完成!!!
2020-12-15 23:19:31:393 AudioDemo:-[ZAEResourceRequestOperation dequeueRequestRanges:]_block_invoke,[Line 149]:
执行本地请求完成:<ZAEResourceRequestOperation: 0x174091260, loadingRequest:0x170014ca0, remoteDataTask:0x0>：427884544-553363383，请求总长度：125478840，本地请求返回数据长度：125478840, 耗时：131847.939014ms
```

4.本地读取操作的队列选择

**选取串行队列**。以上三步完成后，再配合串行队列就可以解决OOM的问题。但是串行队列有性能问题，经过不断的测试发现串行队列容易引起性能问题，特别是缓存了一部分，又要下载一部分，导致需要同时写入和读取，但貌似总是写入操作抢夺成功，如下日志：

```
2020-12-16 10:11:43.589060+0800 AudioDemo[2769:1572353] 读取405733376--405733375数据完成!!!
2020-12-16 10:11:43:589 AudioDemo:-[ZAEResourceRequestOperation dequeueRequestRanges:]_block_invoke,[Line 150]:
执行本地请求完成:(null), error:Error Domain=com.zhenai.emotioncounsel Code=-999 "取消操作" UserInfo={NSLocalizedDescription=取消操作},请求范围：405733376-405798911，请求总长度：65536，本地请求返回数据长度：0, 耗时：29618.003011ms
2020-12-16 10:11:43:589 AudioDemo:-[ZAEResourceRequestOperation dequeueRequestRanges:]_block_invoke,[Line 154]:
本次请求序列：(null)以本地请求结束
2020-12-16 10:11:43.590776+0800 AudioDemo[2769:1572353] 读取406323200--406388735数据完成!!!
2020-12-16 10:11:43:591 AudioDemo:-[ZAEResourceRequestOperation dequeueRequestRanges:]_block_invoke,[Line 150]:
执行本地请求完成:<ZAEResourceRequestOperation: 0x1702ae760, loadingRequest:0x174016cd0, remoteDataTask:0x0>, error:(null),请求范围：406323200-406388735，请求总长度：65536，本地请求返回数据长度：65536, 耗时：23951.740026ms
2020-12-16 10:11:43:610 AudioDemo:-[ZAEResourceRequestOperation dequeueRequestRanges:]_block_invoke,[Line 154]:
本次请求序列：<ZAEResourceRequestOperation: 0x1702ae760, loadingRequest:0x174016cd0, remoteDataTask:0x0>以本地请求结束
2020-12-16 10:11:43:611 AudioDemo:-[ZAEResourceLoader requestOperation:didCompleteWithError:],[Line 91]:
下载完成,error:(null),移除Loading Request：0x174016cd0,op:<ZAEResourceRequestOperation: 0x1702ae760, loadingRequest:0x174016cd0, remoteDataTask:0x0>
2020-12-16 10:11:43:611 AudioDemo:-[ZAEResourceRequestOperation dealloc],[Line 34]:
<ZAEResourceRequestOperation: 0x1702ae760, loadingRequest:0x174016cd0, remoteDataTask:0x0>销毁
2020-12-16 10:11:44.498601+0800 AudioDemo[2769:1572353] 读取427819008--539092859数据完成!!!
2020-12-16 10:11:44:666 AudioDemo:-[ZAEResourceRequestOperation dequeueRequestRanges:]_block_invoke,[Line 150]:
执行本地请求完成:<ZAEResourceRequestOperation: 0x1742b63e0, loadingRequest:0x174017470, remoteDataTask:0x0>, error:(null),请求范围：427819008-539092859，请求总长度：111273852，本地请求返回数据长度：111273852, 耗时：17000.121951ms
2020-12-16 10:11:44:667 AudioDemo:-[ZAEResourceRequestOperation dequeueRequestRanges:],[Line 166]:
```

**选取并发队列**。需要加上product-consume模式，不能无限制的生产，因为系统调用取消操作有时不会那么及时。

另外并发队列只是减轻了串行队列的性能问题，但没有完全解决，因为如果是部分缓存部分需要下载的情况，那么还是有可能导致读操作被阻塞。

1：5s，完全缓存，并发队列，读数据耗时

```
2020-12-27 10:58:53.554633+0800 AudioDemo[4832:1236190] 本地请求完成:<ZAEResourceRequestOperation: 0x1700b4dc0, loadingRequest:0x17401e030, remoteDataTask:0x0, localDataTask:0x174220d40, isCancelled:NO>, error:(null), 请求范围：118685696-118751231，请求总长度：65536，本地请求返回数据长度：65536, 耗时：11.551023ms
2020-12-27 10:58:51.081115+0800 AudioDemo[4832:1236301] 本地请求完成:<ZAEResourceRequestOperation: 0x1740b3fe0, loadingRequest:0x17401cd60, remoteDataTask:0x0, localDataTask:0x17403fa00, isCancelled:NO>, error:(null), 请求范围：7340032-553363383，请求总长度：546023352，本地请求返回数据长度：546023352, 耗时：5453.183055ms
2020-12-27 10:58:54.157903+0800 AudioDemo[4832:1236346] 本地请求完成:<ZAEResourceRequestOperation: 0x1740b4100, loadingRequest:0x17401e220, remoteDataTask:0x0, localDataTask:0x1742208c0, isCancelled:NO>, error:(null), 请求范围：305594368-305659903，请求总长度：65536，本地请求返回数据长度：65536, 耗时：39.137959ms
2020-12-27 10:58:55.466520+0800 AudioDemo[4832:1236348] 本地请求完成:<ZAEResourceRequestOperation: 0x1700b4a60, loadingRequest:0x17401e190, remoteDataTask:0x0, localDataTask:0x1742224e0, isCancelled:NO>, error:(null), 请求范围：450035712-450101247，请求总长度：65536，本地请求返回数据长度：65536, 耗时：10.172963ms
2020-12-27 10:59:27.158853+0800 AudioDemo[4832:1236410] 本地请求完成:<ZAEResourceRequestOperation: 0x1700b9680, loadingRequest:0x17001d6b0, remoteDataTask:0x0, localDataTask:0x174229d80, isCancelled:NO>, error:(null), 请求范围：104267776-406978559，请求总长度：302710784，本地请求返回数据长度：302710784, 耗时：3870.321989ms
2020-12-27 10:59:31.458737+0800 AudioDemo[4832:1236190] 本地请求完成:<ZAEResourceRequestOperation: 0x1740b8a80, loadingRequest:0x17001d760, remoteDataTask:0x0, localDataTask:0x174229e00, isCancelled:NO>, error:(null), 请求范围：118751232-553363383，请求总长度：434612152，本地请求返回数据长度：434612152, 耗时：4573.812008ms
2020-12-27 10:59:39.325969+0800 AudioDemo[4832:1236410] 本地请求完成:<ZAEResourceRequestOperation: 0x1700b9740, loadingRequest:0x17001d7a0, remoteDataTask:0x0, localDataTask:0x174229640, isCancelled:NO>, error:(null), 请求范围：125239296-553363383，请求总长度：428124088，本地请求返回数据长度：428124088, 耗时：5554.340005ms
2020-12-27 10:59:42.040332+0800 AudioDemo[4832:1236190] 本地请求完成:<ZAEResourceRequestOperation: 0x1740b9560, loadingRequest:0x17401ef10, remoteDataTask:0x0, localDataTask:0x17422a060, isCancelled:NO>, error:(null), 请求范围：129040384-553363383，请求总长度：424323000，本地请求返回数据长度：424323000, 耗时：5944.846988ms
```

2：5s，部分缓存，并发队列，读数据耗时

```
2020-12-27 11:10:52.500520+0800 AudioDemo[4835:1237791] 本地请求完成:<ZAEResourceRequestOperation: 0x1740b4fa0, loadingRequest:0x174011bd0, remoteDataTask:0x0, localDataTask:0x17422e420, isCancelled:NO>, error:(null), 请求范围：293984-1516801，请求总长度：1222818，本地请求返回数据长度：1222818, 耗时：24.111986ms
2020-12-27 11:14:48.790028+0800 AudioDemo[4835:1238458] 本地请求完成:<ZAEResourceRequestOperation: 0x1740b82a0, loadingRequest:0x174012360, remoteDataTask:0x113df48e0, localDataTask:0x17042b5e0, isCancelled:NO>, error:(null), 请求范围：349831168-349860103，请求总长度：28936，本地请求返回数据长度：28936, 耗时：22693.680048ms
2020-12-27 11:14:49.307740+0800 AudioDemo[4835:1238458] 本地请求完成:<ZAEResourceRequestOperation: 0x1740b82a0, loadingRequest:0x174012360, remoteDataTask:0x113df5830, localDataTask:0x174230c00, isCancelled:NO>, error:(null), 请求范围：350224384-350240095，请求总长度：15712，本地请求返回数据长度：15712, 耗时：80.983043ms
2020-12-27 11:14:53.432821+0800 AudioDemo[4835:1238410] 本地请求完成:<ZAEResourceRequestOperation: 0x1740b82a0, loadingRequest:0x174012360, remoteDataTask:0x113e51370, localDataTask:0x17042c7a0, isCancelled:NO>, error:(null), 请求范围：353042432-353053727，请求总长度：11296，本地请求返回数据长度：11296, 耗时：2354.549050ms
2020-12-27 11:14:54.806763+0800 AudioDemo[4835:1238458] 本地请求完成:<ZAEResourceRequestOperation: 0x1702a14a0, loadingRequest:0x174013a50, remoteDataTask:0x0, localDataTask:0x17423c340, isCancelled:NO>, error:(null), 请求范围：187564032-188415999，请求总长度：851968，本地请求返回数据长度：851968, 耗时：21.830082ms
2020-12-27 11:14:55.613672+0800 AudioDemo[4835:1238458] 本地请求完成:<ZAEResourceRequestOperation: 0x1740bec60, loadingRequest:0x174013cd0, remoteDataTask:0x0, localDataTask:0x170427f20, isCancelled:NO>, error:(null), 请求范围：197001216-197066751，请求总长度：65536，本地请求返回数据长度：65536, 耗时：8.183002ms
2020-12-27 11:14:54.917223+0800 AudioDemo[4835:1238524] 本地请求完成:(null), error:(null), 请求范围：190185472-190251007，请求总长度：65536，本地请求返回数据长度：65536, 耗时：29.088974ms
2020-12-27 11:14:56.980426+0800 AudioDemo[4835:1238461] 本地请求完成:<ZAEResourceRequestOperation: 0x1740bca40, loadingRequest:0x170016300, remoteDataTask:0x0, localDataTask:0x1742316a0, isCancelled:NO>, error:(null), 请求范围：296353792-296419327，请求总长度：65536，本地请求返回数据长度：65536, 耗时：120.229006ms
2020-12-27 11:15:24.526798+0800 AudioDemo[4835:1238410] 本地请求完成:<ZAEResourceRequestOperation: 0x1702a4e60, loadingRequest:0x170017e80, remoteDataTask:0x0, localDataTask:0x174425ca0, isCancelled:NO>, error:(null), 请求范围：474873856-474939391，请求总长度：65536，本地请求返回数据长度：65536, 耗时：63.163042ms
2020-12-27 11:15:24.547313+0800 AudioDemo[4835:1238585] 本地请求完成:<ZAEResourceRequestOperation: 0x1702a5280, loadingRequest:0x174015040, remoteDataTask:0x0, localDataTask:0x17042e5e0, isCancelled:NO>, error:(null), 请求范围：475201536-475267071，请求总长度：65536，本地请求返回数据长度：65536, 耗时：2.254963ms
```

和读写锁一样，如果需要等待的话，读取操作就慢一些。

在播放未缓存视频时，突然切换下一首，有时候读操作很耗时：概率还比较大。

```
2020-12-27 11:35:16.467090+0800 AudioDemo[4835:1240861] 本地请求完成:<ZAEResourceRequestOperation: 0x1702b8a20, loadingRequest:0x17401d120, remoteDataTask:0x0, localDataTask:0x174632700, isCancelled:NO>, error:(null), 请求范围：0-1，请求总长度：2，本地请求返回数据长度：2, 耗时：3814.440966ms
```

有时候读操作会很慢：

```
2020-12-27 11:55:37.796402+0800 AudioDemo[4835:1244386] 执行本地请求：<NSOperation: 0x174a35000>开始：157810688-157822031，请求总长度：11344，op:<ZAEResourceRequestOperation: 0x1744bad60, loadingRequest:0x174204e20, remoteDataTask:0x115c02d60, localDataTask:0x174a35000, isCancelled:NO>
2020-12-27 11:55:53.277064+0800 AudioDemo[4835:1237693] 等待播放，网络出现问题
2020-12-27 11:56:55.503979+0800 AudioDemo[4835:1244100] 读取157810688-157822031数据完成!!!,请求长度：11344,实际返回：11344,是否取消：0，总耗时:77707.650900ms
2020-12-27 11:56:55.505475+0800 AudioDemo[4835:1244100] 本地请求完成:<ZAEResourceRequestOperation: 0x1744bad60, loadingRequest:0x174204e20, remoteDataTask:0x115c02d60, localDataTask:0x174a35000, isCancelled:NO>, error:(null), 请求范围：157810688-157822031，请求总长度：11344，本地请求返回数据长度：11344, 耗时：77708.058000ms
```

应该是一直在等写操作完成。使用读写锁目前没有看到有等待这么久的。总体来说读写锁还是优于barrier。

新问题：

偶现声音正常但画面卡住。

**选取读写锁**。需要要加上product-consume模式，否则如果渐进式读取选择一次10M，依然会OOM，如果选择1M，虽然不会OOM（偶尔也会彪很高300-400M有OOM的风险），但频繁seek会明显感觉到滑动条很卡，因为CPU消耗太严重。加上product-consume模式后，频繁seek卡的问题，可以得到很好的解决。效果非常棒，避免了读饥饿或写饥饿。

又遇到新的问题：

频繁seek偶尔会发生死锁，主线程无响应。已解决。

无缓存情况下频繁seek偶尔出现画面正常但无声音。

1：5s，完全缓存，读写锁，读数据耗时

```
2020-12-27 09:11:02.886212+0800 AudioDemo[4803:1220972] 本地请求完成:<ZAEResourceRequestOperation: 0x1740b4ca0, loadingRequest:0x174009450, remoteDataTask:0x0, localDataTask:0x17403cb20, isCancelled:NO>, error:(null), 请求范围：242155520-349831167，请求总长度：107675648，本地请求返回数据长度：107675648, 耗时：1206.562042ms
2020-12-27 09:11:05.629485+0800 AudioDemo[4803:1220857] 本地请求完成:<ZAEResourceRequestOperation: 0x1700b4400, loadingRequest:0x1740094f0, remoteDataTask:0x0, localDataTask:0x170226c00, isCancelled:NO>, error:(null), 请求范围：272826368-553363383，请求总长度：280537016，本地请求返回数据长度：280537016, 耗时：2723.701000ms
2020-12-27 09:11:08.503392+0800 AudioDemo[4803:1220857] 本地请求完成:(null), error:Error Domain=com.zhenai.emotioncounsel Code=-999 "取消操作" UserInfo={NSLocalizedDescription=取消操作}, 请求范围：273154048-553363383，请求总长度：280209336，本地请求返回数据长度：1048576, 耗时：59.594989ms
2020-12-27 09:11:09.036530+0800 AudioDemo[4803:1220985] 本地请求完成:<ZAEResourceRequestOperation: 0x1740b53c0, loadingRequest:0x17000de30, remoteDataTask:0x0, localDataTask:0x170226160, isCancelled:NO>, error:(null), 请求范围：277217280-277282815，请求总长度：65536，本地请求返回数据长度：65536, 耗时：8.244991ms
```

可以看到如果没有写数据的操作，仅仅是读的话其实还是挺快的，65536byte只需要8ms。

2：5s，部分缓存，读写锁，读数据耗时

```
2020-12-27 09:33:38.384659+0800 AudioDemo[4809:1224349] 本地请求完成:<ZAEResourceRequestOperation: 0x1740b77c0, loadingRequest:0x17000a490, remoteDataTask:0x10193a480, localDataTask:0x1700311c0, isCancelled:NO>, error:(null), 请求范围：11665408-11730943，请求总长度：65536，本地请求返回数据长度：65536, 耗时：54.841042ms
2020-12-27 09:33:40.679697+0800 AudioDemo[4809:1224484] 本地请求完成:<ZAEResourceRequestOperation: 0x1740b77c0, loadingRequest:0x17000a490, remoteDataTask:0x101844140, localDataTask:0x17403ad60, isCancelled:NO>, error:(null), 请求范围：12910592-12976127，请求总长度：65536，本地请求返回数据长度：65536, 耗时：109.176993ms
2020-12-27 09:33:40.996406+0800 AudioDemo[4809:1224451] 本地请求完成:<ZAEResourceRequestOperation: 0x1740b77c0, loadingRequest:0x17000a490, remoteDataTask:0x101941760, localDataTask:0x17003a980, isCancelled:NO>, error:(null), 请求范围：13238272-13303807，请求总长度：65536，本地请求返回数据长度：65536, 耗时：140.053034ms
2020-12-27 09:33:43.090620+0800 AudioDemo[4809:1224501] 本地请求完成:<ZAEResourceRequestOperation: 0x1740b77c0, loadingRequest:0x17000a490, remoteDataTask:0x101843cb0, localDataTask:0x170033dc0, isCancelled:NO>, error:(null), 请求范围：15007744-15104799，请求总长度：97056，本地请求返回数据长度：97056, 耗时：236.387014ms
2020-12-27 09:34:14.196842+0800 AudioDemo[4809:1224583] 本地请求完成:<ZAEResourceRequestOperation: 0x1700b6440, loadingRequest:0x174007870, remoteDataTask:0x0, localDataTask:0x17403cf00, isCancelled:NO>, error:(null), 请求范围：163708928-163774463，请求总长度：65536，本地请求返回数据长度：65536, 耗时：69.718003ms
2020-12-27 09:37:39.098472+0800 AudioDemo[4809:1225406] 本地请求完成:<ZAEResourceRequestOperation: 0x1740b77c0, loadingRequest:0x17000b750, remoteDataTask:0x0, localDataTask:0x17022e280, isCancelled:NO>, error:(null), 请求范围：177995776-190151099，请求总长度：12155324，本地请求返回数据长度：12155324, 耗时：108.645082ms
```

部分缓存时，一些数据就需要下载及写数据到磁盘，导致读操作有时需要等待，65536byte就需要50-150ms。

10.1新问题，OOM是解决了，但是CPU一直居高不下。

10.2新问题，并发队列，在无seek操作下，从头到尾的播放无缓存的视频，内存会缓慢增长一度增长到400M直到网络问题被暂停了才回落到70M。如果是播放完全缓存的则没有这个情况，内存一直在24M左右。

原来后面的请求超时失败了，但前面的一个请求一直在下载，一直在下载。

或者当前请求没有问题，一直在下载，播放器也一直在正常播放，就会导致内存一直缓慢增长直到播放完成才又缓慢回落。

10.3页面vc有一个属性A是单例，然后调用A的一个耗时方法，然后点击vc返回，问vc会不会被销毁？

会马上销毁，但A的耗时任务仍然在进行，执行完后依然会调用complete回调。其实就跟在页面内发起了一个网络请求一样的。

10.4 偶尔出现

```
2020-12-15 22:29:40.859833+0800 AudioDemo[3260:707689] *** -[AVAssetResourceLoadingRequest finishLoading] was sent to an instance of AVAssetResourceLoadingRequest that was already finished. Ignoring.
```

增加判断即可。

##### 问题11：播放视频时，频繁seek，偶尔出现声音正常但画面卡住或画面正常声音没有的情况。非常难搞。

视频播放非常棘手的问题：声音正常播放但画面卡住或黑屏无画面，画面正常播放但无声音，声画都有但不同步。

经过比对原始视频和缓存视频的区间数据，都是一致的。说明保存区间数据的代码没有问题。而从头至尾播放缓存视频播放到缓存数据没有问题，但是如果seek到未缓存的地方开始播放就会出现声画不同步的情况，感觉应该是seek到临界缓存数据时，音频和视频无法同步导致。以目前的水平无法解决。

11.1 iPhone6plus iOS10.3.3 最大请求2，读写锁，无seek正常播放，有时候会播着播着画面正常但无声音，播了一会可能又恢复正常，卡了一会，继续播放时又画面正常但无声音。

应该还是请求取消策略问题。系统无seek正常播放没有发现这种情况。系统在iPhone6plus iOS10.3.3 上发起的请求，并不是一直取消最前面的。

##### 问题12：响应头信息的回调代码有问题，在iOS10.3.3 iPhone6plus上卡主线程。越旧的机器越明显。

```
2020-12-14 15:42:26.767489+0800 AudioDemo[2433:1309838] --------响应头 code:206, Operation：<ZAEResourceRequestOperation: 0x174093bf0, remoteDataTask:0x14bdacc40>
2020-12-14 15:42:32.476195+0800 AudioDemo[2433:1309838] +++++++++++响应头 code:206, Operation：<ZAEResourceRequestOperation: 0x174093bf0, remoteDataTask:0x14bdacc40>
```

直接卡了6秒。

问题代码：

```objc
__weak typeof(self) weakSelf = self;
[_httpSessionManager setDataTaskDidReceiveResponseBlock:^NSURLSessionResponseDisposition(NSURLSession * _Nonnull session, NSURLSessionDataTask * _Nonnull dataTask, NSURLResponse * _Nonnull response) {
    //子线程
    __block NSURLSessionResponseDisposition disposition = NSURLSessionResponseCancel;
    if (NSThread.isMainThread) {
        disposition = [weakSelf dataTaskDidReceiveResponseWithSession:session dataTask:dataTask response:response];
    } else {
        dispatch_sync(dispatch_get_main_queue(), ^{
            disposition = [weakSelf dataTaskDidReceiveResponseWithSession:session dataTask:dataTask response:response];
        });
    }
    return disposition;
}];
        
- (NSURLSessionResponseDisposition)dataTaskDidReceiveResponseWithSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask response:(NSURLResponse *)response {
    NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
    NSLog(@"--------响应头 code:%d, Operation：%@",httpResponse.statusCode, self);
    
    NSDictionary *headers = [httpResponse allHeaderFields];
    NSDictionary *caseInsensitiveHeaders = [self caseInsensitiveKeyWithDict:headers];
    if (httpResponse.statusCode >= 200 && httpResponse.statusCode < 300) {
        ZAEResourceContentInfoModel *contentInfo = [self contentInfoModelWithHeaderFields:caseInsensitiveHeaders];
        if (![self.mediaCache diskContentInfoExistsWithKey:self.resourceURL.absoluteString]) {
            [self.mediaCache storeContentInfoSync:contentInfo forKey:self.resourceURL.absoluteString];
        }
        self.contentInfo = contentInfo;
        if (self.delgate && [self.delgate respondsToSelector:@selector(requestOperation:didLoadContentInfo:)]) {
            [self.delgate requestOperation:self didLoadContentInfo:contentInfo];
        }
        
        NSLog(@"+++++++++++响应头 code:%d, Operation：%@",httpResponse.statusCode, self);
        return NSURLSessionResponseAllow;
    }
    
    return NSURLSessionResponseCancel;
}
```

主线程要等IO队列操作完才能返回，所以卡住了。

改为：

```
__weak typeof(self) weakSelf = self;
[_httpSessionManager setDataTaskDidReceiveResponseBlock:^NSURLSessionResponseDisposition(NSURLSession * _Nonnull session, NSURLSessionDataTask * _Nonnull dataTask, NSURLResponse * _Nonnull response) {
    //子线程
    NSHTTPURLResponse *httpResponse = (NSHTTPURLResponse *)response;
    NSLog(@"响应头 code:%ld, Operation：%@", httpResponse.statusCode, weakSelf);

    NSDictionary *headers = [httpResponse allHeaderFields];
    NSDictionary *caseInsensitiveHeaders = [weakSelf caseInsensitiveKeyWithDict:headers];
    if (httpResponse.statusCode >= 200 && httpResponse.statusCode < 300) {
        ZAEResourceContentInfoModel *contentInfo = [weakSelf contentInfoModelWithHeaderFields:caseInsensitiveHeaders];
        if (![weakSelf.mediaCache diskContentInfoExistsWithKey:weakSelf.resourceURL.absoluteString]) {
            [weakSelf.mediaCache storeContentInfoSync:contentInfo forKey:weakSelf.resourceURL.absoluteString];
        }
        weakSelf.contentInfo = contentInfo;

        dispatch_async(dispatch_get_main_queue(), ^{
            [weakSelf dataTaskDidReceiveResponseWithSession:session dataTask:dataTask contentInfo:contentInfo];
        });

        return NSURLSessionResponseAllow;
    }

    return NSURLSessionResponseCancel;
}];
```

解决。

##### 问题13：因为iOS10系统，同一时刻可能会存在两个请求，而下载的却是重复的数据段。这里待优化。



##### 问题14：5s,iOS10.3.3系统。读写锁，播放1.5h的音频，疯狂seek快进又快退（特别是网速还不好的情况），直接卡死UI，有时候能够恢复有时隔10s左右等到真播放失败后又自行恢复了。

后面直接用系统播放的也一样，疯狂seek快进又快退，也会突然卡死，直到真播放失败后又自行恢复了。

```
http://1251661065.vod2.myqcloud.com/98deaa00vodgzp1251661065/023401157447398156618029358/MfWImeetaJgA.wav
```

日志：

```
2020-12-17 23:02:18.843146+0800 AudioDemo[4001:832133] 执行本地请求：<NSOperation: 0x174034ce0>开始：127729664-160825343，请求总长度：33095680，op:<ZAEResourceRequestOperation: 0x1700a7ce0, loadingRequest:0x174014710, remoteDataTask:0x0, localDataTask:0x174034ce0, isCancelled:NO>
2020-12-17 23:02:18.843948+0800 AudioDemo[4001:832133] 远程请求完成, error:cancelled, weakSelf：(null)，dataTask:<__NSCFLocalDataTask: 0x13fd44110>{ taskIdentifier: 1 } { completed }
2020-12-17 23:02:19.032940+0800 AudioDemo[4001:832220] 响应头 code:206, Operation：<ZAEResourceRequestOperation: 0x1740a7e60, loadingRequest:0x1700176f0, remoteDataTask:0x13fe6a560, localDataTask:0x174034f80, isCancelled:NO>
2020-12-17 23:02:28.090267+0800 AudioDemo[4001:831634] TouchUpOutside:0.192500,  seekTime:1049.333644
2020-12-17 23:02:28.090407+0800 AudioDemo[4001:831634] seekToTime 1331.078636,是否完成:0
2020-12-17 23:02:28.120611+0800 AudioDemo[4001:831634] seek cmTime
{629600/600 = 1049.333, rounded}
2020-12-17 23:02:28.128934+0800 AudioDemo[4001:831634] seekToTime 1049.333644,是否完成:0
2020-12-17 23:02:28.129970+0800 AudioDemo[4001:831634] TouchUpOutside:0.250000,  seekTime:1362.771000
2020-12-17 23:02:28.130234+0800 AudioDemo[4001:831634] seek cmTime
{817662/600 = 1362.770, rounded}
2020-12-17 23:02:28.132094+0800 AudioDemo[4001:831634] TouchUpOutside:0.413953,  seekTime:2256.495208
2020-12-17 23:02:28.132215+0800 AudioDemo[4001:831634] seekToTime 1362.771000,是否完成:0
2020-12-17 23:02:28.132857+0800 AudioDemo[4001:831634] seek cmTime
{1353897/600 = 2256.495, rounded}
2020-12-17 23:02:28.134917+0800 AudioDemo[4001:831634] TouchUpOutside:0.646512,  seekTime:3524.189117
2020-12-17 23:02:28.135034+0800 AudioDemo[4001:831634] seekToTime 2256.495208,是否完成:0
2020-12-17 23:02:28.135441+0800 AudioDemo[4001:831634] seek cmTime
{2114513/600 = 3524.188, rounded}
2020-12-17 23:02:29.023626+0800 AudioDemo[4001:831634] 播放失败,error:Error Domain=AVFoundationErrorDomain Code=-11819 "Cannot Complete Action" UserInfo={NSLocalizedDescription=Cannot Complete Action, NSLocalizedRecoverySuggestion=Try again later.}
```

问题14.1 读写锁缓存，iphone7plus，iOS14.2 上播放视频，疯狂seek会卡死UI。卡死时CPU为0，内存也正常。

直接系统播放，不管怎样seek不会卡死也不会声画不同步。

视频链接：

```
http://1251661065.vod2.myqcloud.com/98deaa00vodgzp1251661065/c6200b325285890794913485240/saGb5NcsJREA.mp4
```

已经确定是读写锁死锁了。

死锁代码：

```objc
- (ZAERangeRecordTable *)rangeRecordTableFromCacheForKey:(NSString *)key {
    DDLogError(@"读区间表,将要加读锁,线程:%@", [NSThread currentThread]);
    int lockRet = pthread_rwlock_rdlock(&self->_rwlock);
//    int lockRet = pthread_rwlock_tryrdlock(&self->_rwlock);
    if (lockRet == EDEADLK) {
        NSAssert(NO, @"死锁了！！！");
    } else {
        DDLogError(@"读区间表,加读锁code:%d,线程:%@", lockRet, [NSThread currentThread]);
    }
    ZAERangeRecordTable *table = [self diskRangeRecordTableForKey:key];
    DDLogError(@"读区间表,将要解读锁,线程:%@", [NSThread currentThread]);
    pthread_rwlock_unlock(&self->_rwlock);
    return table;
}

- (NSOperation *)mediaDataWithStartOffset:(unsigned long long)startOffset length:(unsigned long long)numberOfBytesToRespondWith forKey:(nonnull NSString *)key didLoadData:(nullable void (^)(NSInteger, NSData * _Nonnull))didLoadDataBlock completion:(nullable void (^)(BOOL, unsigned long long, NSError * _Nullable))completionBlock {
    if (didLoadDataBlock == nil) {
        if (completionBlock) {
            completionBlock(YES, numberOfBytesToRespondWith, nil);
        }
        return [NSOperation new];
    }
    CFAbsoluteTime startTime = CFAbsoluteTimeGetCurrent();
    NSOperation *operation = [NSOperation new];
    dispatch_async(_ioQueue, ^{
    
        DDLogError(@"读mediaData,将要加读锁,线程:%@", [NSThread currentThread]);
        int lockRet = pthread_rwlock_rdlock(&self->_rwlock);
        if (lockRet == EDEADLK) {
            NSAssert(NO, @"死锁了！！！");
        } else {
            DDLogError(@"读mediaData,加读锁code:%d,线程:%@", lockRet, [NSThread currentThread]);
        }
        
        NSString *cachePathForKey = [self defaultCachePathForKey:key];
        NSFileHandle *readingFileHandle = [NSFileHandle fileHandleForReadingAtPath:cachePathForKey];
        
        unsigned long long off = startOffset;
        unsigned long long total = numberOfBytesToRespondWith;
        unsigned long long numberOfBytesResponded = 0;
        NSInteger sliceIndex = 0;
        NSError *error = nil;
        DDLogError(@"1111");
        while (total > 0) {
            DDLogError(@"2222");
//            dispatch_semaphore_wait(self.pcSemaphore, dispatch_time(DISPATCH_TIME_NOW, (int64_t)(30 * NSEC_PER_SEC)));
            dispatch_semaphore_wait(self.pcSemaphore, DISPATCH_TIME_FOREVER);
            DDLogError(@"3333");
            if (operation.isCancelled) {
                DDLogError(@"44444");
                dispatch_semaphore_signal(self.pcSemaphore);
                error = [NSError errorWithCode:-999 msg:@"取消操作"];
                break;
            }
            unsigned long long currReadLength = total;
            if (currReadLength > kMaxLocalDataPerPage) {
                currReadLength = kMaxLocalDataPerPage;
            }
            
            @autoreleasepool {
                NSData *mediaData = [self mediaDataWithStartOffset:off length:currReadLength readingFileHandle:readingFileHandle error:&error];
                if (error) {
                    DDLogError(@"55555");
                    dispatch_semaphore_signal(self.pcSemaphore);
                    break;
                }
                if (didLoadDataBlock) {
                    dispatch_async(dispatch_get_main_queue(), ^{
                        didLoadDataBlock(sliceIndex, mediaData);
                        dispatch_semaphore_signal(self.pcSemaphore);
                    });
                } else {
                    dispatch_semaphore_signal(self.pcSemaphore);
                    NSAssert(NO, @"咋回事啊");
                }
            }
            
            off += currReadLength;
            total -= currReadLength;
            sliceIndex += 1;
            numberOfBytesResponded += currReadLength;
        }
        [readingFileHandle closeFile];

        if (operation.isCancelled && error == nil) {
            error = [NSError errorWithCode:-999 msg:@"取消操作"];
        }
        
        CFAbsoluteTime linkTime = (CFAbsoluteTimeGetCurrent() - startTime);
        DDLogError(@"读取%lld-%lld数据完成!!!,请求长度：%lld,实际返回：%lld,是否取消：%d，总耗时:%fms", startOffset, startOffset + numberOfBytesToRespondWith - 1, numberOfBytesToRespondWith, numberOfBytesResponded, operation.isCancelled, linkTime * 1000.0);
        DDLogError(@"读mediaData,将要解读锁,线程:%@", [NSThread currentThread]);
        pthread_rwlock_unlock(&self->_rwlock);
        
        if (completionBlock) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completionBlock(error == nil ? YES : NO, numberOfBytesResponded, error);
            });
        }
    });
    return operation;
}
```

打印：

```
2020-12-26 11:59:28.421577+0800 AudioDemo[28718:2344773] start op:<ZAEResourceRequestOperation: 0x283d4a4c0, loadingRequest:0x281b68ce0, remoteDataTask:0x0, localDataTask:0x0, isCancelled:NO>, 原始请求：255655936-365690879
2020-12-26 11:59:28.425667+0800 AudioDemo[28718:2344773] 执行本地请求：<NSOperation: 0x282562500>开始：255655936-260357955，请求总长度：4702020，op:<ZAEResourceRequestOperation: 0x283d4a4c0, loadingRequest:0x281b68ce0, remoteDataTask:0x0, localDataTask:0x282562500, isCancelled:NO>
2020-12-26 11:59:28.426131+0800 AudioDemo[28718:2344649] 读mediaData,将要加读锁,线程:<NSThread: 0x280c85f40>{number = 214, name = (null)}
2020-12-26 11:59:28.426539+0800 AudioDemo[28718:2344649] 写MediaData,将要加写锁,线程:<NSThread: 0x28037ce00>{number = 179, name = (null)}
2020-12-26 11:59:28.426747+0800 AudioDemo[28718:2344649] 读mediaData,加读锁code:0,线程:<NSThread: 0x280c85f40>{number = 214, name = (null)}
2020-12-26 11:59:28.427955+0800 AudioDemo[28718:2344649] 1111
2020-12-26 11:59:28.428102+0800 AudioDemo[28718:2344649] 2222
2020-12-26 11:59:28.428217+0800 AudioDemo[28718:2344649] 3333
2020-12-26 11:59:28.430432+0800 AudioDemo[28718:2344645] 写MediaData,将要加写锁,线程:<NSThread: 0x280cc2b00>{number = 155, name = (null)}
2020-12-26 11:59:28.434168+0800 AudioDemo[28718:2344616] 2222
2020-12-26 11:59:28.435470+0800 AudioDemo[28718:2344645] 远程请求完成, error:(null), weakSelf：<ZAEResourceRequestOperation: 0x283d4b2a0, loadingRequest:0x281b5b4c0, remoteDataTask:0x11f6d5f50, localDataTask:0x0, isCancelled:NO>，dataTask:LocalDataTask <769FF02C-435E-41E4-A568-FEEC36578C70>.<1>
2020-12-26 11:59:28.435743+0800 AudioDemo[28718:2344540] 3333
2020-12-26 11:59:28.435915+0800 AudioDemo[28718:2344540] 下载完成,error:(null),移除Loading Request：0x281b5b4c0, op:<ZAEResourceRequestOperation: 0x283d4b2a0, loadingRequest:0x281b5b4c0, remoteDataTask:0x11f6d5f50, localDataTask:0x0, isCancelled:NO>
2020-12-26 11:59:28.436225+0800 AudioDemo[28718:2344540] <ZAEResourceRequestOperation: 0x283d4b2a0, loadingRequest:0x281b5b4c0, remoteDataTask:0x11f6d5f50, localDataTask:0x0, isCancelled:NO>销毁
2020-12-26 11:59:28.439967+0800 AudioDemo[28718:2344645] 2222
2020-12-26 11:59:28.440811+0800 AudioDemo[28718:2344540] 3333
2020-12-26 11:59:28.444460+0800 AudioDemo[28718:2344540] 2222
2020-12-26 11:59:28.444742+0800 AudioDemo[28718:2344645] add Loading Request:0x281b5abf0，op：<ZAEResourceRequestOperation: 0x283d391a0, loadingRequest:0x281b5abf0, remoteDataTask:0x0, localDataTask:0x0, isCancelled:NO>
当前：loadingRequests：(
    "<AVAssetResourceLoadingRequest: 0x281b68ce0, URL request = <NSMutableURLRequest: 0x281b69b40> { URL: http-mine://1251661065.vod2.myqcloud.com/98deaa00vodgzp1251661065/c6200b325285890794913485240/saGb5NcsJREA.mp4 }, request ID = 1376, content information request = (null), data request = <AVAssetResourceLoadingDataRequest: 0x281b69c60, requested offset = 255655936, requested length = 110034944, requests all data to end of resource = NO, current offset = 257753088>>",
    "<AVAssetResourceLoadingRequest: 0x281b5abf0, URL request = <NSMutableURLRequest: 0x281b5b760> { URL: http-mine://1251661065.vod2.myqcloud.com/98deaa00vodgzp1251661065/c6200b325285890794913485240/saGb5NcsJREA.mp4 }, request ID = 1378, content information request = (null), data request = <AVAssetResourceLoadingDataRequest: 0x281b5b5b0, requested offset = 255000576, requested length = 65536, requests all data to end of resource = NO, current offset = 255000576>>"
)
dataOperationDict：{
    0x281b5abf0 = "<ZAEResourceRequestOperation: 0x283d391a0, loadingRequest:0x281b5abf0, remoteDataTask:0x0, localDataTask:0x0, isCancelled:NO>";
    0x281b68ce0 = "<ZAEResourceRequestOperation: 0x283d4a4c0, loadingRequest:0x281b68ce0, remoteDataTask:0x0, localDataTask:0x282562500, isCancelled:NO>";
}
2020-12-26 11:59:28.445280+0800 AudioDemo[28718:2344645] 读区间表,将要加读锁,线程:<NSThread: 0x280c78440>{number = 1, name = main}
```

后再无其他日志。莫名其妙的死锁。初步怀疑是和pcSemaphore信号量共同导致，注释掉信号量相关代码试了很久也没有死锁了。

此时如果断点，堆栈信息：

<img src="https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20201220152927.png" style="zoom:50%;" />

查看日志发现加写锁和解写锁是配对的。最后一次加锁是读mediaData,加的读锁。然而诡异的是为什么此时主线程去加读锁却没有成功，导致进入休眠等待状态。而读mediaData方法里线程还等着主线程发送信号量唤醒，但由于主线程已经休眠，于是读mediaData子线程也就没法继续执行，也没法解读锁，写线程也就进入等待。

这里搞不懂的是读写锁不是可以同时加读锁的吗？为啥主线程却阻塞了。

如果把信号量改为只等待30s,将会看到主线程在加读锁时也等待了30s，直到读mediaData线程解锁。

```
2020-12-26 12:45:01.874347+0800 AudioDemo[28970:2361196] add Loading Request:0x28345e810，op：<ZAEResourceRequestOperation: 0x28137c8a0, loadingRequest:0x28345e810, remoteDataTask:0x0, localDataTask:0x0, isCancelled:NO>
当前：loadingRequests：(
    "<AVAssetResourceLoadingRequest: 0x28345e4e0, URL request = <NSMutableURLRequest: 0x28345e470> { URL: http-mine://1251661065.vod2.myqcloud.com/98deaa00vodgzp1251661065/c6200b325285890794913485240/saGb5NcsJREA.mp4 }, request ID = 2644, content information request = (null), data request = <AVAssetResourceLoadingDataRequest: 0x28345e450, requested offset = 111673344, requested length = 71106560, requests all data to end of resource = NO, current offset = 160956416>>",
    "<AVAssetResourceLoadingRequest: 0x28345e810, URL request = <NSMutableURLRequest: 0x28345e290> { URL: http-mine://1251661065.vod2.myqcloud.com/98deaa00vodgzp1251661065/c6200b325285890794913485240/saGb5NcsJREA.mp4 }, request ID = 2646, content information request = (null), data request = <AVAssetResourceLoadingDataRequest: 0x28345ea10, requested offset = 112001024, requested length = 65536, requests all data to end of resource = NO, current offset = 112001024>>"
)
dataOperationDict：{
    0x28345e4e0 = "<ZAEResourceRequestOperation: 0x28137c780, loadingRequest:0x28345e4e0, remoteDataTask:0x0, localDataTask:0x280a5be00, isCancelled:NO>";
    0x28345e810 = "<ZAEResourceRequestOperation: 0x28137c8a0, loadingRequest:0x28345e810, remoteDataTask:0x0, localDataTask:0x0, isCancelled:NO>";
}
2020-12-26 12:45:01.874819+0800 AudioDemo[28970:2361196] 读区间表,将要加读锁,线程:<NSThread: 0x282340780>{number = 1, name = main}
2020-12-26 12:45:01.874905+0800 AudioDemo[28970:2361196] 2222


2020-12-26 12:45:31.880032+0800 AudioDemo[28970:2361156] 3333
2020-12-26 12:45:31.881952+0800 AudioDemo[28970:2361156] 读取111673344-162301823数据完成!!!,请求长度：50628480,实际返回：50628480,是否取消：0，总耗时:30072.767019ms
2020-12-26 12:45:31.883266+0800 AudioDemo[28970:2361156] 读mediaData,将要解读锁,线程:<NSThread: 0x282c7f540>{number = 438, name = (null)}
2020-12-26 12:45:31.884302+0800 AudioDemo[28970:2361156] 写MediaData,加写锁code:0,线程:<NSThread: 0x282c77a80>{number = 446, name = (null)}
2020-12-26 12:45:31.906985+0800 AudioDemo[28970:2361156] 写MediaData,将要解写锁,线程:<NSThread: 0x282c77a80>{number = 446, name = (null)}
2020-12-26 12:45:31.907870+0800 AudioDemo[28970:2361216] 写MediaData,加写锁code:0,线程:<NSThread: 0x2823c1180>{number = 419, name = (null)}
2020-12-26 12:45:31.924607+0800 AudioDemo[28970:2361156] 写MediaData,将要解写锁,线程:<NSThread: 0x2823c1180>{number = 419, name = (null)}
2020-12-26 12:45:31.925254+0800 AudioDemo[28970:2361150] 读区间表,加读锁code:0,线程:<NSThread: 0x282340780>{number = 1, name = main}
2020-12-26 12:45:31.928500+0800 AudioDemo[28970:2361150] 读区间表,将要解读锁,线程:<NSThread: 0x282340780>{number = 1, name = main}
```

难道是因为读mediaData线程加读锁后，后面又跟了两个写MediaData的线程在等着，此时主线程来了，想加读锁，因为写线程优先级高一些，系统想让写线程先执行于是让读加锁的主线程先等着呢？我感觉应该是这样子，要不然写线程就饿死了。事实确实如此。还是没有理解好pthread_rwlock_rdlock的机制。

**pthread_rwlock_rdlock**

如果当前**没有写线程持有锁**或者**没有写线程阻塞在取锁**时则可以立刻获取到锁，否则请求锁的线程阻塞等待。

一直怀疑是某次写线程没有解写锁导致读线程等待。然后有一次发现日志加写锁和解写锁次数不一致，查了半个多小时，原来是乌龙事件，有一行打印被串了：

```
2020-12-24 23:02:13.240206+0800 AudioDemo[22805:1868765] <ZAEResourceRequestOperation: 0x282a3a8e0, loadingRequest:0x280c18690, remoteDataTask:0x1062c0300, localDataTask:0x0, isCancelled:YES>销毁
2020-12-24 23:02:13.240650+0800 AudioDemo[22805:1868729] 写2020-12-24 23:02:13.241134+0800 AudioDemo[22805:1868765] 远程请求完成, error:cancelled, weakSelf：(null)，dataTask:LocalDataTask <F70C9F6C-22D1-494E-8085-BAB6F2023F2C>.<1>
加锁MediaData code将要解锁，线程：<NSThread: 0x281b87c80>{number = 143, name = (null)} //串的
2020-12-24 23:02:13.241281+0800 AudioDemo[22805:1868728] 写加锁MediaData code:0,线程：<NSThread: 0x281b86c40>{number = 142, name = (null)}
2020-12-24 23:02:13.245148+0800 AudioDemo[22805:1868728] 写加锁MediaData code将要解锁，线程：<NSThread: 0x281b86c40>{number = 142, name = (null)}
```

看来急需一个不会被打断输出的日志工具。

至此，这个问题终于找到原因，这个问题大概花了我3-4天时间查找原因。

如果当前在写模式，然后等待线程为写写写写读读读读，系统会怎样调度？会让前面的写线程都完成才开始读，还是说会穿插执行？

参考：

[一次读锁重入导致的死锁故障](https://blog.thinkeridea.com/201812/go/yi_ci_du_suo_chong_ru_dao_zhi_de_si_suo_gu_zhang.html)

[Code that debugs itself: Fixing a deadlock with a watchdog](https://medium.com/bluecore-engineering/code-that-debugs-itself-fixing-a-deadlock-with-a-watchdog-cd83019cce2e)

去掉所有同步方法的读写锁后，没有出现卡死的现象。但是会有声画不同步的问题。

其他试过的办法：本来还想用子线程来发送信号量解决死锁，结果一样很容易导致读写锁死锁，不过这一次死锁发生后主线程不受影响。

```
if (didLoadDataBlock) {
    dispatch_async(dispatch_get_main_queue(), ^{
        didLoadDataBlock(sliceIndex, mediaData);
        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
            dispatch_semaphore_signal(pcSemaphore);
        });
    });
}
```

不是很明白为啥还会死锁。

最终解决办法：不使用信号量，而是使用一个bool变量进行标记，每次循环开始时先检测变量的值，当didLoadDataBlock回调完成后才修改变量的值。从根本上解决了死锁的可能。

```objc
__block BOOL isFeed = YES;
while (total > 0) {
    if (operation.isCancelled) {
        error = [NSError errorWithCode:-999 msg:@"取消操作"];
        break;
    }
    if (!isFeed) {
        continue;
    }

    unsigned long long currReadLength = total;
    if (currReadLength > kMaxLocalDataPerPage) {
        currReadLength = kMaxLocalDataPerPage;
    }

    @autoreleasepool {
        NSData *mediaData = [self mediaDataWithStartOffset:off length:currReadLength readingFileHandle:readingFileHandle error:&error];
        if (error) {
            break;
        }
        if (didLoadDataBlock) {
            isFeed = NO;
            dispatch_async(dispatch_get_main_queue(), ^{
                didLoadDataBlock(sliceIndex, mediaData);
                isFeed = YES;
            });
        }
    }

    off += currReadLength;
    total -= currReadLength;
    sliceIndex += 1;
    numberOfBytesResponded += currReadLength;
}
```



##### 问题15：5s,iOS10.3.3系统。读写锁缓存，播放视频，loadingRequest都请求完成了，但是系统过了15s才又发起请求，导致缓冲已经完成了，但却没有播放。

```
2020-12-20 19:45:46.828565+0800 AudioDemo[4238:941768] 下载完成,error:(null),移除Loading Request：0x1740128b0, op:<ZAEResourceRequestOperation: 0x1702b49a0, loadingRequest:0x1740128b0, remoteDataTask:0x15b264910, localDataTask:0x17062b2c0, isCancelled:NO>
2020-12-20 19:45:46.828816+0800 AudioDemo[4238:941775] <ZAEResourceRequestOperation: 0x1702b49a0, loadingRequest:0x1740128b0, remoteDataTask:0x15b264910, localDataTask:0x17062b2c0, isCancelled:NO>销毁

---这里差了15s才又发起请求------

2020-12-20 19:46:01.878824+0800 AudioDemo[4238:941897] add Loading Request:0x17001bbf0，op：<ZAEResourceRequestOperation: 0x1702b5540, loadingRequest:0x17001bbf0, remoteDataTask:0x0, localDataTask:0x0, isCancelled:NO>
当前：loadingRequests：(
    "<AVAssetResourceLoadingRequest: 0x17001bbf0, URL request = <NSMutableURLRequest: 0x170012250> { URL: http-mine://vt1.doubanio.com/202001021917/01b91ce2e71fd7f671e226ffe8ea0cda/view/movie/M/301120229.mp4 }, request ID = 1254, content information request = (null), data request = <AVAssetResourceLoadingDataRequest: 0x170014bd0, requested offset = 11075584, requested length = 524288, requests all data to end of resource = NO, current offset = 11075584>>"
)
dataOperationDict：{
    0x17001bbf0 = "<ZAEResourceRequestOperation: 0x1702b5540, loadingRequest:0x17001bbf0, remoteDataTask:0x0, localDataTask:0x0, isCancelled:NO>";
}
2020-12-20 19:46:01.881738+0800 AudioDemo[4238:941897] start op:<ZAEResourceRequestOperation: 0x1702b5540, loadingRequest:0x17001bbf0, remoteDataTask:0x0, localDataTask:0x0, isCancelled:NO>, 原始请求：11075584-11599871
已缓存区间列表：
(
    "type:1,start:0,end:1589387,length:1589388",
    "type:1,start:7864320,end:22397262,length:14532943"
)
拆分请求列表：
(
    "type:1,start:11075584,end:11599871,length:524288"
)
```

##### 问题16：iOS10.3.3 iPhone6plus，最大3请求，播放视频偶现服务器503错误 “java.net.SocketException: Too many open files”。然后视频就播不了了。

等了一会后，服务器又好了。应该还是请求取消策略问题。

##### 问题17：清理缓存时必须确保rta文件和音频文件一起删除。否则如果遗留了rta文件没有删除则会导致实际从本地获取不到数据而播放失败，这个问题还是比较严重的。

解决办法：

1.当清理到规定的最大磁盘缓存的一半以下后，继续查询是否遗漏了rta文件没有删，如果有则继续删除。cia文件和其他两个文件没有关联关系，有没有同时删除没有关系。

2.做好步骤1后，已经在很大程度上降低了问题的出现。但是如果是系统自动清理的Cache目录，则不能保证，因此在拆分请求序列后，请求本地数据如果失败，那么先检查音频文件是否存在，如果不存在则清除url相关的所有缓存，同时将该失败的本地请求转为远程请求继续请求。

##### 问题18：iOS10.3.3 iPhone6plus，正常播放时，一边下载一边缓存，发现内存在缓慢增长，如果网速稳定，感觉如果不下载完，内存会一直缓慢增长，目前观察到会增长到320M。

貌似系统达到一定程度后会发起一些新请求，然后我这边有做控制最大3请求，超出后把它取消，内存就回落了。直接系统播放内存几乎无变化维持在23M左右。

在接收数据的回调方法中加入autoreleasepool也没有用。

```objc
- (void)dataTaskDidReceiveDataWithSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask data:(NSData *)data {
//    DDLogError(@"接收数据LoadingRequest：%ld，Operation：%@", self.loadingRequest.hash, self);
    @autoreleasepool {
        [self.mediaCache storeMediaDataWithContentInfo:self.contentInfo
                                         currentOffset:self.loadingRequest.dataRequest.currentOffset
                                                  data:data
                                                forKey:self.resourceURL.absoluteString
                                            completion:nil];
        
        if (self.delgate && [self.delgate respondsToSelector:@selector(requestOperation:didReceiveData:)]) {
            [self.delgate requestOperation:self didReceiveData:data];
        }
    }
}
```



### 参考

[Audio streaming and caching in iOS using AVAssetResourceLoader and AVPlayer](https://www.codeproject.com/Articles/875105/Audio-streaming-and-caching-in-iOS-using)

[iOS音视频实现边下载边播放](http://sky-weihao.github.io/2015/10/06/Video-streaming-and-caching-in-iOS/)

[AVPlayer支持的视频格式](https://codertw.com/%E7%A8%8B%E5%BC%8F%E8%AA%9E%E8%A8%80/690067/)  原出处： [AVPlayer支持的视频格式](https://www.jianshu.com/p/7373f07f1cbf)

[音视频封装格式、编码格式](https://www.jianshu.com/p/def926938398)  

[iOS AVPlayer 视频缓存的设计与实现](http://chuquan.me/2019/12/03/ios-avplayer-support-cache/)

[可能是目前最好的 AVPlayer 音视频缓存方案](https://mp.weixin.qq.com/s/v1sw_Sb8oKeZ8sWyjBUXGA)

[AVPlayer 视频缓存](https://mochangxing.github.io/2018/08/11/AVPlayer%E7%BC%93%E5%AD%98%E7%9A%84%E5%AE%9E%E7%8E%B0/)  架构图挺好的

[iOS音视频开发-----流媒体](https://blog.csdn.net/szk972092933/article/details/82771631)  系统提供的一种缓存办法，有iOS版本限制。

[iOS 获取视频的任意一帧](https://blog.csdn.net/kevinpake/article/details/25626587)

[IOS音视频（四十五）HTTPS 自签名证书 实现边下边播](https://juejin.cn/post/6844904115416334343)  比较完整的介绍

[AVPlayer 边下边播与最佳实践](https://zltunes.github.io/2019/04/23/avplayer-best-practice/)

[基于AVPlayer实现音视频播放和缓存，支持视频画面的同步输出](https://caisanze.com/post/swift-avplayer/)

[如果视频拖动快进这时候会有声音但画面卡住了](https://github.com/vitoziv/VIMediaCache/issues/19)  非常棘手的问题

[iOS性能优化实践：头条抖音如何实现OOM崩溃率下降50%+](https://www.mdeditor.tw/pl/ptm8)  稍微看下方案

[点播中的流量成本优化](http://www.samirchen.com/video-playback-bandwidth-cost/) 可以看一下。博主很不错，音视频的可以看看。

[腾讯研发总监王辉：十亿级视频播放技术优化揭秘](https://mp.weixin.qq.com/s?__biz=MjM5MDE0Mjc4MA==&mid=2650997049&idx=1&sn=079d954687944e74778df58f31078bd3&chksm=bdbef96a8ac9707cd094f3737f6a32ce849066f50857b0b260fc2804e2eda22d3f6452b3cbfa&scene=27#wechat_redirect)   可以看下大公司的实现方案以及其他考量。当然源码、细节、关键坑等肯定就没有了。

[NicooM3u8Downloader](https://cocoapods.org/pods/NicooM3u8Downloader)  m3u8的缓存	



一次典型的请求:从创建到执行到取消到销毁。

```
2020-12-17 10:24:31.398959+0800 AudioDemo[4018:1759248] add Loading Request:0x17000d970，op：<ZAEResourceRequestOperation: 0x1740aa4a0, loadingRequest:0x17000d970, remoteDataTask:0x0, localDataTask:0x0, isCancelled:NO>
当前：loadingRequests：(
    "<AVAssetResourceLoadingRequest: 0x17000c950, URL request = <NSMutableURLRequest: 0x17000c990> { URL: http-mine://1251661065.vod2.myqcloud.com/98deaa00vodgzp1251661065/c6200b325285890794913485240/saGb5NcsJREA.mp4 }, request ID = 31, content information request = (null), data request = <AVAssetResourceLoadingDataRequest: 0x17000dbe0, requested offset = 0, requested length = 553363384, requests all data to end of resource = YES, current offset = 8456368>>",
    "<AVAssetResourceLoadingRequest: 0x17000d970, URL request = <NSMutableURLRequest: 0x17400fc70> { URL: http-mine://1251661065.vod2.myqcloud.com/98deaa00vodgzp1251661065/c6200b325285890794913485240/saGb5NcsJREA.mp4 }, request ID = 32, content information request = (null), data request = <AVAssetResourceLoadingDataRequest: 0x17400fc30, requested offset = 6356992, requested length = 547006392, requests all data to end of resource = YES, current offset = 6356992>>"
)
dataOperationDict：{
    0x17000c950 = "<ZAEResourceRequestOperation: 0x1700abe80, loadingRequest:0x17000c950, remoteDataTask:0x102024cb0, localDataTask:0x170224640, isCancelled:NO>";
    0x17000d970 = "<ZAEResourceRequestOperation: 0x1740aa4a0, loadingRequest:0x17000d970, remoteDataTask:0x0, localDataTask:0x0, isCancelled:NO>";
}
2020-12-17 10:24:31.407478+0800 AudioDemo[4018:1758891] start op:<ZAEResourceRequestOperation: 0x1740aa4a0, loadingRequest:0x17000d970, remoteDataTask:0x0, localDataTask:0x0, isCancelled:NO>, 原始请求：6356992-553363383
已缓存区间列表：
(
    "type:1,start:0,end:8456367,length:8456368",
    "type:1,start:63045632,end:63067393,length:21762",
    "type:1,start:65863680,end:65929919,length:66240",
    "type:1,start:111542272,end:162518719,length:50976448",
    "type:1,start:168820736,end:168886271,length:65536",
    "type:1,start:192741376,end:192760695,length:19320",
    "type:1,start:200015872,end:200031051,length:15180",
    "type:1,start:200933376,end:200945795,length:12420",
    "type:1,start:202178560,end:202179939,length:1380",
    "type:1,start:212664320,end:212698887,length:34568",
    "type:1,start:224722944,end:224788479,length:65536",
    "type:1,start:225312768,end:225356159,length:43392",
    "type:1,start:241565696,end:241593295,length:27600",
    "type:1,start:250347520,end:250348899,length:1380",
    "type:1,start:327483392,end:327498911,length:15520",
    "type:1,start:336855040,end:336896439,length:41400",
    "type:1,start:337575936,end:337615955,length:40020",
    "type:1,start:339345408,end:339375767,length:30360",
    "type:1,start:388956160,end:389021695,length:65536",
    "type:1,start:389873664,end:389902643,length:28980",
    "type:1,start:393871360,end:393936895,length:65536",
    "type:1,start:394330112,end:394419431,length:89320",
    "type:1,start:400687104,end:401080319,length:393216",
    "type:1,start:401145856,end:447009547,length:45863692",
    "type:1,start:489291776,end:489314779,length:23004",
    "type:1,start:490143744,end:490146503,length:2760"
)
拆分请求列表：
(
    "type:1,start:6356992,end:8456367,length:2099376",
    "type:0,start:8456368,end:63045631,length:54589264",
    "type:1,start:63045632,end:63067393,length:21762",
    "type:0,start:63067394,end:65863679,length:2796286",
    "type:1,start:65863680,end:65929919,length:66240",
    "type:0,start:65929920,end:111542271,length:45612352",
    "type:1,start:111542272,end:162518719,length:50976448",
    "type:0,start:162518720,end:168820735,length:6302016",
    "type:1,start:168820736,end:168886271,length:65536",
    "type:0,start:168886272,end:192741375,length:23855104",
    "type:1,start:192741376,end:192760695,length:19320",
    "type:0,start:192760696,end:200015871,length:7255176",
    "type:1,start:200015872,end:200031051,length:15180",
    "type:0,start:200031052,end:200933375,length:902324",
    "type:1,start:200933376,end:200945795,length:12420",
    "type:0,start:200945796,end:202178559,length:1232764",
    "type:1,start:202178560,end:202179939,length:1380",
    "type:0,start:202179940,end:212664319,length:10484380",
    "type:1,start:212664320,end:212698887,length:34568",
    "type:0,start:212698888,end:224722943,length:12024056",
    "type:1,start:224722944,end:224788479,length:65536",
    "type:0,start:224788480,end:225312767,length:524288",
    "type:1,start:225312768,end:225356159,length:43392",
    "type:0,start:225356160,end:241565695,length:16209536",
    "type:1,start:241565696,end:241593295,length:27600",
    "type:0,start:241593296,end:250347519,length:8754224",
    "type:1,start:250347520,end:250348899,length:1380",
    "type:0,start:250348900,end:327483391,length:77134492",
    "type:1,start:327483392,end:327498911,length:15520",
    "type:0,start:327498912,end:336855039,length:9356128",
    "type:1,start:336855040,end:336896439,length:41400",
    "type:0,start:336896440,end:337575935,length:679496",
    "type:1,start:337575936,end:337615955,length:40020",
    "type:0,start:337615956,end:339345407,length:1729452",
    "type:1,start:339345408,end:339375767,length:30360",
    "type:0,start:339375768,end:388956159,length:49580392",
    "type:1,start:388956160,end:389021695,length:65536",
    "type:0,start:389021696,end:389873663,length:851968",
    "type:1,start:389873664,end:389902643,length:28980",
    "type:0,start:389902644,end:393871359,length:3968716",
    "type:1,start:393871360,end:393936895,length:65536",
    "type:0,start:393936896,end:394330111,length:393216",
    "type:1,start:394330112,end:394419431,length:89320",
    "type:0,start:394419432,end:400687103,length:6267672",
    "type:1,start:400687104,end:401080319,length:393216",
    "type:0,start:401080320,end:401145855,length:65536",
    "type:1,start:401145856,end:447009547,length:45863692",
    "type:0,start:447009548,end:489291775,length:42282228",
    "type:1,start:489291776,end:489314779,length:23004",
    "type:0,start:489314780,end:490143743,length:828964",
    "type:1,start:490143744,end:490146503,length:2760",
    "type:0,start:490146504,end:553363383,length:63216880"
)

执行本地请求：<NSOperation: 0x17022bf00>开始：6356992-8456367，请求总长度：2099376，op:<ZAEResourceRequestOperation: 0x1740aa4a0, loadingRequest:0x17000d970, remoteDataTask:0x0, localDataTask:0x17022bf00, isCancelled:NO>

本地请求完成:<ZAEResourceRequestOperation: 0x1740aa4a0, loadingRequest:0x17000d970, remoteDataTask:0x0, localDataTask:0x17022bf00, isCancelled:NO>, error:(null), 请求范围：6356992-8456367，请求总长度：2099376，本地请求返回数据长度：2099376, 耗时：15.187025ms

执行远程请求：<__NSCFLocalDataTask: 0x1020619d0>{ taskIdentifier: 1 } { running }开始：8456368-63045631，请求总长度：54589264，op:<ZAEResourceRequestOperation: 0x1740aa4a0, loadingRequest:0x17000d970, remoteDataTask:0x1020619d0, localDataTask:0x17022bf00, isCancelled:NO>

响应头 code:206, Operation：<ZAEResourceRequestOperation: 0x1740aa4a0, loadingRequest:0x17000d970, remoteDataTask:0x1020619d0, localDataTask:0x17022bf00, isCancelled:NO>


取消部分 Loading Request:<AVAssetResourceLoadingRequest: 0x17000d970, URL request = <NSMutableURLRequest: 0x17400fc70> { URL: http-mine://1251661065.vod2.myqcloud.com/98deaa00vodgzp1251661065/c6200b325285890794913485240/saGb5NcsJREA.mp4 }, request ID = 32, content information request = (null), data request = <AVAssetResourceLoadingDataRequest: 0x17400fc30, requested offset = 6356992, requested length = 547006392, requests all data to end of resource = YES, current offset = 9178448>> op:<ZAEResourceRequestOperation: 0x1740aa4a0, loadingRequest:0x17000d970, remoteDataTask:0x1020619d0, localDataTask:0x17022bf00, isCancelled:NO>

2020-12-17 10:24:34.468351+0800 AudioDemo[4018:1759249] 将要取消请求序列，当前op：<ZAEResourceRequestOperation: 0x1740aa4a0, loadingRequest:0x17000d970, remoteDataTask:0x1020619d0, localDataTask:0x17022bf00, isCancelled:NO>

2020-12-17 10:24:34.469033+0800 AudioDemo[4018:1759249] <ZAEResourceRequestOperation: 0x1740aa4a0, loadingRequest:0x17000d970, remoteDataTask:0x1020619d0, localDataTask:0x17022bf00, isCancelled:YES>销毁

2020-12-17 10:24:34.492829+0800 AudioDemo[4018:1759171] 远程请求完成, error:cancelled, weakSelf：(null)，dataTask:<__NSCFLocalDataTask: 0x1020619d0>{ taskIdentifier: 1 } { completed }
```



### 其他

Charles出问题，前面请求还好好的，后面突然解析不了host到IP。导致播放失败，如果此时关掉Charles就可以继续播放。

![](https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20210128173301.png)

