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

注意因为重用可能导致的数据错乱问题。

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

什么是离屏渲染？

引起离屏渲染的因素:

1. 使用了Core Graphics里的方法.(任何CG开头的类)
2. 重写了- drawRect方法,即使是空实现.
3. CALayers的shouldRasterize属性设置为YES.
   @property BOOL shouldRasterize; //光栅化
4. CALayers使用了遮罩(mask)和阴影(shadow).
   @property(nullable, strong) CALayer *mask; //遮罩相关
   edgeAntialiasingMask 边缘抗锯齿遮罩
5. 设置Group opacity (UIViewGroupOpacity). 
   `@property BOOL allowsGroupOpacity;`
6. cornerRadius和maskToBounds=YES结合使用实现圆角. 
   `@property BOOL masksToBounds; `
   注：layer.cornerRadius，layer.borderWidth，layer.borderColor并不会发生Offscreen Render，因为这些不需要加入Mask。设置圆角只需要设置cornerRadius的值,但是由于cornerRadius只对背景颜色和layer的border起作用,而不会对layer里的内容起作用,所以一般才需要和masksToBounds配合使用.它们独立作用的时候不会引起离屏渲染。

> 避免图层混合

什么是图层混合？

1. 确保控件的opaque属性设置为true，确保backgroundColor和父视图颜色一致且不透明.关于backgroundColor说明:The default value is nil, which results in a transparent background color.  

2. 如无特殊需要，不要设置低于1的alpha值.  

3. 确保UIImage没有alpha通道.  

> 异步绘制  

适合纯代码的开发.参考:[AsyncDisplayKit](https://github.com/facebookarchive/AsyncDisplayKit)、[YYText](https://github.com/ibireme/YYText)

#### 调试工具的使用

