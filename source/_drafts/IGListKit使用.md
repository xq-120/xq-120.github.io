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



### IGListDiff实现

diff算法是用来比较旧新两个数组的不同，并找出从旧数组变化到新数组需要的操作，这里允许的编辑操作包括：

1. 更新元素（update）
2. 插入一个元素（insert）
3. 删除一个元素（delete）
4. 移动一个元素（move）

界面只需要更新这些变化，而不需要全部更新。

涉及到的数据结构有3个：一个key-value的符号表(symbol table)、两个数组(OA, NA)。

符号表里的key是数组元素的diffIdentifier，value是IGListEntry。

所以IGList才定义了一个协议，数组里的元素必须实现：

```objc
@protocol IGListDiffable

- (nonnull id<NSObject>)diffIdentifier;

- (BOOL)isEqualToDiffableObject:(nullable id<IGListDiffable>)object;

@end
```

IGListEntry：

```objc
/// Used to track data stats while diffing.
struct IGListEntry {
    /// The number of times the data occurs in the old array
    NSInteger oldCounter = 0; //元素出现在旧数组里的次数
    /// The number of times the data occurs in the new array
    NSInteger newCounter = 0; //元素出现在新数组里的次数
    /// The indexes of the data in the old array
    stack<NSInteger> oldIndexes; //元素在旧数组里的index
    /// Flag marking if the data has been updated between arrays by checking the isEqual: method
    BOOL updated = NO;
};
```

IGListRecord：

```objc
/// Track both the entry and algorithm index. Default the index to NSNotFound
struct IGListRecord {
    IGListEntry *entry;
    mutable NSInteger index;

    IGListRecord() {
        entry = NULL;
        index = NSNotFound;
    }
};
```

entry可以让我们快速访问到数据对应的entry，index则用于记录一个新数据在旧数据中的位置（如果存在的情况下）或一个旧数据在新数据中的位置（如果存在的情况下）。

这里OA里面保存的是旧数组里每个元素对应的IGListRecord。

这里NA里面保存的是新数组里每个元素对应的IGListRecord。

可以预见如果一个元素在旧新数组都有出现，则OA，NA里的entry是同一个。

pass1

```objc
// pass 1
// create an entry for every item in the new array
// increment its new count for each occurence
vector<IGListRecord> newResultsArray(newCount); //初始化newResultsArray
for (NSInteger i = 0; i < newCount; i++) {
    id<NSObject> key = IGListTableKey(newArray[i]);
    IGListEntry &entry = table[key];
    entry.newCounter++;

    // add NSNotFound for each occurence of the item in the new array
    entry.oldIndexes.push(NSNotFound);

    // note: the entry is just a pointer to the entry which is stack-allocated in the table
    newResultsArray[i].entry = &entry;
}
```

顺序遍历新数组，为新数组里的每个元素创建一个entry并保存到符号表里和NA里。因为记录的是新数组的所以entry的oldCounter始终是0。

newCounter++

oldIndexes则push NSNotFound。

pass 2

```objc
// pass 2
// update or create an entry for every item in the old array
// increment its old count for each occurence
// record the original index of the item in the old array
// MUST be done in descending order to respect the oldIndexes stack construction
vector<IGListRecord> oldResultsArray(oldCount); //初始化oldResultsArray
for (NSInteger i = oldCount - 1; i >= 0; i--) {
    id<NSObject> key = IGListTableKey(oldArray[i]);
    IGListEntry &entry = table[key];
    entry.oldCounter++;

    // push the original indices where the item occurred onto the index stack
    entry.oldIndexes.push(i);

    // note: the entry is just a pointer to the entry which is stack-allocated in the table
    oldResultsArray[i].entry = &entry;
}
```

倒序遍历旧数组，为每个旧数组里的元素关联一个entry并保存到符号表里和OA里。这个entry可能是新创建的，也可能是符号表里已存在的。为什么会是已存在的？因为如果这个元素在新旧数组都有出现，则在pass1中就已经创建并保存在符号表里。

oldCounter++

oldIndexes压入元素在旧数组里的位置。

pass3

