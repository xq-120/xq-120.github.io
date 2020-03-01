---
title: can't find gem cocoapods (>= 0.a)
date: 2017-09-10 15:37:26
description: 
categories: CocoaPods
tags: CocoaPods
---

## 更新ruby后,pod install出错.
xuequandeiMac:KingDynastyGaming xuequan$ pod install  
提示:

```
/Library/Ruby/Site/2.0.0/rubygems.rb:270:in `find_spec_for_exe': can't find gem cocoapods (>= 0.a) (Gem::GemNotFoundException) from /Library/Ruby/Site/2.0.0/rubygems.rb:298:in `activate_bin_path' from /usr/local/bin/pod:22:in `<main>'
```

解决办法:卸载cocoapods后重新安装
  
1. sudo gem uninstall cocoapods
2. gem install cocoapods
3. pod install
