---
title: 制作CocoaPods私有库
date: 2020-04-15 16:26:53
tags: [私有库]
categories: CocoaPods
---

**1. 创建私有pod spec仓库**

因为制作的是私有库所以就不能把pod sepc文件存放到cocoapods的公开仓库中了，因此需要在GitHub上新建一个repository `XQSpecs`用于保存所有的私有pod spec。

如果已经有自己的私有pod spec仓库，则这一步可跳过。

**2. 创建私有pod库**

主要包含pod源码和pod sepc文件、许可证等。

创建pod库,可以使用命令：`pod lib create xxx`，创建好模板pod，然后基于模板修改。也可以全部自己手动创建。

进入本地目的目录，使用命令：`pod lib create xxx`，然后基于模板修改。

**3. 验证pod库**

pod源码和pod sepc文件都准备好后，别着急发布，先验证pod库的可用性:

`pod lib lint xxx.podspec`或者使用：`pod lib lint AlgorithmUtils.podspec --allow-warnings`，忽略警告。

**4. 提交pod到GitHub，并打tag**

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

**5. 发布**

如果本地还没有刚创建的pod spec私有仓库，那么需要执行：

`pod repo add XQSpecs https://github.com/xq-120/XQSpecs.git`

在本地生成XQSpecs目录。

发布pod:

`pod repo push XQSpecs xxx.podspec`

将.podspec文件推送到pod spec仓库中，成功之后就可以在其他项目里使用该pod了。

**6. 使用**

在其他项目使用：

```
source 'https://github.com/xq-120/XQSpecs.git'

platform :ios,'10.0'
use_frameworks!

target 'adsad' do
   pod 'AlgorithmUtils', '~> 0.1.0'
end
```

**7. 升级**
如果新增了功能或修复了某个bug，这个时候就需要升级pod版本并发布。这个过程基本上是重复上述步骤。

* pod新增功能源码变动，更改pod sepc里的pod版本
* 最好再验证一下pod库
* 提交pod到GitHub，并打tag
* 发布。`pod repo push XQSpecs xxx.podspec`

如果pod修改频繁，就需要频繁升级版本，这无疑非常麻烦，因此可以直接指定git路径和branch，这样就可以避免每次修改后都需要升级版本了：

```
source 'https://github.com/xq-120/XQSpecs.git'

platform :ios,'10.0'
use_frameworks!

target 'adsad' do
  
   pod 'AlgorithmUtils', :git => 'https://github.com/xq-120/AlgorithmUtils.git', :branch => 'master'
     
end
```

还有一种操作：

* 修改源码提交后，删除原来tag

* 在新的提交记录上打tag（tag名称和之前一样）

* 进入本地`~/Library/Caches/CocoaPods/Pods/Release`找到之前下载缓存的pod源码删除
* 进入Xcode工程编辑Podfile文件注释掉pod，执行pod install，再放开注释pod，执行pod install（即重新安装pod）



相关阅读：[制作CocoaPods公开库]([https://xq-120.github.io/2020/04/16/%E5%88%B6%E4%BD%9CCocoaPods%E5%85%AC%E5%BC%80%E5%BA%93/](https://xq-120.github.io/2020/04/16/制作CocoaPods公开库/))

参考：[如何制作一个CocoaPods私有库](https://juejin.im/post/5accdbc86fb9a028ca534f2e)

