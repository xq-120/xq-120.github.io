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

##### 问题1：下载请求超时失败了，但是播放器没有收到任何回调。

频繁seek导致。

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

某次loader下载超时后，系统会再次发起一个新请求的。属正常情况。

##### 问题2：自定义scheme URL <---> 原URL互推。

在原scheme上拼接字符串，这样就可以方便互推。

##### 问题3：代理请求后，iOS14上播放正常，但iOS10上却无法播放。

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

原来是：播放结束会播放下一首导致会resetAudio，从而上一个AVPlayerItem被销毁。因此seek崩溃。把seekToTime提前就可以了。

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

##### 问题6：有时候读取本地缓存特别是0-1区间的2个字节耗时非常严重

```
执行本地请求：0-1，请求总长度：2，实际返回：2, 耗时：4830.241084ms
执行本地请求：0-1，请求总长度：2，实际返回：2, 耗时：13428.519011ms
```

这里不知道为啥读取本地文件耗时这么久（难道是串行IO队列阻塞了？），导致阻塞主线程。

原因是串行IO队列过载了。复现步骤：音频A没有缓存过，音频B已经缓存完成。播放音频A，音频A会请求数据，并往磁盘写入数据。此时切换到音频B，音频A的请求会取消，但是之前的数据还没有写完，音频B的读任务只能排在后面一直等待，直到之前A的数据都写完。这个有点难搞。

暂时解决办法：改为异步子线程读取数据。

这个问题在旧设备特别突出：

```
2020-12-14 16:19:52.889086+0800 AudioDemo[2449:1316027] 执行本地请求：21823488-22397262，请求总长度：573775，实际返回：573775, 耗时：20084.769011ms
2020-12-14 16:19:09.954618+0800 AudioDemo[2449:1316027] 执行本地请求：225722-4470625，请求总长度：4244904，实际返回：4244904, 耗时：34085.933089ms
```



##### 问题7：代理系统请求，iOS10系统。开始播放后，直接seek到结尾处，会播放失败，而系统的则会正常播放完成。与此同时在请求完成之后会卡UI，直到提示播放失败才不卡UI。在此期间还不能切歌，即使切到下一首了，还会导致下一首也失败。难搞。

暂时解决办法：当直接seek到末尾时，取消代理系统请求。这个时候走的就是系统自己的，于是正常播放完成。感觉很trick。（已真正解决）

还是代理系统请求模块有问题，如果不取消之前的请求，就不会有这个问题。但如果不取消之前的请求，就会造成请求数量过多的问题。

最终解决方案：限制最大请求数量为3个，超过则取消较早的请求，问题得到解决。根本原因还是iOS10系统didCancelLoadingRequest不被调用。

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

##### 问题10：播放一个527M，8分54秒的视频，当完全缓存后，下一次播放时会从本地读取，但是由于请求的range是0-end，导致一次性读取整个文件大小到内存，5s直接OOM了。难搞。

渐进式读取本地数据，每次读取16M，问题解决。

##### 问题11：弱网播放时，频繁seek，偶尔出现声音正常但画面卡住的情况。

视频播放非常棘手的问题：声音正常播放但画面卡住或黑屏无画面，画面正常播放但无声音，声画都有但不同步。

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

