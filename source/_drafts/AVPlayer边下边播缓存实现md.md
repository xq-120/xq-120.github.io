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

#### 1.代理系统的请求

7天

系统请求的机制：

在无seek的情况下：先请求0-1的数据段，成功后再请求一大段数据，但只接收一小段数据就cancel掉请求。播放一小段后又请求一大段数据，但接收一小段数据后又cancel掉请求，循环这个操作直到接收所有数据。

在有seek的情况下：会取消之前的下载请求，然后从seek处发起一个新请求。

一个resource对应一个loader，一个loader管理多个loadingRequest，每个loadingRequest对应一个真正的dataTask。一个loader虽然管理多个loadingRequest，但同一时刻只能有一个dataTask下载数据。所以当接收到一个loadingRequest时必须先取消掉之前所有的loadingRequest，确保同一时刻只有一个dataTask在下载数据（这里思路错误，导致后面测试发现很多问题）。正确做法是限制最大请求数，不能全部取消。

经过测试发现在iOS10.3.3中

```
- (void)resourceLoader:(AVAssetResourceLoader *)resourceLoader didCancelLoadingRequest:(AVAssetResourceLoadingRequest *)loadingRequest API_AVAILABLE(macos(10.9), ios(7.0), tvos(9.0)) API_UNAVAILABLE(watchos);
```

didCancelLoadingRequest:方法不会被调用。而在iOS14上，AVAssetResourceLoader在开启下一个请求时，会先调用didCancelLoadingRequest:让你有机会cancel掉之前的。

1.先获取contentInfo信息，再请求数据

没必要

2.loaderDict什么时候移除loader

a.播放新音频时清空loaderDict

b.下载失败时，清除当前loader

3.loadingRequests什么时候移除loadingRequest

a.在addLoadingRequest准备发起一个下载请求时需要取消掉之前所有的loadingRequest，并清空loadingRequests。

b.下载完成后移除当前loadingRequest。

#### 2.缓存数据，并建立区间记录表

1.根据系统的请求范围，查找区间记录表，将系统请求拆分为一个请求列表。[本地请求，远程请求，本地请求...]

2.依次执行请求列表中的每个请求，并将请求到的数据供给播放器。

3.远程请求接收到数据后缓存到音频文件，并合并式更新区间记录表。

音频元数据的保存，音频文件的保存，区间记录表的保存？读写同步问题？

#### 3.缓存清除策略

TODO

#### 4.缓冲估算

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

选取串行队列。以上三步完成后，再配合串行队列就可以解决OOM的问题。但是串行队列有性能问题，经过不断的测试发现串行队列容易引起性能问题，特别是缓存了一部分，又要下载一部分，导致需要同时写入和读取，但貌似总是写入操作抢夺成功，如下日志：

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

选取并发队列。如果选择并发队列，则以上三步也不能解决OOM。还要加上product-consume模式，不能无限制的生产，因为系统调用取消操作有时不会那么及时。

另外并发队列只是减轻了串行队列的性能问题，但没有完全解决，因为如果是部分缓存部分需要下载的情况，那么还是有可能导致读操作被阻塞。

选取读写锁。效果非常棒，避免了读饥饿或写饥饿。又遇到新的问题，频繁seek偶尔会发生死锁，主线程无响应。

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

问题14.1 读写锁缓存，iphone7plus 上播放视频，疯狂seek会卡死UI。卡死时CPU为0，内存也正常。

直接系统播放，不管怎样seek不会卡死也不会声画不同步。

视频链接：

```
http://1251661065.vod2.myqcloud.com/98deaa00vodgzp1251661065/c6200b325285890794913485240/saGb5NcsJREA.mp4
```

目前猜测是读写锁死锁了？

直接断点，堆栈信息：

<img src="https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20201220152927.png" style="zoom:50%;" />

参考：

[一次读锁重入导致的死锁故障](https://blog.thinkeridea.com/201812/go/yi_ci_du_suo_chong_ru_dao_zhi_de_si_suo_gu_zhang.html)

[Code that debugs itself: Fixing a deadlock with a watchdog](https://medium.com/bluecore-engineering/code-that-debugs-itself-fixing-a-deadlock-with-a-watchdog-cd83019cce2e)

去掉所有同步方法的读写锁后，没有出现卡死的现象。但是会有声画不同步的问题。

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

