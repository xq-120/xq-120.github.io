---
title: UITextField笔记
date: 2018-07-09 15:18:50
description: 
categories: 
- UIKit
tags: 
- UITextField
---

## UITextField笔记

### 监听文本输入变化

方法1: addTarget -- UIControlEventValueChanged

```
[self.passwordTextField addTarget:self action:@selector(passwordTextFieldTextDidChanged:) forControlEvents:UIControlEventValueChanged];

```

方法2: 注册通知 -- UITextFieldTextDidChangeNotification

```
[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(passwordTextFieldTextDidChanged:) name:UITextFieldTextDidChangeNotification object:nil];
```

note:

1. 当点击清空按钮时,只有注册通知的方案,回调方法被调用了.
2. 直接对textField的text属性赋值,`textField.text = @"dfsfsd";`是不会触发UIControlEventValueChanged事件,和UITextFieldTextDidChangeNotification通知回调的.可以使用`textField.text = @"";[textField insertText:newString];`
