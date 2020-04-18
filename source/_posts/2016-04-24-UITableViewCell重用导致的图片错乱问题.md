---
title: UITableViewCell重用导致的图片错乱问题
date: 2016-04-24 22:25:40
tags: 
- UITableViewCell
- cell重用问题
categories: 
- UIKit
---

之前做过一个视频信息列表展示的模块，cell很简单就是左边图片，右边文字信息。当时用的SDWebImage加载图片并没有看到图片错乱的情况。但是，如果是自己写的图片下载器，不注意处理是会导致图片错乱的。

今天写了个Demo，验证及解决这个问题。
实验环境：cell依然是左边图片，右边文字信息。图片两张，一张大图片A(风景)，一张小图片B(人物)，**采用自己实现的原始图片下载器**异步下载，block里回调设置cell的图片。要求偶数行的图片是风景，奇数行的图片是人物。
整个界面期望如下:
![期望tableview界面.jpg](http://upload-images.jianshu.io/upload_images/1503319-0c16709bb58a8af1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



但实际可能出现bug，如下图:
![bug界面.jpg](http://upload-images.jianshu.io/upload_images/1503319-396836dd909893b1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



#### 数据错乱原因分析

cell上的数据错乱显然是由于cell的重用导致的。由于图片是异步下载的，下载完成才给cell设置，但是在这个过程中用户可能会上下滑动，滑动的时候会导致cell的重用，比如第0行是设置大图片的，第11行是设置小图片的，用户在滑动的过程中，因为cell的重用第11行的cell可能使用的是第0行的cell，这时第0行的block回调设置的cell和第11行的block回调设置的cell是同一个，这就是问题的关键。

因为图片是异步下载的，你也不知道哪个block会先回调，如果小图片的block先回调那么这个cell的图片就先被设置为小图片，如果后来大图片的block回来了，那么你会看到图片被替换成大图片，这种情况还算比较好，但如果大图片下载失败或者小图片的block最后回调，那么你看到的将是小图片加大图片的文字信息，这时数据就错乱了。

#### 如何解决
如果不重用cell，当然是可以解决该问题的，但是内存肯定会浪费不少。既然已经找到是因为cell的重用导致两个block回调时设置的其实是同一个cell上的imageView。那么我们只需要在下载完成的回调里进行区分，如果不一致则不设置imageView。

有两种办法进行区分：

**1. 通过indexPath来区分**

block里截获的indexPath对象是以前的cell的indexPath。而通过`[tableView indexPathForCell:cell];`则可以获得当前显示的cell的indexPath。如果cell发生了重用的话，那么这两个indexPath将不一致。

```objc
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
    MyTableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"Cell" forIndexPath:indexPath];
    cell.imgView.image = [UIImage imageNamed:@"pl"];
    NSString *urlStr = self.urlArr[indexPath.row];
    __weak typeof(MyTableViewCell *) weakCell = cell;
    [[XQImageDownloader defaultImageDownloader] downloadImageWithUrlString:urlStr completion:^(NSString *imgUrl, UIImage *image, NSError *error) {
        if (!error) {
            NSIndexPath *currentIndexPath = [tableView indexPathForCell:weakCell];
            NSInteger currentRow = currentIndexPath.row;
            NSInteger originalRow = indexPath.row; //这里的indexPath是block截获的.
            if (originalRow != currentRow) {
                NSLog(@"数据错乱，应该设置的是第%ld个cell上的ImageView,但当前设置的是%ld个cell上的ImageView,cell:%p", (long)originalRow, currentRow, weakCell);
                NSLog(@"urlStr:%@, imgUrl：%@", urlStr, imgUrl); //这里的urlStr==imgUrl，想一想为什么
                return ;
            }
            weakCell.imgView.image = image;
        } else {
            NSLog(@"下载图片：%@，error:%@", urlStr, [error localizedDescription]);
        }
    }];
    cell.contentLabel.text = [NSString stringWithFormat:@"这是第%ld个cell:%p", indexPath.row, cell];
    return cell;
}
```
**2. 通过下载的图片URL来区分**

下载前先记录当前imageView应该显示的图片URL，当下载完成时再进行比较。

```objc
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
    MyTableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"Cell" forIndexPath:indexPath];
    cell.imgView.image = [UIImage imageNamed:@"pl"];
    NSString *urlStr = self.urlArr[indexPath.row];
    cell.imageView.xq_imgUrl = [NSURL URLWithString:urlStr]； //记录当前imageView应该显示的图片URL
    __weak typeof(MyTableViewCell *) weakCell = cell;
    [[XQImageDownloader defaultImageDownloader] downloadImageWithUrlString:urlStr completion:^(NSString *imgUrl, UIImage *image, NSError *error) {
        if (!error) {
            if (![weakCell.imageView.xq_imgUrl.absoluteString isEqualToString:imgUrl]) {
                NSLog(@"数据错乱，应该下载的图片url为:%@, 但当前下载的图片url为：%@", weakCell.imageView.xq_imgUrl, imgUrl);
                return ;
            }
            weakCell.imgView.image = image;
        } else {
					  NSLog(@"下载图片：%@，error:%@", urlStr, [error localizedDescription]);
        }
    }];
    cell.contentLabel.text = [NSString stringWithFormat:@"这是第%ld个cell:%p", indexPath.row, cell];
    return cell;
}
```

另外在下载图片之前先把cell的imageView的image置为nil。`cell.imgView.image = nil;`可以防止重用的cell万一图片下载失败而导致显示了以前的图片。

cell上发生数据错乱的控件，大部分都是因为要显示的资源需要异步处理比如图片需要下载后才能设置到imageView上，这时就需要在block中进行区分。少部分同步显示资源的控件（比如UILabel）发生数据错乱则一般是因为你的条件判断有问题导致重用的cell上还留有旧的数据，可以重写`-prepareForReuse`在重用前先清除掉旧数据。

俺在开发过程中遇到的一些数据错乱的场景：

cell上的imageView根据条件判断一会加载本地的图片，一会加载网络图片。

cell上的imageView根据条件判断一会用SDWebImage加载，一会用YYWebImage加载。同一个cell里的imageView千万不要使用两种框架去加载。

有空再详细分析使用SDWebImage加载图片时为什么就不会出现图片错乱。

以上，MADE BY XQ。

