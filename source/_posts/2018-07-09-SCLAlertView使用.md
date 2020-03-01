---
title: SCLAlertView使用
date: 2018-07-09 15:26:27
description: 
categories: 第三方库使用
tags: SCLAlertView
---

## SCLAlertView使用
SCLAlertView设置

```
- (void)showPopView
{
    //提交成功
    SCLAlertView *alertView = [[SCLAlertView alloc] initWithNewWindowWidth:310];
    //去掉顶部圆圈
    [alertView removeTopCircle];
    //按钮水平放置
    [alertView setHorizontalButtons:YES];
    //设置出现动画
    alertView.showAnimationType = SCLAlertViewShowAnimationSimplyAppear;
    alertView.shouldDismissOnTapOutside = NO;
    
    //设置title的字体和颜色
    alertView.labelTitle.font = [UIFont pingFangSCWithSize:16];
    alertView.labelTitle.textColor = [UIColor blackColor];
    
    //设置富文本的subTitle
    alertView.attributedFormatBlock = ^NSAttributedString *(NSString *value) {
        return [NSString attributeString:value subString:@"13812341234" subStrColor:kRedColorRegular font:[UIFont pingFangSCWithSize:16] otherStrColor:[UIColor colorWithHexString:@"#818181"]];
    };
    [alertView setBodyTextFontFamily:@"PingFangSC-Regular" withSize:16];
    //设置普通文本的subTitle
//    alertView.viewText.font = [UIFont pingFangSCWithSize:16];
//    alertView.viewText.textColor = [UIColor colorWithHexString:@"#818181"];

    //设置Buttons, top circle and borders的颜色简便方法
    alertView.customViewColor = [UIColor redColor];
    //统一设置按钮的配置强大方法
    alertView.buttonFormatBlock = ^ NSDictionary *{
        return @{@"backgroundColor": [UIColor cyanColor]};
    };
    
    SCLButton *authPersonalBtn = [alertView addButton:@"取消" actionBlock:^(void) {
        DLog(@"取消按钮被按下");
    }];
    authPersonalBtn.titleLabel.font = [UIFont pingFangSCWithSize:16];
    authPersonalBtn.buttonFormatBlock = ^ NSDictionary *{
        return @{@"backgroundColor": [UIColor whiteColor], @"textColor": [UIColor colorWithHexString:@"#818181"], @"borderWidth": @0};
    };
    
    SCLButton *authCompanyBtn = [alertView addButton:@"去注册" actionBlock:^(void) {
        DLog(@"去注册按钮被按下");
    }];
    authCompanyBtn.titleLabel.font = [UIFont pingFangSCWithSize:16];
    //单独设置某个按钮的配置
    authCompanyBtn.buttonFormatBlock = ^ NSDictionary *{
        return @{@"backgroundColor": [UIColor whiteColor], @"textColor": [UIColor blackColor], @"borderWidth": @0};
    };
    
    [alertView showSuccess:self title:@"手机号码未注册" subTitle:@"手机号码 13812341234 未注册， 请注册后再登录。" closeButtonTitle:nil duration:0.0f];
}
```