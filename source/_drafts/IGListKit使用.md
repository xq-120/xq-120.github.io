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

使用方法可以参考IGListKit 官方demo的MonthSectionController类，或者参考 [IGListKit-Binding-Guide](https://github.com/rnystrom/IGListKit-Binding-Guide)。

提供了一个统一的view-model配置方法：

```
- (void)bindViewModel:(id)viewModel;
```

如果有自己统一的配置方法也可以不用该类。所以一般还是继承IGListSectionController多一些。

### 参考

[Refactoring at Scale – Lessons of Rewriting Instagram’s Feed](https://academy.realm.io/posts/tryswift-ryan-nystrom-refactoring-at-scale-lessons-learned-rewriting-instagram-feed/)

[IGListKit](http://blog.danthought.com/programming/2018/06/22/iglistkit/)

[IGListKit-Github](https://github.com/Instagram/IGListKit)

[IGListKit官方文档](https://instagram.github.io/IGListKit/getting-started.html)

[Diff应用：从LCS到UICollectionView](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&amp;mid=2247484479&amp;idx=1&amp;sn=b0115be0958910e7b9bee48606a81296&amp;chksm=e9d0cfdddea746cbc89d6e78a2e8e547fd670e94f0c15fc10a9bc560459d46b7d998a88de581#rd)  字节的，但主要参考的Noah的博客

[Longest Common Subsequence Diff [Part 1]](https://nghiatran.me/longest-common-subsequence-diff-part-1)  Noah的博客很不错。这一篇主要介绍LCS的实现。

[Diff in iOS [Part 2]](https://nghiatran.me/diff-in-real-world-ios-part-2)

[A better way to update UICollectionView data in Swift with diff framework](https://medium.com/flawless-app-stories/a-better-way-to-update-uicollectionview-data-in-swift-with-diff-framework-924db158db86)

