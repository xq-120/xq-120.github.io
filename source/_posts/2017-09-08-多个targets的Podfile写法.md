---
title: 多个targets的Podfile写法
date: 2017-09-08 11:11:11
description: 
categories: CocoaPods
tags: CocoaPods
comments: true
---

## 多个targets的Podfile写法
项目里面添加了多个target,每次pod install后,其他target就出现找不到`#import <AFNetworking.h>`这样的编译错误,花了三个多小时才找到了原因:是因为Podfile文件只给其中一个target配置了,所以导致了上述的问题.  
正确写法: 

```
platform :ios, '8.0'

def shared_pods
    pod 'AFNetworking', '3.1.0'
    pod 'MJExtension', '3.0.13'
    pod 'SDWebImage', '4.0.0'
    pod 'FMDB', '2.6.2'
    pod 'MJRefresh', '3.1.12'
    pod 'SDCycleScrollView', '1.66'
    pod 'Bugly', '2.4.8'
    pod 'MBProgressHUD', '1.0.0'
    pod 'Masonry', '1.0.2'
    pod 'IQKeyboardManager', '4.0.9'
    pod 'TZImagePickerController', '1.7.9'
    pod 'RealReachability', '1.1.9'
    pod 'MMDrawerController', '0.6.0'
    pod 'UMengAnalytics-NO-IDFA'
end

target 'xxx1' do
    shared_pods
end

target 'xxx2' do
    shared_pods
end

```

参考:[CocoaPods使用注意事项](http://www.jianshu.com/p/aa24cb9644a0)