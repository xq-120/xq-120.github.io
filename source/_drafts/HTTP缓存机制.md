## HTTP缓存机制

HTTP中的缓存是通过一系列的缓存首部来实现的。大致可以分为强制缓存和协商缓存。

注意：即使超出了max-age规定的时间，下一次的请求也可以把验证条件带上的。

1. 如何保证缓存的新鲜度
2. 缓存机制是如何实现的

### Cache-Control

**HTTP/1.1**定义的 [`Cache-Control`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Cache-Control) 头用来区分对缓存机制的支持情况， 请求头和响应头都支持这个属性。通过它提供的不同的值来定义缓存策略。

Cache-Control常见的取值如下：

Cache-Control: no-store

缓存中不得存储任何关于客户端请求和服务端响应的内容。每次由客户端发起的请求都会下载完整的响应内容。

Cache-Control: no-cache

此方式下，每次有请求发出时，缓存会将此请求发到服务器（译者注：该请求应该会带有与本地缓存相关的验证字段），服务器端会验证请求中所描述的缓存是否过期，若未过期（注：实际就是返回304），则缓存才使用本地缓存副本。

Cache-Control: private
Cache-Control: public

"public" 指令表示该响应可以被任何中间人（译者注：比如中间代理、CDN等）缓存。而 "private" 则表示该响应是专用于某单个用户的，中间人不能缓存此响应，该响应只能应用于浏览器私有缓存中，默认是private。服务器也可以指定为public：`Cache-Control: public, max-age=86400` 。

Cache-Control: max-age=31536000

表示资源能够被缓存（保持新鲜）的最大时间。相对[Expires](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Expires)而言，max-age是距离请求发起的时间的秒数。

Cache-Control: must-revalidate

当使用了 "`must-revalidate`" 指令，那就意味着缓存在考虑使用一个陈旧的资源时，必须先验证它的状态，已过期的缓存将不被使用。

### max-age 

The max-age directive states the maximum amount of time in seconds that fetched responses are allowed to be used again (from the time when a request is made). For instance, “max-age=90” indicates that an asset can be reused (remains in the browser cache) for the next 90 seconds.

过期机制中，最重要的指令是 "max-age=`<seconds>`"，表示资源能够被缓存（保持新鲜）的最大时间。相对Expires而言，max-age是距离请求发起的时间的秒数。

Q：`max-age=100` 的起始时间的界定? 

从上面两篇文章来看,应该是从网络请求发起这一刻经过时间100s后,缓存过期.

### Last-Modified/If-Modified-Since

Last-Modified

文件最后修改的时间，只精确到秒，注：这里的更改并非狭义上必须对内容进行相应的更改，哪怕是打开文件再直接进行保存也会刷新该时间。所以才会有Etag的出现。

示例：`Last-Modified: Wed, 21 Oct 2015 07:28:00 GMT` 。

If-Modified-Since

**`If-Modified-Since`** 是一个条件式请求首部，服务器只在所请求的资源在给定的日期时间之后对内容进行过修改的情况下才会将资源返回，状态码为 [`200`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/200) 。如果请求的资源从那时起未经修改，那么返回一个不带有消息主体的 [`304`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/304) 响应，而在 [`Last-Modified`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Last-Modified) 首部中会带有上次修改时间。 不同于  [`If-Unmodified-Since`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/If-Unmodified-Since), `If-Modified-Since` 只可以用在 [`GET`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/GET) 或 [`HEAD`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/HEAD) 请求中。

当与 [`If-None-Match`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/If-None-Match) 一同出现时，它（**`If-Modified-Since`**）会被忽略掉，除非服务器不支持 `If-None-Match`。

### ETag/If-None-Match

ETag

服务器响应请求时，告诉浏览器当前资源在服务器的唯一标识（生成规则由服务器决定）。

If-None-Match

再次请求服务器时，通过此字段通知服务器客户段缓存数据的唯一标识。
服务器收到请求后发现有头If-None-Match 则与被请求资源的唯一标识进行比对，
不同，说明资源有被改动过，则响应整片资源内容，返回状态码200；
相同，说明资源无新修改，则响应HTTP 304，告知浏览器继续使用所保存的cache。



## 断点续传

