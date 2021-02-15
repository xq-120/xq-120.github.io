1. 如何保证缓存的新鲜度
2. 缓存控制原理与实现

## HTTP缓存机制

HTTP中的缓存是通过一系列的缓存首部来实现的。大致可以分为强制缓存和协商缓存。

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

If-Modified-Since

**`If-Modified-Since`** 是一个条件式请求首部，服务器只在所请求的资源在给定的日期时间之后对内容进行过修改的情况下才会将资源返回，状态码为 [`200`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/200) 。如果请求的资源从那时起未经修改，那么返回一个不带有消息主体的 [`304`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/304) 响应，而在 [`Last-Modified`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Last-Modified) 首部中会带有上次修改时间。 不同于  [`If-Unmodified-Since`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/If-Unmodified-Since), `If-Modified-Since` 只可以用在 [`GET`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/GET) 或 [`HEAD`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/HEAD) 请求中。

当与 [`If-None-Match`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/If-None-Match) 一同出现时，它（**`If-Modified-Since`**）会被忽略掉，除非服务器不支持 `If-None-Match`。

### ETag/If-None-Match

ETag

服务器响应请求时，告诉浏览器当前资源在服务器的唯一标识（生成规则由服务器决定）。

If-None-Match

再次请求服务器时，通过此字段通知服务器客户段缓存数据的唯一标识。
服务器收到请求后发现有头If-None-Match 则与被请求资源的唯一标识进行比对，
不同，说明资源又被改动过，则响应整片资源内容，返回状态码200；
相同，说明资源无新修改，则响应HTTP 304，告知浏览器继续使用所保存的cache。



If-Unmodified-Since

HTTP协议中的 **`If-Unmodified-Since`** 消息头用于请求之中，使得当前请求成为条件式请求：只有当资源在指定的时间之后没有进行过修改的情况下，服务器才会返回请求的资源，或是接受 [`POST`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/POST) 或其他 non-[safe](https://developer.mozilla.org/zh-CN/docs/Glossary/safe) 方法的请求。如果所请求的资源在指定的时间之后发生了修改，那么会返回 [`412`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/412) (Precondition Failed) 错误。

常见的应用场景有两种：

- 与 non-[safe](https://developer.mozilla.org/zh-CN/docs/Glossary/safe) 方法如 [`POST`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/POST) 搭配使用，可以用来[优化并发控制](https://en.wikipedia.org/wiki/Optimistic_concurrency_control)，例如在某些wiki应用中的做法：假如在原始副本获取之后，服务器上所存储的文档已经被修改，那么对其作出的编辑会被拒绝提交。
- 与含有 [`If-Range`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/If-Range) 消息头的范围请求搭配使用，用来确保新的请求片段来自于未经修改的文档。

用途：断点续传(一般会指定Range参数). 要想断点续传, 那么文件就一定不能被修改, 否则就不是同一个文件了, 断续还有啥意义?

[If-Unmodified-Since](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/If-Unmodified-Since)

### 断点续传



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

