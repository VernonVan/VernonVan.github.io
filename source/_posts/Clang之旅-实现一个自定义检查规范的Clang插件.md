---
title: Clang 之旅--实现一个自定义检查规范的 Clang 插件
date: 2018-04-17 15:55:23
tags: 
- iOS
- clang & llvm
categories:
- ruanpapa--技术贴
---



Clang 之旅系列文章：
[Clang 之旅--使用 Xcode 开发 Clang 插件](https://www.jianshu.com/p/e3f46d42643b)
[Clang 之旅--[翻译]添加自定义的 attribute](https://www.jianshu.com/p/d277c42f4907)
[Clang 之旅--实现一个自定义检查规范的 Clang 插件](https://www.jianshu.com/p/c27b77f70616)



### 前言

在 Clang 之旅系列文章[开篇](https://www.jianshu.com/p/e3f46d42643b)的时候，我说到过自己接触 Clang 的直接原因就是想实现一个自定义的检查需求：是否有办法在编译阶段检查某个方法的参数与返回值的类型相同，如果类型不一致的话能抛出编译错误的提示。现在我已经根据自己的需求完成了这个插件，这篇文章会讲解这个插件的实现思路，对应的代码在这里：https://github.com/VernonVan/SameTypeClangPlugin



### 具化需求 

首先我先将需求具化一下，之前说的比较宽泛。

试想我们有这么一个函数 `modelOfClass`：

```objective-c
- (__kindof NSObject *)modelOfClass:(Class)modelClass 
{
    if ([modelClass isKindOfClass:[NSString class]]) {
        return [[NSString alloc] init];
    } else if ([modelClass isKindOfClass:[NSArray class]]) {
        return [[NSArray alloc] init];
    }
    return nil;
}
```

`modelOfClass` 接受一个 `Class` 类型的参数，然后会根据 `Class` 对应的类进行不同的操作，最终返回处理好的 `Class` 对应类的实例对象。我们用 `__kindof NSObject *` 返回值类型来保证返回的一定是 `NSObject` 或者其子类，能保证的也只有这样而已。但是，存在这样一种错误的调用方式，但是却能通过编译：

```objective-c
@property (nonatomic, strong) NSString *myString;
@property (nonatomic, strong) NSArray *myArray;

- (void)someMethod
{
    self.myString = [self modelOfClass:[NSString class]];
    self.myArray = [self modelOfClass:[NSString class]];
}
```

可以发现，`someMethod` 中有两行 `modelOfClass` 的函数调用。第一行调用是正确的，`NSString *` 类型的属性 `myString` 调用时传入的是 `[NSString class]`；第二行调用是错误的，`NSArray *` 类型的属性 `myArray` 调用时传入的是 `[NSString class]`。也就是说，在 Objective-C 语言中，并没有一种办法能够检查函数调用时参数类型和返回值类型是完全一致的。



这个需求是从我所在公司的项目中抽象简化出来的，大家看不出来这个函数究竟是用来干什么的，可能会觉得这个需求并不常见，没有什么通用性。但是这篇文章希望读者看了之后能以小见大，举一反三，更重要的是学到怎么样使用通用的方式，根据自己的需求实现自定义检查规范的 Clang 插件。



### 最终效果

我们来看看最终实现的效果：

![演示效果](https://upload-images.jianshu.io/upload_images/698554-c7aa746724799734.GIF?imageMogr2/auto-orient/strip)

最终实现了上面所说的类型检查，同时还给出了对应的修改方法（FixIt），点击修改就能改成正确的参数类型🎉🎉🎉 下面就来说说具体是怎么实现的。



### 抽象语法树（Abstract syntax tree）

抽象语法树，英文简称为 AST，是编译过程中语法分析阶段的产物，也是我们作为外部开发者与 Clang 进行交互的最重要的方式。所以我们最重要的就是学会怎么样阅读、分析语法树。



在命令行中输入以下命令，打印 main.m 文件对应的语法树到命令行中：

```
clang -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator11.3.sdk -fmodules -fsyntax-only -Xclang -ast-dump main.m
```

> 在我写这篇文章时 Xcode 版本是9.3，对应的是 iPhoneSimulator11.3.sdk，你需要进入该目录查看你的 sdk 版本，然后修改 -isysroot 命令后的 sdk 路径



打印出来的语法树如下图：

![AST](https://upload-images.jianshu.io/upload_images/698554-f509375dfc159afe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



编译前端 Clang 首先进行词法分析（Lexical Analysis），把源文件的字符流拆分一个一个的 token；然后 token 进入语法分析（Semantic Analysis），将这些 token 组合成语法树。左边的缩进代表了语法树节点的从属关系，语法树上的每一个节点的名字都能在 Clang 源码中找到对应的类。

从图中挑几个点来解释一下（对应图中的红色标注）：

1. `ObjCImplementationDecl` 节点代表了 Objective-C 类中的 `@implementation` 部分的内容

2. `ObjCMethodDecl` 节点代表了 Objective-C 中的函数定义，我们在 Clang 源码中查看一下对应类的定义

   ![ObjCMethodDecl](https://upload-images.jianshu.io/upload_images/698554-3ece97359f29ec5e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

   Clang 的文档注释可以说相当齐全了，`ObjCMethodDecl`  代表了一个类方法或者实例方法。所有的 `public:` 域中的方法都是我们可以用的，比如说  `Selector getSelector()` 可以获取该方法的 `Selector`，`ArrayRef<ParmVarDecl*> parameters()` 可以获取获取该方法的参数列表等等。

3. 框中的语法块代表了源文件中 `self.myString = [self modelOfClass:[NSString class]];` 语句，`BinaryOperator` 代表了二元操作符（包括赋值的“=”），可以通过 `BinaryOperator` 类的 `Expr *getLHS()` 和 `Expr *getRHS()` 分别取得“=”左右两边的语句。

> 详细的 AST 树的分析可以查看官方的教程：http://clang.llvm.org/docs/IntroductionToTheClangAST.html



那么多种的 AST 节点中应该怎么只获取自己感兴趣的节点呢？

Clang 提供了 `ASTMatcher` 类供我们进行 AST 节点的查找过滤，有一篇专门解释罗列各种各样的 `ASTMatcher` 的[官方文档](http://clang.llvm.org/docs/LibASTMatchersReference.html)可以查看。

![ASTMatcher](https://upload-images.jianshu.io/upload_images/698554-71f15a143a85a6e1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

比如可以用 `objcPropertyDecl` 来匹配到 Objective-C 的类属性，`ASTMatcher` 可以用一种类似链式语法的方式将一系列的 Matcher 串起来，比如可以用 `cxxRecordDecl(unless(hasName("X")))` 来匹配到满足类名不为 `X` 的所有 C++ 类。

> 具体的 ASTMatcher 的使用方法可以查看这篇教程：https://eli.thegreenplace.net/2014/07/29/ast-matchers-and-clang-refactoring-tools



### 实现思路

基础知识铺垫完了，现在我们来拆解一下我们的需求。首先我们需要有一种方式标记需要进行这种检查的函数，总不至于所有函数调用我们都去检查一遍吧😹 这时候就可以想到可以通过 attribute 的方式标记函数！

> 关于 attribute 的知识，可以查看孙源大神的这篇文章：[Clang Attributes 黑魔法小记](https://blog.sunnyxx.com/2016/05/14/clang-attributes/)，讲解了多种常见不常见的 attribute 的使用场景
>
> 另外一篇就是官方关于如何在 Clang 中添加自定义的 attribute 的文档：[How to add an attribute](https://clang.llvm.org/docs/InternalsManual.html#how-to-add-an-attribute)，我自己也翻译了这篇文档，请戳[中文版](https://www.jianshu.com/p/d277c42f4907)。

这里不讲解怎么添加自定义的 attribute，比较简单，就是按最简单的模板添加的。添加完了之后，得在 `modelOfClass` 后面加上一句 `__attribute__((objc_same_type))`，代表 `modelOfClass` 在每次被调用时都会进行自定义的检查，这样才能出现上面演示效果图中的检查结果（objc_same_type 就是我所添加的 attribute 的名字）。

```objective-c
- (__kindof NSObject *)modelOfClass:(Class)modelClass __attribute__((objc_same_type))
```

#### 

具体该怎么检查呢？分成以下几个步骤：

1. 首先判断语法树上的节点是否是赋值语句（Clang 中用 `BinaryOperator` 表征赋值语句）。如果是，进入第 2 步
2. 用 `BinaryOperator` 的 `getLHS()` 、`getRHS()` 函数分别获得左右的表达式
3. 如果左边表达式是 Objective-C 类的属性的话，获取该属性对应的类型 A。进入第 4 步
4. 如果右边表达式是 Objective-C 的函数调用，且被调用的函数是有我们上面所定义 __attribute__((objc_same_type)) 的话（可以通过 `ObjCMethodDecl` 的 `attrs()` 方法获得 Objective-C 函数的所有的 attribute），获取该函数的参数对应的类型 B
5. 对比 A 和 B 的类型是否一致，如果不一致，则弹出类型不一致的编译警告，并提出恰当的修改方法（如效果演示图所示）


> 具体的实现代码和使用方法查看 Github：https://github.com/VernonVan/SameTypeClangPlugin



### 结语

最终花了不到 200 行代码就完成了这个小小的功能，但是却花了我将近一个月的业余时间，中间也做了很多无用功，在错误的道路上走了一段时间才发现自己做的完全是错的，幸好最后还是成功找到了正确的方法。不过，自己也收获了很多的技能点，比如说阅读源码的能力，得益于 LLVM 良好的代码设计和模块化，让我一个门外汉也能比较快速的从庞大的代码中找到自己想要的部分；比如说 CMake 构建工程的知识、C++ 语言以及查找阅读英文文档的能力。收获还是比较多的🍹🍹🍹

接下来如果在 LLVM && Clang 这一块有其他的所得的话，会再撰文分享~