HTTP 协议指出，可以通过 HTTP 请求中的 Range 头指定请求数据的范围，Range 头的使用也很简单，只要指定下面的格式就可以了：

```
Range: bytes=500-999
```

它的意思是，只请求目标文件的第 500 到第 999 这 500 个字节。

比如我有一个1000 bytes 大小的文件需要下载，第一次请求时不用指定 Range 头，表示下载整个文件。但在下载完第 499 个字节后，下载被取消了。那么在下一次请求下载同一个文件时，只需要下载第 500 个字节至第 999 个字节的数据就可以了。原理看上去很简单，但我们需要考虑下面几个问题：

1. 是不是所有的 web 服务器都支持 Range 头？

2. 多次请求之间可能会间隔很长的时间，服务器上的文件发生了变化怎么办？（这个问题非常好）
3. 如何保存下载的部分数据和相关信息？
4. 当我们通过字节操作把一个文件拼成原始大小后，如何验证它和源文件一模一样？（这个问题也非常好）

### Range

**`Range`** 是一个请求首部，告知服务器返回文件的哪一部分。在一个 `Range` 首部中，可以一次性请求多个部分，服务器会以 multipart 文件的形式将其返回。如果服务器返回的是范围响应，需要使用 [`206`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/206) `Partial Content` 状态码。假如所请求的范围不合法，那么服务器会返回 [`416`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/416) `Range Not Satisfiable` 状态码，表示客户端错误。服务器允许忽略 `Range` 首部，从而返回整个文件，状态码用 [`200`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/200) 。

语法：

```
Range: <unit>=<range-start>-
Range: <unit>=<range-start>-<range-end>
Range: <unit>=<range-start>-<range-end>, <range-start>-<range-end>
Range: <unit>=<range-start>-<range-end>, <range-start>-<range-end>, <range-start>-<range-end>
```

示例：

```
Range: bytes=200-1000, 2000-6576, 19000-
```

[Range](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Range)

[range request](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Range_requests)

## 条件请求

### If-Modified-Since

**`If-Modified-Since`** 是一个条件式请求首部，服务器只在所请求的资源在给定的日期时间之后对内容进行过修改的情况下才会将资源返回，状态码为 [`200`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/200) 。如果请求的资源从那时起未经修改，那么返回一个不带有消息主体的 [`304`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/304) 响应，而在 [`Last-Modified`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Last-Modified) 首部中会带有上次修改时间。 不同于  [`If-Unmodified-Since`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/If-Unmodified-Since), `If-Modified-Since` 只可以用在 [`GET`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/GET) 或 [`HEAD`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/HEAD) 请求中。

当与 [`If-None-Match`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/If-None-Match) 一同出现时，它（**`If-Modified-Since`**）会被忽略掉，除非服务器不支持 `If-None-Match`。

[If-Modified-Since](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/If-Modified-Since)

### If-Unmodified-Since

