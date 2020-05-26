复现环境：需要老版本的Xcode工程，不能是Xcode11新搭建的。可下载银联官方Demo。

\#import "UIButton+XQABlock.h"

```
// 处理事件
- (void)handleControlEvent:(UIControlEvents)event withBlock:(void(^)(id sender))block {
    
    NSString *methodName = [UIControl eventName:event];
    
    NSMutableDictionary *opreations = (NSMutableDictionary*)objc_getAssociatedObject(self, &OperationKey);
    
    if(opreations == nil)
    {
        opreations = [[NSMutableDictionary alloc] init];
        objc_setAssociatedObject(self, &OperationKey, opreations, OBJC_ASSOCIATION_RETAIN);
    }
    [opreations setObject:block forKey:methodName];
    
    [self addTarget:self action:NSSelectorFromString(methodName) forControlEvents:event];
    
}

-(void)UIControlEventTouchUpOutside{[self callActionBlock:UIControlEventTouchUpOutside];}

// 设置响应的Block
- (void)callActionBlock:(UIControlEvents)event {
    
    NSMutableDictionary *opreations = (NSMutableDictionary*)objc_getAssociatedObject(self, &OperationKey);
    
    if(opreations == nil) return;
    void(^block)(id sender) = [opreations objectForKey:[UIControl eventName:event]];
    
    if (block) block(self);
    
}
```

测试代码：

```
UIButton *btn = [UIButton buttonWithType:UIButtonTypeCustom];
btn.frame = CGRectMake(100, 100, 100, 100);
btn.backgroundColor = [UIColor redColor];
[btn setTitle:@"Normal" forState:UIControlStateNormal];
[btn setTitle:@"Selected" forState:UIControlStateSelected];
[btn handleControlEvent:UIControlEventTouchUpInside withBlock:^(id sender) {
    printf("00000000\n");
}];
[self.view addSubview:btn];
```

点击按钮后`- (void)callActionBlock:(UIControlEvents)event`方法将不会执行。

符号断点：`-[UIButton callActionBlock:]`

<img src="https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20200526102523.png" style="zoom:50%;" />

注意符号断点：只会断点到UIButton该类自身的方法。如果callActionBlock是父类的方法则不会断点。

编译文件顺序：

<img src="https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20200526102905.png" style="zoom:50%;" />

打印方法列表：

<img src="https://raw.githubusercontent.com/xq-120/cloudImage/master/pictures/20200526102350.png" style="zoom:50%;" />

