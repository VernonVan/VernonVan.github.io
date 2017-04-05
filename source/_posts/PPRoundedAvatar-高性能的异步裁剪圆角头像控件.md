---
title: PPRoundedAvatar--高性能的异步裁剪圆角头像控件
date: 2017-04-01 18:09:16
tags:
- iOS
- Objective-C
categories:
- ruanpapa--技术贴
- ruanpapa&又吉君的日常
---

## 起因

最近的开发任务是进行性能优化，主要是提升列表（UITableView）的滚动流畅性。  

在完成了提前算高、子视图层级优化几个优化步骤之后，滚动的流畅性已经有了明显的提升，平均的帧速率（fps）已经从优化之前49提高到了55。接下来通过 Instruments 的 Time Profile 工具分析，发现圆角头像的裁剪竟然占去了20%+的用户运行时间！图片的处理需要 GPU 和 CPU 的配合，本来就需要大量的处理时间，这本无可厚非。但是这么重的任务都派发到主线程肯定是会造成界面的卡顿的，而且圆角头像在项目中的使用星罗棋布，很多界面中都用到了这个控件，所以呢，对圆角头像的优化就很有必要了。  



## 经过
优化性能，无非是将主线程从繁重的任务中解放出来，将能在后台线程完成的任务都派发到后台线程中。这里选用 NSOperation 进行多线程处理，因为 CoreGraphics 都是线程安全的，于是把图片处理（裁圆角/加边框）的过程都在后台线程中执行好，再在主线程设置图片。接下来说一下核心的技术点：

### 图片处理
我将图片裁剪的过程封装并暴露了两个便利接口，默认的 - (nullable UIImage *)pp_imageByRoundCornerRadius: scaleSize: 会先将图片缩放到scaleSize的大小，再添加上圆角（注意默认的方法是没有边框的），另一个接口 - (nullable UIImage *)pp_imageByRoundCornerRadius: scaleSize: borderWidth: borderColor: 则可以在默认方法的基础上设置边框颜色和边框的宽度。这里没有新建一个专门的处理类，而是拓展了 UIImage 类，具体的头文件如下：
```objective-c
@interface UIImage (PPRoundedAvatar)
/**
 将图片进行圆角处理，默认无边框
 */
- (nullable UIImage *)pp_imageByRoundCornerRadius:(CGFloat)radius
                                        scaleSize:(CGSize)newSize;

/**
 将图片进行圆角处理，并加上边框
 */
- (nullable UIImage *)pp_imageByRoundCornerRadius:(CGFloat)radius
                                        scaleSize:(CGSize)newSize
                                      borderWidth:(CGFloat)borderWidth
                                      borderColor:(nullable UIColor *)borderColor;

/**
 图片加上圆形边框，图片必须得是正方形的，否则直接返回未加边框的原图片
 */
- (nullable UIImage *)pp_imageByRoundBorderedColor:(nullable UIColor *)color
                                       borderWidth:(CGFloat)width;

@end
```
图片进行圆角裁剪和加上边框主要还是采用 CoreGraphics 和 UIBezierPath 的方法，这里按下不表。值得注意的是处理的顺序应该是**先缩放图片，再进行裁剪**，不然边框的宽度会因为缩放而改变。
```objective-c
// 图片圆角裁剪
- (UIImage *)pp_imageByRoundCornerRadius:(CGFloat)radius
                             borderWidth:(CGFloat)borderWidth
                             borderColor:(UIColor *)borderColor
{
    UIImage *scaledImage = [self pp_imageScaledAspectToFillSize:newSize];  // 缩放图片
    UIGraphicsBeginImageContextWithOptions(scaledImage.size, NO, 0);
    CGContextRef context = UIGraphicsGetCurrentContext();
    CGRect rect = CGRectMake(0, 0, scaledImage.size.width, scaledImage.size.height);
    CGContextScaleCTM(context, 1, -1);
    CGContextTranslateCTM(context, 0, -rect.size.height);
    
    CGFloat minSize = MIN(scaledImage.size.width, scaledImage.size.height);
    if (borderWidth < minSize / 2) {
        UIBezierPath *path = [UIBezierPath bezierPathWithRoundedRect:CGRectInset(rect, borderWidth, borderWidth) byRoundingCorners:corners cornerRadii:CGSizeMake(radius, borderWidth)];
        CGContextSaveGState(context);
        [path addClip];
        CGContextDrawImage(context, rect, scaledImage.CGImage);
        CGContextRestoreGState(context);
    }
    
    UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
    image = [image pp_imageByRoundBorderedColor:borderColor borderWidth:borderWidth];   	// 加上边框
    UIGraphicsEndImageContext();
    return image;
}

// 图片加边框
- (UIImage *)pp_imageByRoundBorderedColor:(UIColor *)borderColor
                              borderWidth:(CGFloat)borderWidth
{
    if (self.size.height != self.size.width) {
        return self;
    }
    
    if (!borderColor || borderWidth > self.size.width / 2 || borderWidth < 0) {
        return self;
    }
    
    UIGraphicsBeginImageContextWithOptions(self.size, NO, 0);
    [self drawAtPoint:CGPointZero];
    CGRect rect = CGRectMake(0, 0, self.size.width, self.size.height);
    CGFloat strokeInset = borderWidth / 2;
    CGRect strokeRect = CGRectInset(rect, strokeInset, strokeInset);
    CGFloat radius = self.size.width / 2;
    UIBezierPath *path = [UIBezierPath bezierPathWithRoundedRect:strokeRect byRoundingCorners:UIRectCornerAllCorners cornerRadii:CGSizeMake(radius, borderWidth)];
    path.lineWidth = borderWidth;
    [borderColor setStroke];
    [path stroke];
    UIImage *result = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    return result;
}
```

