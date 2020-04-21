#### Flutter开发环境搭建

参考:[Flutter 开发环境搭建 ( For Mac )](https://segmentfault.com/a/1190000019177539)

**安装IDE开发工具**

`Flutter` 的集成开发环境 `IDE` 有 `Andriod Studio` 、`VS Code`、`IntelliJ`等,就用`Andriod Studio`就可以了.去官网下载`Andriod Studio`,然后安装.这个过程基本没啥问题.

需要注意的是:不同于`Xcode`下载下来后直接拎包入住,创建一个工程点击运行就完事了.`Andriod Studio`下载下来安装后其实是一个毛坯房,还要下载各种各样的插件才能运行一个工程.



**安装flutter**

下载flutter SDK:

可以去[官网](https://flutter.dev/docs/development/tools/sdk/releases)下,但由于你懂的原因会非常慢,可以参考 [Using Flutter in China](https://flutter.io/community/china) 提供的方法--将下载地址中的原域名替换为镜像域名去下载,macos版的如下:

`https://storage.flutter-io.cn/flutter_infra/releases/stable/macos/flutter_macos_v1.7.8+hotfix.4-stable.zip`

下载后解压到一个目录,并获取`flutter/bin` 的完整路径.

把刚刚的路径添加到系统的 PATH 变量中:

`export PATH=$PATH:$HOME/Documents/flutter/bin`

方便后续在任何终端直接使用Flutter 命令，而不用切换到特定目录.



**设置 Flutter 编译**

在终端运行 `flutter doctor` 命令查看是否需要安装其它依赖项来完成安装.按照提示安装即可.

这里主要说下android licenses坑.需要使用`flutter doctor --android-licenses`命令解决,

但执行后报错:

![](https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/image-20200316183031255.png)

原因:tool文件夹没找到.

解决办法:

打开Android studio的偏好设置,安装Android SDK Tools.很隐蔽.

![](https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/image-20200316182922159.png)

安装完成后,再执行`flutter doctor --android-licenses`就好了.

直到 `flutter doctor` 的运行结果都是 `[✓]` ，编译环境工具配置就 `OK` 了。



**创建并运行你的第一个Flutter app**

打开Android studio,新建一个flutter工程.然后运行.

坑:使用安卓模拟器运行时卡在Initializing gradle...

解决办法:等待,提示失败了继续运行等.等到必要的资源下载好了后,就可以了.当然也可以手动在外部下载好.



#### 在既有iOS项目中集成Flutter

按照官方教程操作后.

pod install报错:

```
[!] Invalid `Podfile` file: undefined method `pod' for main:Object.

 #  from /Users/xuequan/Documents/zhenai/zhenaiEmotionCounsel/Podfile:9
 #  -------------------------------------------
 #  flutter_application_path = '../ec_flutter'
 >  load File.join(flutter_application_path, '.ios', 'Flutter', 'podhelper.rb')
 #
 #  -------------------------------------------
```

网上的回答说是要在master channel才行.

执行`flutter doctor -v`命令查看当前是在哪个channel:

```
[✓] Flutter (Channel master, v1.16.1-pre.34, on Mac OS X 10.14.6 18G87, locale
    zh-Hans-CN)
    • Flutter version 1.16.1-pre.34 at /Users/xuequan/Documents/flutter
    • Framework revision e6b0f5f238 (25 hours ago), 2020-03-18 21:56:02 -0400
    • Engine revision 216c420a2c
    • Dart version 2.8.0 (build 2.8.0-dev.15.0 96cf889e6b)
```

或者`flutter channel`:

```
Flutter channels:
* master
  dev
  beta
  stable
```

没改之前我的是stable channel.所以需要切换到master chanel.

执行`flutter channel [<channel-name>]`切换:

```
flutter chanel master		#切换channel
flutter upgrade   #更新channel
```

切换完成后删除`.iOS`文件,重新执行`flutter build iOS`.

继续pod install,上面的报错没了,但报下面的错:

```
### Error

​```
LoadError - library not found for class Digest::MD5 -- digest/md5
/usr/local/Cellar/ruby/2.5.3_1/lib/ruby/2.5.0/digest.rb:16:in `const_missing'
/usr/local/lib/ruby/gems/2.5.0/gems/cocoapods-core-1.8.4/lib/cocoapods-core/source/metadata.rb:45:in `path_fragment'
/usr/local/lib/ruby/gems/2.5.0/gems/cocoapods-core-1.8.4/lib/cocoapods-core/source.rb:114:in `pod_path'
/usr/local/lib/ruby/gems/2.5.0/gems/cocoapods-core-1.8.4/lib/cocoapods-core/source.rb:270:in `search'
/usr/local/lib/ruby/gems/2.5.0/gems/cocoapods-core-1.8.4/lib/cocoapods-core/source/aggregate.rb:83:in `block in search'
/usr/local/lib/ruby/gems/2.5.0/gems/cocoapods-core-1.8.4/lib/cocoapods-core/source/aggregate.rb:83:in `select'
/usr/local/lib/ruby/gems/2.5.0/gems/cocoapods-core-1.8.4/lib/cocoapods-core/source/aggregate.rb:83:in `search'
/usr/local/lib/ruby/gems/2.5.0/gems/cocoapods-1.8.4/lib/cocoapods/resolver.rb:416:in `create_set_from_sources'
/usr/local/lib/ruby/gems/2.5.0/gems/cocoapods-1.8.4/lib/cocoapods/resolver.rb:385:in `find_cached_set'
/usr/local/lib/ruby/gems/2.5.0/gems/cocoapods-1.8.4/lib/cocoapods/resolver.rb:360:in `specifications_for_dependency'
/usr/local/lib/ruby/gems/2.5.0/gems/cocoapods-1.8.4/lib/cocoapods/resolver.rb:165:in `search_for'
/usr/local/lib/ruby/gems/2.5.0/gems/cocoapods-1.8.4/lib/cocoapods/resolver.rb:274:in `block in sort_dependencies'
/usr/local/lib/ruby/gems/2.5.0/gems/cocoapods-1.8.4/lib/cocoapods/resolver.rb:267:in `each'
/usr/local/lib/ruby/gems/2.5.0/gems/cocoapods-1.8.4/lib/cocoapods/resolver.rb:267:in `sort_by'
/usr/local/lib/ruby/gems/2.5.0/gems/cocoapods-1.8.4/lib/cocoapods/resolver.rb:267:in `sort_dependencies'
/usr/local/lib/ruby/gems/2.5.0/gems/molinillo-0.6.6/lib/molinillo/delegates/specification_provider.rb:53:in `block in sort_dependencies'
/usr/local/lib/ruby/gems/2.5.0/gems/molinillo-0.6.6/lib/molinillo/delegates/specification_provider.rb:70:in `with_no_such_dependency_error_handling'
/usr/local/lib/ruby/gems/2.5.0/gems/molinillo-0.6.6/lib/molinillo/delegates/specification_provider.rb:52:in `sort_dependencies'
/usr/local/lib/ruby/gems/2.5.0/gems/molinillo-0.6.6/lib/molinillo/resolution.rb:288:in `initial_state'
/usr/local/lib/ruby/gems/2.5.0/gems/molinillo-0.6.6/lib/molinillo/resolution.rb:210:in `start_resolution'
/usr/local/lib/ruby/gems/2.5.0/gems/molinillo-0.6.6/lib/molinillo/resolution.rb:168:in `resolve'
/usr/local/lib/ruby/gems/2.5.0/gems/molinillo-0.6.6/lib/molinillo/resolver.rb:43:in `resolve'
/usr/local/lib/ruby/gems/2.5.0/gems/cocoapods-1.8.4/lib/cocoapods/resolver.rb:94:in `resolve'
/usr/local/lib/ruby/gems/2.5.0/gems/cocoapods-1.8.4/lib/cocoapods/installer/analyzer.rb:986:in `block in resolve_dependencies'
/usr/local/lib/ruby/gems/2.5.0/gems/cocoapods-1.8.4/lib/cocoapods/user_interface.rb:64:in `section'
/usr/local/lib/ruby/gems/2.5.0/gems/cocoapods-1.8.4/lib/cocoapods/installer/analyzer.rb:984:in `resolve_dependencies'
/usr/local/lib/ruby/gems/2.5.0/gems/cocoapods-1.8.4/lib/cocoapods/installer/analyzer.rb:124:in `analyze'
/usr/local/lib/ruby/gems/2.5.0/gems/cocoapods-1.8.4/lib/cocoapods/installer.rb:410:in `analyze'
/usr/local/lib/ruby/gems/2.5.0/gems/cocoapods-1.8.4/lib/cocoapods/installer.rb:234:in `block in resolve_dependencies'
/usr/local/lib/ruby/gems/2.5.0/gems/cocoapods-1.8.4/lib/cocoapods/user_interface.rb:64:in `section'
/usr/local/lib/ruby/gems/2.5.0/gems/cocoapods-1.8.4/lib/cocoapods/installer.rb:233:in `resolve_dependencies'
/usr/local/lib/ruby/gems/2.5.0/gems/cocoapods-1.8.4/lib/cocoapods/installer.rb:156:in `install!'
/usr/local/lib/ruby/gems/2.5.0/gems/cocoapods-1.8.4/lib/cocoapods/command/install.rb:52:in `run'
/usr/local/lib/ruby/gems/2.5.0/gems/claide-1.0.2/lib/claide/command.rb:334:in `run'
/usr/local/lib/ruby/gems/2.5.0/gems/cocoapods-1.8.4/lib/cocoapods/command.rb:52:in `run'
/usr/local/lib/ruby/gems/2.5.0/gems/cocoapods-1.8.4/bin/pod:55:in `<top (required)>'
/usr/local/bin/pod:23:in `load'
/usr/local/bin/pod:23:in `<main>'
​```

――― TEMPLATE END ――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――

[!] Oh no, an error occurred.
```

感觉已经把cocoapods干废了.其他以前干净的工程也pod install不了了.

`sudo gem update --system`也报错:

```
ERROR:  Loading command: update (LoadError)
	dlopen(/usr/local/Cellar/ruby/2.5.3_1/lib/ruby/2.5.0/x86_64-darwin18/openssl.bundle, 9): Library not loaded: /usr/local/opt/openssl/lib/libssl.1.0.0.dylib
  Referenced from: /usr/local/Cellar/ruby/2.5.3_1/lib/ruby/2.5.0/x86_64-darwin18/openssl.bundle
  Reason: image not found - /usr/local/Cellar/ruby/2.5.3_1/lib/ruby/2.5.0/x86_64-darwin18/openssl.bundle
ERROR:  While executing gem ... (NoMethodError)
    undefined method `invoke_with_build_args' for nil:NilClass
```

猜测应该是load加密库时某些东西找不到导致的.

进入`/usr/local/Cellar`目录发现里面有两个文件夹:`openssl`和`openssl@1.1`.

`openssl`文件夹下包含:

`1.0.2p`、`1.0.2o_1`

`openssl@1.1`文件夹下包含:

`1.1.1d`

应该是上面的某个操作之后openssl库被切换到`openssl@1.1`版本去了.

使用`brew switch <name> <version>`切换本地软件版本:

`brew switch openssl 1.0.2p`

切回`1.0.2p`,再pod install就没问题了.这个是真的坑,网上搜也搜不到,爬坑半天.

其他:

homebrew,rvm,ruby,gem,cocoapods关系



#### 在既有 iOS 项目中添加 Flutter 页面

比如:原生页面弹出flutter页面.





#### 其他

环境变量配置

~/.bash_profile （一般在这个文件中添加用户级环境变量）

当mac上安装了zsh后，修改环境变量就需要在~/.zshrc中修改，比如加入代理:

```
export http_prox=http://10.1.1.1:8080
export https_proxy=http://10.1.1.1:8080
```

如果想要修改立即生效，需要执行:

```
source ~/.zshrc
```

[mac上使用zsh配置环境变量](https://www.cnblogs.com/mingaixin/p/6281795.html)

[Mac查看与修改系统默认shell](https://www.cnblogs.com/Scar007/p/12483284.html)

[Flutter从入门到精通](https://www.kancloud.cn/qtrjjs/flutter)

[flutter跨平台开发一点一滴分析系列文章](https://blog.csdn.net/zl18603543572/article/details/93532582)

[flutter中国](https://flutter.cn/)

[谷歌安卓开发者平台](https://developer.android.google.cn/guide)

