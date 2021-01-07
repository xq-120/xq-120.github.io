### IGListKit demo编译

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



```
- (NSArray<id <IGListDiffable>> *)objectsForListAdapter:(IGListAdapter *)listAdapter;
```

it will only be called when you tell the infrastructure to update.



IGListBindingSectionController

可以参考IGListKit 官方demo的MonthSectionController类，或者参考 [IGListKit-Binding-Guide](https://github.com/rnystrom/IGListKit-Binding-Guide)。

### 参考

[Refactoring at Scale – Lessons of Rewriting Instagram’s Feed](https://academy.realm.io/posts/tryswift-ryan-nystrom-refactoring-at-scale-lessons-learned-rewriting-instagram-feed/)

[IGListKit](http://blog.danthought.com/programming/2018/06/22/iglistkit/)

[IGListKit-Github](https://github.com/Instagram/IGListKit)