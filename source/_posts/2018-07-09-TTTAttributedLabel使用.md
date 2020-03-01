---
title: TTTAttributedLabel使用
date: 2018-07-09 15:28:42
description: 
categories: 第三方库使用
tags: TTTAttributedLabel
---

## TTTAttributedLabel使用

Q1:使用pod管理TTTAttributedLabel,在XIB中使用会导致XIB报错并且一片空白.

A:没找到一种好的解决办法,只能不使用XIB.

Q2:使用TTTAttributedLabel,在聊天cell(xib实现)中,点击链接代理方法不被调用,原因是TTTAttributedLabel的touchesCancelled一直被调用了,touchesEnded没被调用.

A2:代码中有添加点击手势导致label的touch事件总是被cancel.

```
UITapGestureRecognizer *tapGes = [[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(tappedView:)];
tapGes.delegate = self;
tapGes.cancelsTouchesInView = NO; //重要
[self.view addGestureRecognizer:tapGes];
```
> 手势与响应者链
> 
父视图上添加了点击手势,有时(不是每次)会导致父视图上的UIButton的action方法不会被调用.
原因在于如果父视图识别出了手势,那么会cancel掉UITouch.从而第一响应者会收到touchesCancelled消息.基本上那些原本不接受用户事件的控件比如UILabel即使设置userInteractionEnabled=YES,但它的touchesEnded将不会被调用.而UIControl子类控件他们的action方法有时不会被调用.