```objc
// pass 3
// handle data that occurs in both arrays
for (NSInteger i = 0; i < newCount; i++) {
    IGListEntry *entry = newResultsArray[i].entry;

    // grab and pop the top original index. if the item was inserted this will be NSNotFound
    NSCAssert(!entry->oldIndexes.empty(), @"Old indexes is empty while iterating new item %li. Should have NSNotFound", (long)i);
    const NSInteger originalIndex = entry->oldIndexes.top();
    entry->oldIndexes.pop();

    if (originalIndex < oldCount) {  //这里没看懂？？？
        const id<IGListDiffable> n = newArray[i];
        const id<IGListDiffable> o = oldArray[originalIndex];
        switch (option) {
            case IGListDiffPointerPersonality:
                // flag the entry as updated if the pointers are not the same
                if (n != o) {
                    entry->updated = YES;
                }
                break;
            case IGListDiffEquality:
                // use -[IGListDiffable isEqualToDiffableObject:] between both version of data to see if anything has changed
                // skip the equality check if both indexes point to the same object
                if (n != o && ![n isEqualToDiffableObject:o]) {
                    entry->updated = YES;
                }
                break;
        }
    }
    if (originalIndex != NSNotFound
        && entry->newCounter > 0
        && entry->oldCounter > 0) { //在旧新数组都出现
        // if an item occurs in the new and old array, it is unique
        // assign the index of new and old records to the opposite index (reverse lookup)
        newResultsArray[i].index = originalIndex;
        oldResultsArray[originalIndex].index = i;
    }
}
```

处理在旧新数组都出现的元素。在旧新数组中都出现表明该元素可能需要移动或者移动并更新或者位置没变只是单纯的更新或者什么都不用操作。但不管怎样最后都在NA中的record的index记录元素在旧数组中的位置，在OA中的record的index记录元素在新数组中的位置。主要作用为：

1. 为后面的move操作记录必要的信息。
2. 筛选出需要删除的旧元素
3. 筛选出需要插入的新元素

pass4

```objc
// track offsets from deleted items to calculate where items have moved
vector<NSInteger> deleteOffsets(oldCount), insertOffsets(newCount);
NSInteger runningOffset = 0;

// iterate old array records checking for deletes
// incremement offset for each delete
for (NSInteger i = 0; i < oldCount; i++) {
    deleteOffsets[i] = runningOffset;
    const IGListRecord record = oldResultsArray[i];
    // if the record index in the new array doesn't exist, its a delete
    if (record.index == NSNotFound) {
        addIndexToCollection(returnIndexPaths, mDeletes, fromSection, i);
        runningOffset++;
    }

    addIndexToMap(returnIndexPaths, fromSection, i, oldArray[i], oldMap);
}
```

顺序遍历OA，取出record，index为NSNotFound的，则代表该元素不存在NA里，所以需要删除。将其添加到mDeletes操作数组里。

为什么OA里record的index为NSNotFound的元素就是要删除的？因为record的index默认是NSNotFound，如果该元素也存在于NA里那么经过pass3的处理，index必然不是NSNotFound。现在是NSNotFound，说明就不存在于NA里，所以需要删除。

mDeletes里记录了需要删除旧数组里位置i处的元素。

pass5

```objc
// reset and track offsets from inserted items to calculate where items have moved
    runningOffset = 0;

for (NSInteger i = 0; i < newCount; i++) {
    insertOffsets[i] = runningOffset;
    const IGListRecord record = newResultsArray[i];
    const NSInteger oldIndex = record.index;
    // add to inserts if the opposing index is NSNotFound
    if (record.index == NSNotFound) {
        addIndexToCollection(returnIndexPaths, mInserts, toSection, i);
        runningOffset++;
    } else {
        // note that an entry can be updated /and/ moved
        if (record.entry->updated) {
            addIndexToCollection(returnIndexPaths, mUpdates, fromSection, oldIndex);
        }

        // calculate the offset and determine if there was a move
        // if the indexes match, ignore the index
        const NSInteger insertOffset = insertOffsets[i];
        const NSInteger deleteOffset = deleteOffsets[oldIndex];
        if ((oldIndex - deleteOffset + insertOffset) != i) {
            id move;
            if (returnIndexPaths) {
                NSIndexPath *from = [NSIndexPath indexPathForItem:oldIndex inSection:fromSection];
                NSIndexPath *to = [NSIndexPath indexPathForItem:i inSection:toSection];
                move = [[IGListMoveIndexPath alloc] initWithFrom:from to:to];
            } else {
                move = [[IGListMoveIndex alloc] initWithFrom:oldIndex to:i];
            }
            [mMoves addObject:move];
        }
    }

    addIndexToMap(returnIndexPaths, toSection, i, newArray[i], newMap);
}
```

顺序遍历NA。取出record，index为NSNotFound的，代表是新插入的元素。所以需要插入。将其添加到mInserts操作数组里。

为什么NA里record的index为NSNotFound的元素就是要插入的？理由同上。

当index不为NSNotFound则说明该元素可能需要更新或移动。如果需要更新则添加到mUpdates操作数组里。如果需要move则添加到mMoves操作数组里。

mInserts里记录了新元素需要插入在新数组的位置i处。

mUpdates里记录了旧数组的位置oldIndex的元素需要更新。

mMoves里记录的了是从旧数组的位置oldIndex移动到新数组的位置i。

有了mDeletes，mInserts，mUpdates，mMoves就得到了从旧数组到新数组的变化。究竟先应用哪个操作呢？是不是随便都可以。其实不是的，必须按照先mUpdates，再mDeletes，再mInserts，最后mMoves。

