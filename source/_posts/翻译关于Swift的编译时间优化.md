---
title: (翻译)关于Swift的编译时间优化
date: 2017-10-22 21:39:50
tags:
- iOS
- 翻译
categories:
- ruanpapa--技术贴
---



原文链接：[Regarding Swift build time optimizations](https://medium.com/@RobertGummesson/regarding-swift-build-time-optimizations-fc92cdd91e31)



上周，在我读完 [@nickoneill](https://medium.com/@nickoneill) 写的一篇优秀的博文[《为缓慢的Swift编译时间提速》](https://medium.com/swift-programming/speeding-up-slow-swift-build-times-922feeba5780#.k0pngnkns)后，我发现用一个不同的角度去审视 Swift 代码并不是很难的一件事。

可以被认为是简洁的一行代码现在引发了一个新的问题 -- 是否应该把这行代码重构成对应的9行代码以让编译器更容易工作（看看接下来要讲的关于空合运算符(nil coalescing operator)的示例）？到底哪个才是更重要的，简洁的代码还是对编译器友好的代码？这取决于项目的大小和开发者的想法。



#### 慢着。。。这里有一个 Xcode 插件

在展示具体的例子之前，我先想到就是手动查看日志是一件非常耗时的事情。有人提出了用终端命令可以让这件事情变得比较容易，但是我更进一步，把这个用 [Xcode 插件](https://github.com/RobertGummesson/BuildTimeAnalyzer-for-Xcode) 给实现出来了。

![Xcode插件.png](http://upload-images.jianshu.io/upload_images/698554-be809654f87b2336.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对我来说，最初的目的就是找到并修复最耗时的地方，但我现在的想法是让它经历更多的迭代过程。这样的话我就不仅可以让代码编译更有效率，还可以防止第一次进入项目的耗时。



#### 更加惊喜的是

我经常在多个 Git 分支间跳来跳去，等待一个缓慢的项目编译完成往往浪费了大量的时间。我想了好长一段时间为什么我的一个宠物项目会编译地这么缓慢（大概2万行 Swift 代码）。

在我学习了究竟是原因导致的这件事之后，我不得不承认我的确很吃惊，一行代码就需要几秒钟来编译。

让我们看看几个例子。



#### 空合操作符

编译器是很不喜欢这里的第一种方式的。在展开下面两处简写的代码之后，编译时间减少了99.4%。

```swift
// 编译时间: 5238.3ms
return CGSize(width: size.width + (rightView?.bounds.width ?? 0) + (leftView?.bounds.width ?? 0) + 22, height: bounds.height)

// 编译时间: 32.4ms
var padding: CGFloat = 22
if let rightView = rightView {
    padding += rightView.bounds.width
}

if let leftView = leftView {
    padding += leftView.bounds.width
}
return CGSizeMake(size.width + padding, bounds.height)
```



#### ArrayOfStuff + [Stuff]

这个看起来像下面这样：

```swift
return ArrayOfStuff + [Stuff]
// 而不是
ArrayOfStuff.append(stuff)
return ArrayOfStuff
```



我经常这样做，每次都会对所需的编译时间产生影响。下面是最差的一个，这里的编译时间减少了97.9％。

```swift
// 编译时间: 1250.3ms
let systemOptions = [ 7, 14, 30, -1 ]
let systemNames = (0...2).map{ String(format: localizedFormat, systemOptions[$0]) } + [NSLocalizedString("everything", comment: "")]
// 一些中间的代码
labelNames = Array(systemNames[0..<count]) + [systemNames.last!]

// 编译时间: 25.5ms
let systemOptions = [ 7, 14, 30, -1 ]
var systemNames = systemOptions.dropLast().map{ String(format: localizedFormat, $0) }
systemNames.append(NSLocalizedString("everything", comment: ""))
// 一些中间的代码
labelNames = Array(systemNames[0..<count])
labelNames.append(systemNames.last!)
```



#### 三元运算符

仅仅只是把三元运算符替换成 if-else 语句，就让编译时间减少了92.9%。如果将 map 换成 for 循环，就又能减少75%（但是那样的话我的眼睛可就受不了了）。😉

```swift
// 编译时间: 239.0ms
let labelNames = type == 0 ? (1...5).map{type0ToString($0)} : (0...2).map{type1ToString($0)}

// 编译时间: 16.9ms
var labelNames: [String]
if type == 0 {
    labelNames = (1...5).map{type0ToString($0)}
} else {
    labelNames = (0...2).map{type1ToString($0)}
}
```



#### 转换 CGFloat 到 CGFloat

没听懂我在说什么？其实下面例子中值已经是 CGFloat 了，并且有些括号是多余的。在清理完这些冗余之后，编译时间减少了99.9%。

```swift
// 编译时间: 3431.7 ms
return CGFloat(M_PI) * (CGFloat((hour + hourDelta + CGFloat(minute + minuteDelta) / 60) * 5) - 15) * unit / 180

// 编译时间: 3.0ms
return CGFloat(M_PI) * ((hour + hourDelta + (minute + minuteDelta) / 60) * 5 - 15) * unit / 180
```



#### Round()

下面是一个很奇怪的例子，下面的例子中变量是一个局部变量与实例变量的混合。这个问题似乎不是出在四舍五入本身，而是在于结合代码的方法。去掉四舍五入的方法大概能减少 **97.6%** 的构建时间。

```swift
// 编译时间: 1433.7ms
let expansion = a — b — c + round(d * 0.66) + e
// 编译时间: 34.7ms
let expansion = a — b — c + d * 0.66 + e
```



注意：以上所有测试都在MacBool Air(13英寸，2013年中)上进行。



#### 尝试一下吧

不管你是否面临过编译时间太长的问题，编写对编译器友好的代码都是非常有用的。我确信你会在其中找到一些惊喜。作为参考，这里有完整的代码，我的工程中可以5秒内完成编译…

```swift
import UIKit

class CMExpandingTextField: UITextField {

    func textFieldEditingChanged() {
        invalidateIntrinsicContentSize()
    }
    
    override func intrinsicContentSize() -> CGSize {
        if isFirstResponder(), let text = text {
            let size = text.sizeWithAttributes(typingAttributes)
            return CGSize(width: size.width + (rightView?.bounds.width ?? 0) + (leftView?.bounds.width ?? 0) + 22, height: bounds.height)
        }
        return super.intrinsicContentSize()
    }
}
```

