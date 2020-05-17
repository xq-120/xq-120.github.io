---
title: NSMutableArray实现原理
tags:
  - NSMutableArray
categories:
  - Foundation
comments: true
date: 2020-05-17 12:57:57
---



本文基于[Exposing NSMutableArray](https://ciechanow.ski/exposing-nsmutablearray/) 理解和扩展。

C语言中的数组可以说是最基本的数据结构了，其他一些数据结构比如队列，循环队列等都可以基于普通C数组的进一步封装来实现。

**普通C数组**

数组占据的是一段连续的内存空间，并根据顺序存储数据。

优点：时间效率很高

由于数组中的内存是连续的，因此可以根据下标读写任何元素，时间复杂度为O(1)。

缺点：空间效率较低

创建数组时需要先指定数组的容量大小，然后根据大小分配内存。即使只在数组中存储一个元素也需要为所有的数据预先分配内存。因此它的空间效率不是很高，经常会有空闲区域没有被使用。

另一个明显的缺点是：在下标 0 处插入或删除一个元素时，需要移动其它所有的元素。当数据量很大时，性能就变得很差。

**循环队列**

如何避免这个问题呢？答案就是循环队列。循环队列在进行入队和出队操作时，其他元素是不需要移动的。因此可以借鉴这个思想。理解了循环队列基本上就理解了NSMutableArray的底层数据结构。

循环队列里一个很重要的操作就是取模。

例如，对于任何一个自然数，模上5，得到的结果都会在[0,4]之间。

0%5  = 0，1%5  = 1，2%5  = 2，3%5  = 3，4%5  = 4，5%5  = 0，6%5  = 1，7%5  = 2， ......。看到没有，结果已经开始循环了。

循环队列的声明：

```
public class CircularQueue {
  public private(set) var size: Int = 0
  public private(set) var count: Int = 0
  var head: Int = 0
  var tail: Int = 0
  var items: [Int] = []
}
```

一般都会有head和tail两个指针，head总是指向第一个元素，tail总是指向下一个存储的元素的位置。也可以不使用tail，而使用head和count，因为tail = head + count。

初始化时head==tail。另外head可以等于0也可以不等于0这个不重要，因为是循环的，所以无论从哪个点开始都是一样的。

这里初始化：head==tail=0。

<img src="https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/618F53DD5E0AC45A98E4192354D0F63E.jpg" style="zoom:50%;" />

入队1

每次入队时，tail往后移动一个位置。

<img src="https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/488D507C0EC5B5CDA656D769F25C9281.jpg" style="zoom:50%;" />

入4

<img src="https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20200517115455.png" style="zoom:50%;" />

入5

<img src="https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/8C560E41A5516F18BAE25E17928B9A3C.jpg" style="zoom:50%;" />

入队5时，队列就满了，tail的位置又回到了原点。此时就不再允许入队了因为已经满了。

因此每次入队时，tail往后移动一个位置，对应的代码为：

tail = (tail + 1) % size。模size意味着超出size时回到原来的位置。开始循环。这就是取模的魅力。

再来看出队：

这里连续出队1，2，3。每次出队后，head就往后移动一个位置，指向新的第一个元素的位置，对应的代码为：

head = (head + 1) % size。

<img src="https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/6C548BF9E45EA310CDBF27701E5131E7.jpg" style="zoom:50%;" />

再入队3：

<img src="https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/667D39026317F4D61F674BB94052BA4F.jpg" style="zoom:50%;" />

这样head和tail就不断的循环，充分利用了空间，并且每次操作时其他元素是不动的。

上面的数组的内容为[4,5,3]。如果我们想用下标来直接取出一个元素该怎么操作呢。比如a[2] = 3。

利用head就可以轻松计算出实际的下标：realIndex = (head + index) % size 。

即realIndex = （3 + 2）% 5 = 0，原数组下标0即为要取的元素。

利用循环队列结合取模操作，我们既做到了插入或删除一个元素时，不移动其它元素。又可以通过下标来取出元素。

有兴趣的可以刷一下LeetCode的[622. 设计循环队列](https://leetcode-cn.com/problems/design-circular-queue/)，加深理解。

**循环双端队列**

上面的循环队列依然遵守队头出，队尾入的规则。有时候就不太灵活，于是就有循环双端队列这一变种，它可以在两端进行插入和删除操作，但不能在中间操作。

可以刷一下[641. 设计循环双端队列](https://leetcode-cn.com/problems/design-circular-deque/)，由于和循环队列差别不大，这里就不做介绍了。

循环双端队列和NSMutableArray的底层数据结构就更接近了。

**NSMutableArray底层数据结构**

NSMutableArray借鉴了循环双端队列数据结构的思想，只不过NSMutableArray还允许在中间进行插入删除。

NSMutableArray是一个类簇，实际的类型是__NSArrayM。

__NSArrayM的实例布局：

```
@interface __NSArrayM : NSMutableArray
{
    unsigned long long _used;
    unsigned long long _doHardRetain:1;
    unsigned long long _doWeakAccess:1;
    unsigned long long _size:62;
    unsigned long long _hasObjects:1;
    unsigned long long _hasStrongReferences:1;
    unsigned long long _offset:62;
    unsigned long long _mutations;
    id *_list;
}
```

比较重要的几个属性：

_used：当前已使用的内存空间个数，即当前数组中元素的个数。

_size：数组的大小

_offset：数组的第一个元素索引，始终指向数组的第一个元素。相当于上面的head

_list：缓冲区指针，指向分配的那一段连续的内存空间，即元素存储的地方。相当于上面的Array。

_offset + _used就相当于上面的tail。

下标查找时realIndex = (_offset + index) % size 。

**在两端操作，其他元素不需要移动**

在头部进行插入和删除操作及在尾部进行插入和删除操作，同循环双端队列，其他元素是不需要移动的，这就提高了性能。

**在中间操作**

delete at middle或insert at middle

原则：移动较少的一端的元素。

在中间插入或删除时不可避免需要移动元素否则就无法通过下标定位元素了，这里采用移动较少的一端的元素来提高性能。

**扩容**

普通C数组在初始化时容量就已经确定了，所谓的插入和删除操作其实都是在这个确定的容量里进行，并且插入和删除操作并不会对这个确定的容量有什么影响，只有在插入时容量不足了，才需要扩容。

容量不够时，NSMutableArray会扩大到原来的1.625倍，并不是2倍，这个都是有讲究的。扩大后即使元素全部移除也不会再缩减空间。

### 参考

[Exposing NSMutableArray](https://ciechanow.ski/exposing-nsmutablearray/) 译文：[NSMutableArray原理揭露](http://blog.joyingx.me/2015/05/03/NSMutableArray%20%E5%8E%9F%E7%90%86%E6%8F%AD%E9%9C%B2/)

[NSDictionary和NSMutableArray](https://blog.csdn.net/Deft_MKJing/article/details/82732833)

[数据结构 第7讲 循环队列](https://www.jianshu.com/p/9ba8a65464dd)

[看完这篇你还不知道这些队列，我这些图白作了](https://juejin.im/post/5d5fb74fe51d45620346b8d0) 讲解了各种队列。

[堆、栈和队列如何理解与应用？](https://zhuanlan.zhihu.com/p/72007079) 主要讲数据结构中的堆和栈。

[怎样深入理解堆和栈](https://zhuanlan.zhihu.com/p/66922957) 主要讲操作系统中**内存分配方式**的堆和栈。

[常见数据结构 (一)- 栈, 队列, 堆, 哈希表](https://juejin.im/entry/58330cbfa0bb9f005a0fed62)

**iOS逆向**

知识储备

[class-dump和MachO文件](https://chinafishnews.github.io/2018/05/29/class-dump%E5%92%8CMachO%E6%96%87%E4%BB%B6/)

[iOS逆向工具之class-dump(MacOS)介绍](https://www.jianshu.com/p/6b6691d52e8a) 本文作者对iOS逆向有较深入研究。

class-dump安装目录：`/usr/local/bin`

`class-dump -H /System/Library/Frameworks/CoreFoundation.framework -o 目的文件夹`

Xcode11不能使用下面的CoreFoundation路径。（网上的教程都是下面的路径，但实验证明不行）

`/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks/CoreFoundation.framework`

库的路径整了一天，看来是路径变了。

别人整好的：关于iOS SDK的所有头文件，早有专人建立了一个在线网站去分析，点击跳转：[iOS Runtime Headers](http://developer.limneos.net/)

[macOS_headers](https://github.com/w0lfschild/macOS_headers) macOS上的Frameworks class-dump。

另一款工具：[dsdump](https://derekselander.github.io/dsdump/)：An improved nm + Objective-C & Swift class-dump

