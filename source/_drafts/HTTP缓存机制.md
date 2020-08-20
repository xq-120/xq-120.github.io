[HTTP Cache Headers - A Complete Guide](https://www.keycdn.com/blog/http-cache-headers)

max-age  
The max-age directive states the maximum amount of time in seconds that fetched responses are allowed to be used again (from the time when a request is made). For instance, “max-age=90” indicates that an asset can be reused (remains in the browser cache) for the next 90 seconds.

[HTTP 缓存](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Caching_FAQ)

过期机制中，最重要的指令是 "max-age=`<seconds>`"，表示资源能够被缓存（保持新鲜）的最大时间。相对Expires而言，max-age是距离请求发起的时间的秒数。

`max-age=100`的起始时间的界定?  
从上面两篇文章来看,应该是从网络请求发起这一刻经过时间100s后,缓存过期.

[彻底弄懂HTTP缓存机制及原理](https://www.cnblogs.com/chenqf/p/6386163.html)

HTTP请求响应时间统计  
[简单剖析用户请求响应过程](https://www.linpx.com/p/a-simple-analysis-of-the-user-request-response-process.html)

