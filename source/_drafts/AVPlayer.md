AVPlayer简单使用

```
//
//  ViewController.m
//  AudioDemo
//
//  Created by 薛权 on 2018/7/20.
//  Copyright © 2018年 zhenai. All rights reserved.
//

#import "ViewController.h"
#import <AVFoundation/AVFoundation.h>

static NSString *musicUrl = @"http://sc1.111ttt.cn/2014/1/09/24/2242313311.mp3";

@interface ViewController ()

@property (weak, nonatomic) IBOutlet UIButton *playerBtn;
@property (weak, nonatomic) IBOutlet UISlider *progressSlider;
@property (weak, nonatomic) IBOutlet UIButton *recordBtn;
@property (weak, nonatomic) IBOutlet UILabel *statusLabel;
@property (weak, nonatomic) IBOutlet UILabel *currentTimeLabel;
@property (weak, nonatomic) IBOutlet UILabel *totalTimeLabel;

@property (nonatomic, strong) AVAudioPlayer *audioPlayer;
@property (nonatomic, strong) AVPlayer *player;

@property (nonatomic, copy) NSString *currentPlayUrl;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self.playerBtn setTitle:@"▷" forState:UIControlStateNormal];
    [self.playerBtn setTitle:@"||" forState:UIControlStateSelected];
    self.playerBtn.layer.cornerRadius = 25;
    self.recordBtn.layer.cornerRadius = 25;
    
    self.progressSlider.value = 0;
    self.progressSlider.maximumValue = 1;
    
    self.currentPlayUrl = musicUrl;
    
    [self.progressSlider addTarget:self action:@selector(sliderValueChanged:) forControlEvents:UIControlEventValueChanged|UIControlEventTouchUpInside];
    [self.progressSlider setThumbImage:[UIImage imageNamed:@"progress_thumb"] forState:UIControlStateNormal];
}

- (IBAction)playerBtnClicked:(UIButton *)sender {
    self.playerBtn.selected = !self.playerBtn.selected;
    
//    [self playLocalMusicWithBtnSelectedStatus:self.playerBtn.selected];
    [self playNetworkMusicWithBtnSelectedStatus:self.playerBtn.selected];
}

- (AVPlayer *)player
{
    if (_player == nil) {
        AVPlayerItem *item = [[AVPlayerItem alloc] initWithURL:[NSURL URLWithString:self.currentPlayUrl]];
        [item addObserver:self forKeyPath:@"status" options:NSKeyValueObservingOptionNew context:nil];
        [item addObserver:self forKeyPath:@"loadedTimeRanges" options:NSKeyValueObservingOptionNew context:nil];
        _player = [[AVPlayer alloc] initWithPlayerItem:item];
        if (@available(iOS 10.0, *)) {
            [_player setAutomaticallyWaitsToMinimizeStalling:NO];
        } else {
            // Fallback on earlier versions
        }
        __weak typeof(self) weakSelf = self;
        CMTime interval = CMTimeMakeWithSeconds(1, NSEC_PER_SEC);
        [_player addPeriodicTimeObserverForInterval:interval queue:dispatch_get_main_queue() usingBlock:^(CMTime time) {
            //当前播放的时间
            float current = CMTimeGetSeconds(time);
            weakSelf.currentTimeLabel.text = [weakSelf formatterTime:current];
            
            //总时间
            float total = CMTimeGetSeconds(item.duration);
            
            NSLog(@"current:%f,total:%f", current, total);
            
            if (current) {
                weakSelf.totalTimeLabel.text = [weakSelf formatterTime:total];
                float progress = current / total;
                weakSelf.progressSlider.value = progress;
            }
        }];
        
        [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(playEnd:) name:AVPlayerItemDidPlayToEndTimeNotification object:_player.currentItem];
    }
    
    return _player;
}

- (AVAudioPlayer *)audioPlayer
{
    if (_audioPlayer == nil) {
        NSString *soundPath = [[NSBundle mainBundle] pathForResource:@"平凡之路－朴树" ofType:@"mp3"];
        NSURL *soundUrl = [NSURL fileURLWithPath:soundPath];
        //初始化播放器对象
        NSError *err = nil;
        _audioPlayer = [[AVAudioPlayer alloc] initWithContentsOfURL:soundUrl error:&err];
        if (err) {
            return nil;
        }
        //设置声音的大小
        _audioPlayer.volume = 0.5;//范围为（0到1）；
        //设置循环次数，如果为负数，就是无限循环
        _audioPlayer.numberOfLoops = 0;
        //设置播放进度
        _audioPlayer.currentTime = 0;
    }
    
    return _audioPlayer;
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary*)change context:(void *)context {
    AVPlayerItem *songItem = object;
    
    if ([keyPath isEqualToString:@"status"]) {
        switch (songItem.status) {
            case AVPlayerStatusUnknown:
                NSLog(@"KVO：未知状态，此时不能播放");
                break;
            case AVPlayerStatusReadyToPlay:
            {
                NSLog(@"KVO：准备完毕，可以播放");
                self.statusLabel.text = @"准备完毕，可以播放";
                dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
                    self.statusLabel.text = nil;
                });
                break;
            }
            case AVPlayerStatusFailed:
                NSLog(@"KVO：加载失败，网络或者服务器出现问题");
                break;
            default:
                break;
        }
    }
    
    if ([keyPath isEqualToString:@"loadedTimeRanges"]) {
        NSArray *array = songItem.loadedTimeRanges;
        CMTimeRange timeRange = [array.firstObject CMTimeRangeValue]; //本次缓冲的时间范围
        NSTimeInterval totalBuffer = CMTimeGetSeconds(timeRange.start) + CMTimeGetSeconds(timeRange.duration); //缓冲总长度
        float totalTime = CMTimeGetSeconds(self.player.currentItem.duration);
        float bufferProgress = 0;
        if (totalTime > 0) {
            bufferProgress = totalBuffer/totalTime;
        }
        NSLog(@"共缓冲%.3f, 进度:%.3f",totalBuffer, bufferProgress);
    }
}

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
- (void)playNetworkMusicWithBtnSelectedStatus:(BOOL)isSelected
{
    if (isSelected) {
        [self.player play];
    } else {
        [self.player pause];
    }
}

- (NSString *)formatterTime:(float)time
{
    NSString *durationString;
    
    int seconds = MAX(0, time);
    
    int s = seconds;
    int m = s / 60;
    int h = m / 60;
    
    s = s % 60;
    m = m % 60;

    durationString = [NSString stringWithFormat:@"%02d:%02d:%02d",h,m,s];
    return durationString;
}

- (void)playEnd:(id)sender
{
    NSLog(@"播放完成");
    self.playerBtn.selected = NO;
}

- (void)playLocalMusicWithBtnSelectedStatus:(BOOL)isSelected
{
    if (isSelected) {
        //准备播放
        [self.audioPlayer prepareToPlay];
        [self.audioPlayer play];
    } else {
        [self.audioPlayer pause];
    }
}

- (IBAction)recordBtnClicked:(UIButton *)sender {
    
}

- (void)sliderValueChanged:(UISlider *)sender {
    
//    计算时间
    CMTime duration = self.player.currentItem.duration;
    Float64 second = CMTimeGetSeconds(duration);
    int time = sender.value * second;

    NSLog(@"--UISlider:%.3f  time:%d", sender.value, time);

    //搜寻到指定时间
    [self.player seekToTime:CMTimeMake(time, 1)];
}

@end

```

