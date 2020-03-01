---
title: UIBarButtonItem坑
date: 2018-12-19 21:34:06
description: 
categories: UIKit
tags: UIBarButtonItem
comments: true
---

使用系统设置,在iOS11.4上会出现点击时,文字变大,颜色变为蓝色.所以使用按钮还是靠谱些.

```
UIBarButtonItem *allReadItem = [[UIBarButtonItem alloc] initWithTitle:@"全部已读" style:UIBarButtonItemStylePlain target:self action:@selector(allReadItemDidClicked:)];
[allReadItem setTitleTextAttributes:@{NSFontAttributeName : [UIFont pingFangSCWithSize:14], NSForegroundColorAttributeName : [UIColor colorWithHexString:@"#333333"]} forState:UIControlStateNormal];
self.navigationItem.rightBarButtonItem = allReadItem;
```

解决办法使用自定义按钮:

```
UIButton *btn = [UIButton buttonWithType:UIButtonTypeCustom];
    btn.titleLabel.font = [UIFont pingFangSCWithSize:14];
    [btn setTitle:@"全部已读" forState:UIControlStateNormal];
    [btn setTitleColor:[UIColor colorWithHexString:@"#333333"] forState:UIControlStateNormal];
    [btn addTarget:self action:@selector(allReadItemDidClicked:) forControlEvents:UIControlEventTouchUpInside];
    [btn sizeToFit]; //需要设置一下否则在iOS10会没有大小.
    UIBarButtonItem *allReadItem = [[UIBarButtonItem alloc] initWithCustomView:btn];
    self.navigationItem.rightBarButtonItem = allReadItem;
```