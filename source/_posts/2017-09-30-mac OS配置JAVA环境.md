---
title: mac OS配置JAVA环境
date: 2017-09-30 15:12:24
description: 
categories:
tags:
---

## mac OS配置JAVA环境

* 下载安装最新版的JDK安装包:<http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html>,安装完成后查看下Java的版本后面配置要用到:`java -version`

	```
	admindeMacBook-Pro:未命名文件夹 admin$ java -version
	java version "1.8.0_144"
	Java(TM) SE Runtime Environment (build 1.8.0_144-b01)
	Java HotSpot(TM) 64-Bit Server VM (build 25.144-b01, mixed mode)
	```
* 配置环境变量：在命令行中打开`sudo vim ~/.bash_profile`，在文件后面加入下面内容：

	```
	export JAVA_HOME="/Library/Java/JavaVirtualMachines/jdk1.8.0_144.jdk/Contents/Home" 
	export PATH="$JAVA_HOME/bin:$PATH" 
	export CLASSPATH=".:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar"
	```
	note:`/jdk1.8.0_144.jdk`里面的版本字符串需要替换为上面提到的版本字符串.
	完成后保存,并使用`source ~/.bash_profile`应用该变更。
