[IGListKit](http://blog.danthought.com/programming/2018/06/22/iglistkit/)

IGListKit demo编译

1. 可能需要升级bundler

`gem update --system`

[How can I remove default version of bundler?](https://stackoverflow.com/questions/57306611/how-can-i-remove-default-version-of-bundler)

2. 在macOS 10.15上`scripts/version.sh` 脚本需要修改为：

```
# exec defaults read "$(pwd)/Source/Info" CFBundleShortVersionString
/usr/libexec/PlistBuddy -c "print CFBundleShortVersionString" "$(pwd)/Source/Info.plist"
```

[Fix defaults cmd issue on macOS 10.15](https://github.com/Instagram/IGListKit/pull/1423)

3. 编译错误：`../../scripts/lint.sh: line 8: swiftlint: command not found`

解决办法：

`brew install swiftlint`

