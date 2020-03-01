---
title: SDWebImage内存暴涨分析与解决
date: 2018-02-14 15:01:11
description: 
categories: 第三方库使用
tags: SDWebImage
comments: true
---

使用SDWebImage加载图片,当列表有很多页,图片很多时,可能会导致内存暴涨:
![内存暴涨](http://7xqoji.com1.z0.glb.clouddn.com/SDWebImage_%E5%86%85%E5%AD%98%E6%9A%B4%E6%B6%A8.jpg)

产生内存暴涨的代码:

```
//从磁盘获取图片,并缓存到内存.
- (nullable UIImage *)imageFromDiskCacheForKey:(nullable NSString *)key {
    UIImage *diskImage = [self diskImageForKey:key];
    if (diskImage && self.config.shouldCacheImagesInMemory) {
        NSUInteger cost = SDCacheCostForImage(diskImage);
        [self.memCache setObject:diskImage forKey:key cost:cost];
    }

    return diskImage;
}

//解压图片
- (nullable UIImage *)diskImageForKey:(nullable NSString *)key {
    NSData *data = [self diskImageDataBySearchingAllPathsForKey:key];
    if (data) {
        UIImage *image = [[SDWebImageCodersManager sharedInstance] decodedImageWithData:data];
        image = [self scaledImageForKey:key image:image];
        if (self.config.shouldDecompressImages) {
            image = [[SDWebImageCodersManager sharedInstance] decompressedImageWithImage:image data:&data options:@{SDWebImageCoderScaleDownLargeImagesKey: @(NO)}];
        }
        return image;
    } else {
        return nil;
    }
}

```
关闭缓存图片到内存的功能:
`[[SDImageCache sharedImageCache].config setShouldCacheImagesInMemory:NO];`

效果如下:
![内存正常](http://7xqoji.com1.z0.glb.clouddn.com/SDWebImage_%E5%86%85%E5%AD%98%E6%AD%A3%E5%B8%B8.jpg)
此时虽然内存不再异常,但滑动过程中会明显看到图片填充时的闪动,用户体验想当不好.

比较好的办法:

```
- (void)initSDWebImage
{
    [[SDImageCache sharedImageCache].config setShouldDecompressImages:NO];
    [[SDWebImageDownloader sharedDownloader] setShouldDecompressImages:NO];
    [[SDImageCache sharedImageCache] setMaxMemoryCost:20 * 1024 * 1024];
}
```
设置一个内存缓存最大值,需要注意的是系统并不会精确的使用该值进行限定.这样设置之后,内存基本在70-90M之间浮动.不会再像之前飙到3-400M还停不下来.
如果设置为10 * 1024 * 1024,那么内存基本在30-50M之间.并且用户体验还行.