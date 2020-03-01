---
title: layoutSubviews的调用时机
date: 2017-08-20 23:20:24
description: 
categories:
tags:
---

## layoutSubviews的调用时机

方法说明:override point. called by `layoutIfNeeded` automatically. 
As of iOS 6.0, when constraints-based layout is used the base implementation applies the constraints-based layout, 
otherwise it does nothing.  
该方法由`layoutIfNeeded`自动调用,所以我们最好不要手动去调用它.因此在需要重新布局子视图时,可以调用`[self layoutIfNeeded]`或`[self setNeedLayout];`


## 手动触发layoutSubviews

调用`[self layoutIfNeeded]`或`[self setNeedLayout];`  
> `layoutIfNeeded`与`setNeedLayout`的区别:  
> 苹果的说明:这两个方法Allows you to perform layout before the drawing cycle happens. -layoutIfNeeded forces layout early  
> 根据上面所说`layoutIfNeeded`内部是直接调用了`layoutSubviews`,所以调用`layoutIfNeeded`会马上执行`layoutSubviews`方法.但是调用`setNeedLayout`并不会马上执行`layoutSubviews`,而是等到一个runloop结束时调用.


## 自动触发的layoutSubviews

如果一个viewA还没有被添加到父视图上面去,那么无论你对它进行什么操作(设置frame,添加子视图等等),它的`layoutSubviews`方法始终是不会被调用的.  
下面情况将会自动触发viewA的`layoutSubviews`方法:  

1. viewA被添加到父视图上.(viewA的和viewA所有子视图的都会被触发).

2. viewA addSubview会触发`layoutSubviews`.(viewA的和刚添加的子视图的会被触发,viewA其他子视图不会).

3. viewA上的子视图从viewA上移除,会触发`layoutSubviews`.(只有viewA的会被触发, viewA的子视图不会被触发)

4. 设置viewA的Frame会触发`layoutSubviews`，当然前提是frame的值设置前后发生了变化.(只有viewA的会被触发, viewA的子视图的不会被触发)

5. viewA上的子视图大小(x,y改变没影响)改变时也会触发viewA的`layoutSubviews`事件.(viewA的和该子视图的,其他子视图不会)

6. 如果viewA是一个UIScrollView,那么viewA在滚动时会触发它的layoutSubviews方法.(只有viewA的会被触发, viewA的子视图的不会被触发)

7. 旋转Screen会触发viewController的primary view的layoutSubviews事件.

注意:  

* init初始化不会触发`layoutSubviews`,因为此时View还没有被添加到父视图上去.  
* 一个View的子视图指的是它的直系子视图.


## 参考

[When does layoutSubviews get called?](http://blog.logichigh.com/2011/03/16/when-does-layoutsubviews-get-called/)