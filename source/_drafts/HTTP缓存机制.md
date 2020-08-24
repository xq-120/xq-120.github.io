1. 如何保证缓存的新鲜度
2. 缓存控制原理与实现

max-age 
The max-age directive states the maximum amount of time in seconds that fetched responses are allowed to be used again (from the time when a request is made). For instance, “max-age=90” indicates that an asset can be reused (remains in the browser cache) for the next 90 seconds.

过期机制中，最重要的指令是 "max-age=`<seconds>`"，表示资源能够被缓存（保持新鲜）的最大时间。相对Expires而言，max-age是距离请求发起的时间的秒数。

`max-age=100`的起始时间的界定? 
从上面两篇文章来看,应该是从网络请求发起这一刻经过时间100s后,缓存过期.



Last-Modified

文件最后修改的时间，注：这里的更改并非狭义上必须对内容进行相应的更改，哪怕是打开文件再直接进行保存也会刷新该时间。所以才会有Etag的出现。



If-Modified-Since

**`If-Modified-Since`** 是一个条件式请求首部，服务器只在所请求的资源在给定的日期时间之后对内容进行过修改的情况下才会将资源返回，状态码为 [`200`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/200) 。如果请求的资源从那时起未经修改，那么返回一个不带有消息主体的 [`304`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/304) 响应，而在 [`Last-Modified`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Last-Modified) 首部中会带有上次修改时间。 不同于  [`If-Unmodified-Since`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/If-Unmodified-Since), `If-Modified-Since` 只可以用在 [`GET`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/GET) 或 [`HEAD`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/HEAD) 请求中。

当与 [`If-None-Match`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/If-None-Match) 一同出现时，它（**`If-Modified-Since`**）会被忽略掉，除非服务器不支持 `If-None-Match`。



If-Unmodified-Since

HTTP协议中的 **`If-Unmodified-Since`** 消息头用于请求之中，使得当前请求成为条件式请求：只有当资源在指定的时间之后没有进行过修改的情况下，服务器才会返回请求的资源，或是接受 [`POST`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods/POST) 或其他 non-[safe](https://developer.mozilla.org/zh-CN/docs/Glossary/safe) 方法的请求。如果所请求的资源在指定的时间之后发生了修改，那么会返回 [`412`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/412) (Precondition Failed) 错误。

用途：断点续传(一般会指定Range参数). 要想断点续传, 那么文件就一定不能被修改, 否则就不是同一个文件了, 断续还有啥意义?

## 参考

[HTTP Cache Headers - A Complete Guide](https://www.keycdn.com/blog/http-cache-headers)

[HTTP 缓存](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Caching_FAQ)  MDN web docs

[HTTP 缓存](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching?hl=zh-cn) Web Fundamentals

[彻底弄懂HTTP缓存机制及原理](https://www.cnblogs.com/chenqf/p/6386163.html) 4星

HTTP请求响应时间统计  
[简单剖析用户请求响应过程](https://www.linpx.com/p/a-simple-analysis-of-the-user-request-response-process.html)

[HTTP缓存控制小结](https://imweb.io/topic/5795dcb6fb312541492eda8c)  4星

[漫谈Web缓存](https://segmentfault.com/a/1190000006671795)

[【译】缓存的最佳实践以及max-age的陷阱](https://segmentfault.com/a/1190000018576830)