#### 1.接管系统的请求

7天

系统请求的机制：

在无seek的情况下：先请求0-1的数据段，成功后再请求一大段数据，但只接收一小段数据就cancel掉请求。播放一小段后又请求一大段数据，但接收一小段数据后又cancel掉请求，循环这个操作直到接收所有数据。

在有seek的情况下：会取消之前的下载请求，然后从seek处发起一个新请求。

一个resource对应一个loader，一个loader管理多个loadingRequest，每个loadingRequest对应一个真正的dataTask。一个loader虽然管理多个loadingRequest，但同一时刻只能有一个dataTask下载数据。所以当接收到一个loadingRequest时必须先取消掉之前所有的loadingRequest，确保同一时刻只有一个dataTask在下载数据。

经过测试发现在iOS10.3.3中

```
- (void)resourceLoader:(AVAssetResourceLoader *)resourceLoader didCancelLoadingRequest:(AVAssetResourceLoadingRequest *)loadingRequest API_AVAILABLE(macos(10.9), ios(7.0), tvos(9.0)) API_UNAVAILABLE(watchos);
```

didCancelLoadingRequest:方法不会被调用（加上使用缓存后，又会被调用了，真操了）。而在iOS14上，AVAssetResourceLoader在开启下一个请求时，会先调用didCancelLoadingRequest:让你有机会cancel掉之前的。为了兼容系统，当接收到一个loadingRequest时必须先取消掉之前所有的loadingRequest。