### 圆角头像控件
这里采用 NSOperation + NSOperationQueue 的方式进行多线程处理，在图片、边框等属性的 set 方法里调用 setNeedsLayout 方法刷新布局，同时设置 _isNeedTransform 标记位为 YES，表示需要刷新，可以提高性能。
```objective-c
- (void)setAvatarImage:(UIImage *)avatarImage
{
    if (_avatarImage != avatarImage) {
        _avatarImage = avatarImage;
        
        _isNeedTransform = YES;      // 需要刷新的标识
        [self setNeedsLayout];
    }
}

- (void)setBorderHidden:(BOOL)borderHidden
{
    if (_borderHidden != borderHidden) {
        _borderHidden = borderHidden;
        
        _isNeedTransform = YES;		// 需要刷新的标识
        [self setNeedsLayout];
    }
}

+ (NSOperationQueue *)sharedTransformQueue
{
    static NSOperationQueue *transformQueue;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        transformQueue = [[NSOperationQueue alloc] init];
        transformQueue.name = @"io.github.vernonvan.PPRoundedAvatar.sharedOperationQueue";
        transformQueue.maxConcurrentOperationCount = 20;
    });
    return transformQueue;
}

- (void)layoutSubviews
{
    [super layoutSubviews];
    
    if (!self.avatarImage && !self.imageBackgroundColor) {
        return;
    }
    
    if (self.bounds.size.width <= 0 || self.bounds.size.height <= 0) {
        return;
    }

    if (_isNeedTransform || !CGSizeEqualToSize(self.bounds.size, self.imageView.image.size)) {
        [self transformImage];
    }
}

- (void)transformImage
{
    UIImage *startImage = self.avatarImage;
    
    NSBlockOperation *transformOperation = [[NSBlockOperation alloc] init];
    __weak NSBlockOperation *weakTransformOperation = transformOperation;
    [transformOperation addExecutionBlock:^{
        NSBlockOperation *strongTransformOperation = weakTransformOperation;
        
        if ([strongTransformOperation isCancelled]) {
            return;
        }
        
        UIImage *transformedImage = nil;
        if (self.avatarImage) {
            transformedImage = [self.avatarImage pp_imageByRoundCornerRadius:self.bounds.size.width scaleSize:self.bounds.size];
        } else if (self.imageBackgroundColor) {
            transformedImage = [UIImage pp_roundImageWithColor:self.imageBackgroundColor radius:self.bounds.size.width / 2];
        }
        
        if (!self.borderHidden) {
            transformedImage = [transformedImage pp_imageByRoundBorderedColor:self.borderColor borderWidth:self.borderWidth];
        }
        
        dispatch_async(dispatch_get_main_queue(), ^{
            if ([strongTransformOperation isCancelled]) {
                return;
            }
            if (self.avatarImage == startImage) {  // 1
                _isNeedTransform = NO;
                [self setImage:transformedImage forState:UIControlStateNormal];
            }
        });
    }];
    
    [[self.class sharedTransformQueue] addOperation:transformOperation];
}
```
在上面有一个标注了1的注释点，这里的条件判断是为了避免多线程时序性的问题而加的。考虑这样的一种常见情况：圆角头像控件是表格上的单元格（cell）上的子视图，某个 cell 被滑到屏幕上，于是开始头像的裁剪（这里将这个头像称为旧头像），然后在这个裁剪尚未完成的时候，这个 cell 被滑出了屏幕，然后根据新的图片裁剪圆角（这个头像称为新头像），可能出现新头像裁剪早于旧头像完成的情况，就会导致控件先设置头像为新头像，然后被慢悠悠才完成裁剪的旧头像覆盖的问题。所以这里用这个条件避免这个问题。



## 结果
将上述的操作都封装隐藏好，现在的圆角头像控件 **PPRoundedAvatar** 的使用就很简单了，
```objective-c
PPRoundedAvatar *avatar = [[PPRoundedAvatar alloc] initWithFrame:CGRectMake(0, 0, 100, 100)];
avatar.avatarImage = [UIImage imageNamed:@"example.png"];       // 头像图片
avatar.borderWidth = 1.0;                                       // 边框宽度
avatar.borderColor = UIColor.blackColor;                        // 边框颜色
avatar.borderHidden = NO;                                       // 显示边框
avatar.imageBackgroundColor = UIColor.grayColor;                // 背景颜色
[self.view addSubview:avatar];
```
具体的代码已经丢到了 [github](https://github.com/VernonVan/PPRoundedAvatar) 上了，同时支持 Cocoapods 导入项目。
有需要的可以 clone 进行使用，也欢迎 pull request 完善这个控件。



## 插叙
技术上的问题就说到了这里，可是做一个开源项目不仅在技术上需要反复斟酌，而且在 github 的展示、demo 的展示等地方都需要有界面的设计，可苦了我这直男审美....../(ㄒoㄒ)/~~
这也导致了我家又吉君实在是看不下去了，终于勇敢地站了出来帮我把界面的设计给做好了，当中的你来我往又是另外的[故事](https://vernonvan.github.io/2017/04/01/ruanpapa%E5%92%8C%E5%8F%88%E5%90%89%E5%90%9B%E7%9A%84%E6%97%A5%E5%B8%B8%E4%B9%8B%E4%B8%80/)。真的是对我家又吉君感激涕零~~~
呐，又吉君，你帮我画一辈子设计好不好吖，我带你去吃鸡锁骨、卷饼、酱香饼、枣糕还有烤冷面哦~哈哈哈哈啊哈哈哈