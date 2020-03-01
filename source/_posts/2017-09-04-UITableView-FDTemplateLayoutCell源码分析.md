---
title: UITableView-FDTemplateLayoutCell源码分析
date: 2017-09-04 11:12:24
description: 
categories: 第三方库使用
tags: UITableView-FDTemplateLayoutCell
comments: false
---

## 高度的计算
通过从可重用队列里出列一个cell,并将该cell配置好要展示的数据后(ps:在使用时如果有一些会影响cell高度的代码没有包括在配置block里将导致计算不准),开始计算cell的高度:

1. 给cell添加约束:
`NSLayoutConstraint *widthFenceConstraint = [NSLayoutConstraint constraintWithItem:cell.contentView attribute:NSLayoutAttributeWidth relatedBy:NSLayoutRelationEqual toItem:nil attribute:NSLayoutAttributeNotAnAttribute multiplier:1.0 constant:contentViewWidth];`
对于10.2以上系统还需添加cell上下左右的约束:

	```
	if (isSystemVersionEqualOrGreaterThen10_2) {
	            // To avoid confilicts, make width constraint softer than required (1000)
	            widthFenceConstraint.priority = UILayoutPriorityRequired - 1;
	            
	            // Build edge constraints
	            NSLayoutConstraint *leftConstraint = [NSLayoutConstraint constraintWithItem:cell.contentView attribute:NSLayoutAttributeLeft relatedBy:NSLayoutRelationEqual toItem:cell attribute:NSLayoutAttributeLeft multiplier:1.0 constant:0];
	            NSLayoutConstraint *rightConstraint = [NSLayoutConstraint constraintWithItem:cell.contentView attribute:NSLayoutAttributeRight relatedBy:NSLayoutRelationEqual toItem:cell attribute:NSLayoutAttributeRight multiplier:1.0 constant:accessroyWidth];
	            NSLayoutConstraint *topConstraint = [NSLayoutConstraint constraintWithItem:cell.contentView attribute:NSLayoutAttributeTop relatedBy:NSLayoutRelationEqual toItem:cell attribute:NSLayoutAttributeTop multiplier:1.0 constant:0];
	            NSLayoutConstraint *bottomConstraint = [NSLayoutConstraint constraintWithItem:cell.contentView attribute:NSLayoutAttributeBottom relatedBy:NSLayoutRelationEqual toItem:cell attribute:NSLayoutAttributeBottom multiplier:1.0 constant:0];
	            edgeConstraints = @[leftConstraint, rightConstraint, topConstraint, bottomConstraint];
	            [cell addConstraints:edgeConstraints];
	        }
	```

2. 调用系统API`systemLayoutSizeFittingSize`计算出高度:`fittingHeight = [cell.contentView systemLayoutSizeFittingSize:UILayoutFittingCompressedSize].height;`
3. 移除之前添加的约束:  

	```
    [cell.contentView removeConstraint:widthFenceConstraint];
    if (isSystemVersionEqualOrGreaterThen10_2) {
        [cell removeConstraints:edgeConstraints];
    }
	```

最终得到cell的高度.

## 高度缓存的实现
通过给UITableView增加一个类别`(FDIndexPathHeightCache)`:  

```
@interface UITableView (FDIndexPathHeightCache)
/// Height cache by index path. Generally, you don't need to use it directly.
@property (nonatomic, strong, readonly) FDIndexPathHeightCache *fd_indexPathHeightCache;
@end
```
类别里有一个`FDIndexPathHeightCache`类型的属性`fd_indexPathHeightCache`是只读的,用于操作缓存:设置缓存,取出缓存,清除缓存,查询某个缓存是否存在等.`FDIndexPathHeightCache`内部扩展定义了一个用于保存缓存的二维数组.以后取缓存时是从该二维数组取得.默认高度是-1,即该行cell还未缓存高度.

## 缓存高度的清除时机
定义了一个类别`FDIndexPathHeightCacheInvalidation`,该类别通过method swizzle交换了一些系统方法的实现.

```
+ (void)load {
    // All methods that trigger height cache's invalidation
    SEL selectors[] = {
        @selector(reloadData),
        @selector(insertSections:withRowAnimation:),
        @selector(deleteSections:withRowAnimation:),
        @selector(reloadSections:withRowAnimation:),
        @selector(moveSection:toSection:),
        @selector(insertRowsAtIndexPaths:withRowAnimation:),
        @selector(deleteRowsAtIndexPaths:withRowAnimation:),
        @selector(reloadRowsAtIndexPaths:withRowAnimation:),
        @selector(moveRowAtIndexPath:toIndexPath:)
    };
    
    for (NSUInteger index = 0; index < sizeof(selectors) / sizeof(SEL); ++index) {
        SEL originalSelector = selectors[index];
        SEL swizzledSelector = NSSelectorFromString([@"fd_" stringByAppendingString:NSStringFromSelector(originalSelector)]);
        Method originalMethod = class_getInstanceMethod(self, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(self, swizzledSelector);
        method_exchangeImplementations(originalMethod, swizzledMethod);
    }
}

- (void)fd_reloadData {
    if (self.fd_indexPathHeightCache.automaticallyInvalidateEnabled) {
        [self.fd_indexPathHeightCache enumerateAllOrientationsUsingBlock:^(FDIndexPathHeightsBySection *heightsBySection) {
            [heightsBySection removeAllObjects];
        }];
    }
    FDPrimaryCall([self fd_reloadData];);
}
```
当调用这些可能会影响cell高度的方法时,就会将缓存清除.

## bug
在iOS 10.2以上可能会出现崩溃,崩溃在`systemLayoutSizeFittingSize:`处.一种解决办法:设置cell中label的preferredMaxLayoutWidth属性.