---
title: navigationBar对self.view的frame影响
date: 2018-06-10 12:49:28
description: 
categories: UIViewController
tags: translucent属性
---

### navigationBar对self.view的frame影响

在iOS7及以上系统,`self.navigationController.navigationBar.hidden = NO;`的情况下.

**设置translucent为YES**.即`self.navigationController.navigationBar.translucent = YES;`

> 不隐藏状态栏

self.view的位置和大小如下:

![](http://7xqoji.com1.z0.glb.clouddn.com/self.view%E7%9A%84%E4%BD%8D%E7%BD%AE%E5%92%8C%E5%A4%A7%E5%B0%8F1@2x.png)

> 隐藏状态栏

![](http://7xqoji.com1.z0.glb.clouddn.com/self.view%E7%9A%84%E4%BD%8D%E7%BD%AE%E5%92%8C%E5%A4%A7%E5%B0%8F2@2x.png)

执行流:
![](http://7xqoji.com1.z0.glb.clouddn.com/navigationBar%E5%AF%B9self.view%E7%9A%84frame%E5%BD%B1%E5%93%8D_3.png)

translucent为YES时,可以在viewDidLoad中子视图可以使用self.view的frame信息.

**设置translucent为NO**,即`self.navigationController.navigationBar.translucent = NO;`

> 不隐藏状态栏

self.view的位置和大小如下:

![](http://7xqoji.com1.z0.glb.clouddn.com/navigationBar%E5%AF%B9self.view%E7%9A%84frame%E5%BD%B1%E5%93%8D_4.png)

> 隐藏状态栏

![](http://7xqoji.com1.z0.glb.clouddn.com/navigationBar%E5%AF%B9self.view%E7%9A%84frame%E5%BD%B1%E5%93%8D_5.png)

执行流:

![](http://7xqoji.com1.z0.glb.clouddn.com/navigationBar%E5%AF%B9self.view%E7%9A%84frame%E5%BD%B1%E5%93%8D_6.png)

self.view在viewDidLoad,viewWillAppear中的origin,size都不对.在viewWillLayoutSubviews,viewDidLayoutSubviews中size是对的,origin不对.在viewDidAppear中origin,size才都对.

因此在这种情况下,self.view的子视图布局最好在viewWillLayoutSubviews中进行调整.

当设置`self.navigationController.navigationBar.hidden = YES;`时,则navigationBar的translucent属性不再对self.view的frame产生影响.

#### translucent属性
由于translucent属性对self.view的frame会产生影响,所以有必要查看它的说明.

官方说明:

```
The default value is YES. If the navigation bar has a custom background image, the default is YES if any pixel of the image has an alpha value of less than 1.0, and NO otherwise.

If you set this property to YES on a navigation bar with an opaque custom background image, the navigation bar applies a system-defined opacity of less than 1.0 to the image.

If you set this property to NO on a navigation bar with a translucent custom background image, the navigation bar provides an opaque background for the image using black if the navigation bar has UIBarStyleBlack style, white if the navigation bar has UIBarStyleDefault, or the navigation bar’s barTintColor if a custom value is defined.
```
大意为:
该属性默认为YES.但是如果给导航栏设置了一张自定义的背景图片,如果该图片有一个alpha<1的像素.那么该值就为YES,否则为NO.(设置导航栏的背景图片会影响translucent的默认值)

另外如果手动设置了该属性,并且设置了导航栏的背景图,则系统可能会对背景图进行处理:

1. 如果设置该属性为YES,但是提供了一张不透明的背景图,系统会对该背景图进行半透明处理.
2. 如果设置该属性为NO,但是提供了一张半透明的背景图,则系统会对该背景图进行不透明处理.具体是根据导航栏的style或者barTintColor进行处理.

#### 总结

在iOS7及以上系统:

1. 当导航栏没有被隐藏时,且translucent属性设置为YES,那么,VC的self.view的位置是从(0,0)开始,大小是整个屏幕宽高.而当translucent属性设置为NO时,那么VC的self.view的位置是从导航栏左下角开始,宽是屏幕的宽,高是整个屏幕减去导航栏的CGRectGetMaxY.

2. 导航栏的y值会随着状态栏隐藏与否而有20pt的偏移.

3. 当导航栏被隐藏时,VC的self.view的位置是从(0,0)开始,大小是整个屏幕宽高.

4. self.view的子视图布局推荐在viewWillLayoutSubviews中进行.因为此时不管translucent为何值,self.view的size都是正确的.

---
吐槽Jekyll

写在

\`\`\`
 
Xcode的日志

\`\`\`

这里面的日志,包含了View的frame打印,居然导致Jekyll构建页面失败,说什么变量未正确关闭,最后只好一张张截图.这解析功能也太差了吧.