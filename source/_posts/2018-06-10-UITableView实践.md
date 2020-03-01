---
title: UITableView实践
date: 2018-06-10 18:12:34
description: 
categories: UITableView
tags: 
---

### UITableView实践

#### contentInset
设置contentInset,会影响plain模式的组头位置.利用该特性取消plain模式组头吸顶效果.

```
//设置contentInset,会影响plain模式的组头位置.
self.tableView.contentInset = UIEdgeInsetsMake(-20, 0, 0, 0);
```
在设置界面,cell上的分割线显示是个头疼的问题.UI经常设计为组头第一个cell顶部,组尾最后一个cell底部不显示分割线.虽然plain模式刚好是这个效果,但是plain模式中组头具有吸顶效果,这里又不用它吸顶.使用group模式分割线的显示隐藏更蛋疼,自定义cell又有点麻烦.

示意图:

<img src="http://7xqoji.com1.z0.glb.clouddn.com/UITableView%E5%AE%9E%E8%B7%B5_%20contentInset_%E5%8F%96%E6%B6%88%E5%90%B8%E9%A1%B6.PNG" width="160" hegiht="284" />

利用上面的办法就可以取消plain模式组头吸顶效果.

实现代码如下:

```
- (void)scrollViewDidScroll:(UIScrollView *)scrollView
{
    if (scrollView == _tableView) {
        //去掉UItableview的section的headerview黏性
        CGFloat sectionHeaderHeight = 10;
        if (scrollView.contentOffset.y <= sectionHeaderHeight && scrollView.contentOffset.y >= 0) {
            scrollView.contentInset = UIEdgeInsetsMake(-scrollView.contentOffset.y, 0, 0, 0);
        } else if (scrollView.contentOffset.y >= sectionHeaderHeight) {
            scrollView.contentInset = UIEdgeInsetsMake(-sectionHeaderHeight, 0, 0, 0);
        }
    }
}


- (void)scrollViewWillEndDragging:(UIScrollView *)scrollView withVelocity:(CGPoint)velocity targetContentOffset:(inout CGPoint *)targetContentOffset
{
    CGFloat sectionHeaderHeight = 10;
    //解决停止时,第一个组头不出现的bug.
    if ((*targetContentOffset).y == sectionHeaderHeight) {
        //计算UIScrollView结束拖动所需的时间？这里暂时写死为0.85
        [UIView animateWithDuration:0.85 delay:0 usingSpringWithDamping:1 initialSpringVelocity:velocity.y options:UIViewAnimationOptionTransitionNone animations:^{
            scrollView.contentOffset = CGPointZero;
        } completion:^(BOOL finished) {

        }];
    }
}

```
网上找到的大都只实现了scrollViewDidScroll方法.但是只实现scrollViewDidScroll方法有bug:一直往上滑,松手后,第一个组头会不出现.所以这里解决一下这个问题.
