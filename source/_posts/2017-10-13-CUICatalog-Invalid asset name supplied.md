---
title: CUICatalog-Invalid asset name supplied
date: 2017-10-13 10:12:24
description: 
categories: Warning
tags:
---

## CUICatalog-Invalid asset name supplied
iOS11中,
`UIImage *highlightedImage = [UIImage imageNamed:highlightedImageName];`如果图片名是nil,或者不存在对应的图片,那么控制台会打印下面的提示信息.一般是因为传了nil或要找的资源不存在导致.

2017-09-27 16:46:33.399319+0800 KingDynastyGaming[48313:2122542] [framework] CUICatalog: Invalid asset name supplied: '(null)'

问题就是要找到哪行代码导致出现了提示信息.  
调试办法:打符号断点:`+[UIImage imageNamed:]`.在出现这条信息的操作之前使能断点,这样根据函数调用栈就可以看到调用该方法的位置,使用命令`po $arg3`还可以查看图片的名字即第三个参数的值.

### 调试命令参数

The exception object is passed in as the first argument to `objc_exception_throw`. LLDB provides `$arg1..$argn` variables to refer to arguments in the correct calling convention, making it simple to print the exception details:

```
(lldb) po $arg1
(lldb) po [$arg1 name]
(lldb) po [$arg1 reason]
```
Make sure to select the objc_exception_throw frame in the call stack before executing these commands. See the "Advanced Debugging and the Address Sanitizer" in the WWDC15 session videos to see this performed on stage.

On the iPhone Simulator, all function arguments are passed on the stack, so the syntax is considerably more horrible. The shortest expression I could construct that gets to it is `*(id *)($ebp + 8)`. To make things less painful, I suggest using a convenience variable:

```
(gdb) set $exception = *(id *)($ebp + 8)
(gdb) po $exception
(gdb) po [$exception name]
(gdb) po [$exception reason]
```
You can also set `$exception` automatically whenever the breakpoint is triggered by adding a command list to the `objc_exception_throw` breakpoint.

(Note that in all cases I tested, the exception object was also present in the eax and edx registers at the time the breakpoint hit. I'm not sure that'll always be the case, though.)

Added from comment below:

In lldb, select the stack frame for objc_exception_throw and then enter this command:

`(lldb) po *(id *)($esp + 4)`

### 参考

[Xcode/LLDB: How to get information about an exception that was just thrown?](https://stackoverflow.com/questions/3327828/xcode-lldb-how-to-get-information-about-an-exception-that-was-just-thrown)

[Inspecting Obj-C parameters in gdb](http://www.clarkcox.com/blog/2009/02/04/inspecting-obj-c-parameters-in-gdb/)

[【OC刨根问底】-Runtime简单粗暴理解](http://www.jianshu.com/p/f900de4a1495)

[OC 运行时 (二)](http://blog.csdn.net/qq_27074387/article/details/50352613)