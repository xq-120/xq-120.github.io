---
title: 制作CocoaPods公开库
date: 2020-04-16 00:18:19
tags: [公开库]
categories: CocoaPods
---

**1. 创建要公开的pod库**

主要包含pod源码和pod sepc文件、许可证等。

这个过程可以先在本地创建好，再到GitHub创建对应的仓库，然后将本地pod添加到远程仓库。

也可以先在GitHub创建好仓库，再clone到本地，然后在clone的目录下创建pod。

pod sepc文件的说明可以参考：[CocoaPods 系列之三 Podspec 语法说明](https://www.jianshu.com/p/75e19c92df50)

**2. 验证pod库**

pod源码和pod sepc文件都准备好后，别着急发布，先验证pod库的可用性:

`pod spec lint xxx.podsepc`

验证库坑:

`pod spec lint name.podsepc`

一直提示:CDN: trunk URL couldn't be downloaded

```
 -> XQSheet (2.0.0)
    - ERROR | [iOS] unknown: Encountered an unknown error (CDN: trunk URL couldn't be downloaded: https://raw.githubusercontent.com/CocoaPods/Specs/master/Specs/4/2/1/JKPresentationController/1.0.0/JKPresentationController.podspec.json, error: Failed to open TCP connection to raw.githubusercontent.com:443 (Connection refused - connect(2) for "raw.githubusercontent.com" port 443)) during validation.
```

解决办法：指定source。

`pod spec lint name.podsepc --sources='https://github.com/CocoaPods/Specs.git'`

**3.提交pod到GitHub，并打tag**

这个就是平时开发提交代码，打tag。

前面这几步和制作私有库是一样的都是准备工作。

**4. 发布**

要想发布一个公开库，需要先注册一个 CocoaPods 账号。

查看有没有注册过：

`pod trunk me`

如果没有那么先注册：

`pod trunk register EMAIL [NAME]`

eg:

`pod trunk register orta@cocoapods.org 'Orta Therox' --description='macbook air'`

注册好后，开始发布，使用命令：

`pod trunk push xxx.podspec`

将pod spec文件推送到cocoapods的官方仓库，这样别人才能找得到。

推送库坑:

`pod trunk push XQSheet.podspec`

```
Updating spec repo `trunk`

CocoaPods 1.9.1 is available.
To update use: `sudo gem install cocoapods`

For more information, see https://blog.cocoapods.org and the CHANGELOG for this version at https://github.com/CocoaPods/CocoaPods/releases/tag/1.9.1

Validating podspec
 -> XQSheet (2.0.0)
    - ERROR | [iOS] unknown: Encountered an unknown error (CDN: trunk URL couldn't be downloaded: https://raw.githubusercontent.com/CocoaPods/Specs/master/Specs/4/2/1/JKPresentationController/1.0.0/JKPresentationController.podspec.json, error: Failed to open TCP connection to raw.githubusercontent.com:443 (Connection refused - connect(2) for "raw.githubusercontent.com" port 443)) during validation.

[!] The spec did not pass validation, due to 1 error.
```

一直卡在这里,最后让终端翻墙后好了.

**5. 验证结果**

推送到CocoaPods成功后，我们可以搜索刚才制作的pod。此时一般是搜不到的，因为你本地的搜索索引文件，pod spec仓库都是旧的。

发布成功后搜索不到解决办法:

```
//删除本地索引
rm ~/Library/Caches/CocoaPods/search_index.json

//搜索
pod search [库名]

//更新spec仓库
pod repo update
```

**扩展**

上述步骤每次操作也都挺繁琐的，可以考虑自动化，参考：[一行命令发布 Pod 框架](https://ripperhe.com/2017/03/30/fastlane-pod/)



**其他**

上述pod是托管到某个平台上的，如果不想上传到服务器，也可以托管在本地玩玩。

安装本地pod

假设已经有了一个本地pod。`JKPresentationController.podspec`如下:

```
spec.source       = { :git => "./JKPresentationController/", :tag => "#{spec.version}" }
spec.source_files  = "JKPresentationController/*.swift"
```

git后面是源码本地路径.

Podfile文件:

```
source 'https://github.com/CocoaPods/Specs.git'

platform :ios, '10.0'
use_frameworks!

target 'JKPresentationControllerDemo' do

pod 'JKPresentationController', :path => '../'
pod 'SnapKit', '~> 5.0.0'

end
```

path后面是`JKPresentationController.podspec`所在的路径.

执行pod install后,Pods工程下会出现一个`Development Pods`目录.该目录中包含你的源码文件夹.



相关阅读：[制作CocoaPods私有库](https://xq-120.github.io/2020/04/15/%E5%88%B6%E4%BD%9CCocoaPods%E7%A7%81%E6%9C%89%E5%BA%93/#more)







