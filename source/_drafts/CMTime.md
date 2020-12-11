### CMTime定义

```c
/*!
	@typedef	CMTime
	@abstract	Rational time value represented as int64/int32.
*/
typedef struct
{
	CMTimeValue	value;		/*! @field value The value of the CMTime. value/timescale = seconds. */
	CMTimeScale	timescale;	/*! @field timescale The timescale of the CMTime. value/timescale = seconds.  */
	CMTimeFlags	flags;		/*! @field flags The flags, eg. kCMTimeFlags_Valid, kCMTimeFlags_PositiveInfinity, etc. */
	CMTimeEpoch	epoch;		/*! @field epoch Differentiates between equal timestamps that are actually different because
												 of looping, multi-item sequencing, etc.  
												 Will be used during comparison: greater epochs happen after lesser ones. 
												 Additions/subtraction is only possible within a single epoch,
												 however, since epoch length may be unknown/variable. */
} CMTime API_AVAILABLE(macos(10.7), ios(4.0), tvos(9.0), watchos(6.0));
```

value：int64整型。代表当前第几帧。在初始化的时候如果传入一个小数其实会被强转为int64整型，这里需要注意一下。比如0.2会变为0。

timescale：int32整型。时间刻度，代表1秒钟有多少帧。

flags：表示CMTime的状态，比如kCMTimeFlags_Valid（合法CMTime），kCMTimeFlags_PositiveInfinity（正无穷大）等。

epoch：看不懂

value/timescale 就是CMTime表示的时间（s）。

```
A CMTime is represented as a rational number, with a numerator (an int64_t value), and a denominator (an int32_t timescale). Conceptually, the timescale specifies the fraction of a second each unit in the numerator occupies. Thus if the timescale is 4, each unit represents a quarter of a second; if the timescale is 10, each unit represents a tenth of a second, and so on. In addition to a simple time value, a CMTime can represent non-numeric values: +infinity, -infinity, and indefinite. Using a flag CMTime indicates whether the time been rounded at some point.
```

CMTime是以两个整数比的形式来表示一个有理数。其中timescale表示1秒钟有timescale帧，那么1帧就代表（1/timescale）秒。如果timescale=4，那么1帧就代表0.25秒。

为什么会有CMTime？

虽然浮点类型在大多数场景下都能满足要求，但是在音视频领域浮点类型的误差在累加的情况下会越来越大，所以需要另辟蹊径来解决这个问题，CMTime类型于是诞生。

### CMTime的创建

```c
CM_EXPORT 
CMTime CMTimeMake(
				int64_t value,		/*! @param value		Initializes the value field of the resulting CMTime. */
				int32_t timescale)	/*! @param timescale	Initializes the timescale field of the resulting CMTime. */
							API_AVAILABLE(macos(10.7), ios(4.0), tvos(9.0), watchos(6.0));
							
/*!
	@function	CMTimeMakeWithSeconds
	@abstract	Make a CMTime from a Float64 number of seconds, and a preferred timescale.
	@discussion	The epoch of the result will be zero.  If preferredTimescale is <= 0, the result
				will be an invalid CMTime.  If the preferred timescale will cause an overflow, the
				timescale will be halved repeatedly until the overflow goes away, or the timescale
				is 1.  If it still overflows at that point, the result will be +/- infinity.  The
				kCMTimeFlags_HasBeenRounded flag will be set if the result, when converted back to
				seconds, is not exactly equal to the original seconds value.
	@result		The resulting CMTime.
*/
CM_EXPORT 
CMTime CMTimeMakeWithSeconds(
				Float64 seconds,
				int32_t preferredTimescale)
							API_AVAILABLE(macos(10.7), ios(4.0), tvos(9.0), watchos(6.0));
```

eg:

```
CMTime time = CMTimeMake(1345, 600);
CMTime time2 = CMTimeMakeWithSeconds(2.0, 600);
CMTime time3 = CMTimeMake(0.2, 1); //这里实际代表是0s而不是0.2s
Float64 numTime = CMTimeGetSeconds(time); //2.2416666666666667s
Float64 numTime2 = CMTimeGetSeconds(time2); //2s
Float64 numTime3 = CMTimeGetSeconds(time3); //0s
//以下CMTime都代表5秒
CMTime t1 =CMTimeMake(5, 1);
CMTime t2 =CMTimeMake(3000, 600);
CMTime t3 =CMTimeMake(5000, 1000);
```

注：在处理视频内容时常见的时间刻度为600，这是大部分常用视频帧率24FPS、25FPS、30FPS的公倍数。音频数据常见的时间刻度就是采样率，比如44 100(44.1kHZ)或48 000(kHZ)。

CMTime还有一些加减乘操作和内省比较操作，这里就不举例了。

### 参考

[CMTime官方文档](https://developer.apple.com/documentation/coremedia/cmtime-u58?language=objc)

[短视频从无到有 (三)CMTime详解](https://juejin.cn/post/6844904144478666759)

[How to precisely jump to a specific time in AVPlayer](https://medium.com/@chauyan/how-to-precisely-jump-to-a-specific-time-in-avplayer-5b8cec637afe)

