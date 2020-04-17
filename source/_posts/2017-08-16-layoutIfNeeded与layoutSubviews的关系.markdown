---
title: layoutIfNeeded与layoutSubviews的关系
date: 2017-08-16 15:12:24
description: 
categories:
- UIKit
tags:
- layoutIfNeeded
---

动画`-layoutIfNeeded`父控制器的self.view,子控制器的子视图如果重写了`-layoutSubviews`也将动画布局.  
需求:导航栏根据子VC的tableview的位移动画.

```
if (contentOffsetY > 0) {
    //上滑
    
    [self.headerContainerView mas_updateConstraints:^(MASConstraintMaker *make) {
        make.top.equalTo(self.view.mas_topMargin).with.offset(startY - endY);
    }];
    
    [self.view setNeedsUpdateConstraints];
    [self.view updateConstraintsIfNeeded];
    
    [UIView animateWithDuration:0.3 animations:^{
        self.contentScrollView.contentInset = UIEdgeInsetsMake(-endY, 0, 0, 0);
        
        [self.view layoutIfNeeded]; //问题点
        
        self.headerContainerView.headerView.matchOtherDesBgView.alpha = 0;
        self.headerContainerView.headerView.team1NameLabel.alpha = 0;
        self.headerContainerView.headerView.team2NameLabel.alpha = 0;
        
        self.headerContainerView.headerView.teamLogoBgView.transform = CGAffineTransformMakeScale(0.5, 0.5);
        CGAffineTransform currentT = self.headerContainerView.headerView.teamLogoBgView.transform;
        CGAffineTransform translation = CGAffineTransformTranslate(currentT, 0, teamLogoBgViewOffsetY);
        self.headerContainerView.headerView.teamLogoBgView.transform = translation;
    }];
}
```
由于需要动画的改变父控制器里的view位移,又因为使用了xib,所以动画时需要在动画block里写`[self.view layoutIfNeeded];`但这句代码导致了子控制器的视图的`-layoutSubviews`方法被调用.于是子控制器的视图在布局时也会动画的进行(而这种效果不是你需要的从而变成了bug.)   
这里的bug就是:子控制器里的tableViewCell因为重写了`-layoutSubviews`,而cell上的子控件`KDGMatchDetailItemView`也重写了`-layoutSubviews`.所以`KDGMatchDetailItemView`在布局时出现动画的往右移动.  
解决办法:当导航栏收缩后,就不要执行`[self.view layoutIfNeeded];`加个判断条件:

```
if (contentOffsetY > 0  && self.contentScrollView.contentInset.top != -endY) {
	//动画代码
}
```

系统的`-layoutIfNeeded`的实现应该是调用了视图的`-layoutSubviews`方法.父视图`-layoutSubviews`方法的调用可能会引起子视图`-layoutSubviews`方法的调用.