1.先获取contentInfo信息，再请求数据

没必要

2.loaderDict什么时候移除loader

a.播放新音频时清空loaderDict

b.下载失败时，清除当前loader

3.loadingRequests什么时候移除loadingRequest

a.在addLoadingRequest准备发起一个下载请求时需要取消掉之前所有的loadingRequest，并清空loadingRequests。

b.下载完成后移除当前loadingRequest。

##### 问题0：在缓存到80%的时候，出现range的left>right的现象。

```
[headers setValue:[NSString stringWithFormat:@"bytes=%lld-%ld", loadingRequest.dataRequest.requestedOffset, loadingRequest.dataRequest.requestedLength-1] forKey:@"range"];

<AVAssetResourceLoadingDataRequest: 0x281423340, requested offset = 32178176, requested length = 5897261, requests all data to end of resource = YES, current offset = 32178176>
2020-12-02 22:08:14.738839+0800 AudioDemo[19856:1539854] headers:{
    range = "bytes=32178176-5897260";
}
```

后台返回了 `416 Range Not Satisfiable` 错误。	

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

某次loader下载超时后，系统会再次发起一个新请求的。

##### 问题2：自定义scheme URL <---> 原URL互推。

在原scheme上拼接字符串，这样就可以方便互推。

##### 问题3：自定义后，iOS14上播放正常，但iOS10上却无法播放。

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

##### 问题4：自定义后，在iOS10.3.3上，播放结束后，马上seek到0会崩溃

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

#### 2.缓存数据，并建立区间记录表

1.根据系统的请求范围，查找区间记录表，将系统请求拆分为一个请求列表。[本地请求，远程请求，本地请求...]

2.依次执行请求列表中的每个请求，并将请求到的数据供给播放器。

3.远程请求接收到数据后缓存到音频文件，并合并式更新区间记录表。

音频元数据的保存，音频文件的保存，区间记录表的保存？读写同步问题？

##### 问题5：拿缓存的文件播放时，在某一秒滋了一声。但原URL播放则没有，难搞。

原因：缓存的文件二进制数据和原文件二进制数据有几个地方不相同导致。不知道什么原因导致。

##### 问题6：有时候读取本地缓存特别是0-1区间的2个字节耗时非常严重

```
执行本地请求：0-1，请求总长度：2，实际返回：2, 耗时：4830.241084ms
执行本地请求：0-1，请求总长度：2，实际返回：2, 耗时：13428.519011ms
```

这里不知道为啥读取本地文件耗时这么久（难道是串行IO队列阻塞了？），导致阻塞主线程，暂时解决办法：异步读取数据。

是串行IO队列过载了。复现步骤：音频A没有缓存过，音频B已经缓存完成。播放音频A，音频A会请求数据，并往磁盘写入数据。此时切换到音频B，音频A的请求会取消，但是之前的数据还没有写完，音频B的读任务只能排在后面一直等待，直到之前A的数据都写完。这个有点难搞。

##### 问题7：边下边播时，开始播放后，直接seek到结尾处，会播放失败，而系统的则会正常播放完成。与此同时在请求完成之后会卡UI，直到提示播放失败才不卡UI。在此期间还不能切歌，即使切到下一首了，还会导致下一首也失败。难搞。

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

暂时解决办法：当直接seek到末尾时，取消代理系统请求。这个时候走的就是系统自己的，于是正常播放完成。感觉很trick。

#### 3.缓存清除策略



参考:

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

