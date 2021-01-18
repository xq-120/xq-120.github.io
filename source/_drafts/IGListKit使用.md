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



Diff

```swift
// Test
let a = ["A", "D", "F", "G", "T"]
let b = ["A", "F", "O", "X", "T"]
let diff = a.diff(b)
let c = a.apply(diff)
print(c)

let reload1 = DiffTransform.reload(4, a[4]) // at (i,j) = (5,5)
let delete2 = DiffTransform.delete(3, a[3]) // at (4,4)
let insert3 = DiffTransform.insert(3, b[3]) // at (3,4)
let insert4 = DiffTransform.insert(2, b[2]) // at (3,3)
let reload5 = DiffTransform.reload(2, a[2]) // at (3,2)
let delete6 = DiffTransform.delete(1, a[1]) // at (2,1)
let reload7 = DiffTransform.reload(0, a[0]) // at (1,1)
var diff2 = Diff<String>.init()
diff2.append(item: reload1)
diff2.append(item: delete2)
diff2.append(item: insert3)
diff2.append(item: insert4)
diff2.append(item: delete6)
diff2.append(item: reload7)
let d = a.apply(diff2)
print(d)
```

为什么获取插入或删除的转换时需要排序索引？

```swift
// Diff Container

struct Diff<T>: CustomStringConvertible {
    
    typealias Element = DiffTransform<T>
    
    private var _result: [Element] = []
    var result: [Element] {
        return self._result
    }
    
    var insertion: [Element] {
        return self.result.filter { (obj) -> Bool in
            return obj.isInsertion
        }.sorted { (obj1, obj2) -> Bool in
            return obj1.index < obj2.index //升序
        }
    }
    
    var deletion: [Element] {
        return self.result.filter { (obj) -> Bool in
            return obj.isDeletion
        }.sorted { (obj1, obj2) -> Bool in
            return obj1.index > obj2.index //降序
        }
    }
    
    var reload: [Element] {
        return self.result.filter { (obj) -> Bool in
            return obj.isReload
        }
    }
    
    mutating func append(item: Element) {
        self._result.append(item)
    }
    
    var description: String {
        return ""
    }
}
```

因为对于数组的插入操作：

插入的index范围只能是 0...arr.count

```
var arr = [1, 4, 5]
arr.insert(31, at: 0) //插入的index范围只能是 0...arr.count
print(arr)
```

所以对insertion操作的index进行一个升序操作确保不越界

同理，对于deletion操作的index进行一个降序操作也是确保不越界。这样就会先删除index大的，删除后数组个数变小，再删除index小的就不会有问题。

应用diff的顺序

```swift
// Apply Diff
func apply(_ diff: Diff<Element>) -> [Element] {
    var copy = self

    // Delete first
    for delete in diff.deletion {
        copy.remove(at: delete.index)
    }

    // Insert
    for insert in diff.insertion {
        copy.insert(insert.value, at: insert.index)
    }

    return copy
}
```

先对旧数组进行删除操作，再对旧数组进行插入操作。这样旧数组就变成了新数组。



### Levenshtein距离

莱文斯坦距离，又称Levenshtein距离，是编辑距离的一种。指两个字符串之间，由一个转成另一个所需的最少编辑操作次数。

允许的编辑操作包括：

1. 将一个字符替换成另一个字符

2. 插入一个字符

3. 删除一个字符

由俄罗斯科学家[Vladimir Levenshtein](https://en.wikipedia.org/wiki/Vladimir_Levenshtein)于1965年提出了这一概念。而Wagner-Fischer算法是levenshtein距离的高效解决办法。

![](https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/equation1.png)

[莱文斯坦距离 知乎](https://www.zhihu.com/question/315634571/answer/620984468)  

[莱文斯坦距离 zh-wiki](https://zh.wikipedia.org/wiki/%E8%90%8A%E6%96%87%E6%96%AF%E5%9D%A6%E8%B7%9D%E9%9B%A2)

[Levenshtein distance 编辑距离算法](https://hddhyq.github.io/2018/12/08/Levenshtein-distance-%E7%BC%96%E8%BE%91%E8%B7%9D%E7%A6%BB%E7%AE%97%E6%B3%95/)  挺不错的

### 参考

[Refactoring at Scale – Lessons of Rewriting Instagram’s Feed](https://academy.realm.io/posts/tryswift-ryan-nystrom-refactoring-at-scale-lessons-learned-rewriting-instagram-feed/)

[IGListKit](http://blog.danthought.com/programming/2018/06/22/iglistkit/)

[IGListKit-Github](https://github.com/Instagram/IGListKit)

[IGListKit官方文档](https://instagram.github.io/IGListKit/getting-started.html)

[Diff应用：从LCS到UICollectionView](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&amp;mid=2247484479&amp;idx=1&amp;sn=b0115be0958910e7b9bee48606a81296&amp;chksm=e9d0cfdddea746cbc89d6e78a2e8e547fd670e94f0c15fc10a9bc560459d46b7d998a88de581#rd)  字节的，但主要参考的Noah的博客

[Longest Common Subsequence Diff [Part 1]](https://nghiatran.me/longest-common-subsequence-diff-part-1)  Noah的博客很不错。这一篇主要介绍LCS的实现。

[Diff in iOS [Part 2]](https://nghiatran.me/diff-in-real-world-ios-part-2) 这一篇主要介绍LCS在iOS中的应用。

[A better way to update UICollectionView data in Swift with diff framework](https://medium.com/flawless-app-stories/a-better-way-to-update-uicollectionview-data-in-swift-with-diff-framework-924db158db86)

[Wagner–Fischer algorithm--wiki](https://en.wikipedia.org/wiki/Wagner%E2%80%93Fischer_algorithm)

[Heckel算法介绍](https://gist.github.com/ndarville/3166060)

