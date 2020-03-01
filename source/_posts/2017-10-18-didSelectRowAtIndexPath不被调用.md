---
title: didSelectRowAtIndexPath不被调用
date: 2017-10-18 15:01:11
description: 
categories: UITableView
tags: UITableView
comments: true
---

## didSelectRowAtIndexPath不被调用

vc.view上添加一个UITableView,并且vc.view上添加一个UITapGestureRecognizer手势,那么`- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath`代理方法将不再被调用.

这是由于`UIGestureRecognizer`有一个属性:  
`@property(nonatomic) BOOL cancelsTouchesInView;`  
官方的注释:default is YES. causes touchesCancelled:withEvent: or pressesCancelled:withEvent: to be sent to the view for all touches or presses recognized as part of this gesture immediately before the action method is called.  
该属性默认是YES,在action方法被调用之前系统会迅速调用View的touchesCancelled:withEvent: or pressesCancelled:withEvent:方法.touches事件将立即被cancel.所有的touches或presses对象将转而被用于手势识别.因此当一个View上添加了一个手势那么该View的touchesEnded方法将不会被调用(只会touchesBegan->touchesCancelled).

可以推断出:如果一个button上又添加一个tap手势,则最后只会有tap响应,button的target则不会被调用.这是由于手势识别时会cancel掉后面的Touches.因此当用户触摸一个控件时,手势的处理优先于触摸事件的处理.

因此,要想didSelectRowAtIndexPath方法被调用,有两种解决办法:  

1. 设置`tap.cancelsTouchesInView = NO;`让touch事件继续进行,从而让view可以继续处理.
2. 实现`UIGestureRecognizerDelegate`的代理方法

	```
	- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldReceiveTouch:(UITouch *)touch {
	    if ([touch.view isKindOfClass:NSClassFromString(@"UITableViewCellContentView")]) {
	        return NO;
	    }
	    return YES;
	}
	``` 
	当点击到cell.contentView上时,让tap手势不处理touches.

### 参考:

[iOS的事件处理](https://joakimliu.github.io/2017/01/15/iOS的事件处理/)

Event Handling Guide for iOS 