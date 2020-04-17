---
title: iOS UIApplicationState状态变化
date: 2018-12-09 23:48:59
description: 
categories: 
- UIKit
tags: 
- UIApplicationState
comments: true
---

实验设备:iPhone5s,10.3.3系统.

应用状态:

```
typedef NS_ENUM(NSInteger, UIApplicationState) {
    UIApplicationStateActive, //0
    UIApplicationStateInactive, //1
    UIApplicationStateBackground //2
} NS_ENUM_AVAILABLE_IOS(4_0);
```

APP未启动,点击图片执行顺序:

```
默认	23:07:51.443320 +0800	pictureInMemory	-[AppDelegate application:didFinishLaunchingWithOptions:]  应用状态:1

默认	23:07:51.515796 +0800	pictureInMemory	-[AppDelegate applicationDidBecomeActive:]  应用状态:0
```

在应用处于UIApplicationStateActive时,分别进行如下操作:
#### 1.按下home键

```
默认	23:09:50.067690 +0800	pictureInMemory	-[AppDelegate applicationWillResignActive:]  应用状态:0
默认	23:09:50.602224 +0800	pictureInMemory	-[AppDelegate applicationDidEnterBackground:]  应用状态:2
```

1.1处于后台时,再点击图标

```
默认	23:11:34.897752 +0800	pictureInMemory	-[AppDelegate applicationWillEnterForeground:]  应用状态:2
默认	23:11:35.218295 +0800	pictureInMemory	-[AppDelegate applicationDidBecomeActive:]  应用状态:0
```

1.2处于后台时,双击home进入多任务界面点击自身

```
默认	23:19:12.051002 +0800	pictureInMemory	-[AppDelegate applicationWillEnterForeground:]  应用状态:2
默认	23:19:13.286377 +0800	pictureInMemory	-[AppDelegate applicationDidBecomeActive:]  应用状态:0
```

#### 2. 按下锁屏键

```
默认	23:13:07.542846 +0800	pictureInMemory	-[AppDelegate applicationWillResignActive:]  应用状态:0
默认	23:13:07.543049 +0800	pictureInMemory	-[AppDelegate applicationDidEnterBackground:]  应用状态:2
```

2.1解锁

```
默认	23:17:48.176320 +0800	pictureInMemory	-[AppDelegate applicationWillEnterForeground:]  应用状态:2
默认	23:17:48.872183 +0800	pictureInMemory	-[AppDelegate applicationDidBecomeActive:]  应用状态:0
```

#### 3. 双击home进入多任务界面

```
默认	23:14:28.340768 +0800	pictureInMemory	-[AppDelegate applicationWillResignActive:]  应用状态:0
```

3.1多任务点击其他APP

```
默认	23:15:14.853798 +0800	pictureInMemory	-[AppDelegate applicationDidEnterBackground:]  应用状态:2
```

3.2多任务点击自身

```
默认	23:15:53.643633 +0800	pictureInMemory	-[AppDelegate applicationDidBecomeActive:]  应用状态:0
```



