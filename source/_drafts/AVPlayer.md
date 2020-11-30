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

参考:

[iOS音视频实现边下载边播放](http://sky-weihao.github.io/2015/10/06/Video-streaming-and-caching-in-iOS/)

[AVPlayer支持的视频格式](https://codertw.com/%E7%A8%8B%E5%BC%8F%E8%AA%9E%E8%A8%80/690067/)  原出处： [AVPlayer支持的视频格式](https://www.jianshu.com/p/7373f07f1cbf)

[音视频封装格式、编码格式](https://www.jianshu.com/p/def926938398)  

[iOS AVPlayer 视频缓存的设计与实现](http://chuquan.me/2019/12/03/ios-avplayer-support-cache/)

[可能是目前最好的 AVPlayer 音视频缓存方案](https://mp.weixin.qq.com/s/v1sw_Sb8oKeZ8sWyjBUXGA)

[AVPlayer 视频缓存](https://mochangxing.github.io/2018/08/11/AVPlayer%E7%BC%93%E5%AD%98%E7%9A%84%E5%AE%9E%E7%8E%B0/)  架构图挺好的

[iOS音视频开发-----流媒体](https://blog.csdn.net/szk972092933/article/details/82771631)  系统提供的一种缓存办法，有iOS版本限制。