NSMutableArray借鉴了循环队列数据结构的思想，只不过NSMutableArray还允许在头部或中间进行插入删除。由于使用场景不一样自然不可能像队列那样严格的遵守先入先出。

[环形缓冲区应用实例](https://juejin.im/post/5e55108051882549361e580a)

[数据结构 第7讲 循环队列](https://www.jianshu.com/p/9ba8a65464dd)

[看完这篇你还不知道这些队列，我这些图白作了](https://juejin.im/post/5d5fb74fe51d45620346b8d0)

[堆、栈和队列如何理解与应用？](https://zhuanlan.zhihu.com/p/72007079) 主要讲数据结构中的堆和栈

[怎样深入理解堆和栈](https://zhuanlan.zhihu.com/p/66922957) 主要讲操作系统中**内存分配方式**的堆和栈。

[常见数据结构 (一)- 栈, 队列, 堆, 哈希表](https://juejin.im/entry/58330cbfa0bb9f005a0fed62)

**在两端操作**

Insert at end

和普通C数组一样。在最后一个元素的后面插入即可。

insert at start 

一直在左边插入，如果左边满了，则跑到最右边插入，更新offset。其他元素不需要移动。

delete at end

直接删除，其他元素不需要移动。

delete at start 

直接删除，更新offset，其他元素不需要移动。

**在中间操作**

delete at middle

insert at middle

原则：移动较少的一端的元素。

**扩容**

容量不够时，会扩大到原来的1.625倍，并不是2倍，这个都是有讲究的。扩大后即使元素全部移除也不会缩减空间。

### 参考

[NSDictionary和NSMutableArray](https://blog.csdn.net/Deft_MKJing/article/details/82732833)

[NSMutableArray](http://blog.joyingx.me/2015/05/03/NSMutableArray%20%E5%8E%9F%E7%90%86%E6%8F%AD%E9%9C%B2/)

[Exposing NSMutableArray](https://ciechanow.ski/exposing-nsmutablearray/)

**iOS逆向**

知识储备

[class-dump和MachO文件](https://chinafishnews.github.io/2018/05/29/class-dump%E5%92%8CMachO%E6%96%87%E4%BB%B6/)

[iOS逆向工具之class-dump(MacOS)介绍](https://www.jianshu.com/p/6b6691d52e8a) 本文作者对iOS逆向有较深入研究。

class-dump安装目录：`/usr/local/bin`

`class-dump -H /System/Library/Frameworks/CoreFoundation.framework -o 目的文件夹`

Xcode11不能使用下面的CoreFoundation路径。（网上的教程都是下面的路径，但实验证明不行）

`/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/CoreFoundation.framework`

库的路径整了一天，看来是路径变了。

别人整好的：

关于iOS SDK的所有头文件，早有专人建立了一个在线网站去分析，点击跳转：[iOS Runtime Headers](http://developer.limneos.net/)

[macOS_headers](https://github.com/w0lfschild/macOS_headers) macOS上的Frameworks class-dump。

另一款工具：[dsdump](https://derekselander.github.io/dsdump/)：An improved nm + Objective-C & Swift class-dump