```objc
result = IGListDiff(oldViewModels, self.viewModels, IGListDiffEquality);
        
[result.updates enumerateIndexesUsingBlock:^(NSUInteger oldUpdatedIndex, BOOL *stop) {
    id identifier = [oldViewModels[oldUpdatedIndex] diffIdentifier];
    const NSInteger indexAfterUpdate = [result newIndexForIdentifier:identifier];
      if (indexAfterUpdate != NSNotFound) {
          UICollectionViewCell<IGListBindable> *cell = [collectionContext cellForItemAtIndex:oldUpdatedIndex
                                                                           sectionController:self];
          [cell bindViewModel:self.viewModels[indexAfterUpdate]];
      }
	}];

  if (IGListExperimentEnabled(self.collectionContext.experiments, IGListExperimentInvalidateLayoutForUpdates)) {
      [batchContext invalidateLayoutInSectionController:self atIndexes:result.updates];
  }
  [batchContext deleteInSectionController:self atIndexes:result.deletes];
  [batchContext insertInSectionController:self atIndexes:result.inserts];

  for (IGListMoveIndex *move in result.moves) {
      [batchContext moveInSectionController:self fromIndex:move.from toIndex:move.to];
  }

  self.state = IGListDiffingSectionStateUpdateApplied;
} completion:^(BOOL finished) {
  self.state = IGListDiffingSectionStateIdle;
  if (completion != nil) {
      completion(YES);
  }
}];

- (void)ig_applyBatchUpdateData:(IGListBatchUpdateData *)updateData {    
    [self deleteItemsAtIndexPaths:updateData.deleteIndexPaths];
    [self insertItemsAtIndexPaths:updateData.insertIndexPaths];
    [self reloadItemsAtIndexPaths:updateData.updateIndexPaths];

    for (IGListMoveIndexPath *move in updateData.moveIndexPaths) {
        [self moveItemAtIndexPath:move.from toIndexPath:move.to];
    }

    for (IGListMoveIndex *move in updateData.moveSections) {
        [self moveSection:move.from toSection:move.to];
    }

    [self deleteSections:updateData.deleteSections];
    [self insertSections:updateData.insertSections];
}
```



### 问题

#### 为什么pass 2 必须是倒序遍历旧数组？

这是因为 `IGListEntry` 里定义的oldIndexes是栈结构

```c++
/// Used to track data stats while diffing.
struct IGListEntry {
    /// The number of times the data occurs in the old array
    NSInteger oldCounter = 0;
    /// The number of times the data occurs in the new array
    NSInteger newCounter = 0;
    /// The indexes of the data in the old array
    stack<NSInteger> oldIndexes;
    /// Flag marking if the data has been updated between arrays by checking the isEqual: method
    BOOL updated = NO;
};
```

而在pass 3里处理公共元素是顺序遍历newResultsArray的。

```c++
// pass 3
// handle data that occurs in both arrays
for (NSInteger i = 0; i < newCount; i++) {
    IGListEntry *entry = newResultsArray[i].entry;

    // grab and pop the top original index. if the item was inserted this will be NSNotFound
    NSCAssert(!entry->oldIndexes.empty(), @"Old indexes is empty while iterating new item %li. Should have NSNotFound", (long)i);
    const NSInteger originalIndex = entry->oldIndexes.top();
    entry->oldIndexes.pop();

    if (originalIndex < oldCount) {
        const id<IGListDiffable> n = newArray[i];
        const id<IGListDiffable> o = oldArray[originalIndex];
        switch (option) {
            case IGListDiffPointerPersonality:
                // flag the entry as updated if the pointers are not the same
                if (n != o) {
                    entry->updated = YES;
                }
                break;
            case IGListDiffEquality:
                // use -[IGListDiffable isEqualToDiffableObject:] between both version of data to see if anything has changed
                // skip the equality check if both indexes point to the same object
                if (n != o && ![n isEqualToDiffableObject:o]) {
                    entry->updated = YES;
                }
                break;
        }
    }
    if (originalIndex != NSNotFound
        && entry->newCounter > 0
        && entry->oldCounter > 0) {
        // if an item occurs in the new and old array, it is unique
        // assign the index of new and old records to the opposite index (reverse lookup)
        newResultsArray[i].index = originalIndex;
        oldResultsArray[originalIndex].index = i;
    }
}
```

由于栈先入后出的特点，所以pass 2 必须是倒序遍历旧数组。要不然originalIndex 索引值可能会不正确。

eg: t 的originalIndex 

```
kitten
sitting
```



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

[IGListKit 学习(1) 一种高效的 diff 算法](https://sunshineyg888.github.io/2019/12/15/IGListKit%E5%AD%A6%E4%B9%A0(1)--%E9%AB%98%E6%95%88%E7%9A%84diff%E7%AE%97%E6%B3%95/)   不错

