NSMutableArray借鉴了循环队列数据结构的思想，只不过NSMutableArray还允许在头部或中间进行插入删除。由于使用场景不一样自然不可能像队列那样严格的遵守先入先出。

[环形缓冲区应用实例](https://juejin.im/post/5e55108051882549361e580a)

[数据结构 第7讲 循环队列](https://www.jianshu.com/p/9ba8a65464dd)

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

逆向工具

关于iOS SDK的所有头文件，早有专人建立了一个在线网站去分析，点击跳转：[iOS Runtime Headers](http://developer.limneos.net/)

[class-dump和MachO文件](https://chinafishnews.github.io/2018/05/29/class-dump%E5%92%8CMachO%E6%96%87%E4%BB%B6/)

安装目录：`/usr/local/bin`
[dsdump](https://derekselander.github.io/dsdump/)：An improved nm + Objective-C & Swift class-dump

[iOS逆向工具之class-dump(MacOS)介绍](https://www.jianshu.com/p/6b6691d52e8a) 本文作者对iOS逆向有较深入研究。

`class-dump -H /System/Library/Frameworks/CoreFoundation.framework -o dsym文件`

库的路径找了一天。最后在这里看到了：[Manipulating Objective-C Program](https://dotblogs.com.tw/cmd4shell/2012/10/18/77570)

