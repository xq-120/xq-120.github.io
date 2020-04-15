---
title: 基于GitHub制作CocoaPods私有库
date: 2020-04-15 16:26:53
tags: [私有库]
categories: CocoaPods
---

1.创建pod spec私有仓库

因为制作的是私有库所以就不能把pod sepc文件放到cocoapods的公开仓库中了，因此需要在GitHub上新建一个repository `XQSpecs`用于保存所有的私有pod spec.

2.创建pod库

主要是pod sepc文件和源码。

创建pod库,可以使用命令：`pod lib create xxx`，创建好模板pod，然后基于模板修改。也可以全部自己手动创建。

进入本地目的目录，使用命令：`pod lib create xxx`，然后基于模板修改。

3.验证pod库

发布之前先验证pod库的可用性。

`pod lib lint xxx.podspec`或者使用：`pod lib lint AlgorithmUtils.podspec --allow-warnings`，忽略警告。

4.提交源码到GitHub，并打tag

在GitHub上新建一个repository用于保存pod.

创建完成后，回到本地pod目录执行：

```
git remote add origin https://github.com/xq-120/AlgorithmUtils.git
git push -u origin master
```

将已存在的本地git目录关联到远程地址。

提交代码到远程：

```
git add .
git commit -a -m "0.1.0"
git push
```

打tag并推送到远程:

```
git tag 0.1.0
git push origin 0.1.0
```

5.发布

如果本地还没有刚创建的pod spec私有仓库，那么需要执行：

`pod repo add XQSpecs https://github.com/xq-120/XQSpecs.git`

在本地生成XQSpecs目录。

发布pod:

`pod repo push XQSpecs xxx.podspec`

将.podspec文件推送到pod spec仓库中，成功之后就可以在其他项目里使用该pod了。

6.使用

在其他项目使用：

```
source 'https://github.com/xq-120/XQSpecs.git'

platform :ios,'10.0'
use_frameworks!

target 'adsad' do
   pod 'AlgorithmUtils', '~> 0.1.0'
end
```

如果pod修改频繁，就需要频繁升级版本，这无疑非常麻烦，因此可以直接指定git路径和branch，这样就可以避免每次修改后都需要升级版本了：

```
source 'https://github.com/xq-120/XQSpecs.git'

platform :ios,'10.0'
use_frameworks!

target 'adsad' do
  
   pod 'AlgorithmUtils', :git => 'https://github.com/xq-120/AlgorithmUtils.git', :branch => 'master'
     
end
```



参考：[如何制作一个CocoaPods私有库](https://juejin.im/post/5accdbc86fb9a028ca534f2e)