HTTP协议中的 **`If-Unmodified-Since`** 消息头用于请求之中，使得当前请求成为条件式请求：只有当资源在指定的时间之后没有进行过修改的情况下，服务器才会返回请求的资源，或是接受 [`POST`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/POST) 或其他 non-[safe](https://developer.mozilla.org/zh-CN/docs/Glossary/safe) 方法的请求。如果所请求的资源在指定的时间之后发生了修改，那么会返回 [`412`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/412) (Precondition Failed) 错误。

常见的应用场景有两种：

- 与 non-[safe](https://developer.mozilla.org/zh-CN/docs/Glossary/safe) 方法如 [`POST`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/POST) 搭配使用，可以用来[优化并发控制](https://en.wikipedia.org/wiki/Optimistic_concurrency_control)，例如在某些wiki应用中的做法：假如在原始副本获取之后，服务器上所存储的文档已经被修改，那么对其作出的编辑会被拒绝提交。
- 与含有 [`If-Range`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/If-Range) （xq注：我感觉这里应该是Range）消息头的范围请求搭配使用，用来确保新的请求片段来自于未经修改的文档。

用途：断点续传(一般会指定Range参数). 要想断点续传, 那么文件就一定不能被修改, 否则就不是同一个文件了, 断续还有啥意义?

[If-Unmodified-Since](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/If-Unmodified-Since)

### If-Match

请求首部 **`If-Match`** 的使用表示这是一个条件请求。在请求方法为 [`GET`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/GET) 和 [`HEAD`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/HEAD) 的情况下，服务器仅在请求的资源满足此首部列出的 `ETag`值时才会返回资源。而对于 [`PUT`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/PUT) 或其他非安全方法来说，只有在满足条件的情况下才可以将资源上传。

以下是两个常见的应用场景：

- 对于 [`GET`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/GET) 和 [`HEAD`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/HEAD) 方法，搭配  [`Range`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Range)首部使用，可以用来保证新请求的范围与之前请求的范围是对同一份资源的请求。如果  ETag 无法匹配，那么需要返回 [`416`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/416)` `(Range Not Satisfiable，范围请求无法满足) 响应。
- 对于其他方法来说，尤其是 [`PUT`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/PUT), `If-Match` 首部可以用来避免[更新丢失问题](https://www.w3.org/1999/04/Editing/#3.1)。它可以用来检测用户想要上传的不会覆盖获取原始资源之后做出的更新。如果请求的条件不满足，那么需要返回 [`412`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/412) (Precondition Failed，先决条件失败) 响应。

[If-Match](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/If-Match)

### If-None-Match

再次请求服务器时，通过此字段通知服务器客户段缓存数据的唯一标识。
服务器收到请求后发现有头If-None-Match 则与被请求资源的唯一标识进行比对，
如果If-None-Match判断为true，即不同，说明资源有被改动过，则响应整片资源内容，返回状态码200；
如果If-None-Match判断为false，即相同，说明资源无新修改，则响应HTTP 304，告知浏览器继续使用所保存的cache。

[If-None-Match](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/If-None-Match)

### If-Range

当（中断之后）重新开始请求更多资源片段的时候，必须确保自从上一个片段被接收之后该资源没有进行过修改。

**`If-Range`** HTTP 请求头字段用来使得 **`Range`** 头字段在一定条件下起作用：当字段值中的条件得到满足时，**`Range`** 头字段才会起作用，同时服务器回复[`206`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/206) 部分内容状态码，以及**`Range`** 头字段请求的相应部分；如果字段值中的条件没有得到满足，服务器将会返回 [`200`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/200) `OK` 状态码，并返回完整的请求资源。

字段值中既可以用 [`Last-Modified`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Last-Modified) 时间值用作验证，也可以用[`ETag`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/ETag)标记作为验证，但不能将两者同时使用。

**`If-Range`** 头字段通常用于断点续传的下载过程中，用来自从上次中断后，确保下载的资源没有发生改变。

示例：

```
If-Range: Wed, 21 Oct 2015 07:28:00 GMT
```

和If-Match 、 If-Unmodified-Since区别：

```
The "If-Range" header field provides a special conditional request
mechanism that is similar to the If-Match and If-Unmodified-Since
header fields but that instructs the recipient to ignore the Range
header field if the validator doesn't match, resulting in transfer of
the new selected representation instead of a 412 (Precondition
Failed) response.
```

## 参考

[HTTP Cache Headers - A Complete Guide](https://www.keycdn.com/blog/http-cache-headers)

[HTTP 缓存](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Caching_FAQ)  MDN web docs  挺不错的

[HTTP 缓存](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching?hl=zh-cn) Web Fundamentals

[彻底弄懂HTTP缓存机制及原理](https://www.cnblogs.com/chenqf/p/6386163.html) 4星

HTTP请求响应时间统计  
[简单剖析用户请求响应过程](https://www.linpx.com/p/a-simple-analysis-of-the-user-request-response-process.html)

[HTTP缓存控制小结](https://imweb.io/topic/5795dcb6fb312541492eda8c)  4星	

[漫谈Web缓存](https://segmentfault.com/a/1190000006671795)

[【译】缓存的最佳实践以及max-age的陷阱](https://segmentfault.com/a/1190000018576830)

[rfc7232-Hypertext Transfer Protocol (HTTP/1.1): Conditional Requests](https://tools.ietf.org/html/rfc7232)

[C# 文件下载之断点续传](https://www.cnblogs.com/sparkdev/p/6141539.html)  非常棒

