---
title: iOS 截图的那些事儿
date: 2018-05-28 21:50:57
tags: 
- iOS
categories:
- ruanpapa--技术贴
---

同时按下 Home 键和电源键，咔嚓一声，就得到了一张手机的截图，这操作想必 iPhone 用户再熟悉不过了。我们作为研发人员，面对的是一个个的 View，那么该怎么用代码对 View 进行截图呢？
这篇文章主要讨论的是如何在包括 UIWebView 和 WKWebView 的网页中进行长截图，对应的示例代码在这儿：https://github.com/VernonVan/PPSnapshotKit。

### UIWebView 截图

对 UIWebView 截图比较简单，`renderInContext` 这个方法相信大家都不会陌生，这个方法是 CALayer 的一个实例方法，可以用来对大部分 View 进行截图。我们知道，UIWebView 承载内容的其实是作为其子 View 的 UIScrollView，所以对 UIWebView 截图应该对其 scrollView 进行截图。具体的截图方法如下：

```objective-c
- (void)snapshotForScrollView:(UIScrollView *)scrollView
{
    // 1. 记录当前 scrollView 的偏移和位置
	CGPoint currentOffset = scrollView.contentOffset;
    CGRect currentFrame = scrollView.frame;

    scrollView.contentOffset = CGPointZero;
    // 2. 将 scrollView 展开为其实际内容的大小
    scrollView.frame = CGRectMake(0, 0, scrollView.contentSize.width, scrollView.contentSize.height);

    // 3. 第三个参数设置为 0 表示设置为屏幕的默认缩放因子
    UIGraphicsBeginImageContextWithOptions(scrollView.contentSize, YES, 0);
    [scrollView.layer renderInContext:UIGraphicsGetCurrentContext()];
    UIImage *snapshotImage = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
	
    // 4. 重新设置 scrollView 的偏移和位置，还原现场
    scrollView.contentOffset = currentOffset;
    scrollView.frame = currentFrame;
}
```

### WKWebView 截图

 虽然 WKWebView 里也有 scrollView，但是直接对这个 scrollView 截图得到的是一片空白的，具体原因不明。一番 Google 之后可以看到好些人提到 `drawViewHierarchyInRect` 方法， 可以看到这个方法是 iOS 7.0 开始引入的。官方文档中描述为：
> Renders a snapshot of the complete view hierarchy as visible onscreen into the current context.

注意其中的 **visible onscreen**，也就是将屏幕中可见部分渲染到上下文中，这也解释了为什么对 WKWebView 中的 scrollView 展开为实际内容大小，再调用 `drawViewHierarchyInRect` 方法总是得到一张不完整的截图（只有屏幕可见区域被正确截到，其他区域为空白）。

不过，这样倒是给我们提供了一个思路，可以将 WKWebView 按屏幕高度裁成 n 页，然后将 WKWebView 一页一页的往上推，每推一页就调用一次 `drawViewHierarchyInRect` 将当前屏幕的截图渲染到上下文中，最后调用 `UIGraphicsGetImageFromCurrentImageContext ` 从上下文中获取的图片即为完整截图。

核心代码如下（代码为演示用途，完整代码请从[这里](https://github.com/VernonVan/PPSnapshotKit)查看)：

```objective-c
- (void)snapshotForWKWebView:(WKWebView *)webView
{
    // 1
    UIView *snapshotView = [webView snapshotViewAfterScreenUpdates:YES];
    [webView.superview addSubview:snapshotView];

    // 2
    CGPoint currentOffset = webView.scrollView.contentOffset;
    ...

    // 3
    UIView *containerView = [[UIView alloc] initWithFrame:webView.bounds];
    [webView removeFromSuperview];
    [containerView addSubview:webView];

    
    // 4
    CGSize totalSize = webView.scrollView.contentSize;
    NSInteger page = ceil(totalSize.height / containerView.bounds.size.height);

    webView.scrollView.contentOffset = CGPointZero;
    webView.frame = CGRectMake(0, 0, containerView.bounds.size.width, webView.scrollView.contentSize.height);

    UIGraphicsBeginImageContextWithOptions(totalSize, YES, UIScreen.mainScreen.scale);
    [self drawContentPage:0 maxIndex:page completion:^{
        UIImage *snapshotImage = UIGraphicsGetImageFromCurrentImageContext();
        UIGraphicsEndImageContext();

        // 8
        [webView removeFromSuperview];
        ...
    }];
}

- (void)drawContentPage(NSInteger)index maxIndex:(NSInteger)maxIndex completion:(dispatch_block_t)completion
{
    // 5
    CGRect splitFrame = CGRectMake(0, index * CGRectGetHeight(containerView.bounds), containerView.bounds.size.width, containerView.frame.size.height);
    CGRect myFrame = webView.frame;
    myFrame.origin.y = -(index * containerView.frame.size.height);
    webView.frame = myFrame;

    // 6
    [targetView drawViewHierarchyInRect:splitFrame afterScreenUpdates:YES];

    // 7
    if (index < maxIndex) {
        [self drawContentPage:index + 1 maxIndex:maxIndex completion:completion];
    } else {
        completion();
    }
}
```

代码注意项如下（对应代码注释中的序号）：
1. 为了截图时对 frame 进行操作不会出现闪屏等现象，我们需要盖一个“假”的 webView 到现在的位置上，并将真正的 webView “摘下来”。调用 `snapshotViewAfterScreenUpdates` 即可得到这样一个“假”的 webView
2. 保存真正的 webView 的偏移、位置等信息，以便截图完成之后“还原现场”
3. 用一个新的视图承载“真正的” webView，这个视图也是绘图所用到的上下文
4. 将 webView 按照实际内容高度和屏幕高度分成 page 页
5. 得到每一页的实际位置，并将 webView 往上推到该位置
6. 调用 `drawViewHierarchyInRect` 将当前位置的 webView 渲染到上下文中
7. 如果还未到达最后一页，则递归调用 `drawViewHierarchyInRect` 方法进行渲染；如果已经渲染完了全部页，则回调通知截图完成
8. 调用 `UIGraphicsGetImageFromCurrentImageContext` 方法从当前上下文中获取到完整截图，将第 2 步中保存的信息重新赋予到 webView 上，“还原现场”

注意：我们的截图方法中有对 webView 的 frame 进行操作，如果其他地方如果有对 frame 进行操作的话，是会影响我们截图的。所以在截图时应该禁用掉其他地方对 frame 的改变，就像这样：

```objective-c
- (void)layoutWebView
{
    if (!_isCapturing) {
        self.wkWebView.frame = [self frameForWebView];
    }
}
```

### 结语

当前 WKWebView 的使用越来越广泛了，我随意查看了内存占用：打开同样一个网页，UIWebView 直接占用了 160 MB 内存，而 WKWebView 只占用了 40 MB 内存，差距是相当明显的。如果我们的业务中用到了 WKWebView 且有截图需求的话，那么还是得老老实实完成的。

最后，本文对应的代码在这儿：https://github.com/VernonVan/PPSnapshotKit