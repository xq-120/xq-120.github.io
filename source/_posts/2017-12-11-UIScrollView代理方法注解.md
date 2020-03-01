---
title: UIScrollView代理方法
date: 2017-12-11 10:16:24
description: 
categories: UIKit
tags: UIScrollView
---

将要开始发生拖拽.
`- (void)scrollViewWillBeginDragging:(UIScrollView *)scrollView`

将要结束拖拽,velocity:结束时的速度;targetContentOffset:想要滚动到的指定位置.(使用该方法可以让UIScrollView停在我们想要的地方)
`- (void)scrollViewWillEndDragging:(UIScrollView *)scrollView withVelocity:(CGPoint)velocity targetContentOffset:(inout CGPoint *)targetContentOffset`

拖拽结束,decelerate=YES:表明拖拽结束的时候,scrollView还有速度,将会减速滑动一段距离,最终停止时会回调scrollViewDidEndDecelerating方法;=NO:表明拖拽结束,scrollView也随即停止,此时不会再回调scrollViewDidEndDecelerating方法.
`- (void)scrollViewDidEndDragging:(UIScrollView *)scrollView willDecelerate:(BOOL)decelerate`

已经结束减速.表明滚动停止.只有在停止前有速度的时候才会被回调.
`- (void)scrollViewDidEndDecelerating:(UIScrollView *)scrollView`

已经滚动.scrollView处于滚动状态时,该方法会被频繁调用,因此该方法里面不应该有太复杂的处理.动画scrollToItemAtIndexPath:atScrollPosition:animated:;并不会使该代理方法调用.  
`- (void)scrollViewDidScroll:(UIScrollView *)scrollView`

called when setContentOffset/scrollRectVisible:animated: finishes. not called if not animating. 因为动画原因滚动结束.
`- (void)scrollViewDidEndScrollingAnimation:(UIScrollView *)scrollView`