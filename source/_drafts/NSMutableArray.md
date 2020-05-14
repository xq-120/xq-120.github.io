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

容量不够时，会扩大到原来的1.56倍，并不是2倍，这个都是有讲究的。扩大后即使元素全部移除也不会缩减空间。

### 参考

[NSDictionary和NSMutableArray](https://blog.csdn.net/Deft_MKJing/article/details/82732833)

[NSMutableArray](http://blog.joyingx.me/2015/05/03/NSMutableArray%20%E5%8E%9F%E7%90%86%E6%8F%AD%E9%9C%B2/)

[Exposing NSMutableArray](https://ciechanow.ski/exposing-nsmutablearray/)

