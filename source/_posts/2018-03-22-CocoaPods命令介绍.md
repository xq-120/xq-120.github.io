---
title: CocoaPods命令介绍
date: 2018-03-22 16:42:24
description: 
categories: CocoaPods
comments: true
---

### pod install和pod update
#### pod install
每次运行pod install命令，并且有下载、安装新的库时，CocoaPods会把这些安装的库的版本都写在Podfile.lock文件里面。这个文件记录你每个安装库的版本号，并且锁定了这些版本。

后面再次运行的时候,它会解析Podfile.lock文件,对于存在在Podfile.lock文件中的则安装Podfile.lock文件中的版本的库而不会去检测该库是否有最新的版本可用,对于不在Podfile.lock文件中的,则根据Podfile文件指定的库版本下载安装.

pod install不会去更新已存在的库的版本.

#### pod update
当你运行 pod update PODNAME 命令时，CocoaPods会帮你更新到这个库的新版本，此时CocoaPods 会忽略Podfile.lock里面的限制，它会更新到这个库尽可能的新版本，**只要符合Podfile里面的版本限制**。

如果你运行pod update，后面没有跟库的名字，CocoaPods就会更新每一个Podfile里面的库到尽可能的最新版本。

pod update默认是更新repo的，而pod install是不更新的。所以我们有时候pod update时会特别慢.

举例:  
当你在Podfile文件中新增一个库时,你应该使用pod install而不是pod update.(因为pod update会更新已存在的库而我们的目的只是想安装一个库所以应该使用pod install)

你只有在想更新某个库的时候才使用pod update.

### pod setup
所有的项目的Podspec文件都托管在`https://github.com/CocoaPods/Specs`。第一次执行pod setup时，CocoaPods会将这些podspec文件clone到本地的 `~/.cocoapods/repos`目录下，这个文件比较大。所以第一次更新时非常慢，通常需要使用国内的镜像服务器或翻墙。如果clone已存在,则会更新repos文件.

```
xuequandeMacBook-Pro:~ xuequan$ pod setup
Setting up CocoaPods master repo
  $ /usr/bin/git -C /Users/xuequan/.cocoapods/repos/master fetch origin
  --progress
  remote: Counting objects: 38172, done.        
  remote: Compressing objects: 100% (117/117), done.        
  remote: Total 38172 (delta 13706), reused 13672 (delta 13672), pack-reused 24374        
  Receiving objects: 100% (38172/38172), 4.09 MiB | 560.00 KiB/s, done.
  Resolving deltas: 100% (26181/26181), completed with 3595 local objects.
  From https://github.com/CocoaPods/Specs
     cc16dc96ff1..f42ce3f56e0  master     -> origin/master
  $ /usr/bin/git -C /Users/xuequan/.cocoapods/repos/master rev-parse
  --abbrev-ref HEAD
  master
  $ /usr/bin/git -C /Users/xuequan/.cocoapods/repos/master reset --hard
  origin/master
  HEAD is now at f42ce3f56e0 [Add] HLDevice 1.0.0
warning: inexact rename detection was skipped due to too many files.
warning: you may want to set your diff.renameLimit variable to at least 3856 and retry the command.

CocoaPods 1.4.0 is available.
To update use: `sudo gem install cocoapods`

For more information, see https://blog.cocoapods.org and the CHANGELOG for this version at https://github.com/CocoaPods/CocoaPods/releases/tag/1.4.0

Setup completed
```

### pod search 
执行该命令后,CocoaPods会在`~/Library/Caches/CocoaPods`目录下创建`search_index.json`文件(目前16.6M).当你`pod search`一个内容时，就在这个本地目录下进行搜索。所以一些较新的pod库可能会搜索不到.比如目前刚出的`LKImageKit`.
解决办法:  
删除`~/Library/Caches/CocoaPods`目录下的`search_index.json`文件.
再次执行`pod search 'xxx'`, CocoaPods会重新创建该文件.

```
xuequandeMacBook-Pro:~ xuequan$ pod search LKImageKit
Creating search index for spec repo 'artsy'.. Done!
Creating search index for spec repo 'brion'.. Done!
Creating search index for spec repo 'master'.. Done!
```

```
-> LKImageKit (5.3.1)
   LKImageKit
   pod 'LKImageKit', '~> 5.3.1'
   - Homepage: https://github.com/Tencent/LKImageKit
   - Source:   https://github.com/Tencent/LKImageKit.git
   - Versions: 5.3.1 [master repo]
(END)
```

### Podfile.lock文件
该文件在第一次运行pod install后生成,用于跟踪每一个安装的Pod库的版本.当修改了Podfile文件中某个库的版本或pod update被执行,将生成新的Podfile.lock文件.

Q:如何保证项目中的成员使用的pod库版本一致?  
A:提交Podfile.lock文件.As a reminder, even if your policy is not to commit the Pods folder into your shared repository, you should always commit & push your Podfile.lock file.

总结下就是:

* 忽略Pods文件夹
* 尽量在Podfile文件中指明库的版本
* 提交Podfile.lock文件

注意:仅在Podfile中写死库的版本并不能保证团队各成员使用的库的版本一致.这是因为第三方库自己有可能也依赖某个库.因此提交Podfile.lock文件才是正解.

### Manifest.lock
Manifest.lock 是 Podfile.lock 的副本，每次只要生成 Podfile.lock 时就会生成一个一样的 Manifest.lock 存储在 Pods 文件夹下。在每次项目 Build 的时候，会跑一下脚本检查一下 Podfile.lock 和 Manifest.lock 是否一致：

Build phases -> [CP] Check Pods Manifest.lock

```
diff "${PODS_PODFILE_DIR_PATH}/Podfile.lock" "${PODS_ROOT}/Manifest.lock" > /dev/null
if [ $? != 0 ] ; then
    # print error to STDERR
    echo "error: The sandbox is not in sync with the Podfile.lock. Run 'pod install' or update your CocoaPods installation." >&2
    exit 1
fi
# This output is used by Xcode 'outputs' to avoid re-running this script phase.
echo "SUCCESS" > "${SCRIPT_OUTPUT_FILE_0}"
```
因此如果你有提交Pods文件夹,那么Manifest.lock也有可能冲突.解决冲突的时候不要忘记它,并且确保Manifest.lock和Podfile.lock一致.

### pod库版本号限定
eg:  
`pod 'AFNetworking', '~> 1.0'` 版本号可以是1.0，可以是1.1 ... 1.9，但必须小于2.

详细介绍如下:

*	`= 0.1 Version 0.1.`
*	`> 0.1 Any version higher than 0.1.`
*	`>= 0.1 Version 0.1 and any higher version.`
*	`< 0.1 Any version lower than 0.1.`
*	`<= 0.1 Version 0.1 and any lower version.`
*	`~> 0.1.2 Version 0.1.2 and the versions up to 0.2, not including 0.2. This operator works based on the last component that you specify in your version requirement. The example is equal to >= 0.1.2 combined with < 0.2.0 and will always match the latest known version matching your requirements.`

`pod 'AFNetworking',`  // 不指定版本号，任何版本都可以

### 参考 
[pod install vs. pod update](https://guides.cocoapods.org/using/pod-install-vs-update.html)

[podfile文件语法规则](https://guides.cocoapods.org/syntax/podfile.html#pod)

[深入理解 CocoaPods](https://objccn.io/issue-6-4/)	