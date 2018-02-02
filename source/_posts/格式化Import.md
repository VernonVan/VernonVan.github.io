---
title: Source Editor Extension — Xcode 格式化 Import 的插件
date: 2018-02-01 17:06:35
tags: 
- iOS
categories:
- ruanpapa--技术贴
---

## 背景
Xcode 秉承了 Apple 封闭的传统，提供的可自定义的选项比起其他 IDE 来说是比较少的，不过在 Xcode 7 之前（包含 Xcode 7）我们还是可以通过插件实现 Xcode 的自定义，甚至还出现了像  [Alcatraz](https://github.com/alcatraz/Alcatraz) 的专门的插件管理工具，开源社区中也有诸如 [VVDocumenter-Xcode](https://link.jianshu.com/?t=https://github.com/onevcat/VVDocumenter-Xcode)、[CocoaPods](https://github.com/CocoaPods/CocoaPods) 等知名的插件，不过这些便利随着 Xcode 8 的发布成为了过去式。
出于安全性考虑（比如说 Xcode ghost 事件），Apple 从 Xcode 8 开始不再支持第三方的插件。Apple 方面提供了基于 App Extension 的解决方案 -- Xcode Source Editor Extension，这是一个相当简单的方案，能且仅能完成有限的文本编辑辅助，很大部分之前第三方插件能完成的任务都没办法实现了。聊胜于无吧 😑
（本文会介绍 Source Editor Extension 的开发以及分发相关的知识，本文对应的 Demo 在：https://github.com/VernonVan/PPImportArrangerExtension）



## 创建插件

1. 创建一个 Cocoa App：Source Editor Extension 不能独立存在，必须依附于 Cocoa App。
  ![Cocoa App](http://upload-images.jianshu.io/upload_images/698554-a00fabcbca0f7353.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  ​

2. File -> New -> Target -> Xcode Source Editor Extension 添加一个 Target，并激活这个 Target。
  ![Xcode Source Editor Extension](http://upload-images.jianshu.io/upload_images/698554-0c32ca262d291557.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  ![激活 target](http://upload-images.jianshu.io/upload_images/698554-b6a7d64355a372d4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这样就创建好了一个可运行的 Source Editor Extension，相当的简单。🧐



## 关键概念

![文件结构](http://upload-images.jianshu.io/upload_images/698554-0e6f169643d8a20f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- SourceEditorExtension 类：遵循 XCSourceEditorExtension 协议的类，XCSourceEditorExtension 协议的头文件如下：

```Objective-C
@protocol XCSourceEditorExtension <NSObject>

@optional

- (void)extensionDidFinishLaunching;

@property (readonly, copy) NSArray <NSDictionary <XCSourceEditorCommandDefinitionKey, id> *> *commandDefinitions;

@end
```
XCSourceEditorExtension 协议只有一个方法和一个属性，extensionDidFinishLaunching 方法是用来在插件加载好后是对插件进行一些准备工作的，根据 [WWDC](https://developer.apple.com/videos/play/wwdc2016/414/) 的说法，各个插件与 Xcode 本身的初始化过程是在不同进程上进行的，同样地，插件的崩溃并不会引起 Xcode 的崩溃。commandDefinitions 属性则可以动态返回插件的菜单项。

SourceEditorCommand 类：遵循 XCSourceEditorCommand 协议的类，实现插件功能的核心类，对应到插件的菜单项，可以一个菜单项对应到一个 Command 类，也可以多个菜单项对应到一个 Command 类，XCSourceEditorCommand 协议头文件定义如下：
```Objective-C
@protocol XCSourceEditorCommand <NSObject>

@required

- (void)performCommandWithInvocation:(XCSourceEditorCommandInvocation *)invocation completionHandler:(void (^)(NSError * _Nullable nilOrError))completionHandler;

@end
```
XCSourceEditorCommandInvocation 类型的参数 invocation 主要是点击的菜单项的标识、当前文本信息（文本字符串数组、选中区间等）以及点击取消按钮的回调事件，completionHandler 参数则是用来通知 Xcode 本插件已经完成了自己的操作，需要保证一定要调用 completionHandler！否则会出现下图所示的提示，然后菜单项就会变灰不能再点击：
![插件 busy](http://upload-images.jianshu.io/upload_images/698554-f57b5a31a603f5f3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![菜单项变灰](http://upload-images.jianshu.io/upload_images/698554-8e6d0b4a636883f2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- Info.plist：Info.plist 文件用于静态配置插件对应的菜单项，如下图所示，XCSourceEditorExtensionPrincipalClass 对应到上文说的 XCSourceEditorExtension 类，XCSourceEditorCommandDefinitions 指定菜单项，XCSourceEditorCommandClassName 对应到上文说的 SourceEditorCommand 类，XCSourceEditorCommandIdentifier 是每个具体菜单项的标识，XCSourceEditorCommandName 是菜单项的描述。

![Info.plist](http://upload-images.jianshu.io/upload_images/698554-54682412d2c5b04a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 保证 TARGETS 组下的两个 Target 用的同一个签名。


## 实现步骤

本 Demo 要实现的功能就是按照字母顺序重新排列当前文件的所有 Import，强迫症们一定知道我在说什么🤣，先来看一下效果：
![效果图](http://upload-images.jianshu.io/upload_images/698554-58e34917de432fbd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![演示效果](http://upload-images.jianshu.io/upload_images/698554-9638491f62073029.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以点击 Editor -> ImportArranger -> Arrange Imports 重新排列所有的 Imports，甚至还可以为其设置快键键。

实现步骤反而没有什么可说的，主要是操作 invocation.buffer.lines 和 invocation.buffer.selections，分别对应的是当前文件的所有行和当前文件的选择区域，都是可变类型的数组，做完自定义的操作后操作数组即可更新当前文件。注意：**不管是哪条执行路径，一定要保证调用到 completionHandler**。其他需要留意的地方都在代码中的注释中给出：
```Objective-C
- (void)performCommandWithInvocation:(XCSourceEditorCommandInvocation *)invocation completionHandler:(void (^)(NSError *_Nullable nilOrError))completionHandler
{
    NSMutableArray<NSString *> *lines = invocation.buffer.lines;
    if (!lines || !lines.count) {
        completionHandler(nil);
        return;
    }

    NSMutableArray<NSString *> *importLines = [[NSMutableArray alloc] init];
    NSInteger firstLine = -1;
    for (NSUInteger index = 0, max = lines.count; index < max; index++) {
        NSString *line = lines[index];
        NSString *pureLine = [line stringByReplacingOccurrencesOfString:@" " withString:@""];       // 去掉多余的空格，以防被空格干扰没检测到 #import
        // 支持 Objective-C、Swift、C 语言的导入方式
        if ([pureLine hasPrefix:@"#import"] || [pureLine hasPrefix:@"import"] || [pureLine hasPrefix:@"@class"]
            || [pureLine hasPrefix:@"@import"] || [pureLine hasPrefix:@"#include"]) {     
            [importLines addObject:line];
            if (firstLine == -1) {
                firstLine = index;      // 记住第一行 #import 所在的行数，用来等下重新插入的位置
            }
        }
    }

    if (!importLines.count) {
        completionHandler(nil);
        return;
    }

    [invocation.buffer.lines removeObjectsInArray:importLines];

    NSArray *noRepeatArray = [[NSSet setWithArray:importLines] allObjects];         // 去掉重复的 #import
    NSMutableArray<NSString *> *sortedImports = [[NSMutableArray alloc] initWithArray:[noRepeatArray sortedArrayUsingSelector:@selector(caseInsensitiveCompare:)]];

    // 引用系统文件在前，用户自定义的文件在后
    NSMutableArray *systemImports = [[NSMutableArray alloc] init];
    for (NSString *line in sortedImports) {
        if ([line containsString:@"<"]) {
            [systemImports addObject:line];
        }
    }
    if (systemImports.count) {
        [sortedImports removeObjectsInArray:systemImports];
        [sortedImports insertObjects:systemImports atIndexes:[NSIndexSet indexSetWithIndexesInRange:NSMakeRange(0, systemImports.count)]];
    }

    if (firstLine >= 0 && firstLine < invocation.buffer.lines.count) {
        // 重新插入排好序的 #import 行
        [invocation.buffer.lines insertObjects:sortedImports atIndexes:[NSIndexSet indexSetWithIndexesInRange:NSMakeRange(firstLine, sortedImports.count)]];
        // 选中所有 #import 行
        [invocation.buffer.selections addObject:[[XCSourceTextRange alloc] initWithStart:XCSourceTextPositionMake(firstLine, 0) end:XCSourceTextPositionMake(firstLine + sortedImports.count, sortedImports.lastObject.length)]];
    }

    completionHandler(nil);
}
```

选择这个插件作为当前 Scheme，选择 Xcode 运行，然后就会弹出一个黑色的 Xcode 供你调试了。
![image.png](http://upload-images.jianshu.io/upload_images/698554-8fb83256d9e38800.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![调试插件](http://upload-images.jianshu.io/upload_images/698554-7410ea37b33422fd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



## 分发
插件开发测试完成之后，最重要的当然是将插件分发出去，供他人使用。Apple 在  [WWDC](https://developer.apple.com/videos/play/wwdc2016/414/) 说到 Xcode Source Editor Extension 是可以上架 Mac App Store 的，不过受限于 Source Editor Extension 功能实在太少，目前也没有在 Mac App Store 上看到很火的插件。更多是直接把 .app 文件上传到 Github 上供人下载（这里有人整理了一些不错的插件：https://github.com/theswiftdev/awesome-xcode-extensions），具体步骤如下：



#### 打包

测试完成后，找到 Products 下面的 .app 文件，注意需要保证上文中说的两个签名是一致的。然后就可以把这个 .app 上传到个人网站或者 Github 上供人下载使用了。
![.app 文件](http://upload-images.jianshu.io/upload_images/698554-5202f8ddea721d38.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



#### 安装

当我们下载好了一个 .app 格式的插件之后，直接双击这个 .app 文件，然后在 系统偏好设置-> 扩展 -> Xcode Source Editor Extension 勾选该插件，最后重启 Xcode 就可以在 Editor 菜单中找到该插件了。
![勾选插件](http://upload-images.jianshu.io/upload_images/698554-b35bafce22cccf86.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

还可以在 Xcode 中为插件的菜单项设置快捷键。
![设置快键键](http://upload-images.jianshu.io/upload_images/698554-d5f2a3622205f490.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



## 结语

至少现有的 Xcode Source Editor Extension 还是比较受限的，接口少的可怜，可想象的空间不是很多，大部分之前第三方插件能做的事情都没办法完成了🤷‍♀️。还是默默希望 Apple 能以更加开放的姿态，提供更多的接口给开发者，Xcode 没办法满足所有人的喜好，起码，能让喜欢折腾的人把它变得更好 :-D