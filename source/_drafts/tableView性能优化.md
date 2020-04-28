---
title: tableView性能优化
tags:
  - 标签1
  - 标签2
  - 标签3
categories:
  - 分类1
  - 分类2
comments: false
date: 2020-04-16 23:30:52
---

计算机最宝贵的资源就是CPU和内存，因此所谓的优化其实就是在"空间"和"时间"之间权衡，以空间换取更短的时间响应。表现在可能会增加额外的内存开销如：cell的重用、高度的缓存甚至布局的缓存等。

#### 优化  

> cell的重用

注意重用可能导致的数据错乱问题。

参考：[UITableViewCell重用导致的图片错乱问题](https://xq-120.github.io/2016/04/24/UITableViewCell%E9%87%8D%E7%94%A8%E5%AF%BC%E8%87%B4%E7%9A%84%E5%9B%BE%E7%89%87%E9%94%99%E4%B9%B1%E9%97%AE%E9%A2%98/)

> 提前计算好cell的高度并且缓存起来

**UITableView代理方法的执行顺序** 
系统会先调用numberOfRowsInSection来获取cell的行数,然后再多次(和numberOfRow正相关)调用heightForRow来确定contentSize及cell的位置,最后才会调用cellForRow显示当前屏幕的cell. 
由于heightForRow会频繁的调用,因此该方法里一定不要进行大量重复的计算.所以cell的高度需要进行缓存,然后在heightForRow方法中直接返回.可采用的策略:网络请求完成后就计算好每个cell的高度,并缓存到对应的model中.更有甚者会在此时连cell的子视图布局都计算好.

cell的高度计算与缓存问题？

> 避免阻塞主线程 

比如图片的下载，图片的处理等其他耗时操作应该放在子线程执行，如果可以应该缓存起来。

> 按需加载 

比如还在滚动时,不去加载显示图片.当滚动停止时才加载显示图片.这种方式在滚动时会看不到图片,因此需要结合实际需求使用.

> 不要动态的add或remove子控件 

子控件最好在初始化时就添加完成,可采用hidden来控制是否显示。复杂的cell可以采用[IGListKit](https://github.com/Instagram/IGListKit)，而不是各种remake约束。

> 尽可能重用创建时开销比较大的对象 

比如NSDateFormatter和NSCalendar等对象.这些对象初始化非常的慢,我们可以把它们加入到类的属性当中,或者创建单例来使用.

> 避免离屏渲染

**什么是离屏渲染**

GPU渲染机制：

CPU 计算好显示内容提交到 GPU，GPU 渲染完成后将渲染结果放入帧缓冲区，随后视频控制器会按照 VSync 信号逐行读取帧缓冲区的数据，经过可能的数模转换传递给显示器显示。

GPU屏幕渲染有以下两种方式：

On-Screen Rendering意为当前屏幕渲染，指的是GPU的渲染操作是在当前用于显示的屏幕缓冲区中进行。

Off-Screen Rendering意为离屏渲染，指的是GPU在当前屏幕缓冲区以外新开辟一个缓冲区进行渲染操作。

**为什么要有离屏渲染**

因为有些效果不能直接在当前屏幕渲染，所以需要有离屏渲染。貌似是废话。为什么圆角，阴影，遮罩等效果不能直接在当前屏幕渲染？

**离屏渲染代价**

相比于当前屏幕渲染，离屏渲染的代价是很高的，主要体现在两个方面：

* 创建新缓冲区

要想进行离屏渲染，首先要创建一个新的缓冲区。

* 上下文切换

离屏渲染的整个过程，需要多次切换上下文环境：先是从当前屏幕（On-Screen）切换到离屏（Off-Screen）；等到离屏渲染结束以后，将离屏缓冲区的渲染结果显示到屏幕上有需要将上下文环境从离屏切换到当前屏幕。而上下文环境的切换是要付出很大代价的。

**引起离屏渲染的因素**

1. 使用了Core Graphics里的方法.(任何CG开头的类)

2. 重写了- drawRect方法，即使是空实现。

3. 设置CALayers的以下属性：
  
   光栅化
   
   `@property BOOL shouldRasterize;` //shouldRasterize属性设置为YES.
   
   遮罩
   
   `@property(nullable, strong) CALayer *mask;` //遮罩相关
   
   阴影(shadow)
   
   `@property(nullable) CGColorRef shadowColor;`等
   
   抗锯齿
   
   `@property CAEdgeAntialiasingMask edgeAntialiasingMask;` //边缘抗锯齿遮罩
   
   组不透明（Group opacity）
   `@property BOOL allowsGroupOpacity;`
   
4. cornerRadius和maskToBounds=YES结合使用实现圆角. 
   `@property BOOL masksToBounds; `
   注：layer.cornerRadius，layer.borderWidth，layer.borderColor并不会发生Offscreen Render，因为这些不需要加入Mask。设置圆角只需要设置cornerRadius的值,但是由于cornerRadius只对背景颜色和layer的border起作用,而不会对layer里的内容起作用,所以一般才需要和masksToBounds配合使用.它们独立作用的时候不会引起离屏渲染。

参考：

[为什么产生离屏渲染](https://www.jianshu.com/p/aa8dc1a61c91)

[OpenGL 图片从文件渲染到屏幕的过程](https://juejin.im/post/5d732ea96fb9a06b2d77f76a)

[GPU屏幕渲染——离屏渲染](http://imgtec.eetrend.com/d6-imgtec/blog/2018-08/17019.html)

> 避免图层混合

**什么是图层混合**

如果屏幕的一块区域上有多个图层（layer），每个图层都会有一定的透明度，那么最后这块区域的显示效果就是这些图层共同作用的结果，这种结果需要cpu对每个图层的颜色进行计算，需要消耗更多的cpu资源。如果我们把最上层的layer设定为不透明，那么cpu就不需要计算底层的layer的色值，这样就可以节约cpu的计算量，节约资源。

**应对措施**

1. 确保控件的opaque属性设置为true，确保backgroundColor不透明.关于backgroundColor说明:The default value is nil, which results in a transparent background color.  
2. 如无特殊需要，不要设置低于1的alpha值.  
3. 确保UIImage没有alpha通道.  

参考：[ios性能优化--label上汉字图层混合问题](https://www.jianshu.com/p/b8ee7a40e219)

> 异步绘制  

适合纯代码的开发.

参考：

[AsyncDisplayKit](https://github.com/facebookarchive/AsyncDisplayKit)、[YYText](https://github.com/ibireme/YYText)

[AsyncDisplayKit介绍（一）原理和思路](https://medium.com/jike-engineering/asyncdisplaykit%E4%BB%8B%E7%BB%8D-%E4%B8%80-6b871d29e005)

#### 调试工具的使用

