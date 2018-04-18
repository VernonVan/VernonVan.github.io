---
title: Clang 之旅--使用 Xcode 开发 Clang 插件
date: 2018-03-16 14:44:05
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

最近在跟老大的聊天中聊到了一个比较特殊的需求：是否有办法在编译阶段检查某个方法的参数与返回值的类型相同，如果类型不一致的话能抛出编译错误的提示。这似乎已经不是 Objective-C 或者 Swift 的语言语法本身所能解决的了，老大还指点了可以从编译器等底层中进行研究。于是，我踏进了 Clang 和 LLVM 的大门。

我打算将 Clang 的研究心得分为几篇文章来写，这是 Clang 之旅的第一篇，主要讲如何用 Xcode 编译 Clang，以及实现一个简单的 Clang 插件并挂载到 Xcode 中参与编译流程，算是进入 Clang 的门槛。只是，这门槛就狠狠地让我吃了苦头，Google 找到好几篇博客讲怎么编译 Clang 的，但是也有一些年头了，版本比较旧，编译出来的 Clang 不能运行在现在的系统上；还有一些写的比较含糊，漏了某些关键步骤，导致花了好几个小时跟着教程做下来最后还是一堆 error；而且试错的成本还是比较高的，下载的源码有1G多（考虑从 Github 下载的速度🙄，需要挂个代理），完整编译出来有20G左右，我的15款 Macbook Pro 大概需要疯狂编译2个小时…...如果不能接受这些的话，还是别尝试了，很遗憾，你连见到 Clang 真容的机会都没有┑(￣Д ￣)┍

![llvm大小](https://upload-images.jianshu.io/upload_images/698554-49318ad53d98d6ef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### 编译源码

#### 准备工作

Clang 需要用 CMake 来编译，CMake 的安装方法可以参考这篇文章：[Mac 安装 CMake & CMake Command Line Tools](https://www.jianshu.com/p/7466c85d5d6b)，建议对 CMake 完全不了解的同学可以先补充一点 CMake 的基本知识，这样能更容易理解接下来要做的事情，CMake 的入门知识可以参考：[CMake 入门实战](http://www.hahack.com/codes/cmake/)



#### 下载源码

首先创建 LLVM 的源码路径及编译路径：

```
cd /opt
sudo mkdir llvm
sudo chown `whoami` llvm	// 将 llvm 目录的所有者指定为当前用户
cd llvm
export LLVM_HOME=`pwd`		// 设置当前目录(/opt/llvm)为 LLVM_HOME 目录
```

接下来从 Github clone 源代码（注意这几条语句中的 release_60，在当前时间2018.3.18时，我试过了 release_33、release_39，编译出来的 Clang 插件在运行的时候都会报 NSUUID 的 Nullability 错误，应该是这些版本不支持 Objective-C 后来加的 Nullability 特性，所以我下载了当前最新的 release_60 分支。一般来说，最新分支是兼容已有特性的，所以优先下载最新分支，分支查看可以参照下图）：

```
git clone -b release_60 git@github.com:llvm-mirror/llvm.git llvm
git clone -b release_60 git@github.com:llvm-mirror/clang.git llvm/tools/clang
git clone -b release_60 git@github.com:llvm-mirror/clang-tools-extra.git llvm/tools/clang/tools/extra
git clone -b release_60 git@github.com:llvm-mirror/compiler-rt.git llvm/projects/compiler-rt
```

![llvm最新分支.png](https://upload-images.jianshu.io/upload_images/698554-5e54bd18bac5a151.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



#### 编译源码

生成 Xcode 工程（也可以直接用命令行编译，不过大家平时可能看习惯了 Xcode 工程，所以用 Xcode 编译比较习惯）

```
mkdir llvm_build; cd llvm_build
cmake -G Xcode ../llvm -DCMAKE_BUILD_TYPE:STRING=MinSizeRel
```

生成的文件如下：

![Xcode工程.png](https://upload-images.jianshu.io/upload_images/698554-975fe9e218257288.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

打开 Xcode 工程，选择自动创建 Schemes：

![自动创建Schemes.png](https://upload-images.jianshu.io/upload_images/698554-dd7980d5fe52b689.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后编译 Clang 和 libClang（可以随时终止编译，再次点击编译会从上次停止的地方继续进行）：

![编译Clang和libClang](https://upload-images.jianshu.io/upload_images/698554-cad3858632e8185b.JPG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里可能需要1个多小时才能完成编译，如无意外，编译成功！



### 编写你的第一个插件

这个插件实现的功能就是打印语法树上所有节点的类名以及父类名，创建 Clang 插件的整体步骤如下图：

![创建插件.png](https://upload-images.jianshu.io/upload_images/698554-214742ce207ea7ce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


首先修改源代码目录 /opt/llvm/llvm/tools/clang/tools 下的 CMakeLists.txt 文件，添加一个新的编译目标，直接在 CMakeLists.txt 的最后面添加上一行，如下图：
![添加新的编译目标.png](https://upload-images.jianshu.io/upload_images/698554-db674e4e4ade824d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


然后在 tools 目录下添加 MyPlugin 文件夹，文件夹里面新增两个文件 CMakeLists.txt 和 MyPlugin.cpp，这里先不讲解具体文件中的内容，目的是想让插件跑起来，看到运行效果。

   CMakeLists.txt 文件如下：

   ```cmake
   add_llvm_loadable_module(MyPlugin 
   MyPlugin.cpp
   PLUGIN_TOOL clang
   )

   if(LLVM_ENABLE_PLUGINS AND (WIN32 OR CYGWIN))
     target_link_libraries(MyPlugin PRIVATE
       clangAST
       clangBasic
       clangFrontend
       clangLex
       LLVMSupport
       )
   endif()
   ```

   MyPlugin.cpp 文件如下：

   ```c++
   #include <iostream>
   #include "clang/AST/AST.h"
   #include "clang/AST/ASTConsumer.h"
   #include "clang/AST/RecursiveASTVisitor.h"
   #include "clang/Frontend/CompilerInstance.h"
   #include "clang/Frontend/FrontendPluginRegistry.h"
   using namespace clang;
   using namespace std;
   using namespace llvm;
   namespace MyPlugin
   {
       class MyASTVisitor: public
       RecursiveASTVisitor < MyASTVisitor >
       {
   private:
           ASTContext *context;
   public:
           void setContext(ASTContext &context)
           {
               this->context = &context;
           }

           bool VisitDecl(Decl *decl)
           {
               if (isa < ObjCInterfaceDecl > (decl)) {
                   ObjCInterfaceDecl *interDecl = (ObjCInterfaceDecl *)decl;
                   if (interDecl->getSuperClass()) {
                       string interName = interDecl->getNameAsString();
                       string superClassName = interDecl->getSuperClass()->getNameAsString();

                       cout << "-------- ClassName:" << interName << " superClassName:" << superClassName << endl;
                   }
               }

               return true;
           }
       };
       
       class MyASTConsumer: public ASTConsumer
       {
   private:
           MyASTVisitor visitor;
           void HandleTranslationUnit(ASTContext &context)
           {
               visitor.setContext(context);
               visitor.TraverseDecl(context.getTranslationUnitDecl());
           }
       };
       class MyASTAction: public PluginASTAction
       {
   public:
           unique_ptr < ASTConsumer > CreateASTConsumer(CompilerInstance & Compiler, StringRef InFile) {
               return unique_ptr < MyASTConsumer > (new MyASTConsumer);
           }
           bool ParseArgs(const CompilerInstance &CI, const std::vector < std::string >& args)
           {
               return true;
           }
       };
   }
   static clang::FrontendPluginRegistry::Add
   < MyPlugin::MyASTAction > X("MyPlugin",
                               "MyPlugin desc");
   ```

再次在 llvm_build 目录下 CMake 一下

```cmake
cmake -G Xcode ../llvm -DCMAKE_BUILD_TYPE:STRING=MinSizeRel
```

然后重新打开 LLVM.xcodeproj 工程，会发现多了一个 MyPlugin 的编译目标，选中进行编译。

![编译myPlugin.png](https://upload-images.jianshu.io/upload_images/698554-a1ec081353964edc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

编译成功之后，就可以得到一个 MyPlugin.dylib 的 Clang 插件了~为了方便，我将 MyPlugin.dylib 放在桌面上：

![MyPlugin插件.png](https://upload-images.jianshu.io/upload_images/698554-3b17229fdf5192e2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### 使用插件

#### 命令行中使用插件

首先用命令行对单文件测试一下刚刚生成的 Clang 插件是否正确，新建一个测试用文件 test.m 放在桌面，test.m 如下：

```objective-c
#import<UIKit/UIKit.h>
@interface ViewController : UIViewController
@end
@implementation ViewController
- (instancetype)init
{
	if(self = [super init]){
	}
	return self;
}
@end
```

现在我的 test.m 和 MyPlugin.dylib 都在桌面上了（当然也可以放在不同的目录下，只要在待会用到这两个文件的地方指定各自的绝对路径就行，这里是为了方便叙述）

![文件结构](https://upload-images.jianshu.io/upload_images/698554-5bd1c283af274180.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

接着命令行 cd 到桌面，然后执行以下命令就可以看到结果了：

```
/opt/llvm/llvm_build/Debug/bin/clang -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator11.2.sdk -Xclang -load -Xclang ./MyPlugin.dylib -Xclang -add-plugin -Xclang MyPlugin -c ./test.m
```

> 注意：
>
> 1. 我编译出来的 clang 在 /opt/llvm/llvm_build/Debug/bin/clang 目录中，如果你与我的路径不一样则指定为你对应的路径
>
> 2. 在我写这篇文章时 Xcode 版本是9.2，对应的是 iPhoneSimulator11.2.sdk，你需要进入该目录查看你的 sdk 版本

如无意外，命令行中会出现一大堆输出：

![命令行输出](https://upload-images.jianshu.io/upload_images/698554-0c3a4024f8974640.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



#### Xcode 中使用插件

接下来讲怎么样在 Xcode 使用我们刚刚编译出来的插件（随着 Xcode 变得封闭，插件挂载到 Xcode 上运行在未来的版本中可能会被禁止）。

首先 hack Xcode，才能使 Xcode 指向我们自己编译的 Clang：

下载 [XcodeHacking.zip](http://www.njiang.cn/uploads/2017/03/01/XcodeHacking.zip) 并解压，里面有 HackedBuildSystem.xcspec 和 HackedClang.xcplugin 两个文件，这里可能需要修改一下 HackedClang.xcplugin/Contents/Resources/HackedClang.xcspec 文件，将 ExecPath 的值修改为你编译出来的 Clang 的目录：
![修改HackedClang.xcspec](https://upload-images.jianshu.io/upload_images/698554-cd4d800cbebe8e09.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后 cd 到解压的 XcodeHacking 目录，将这两个文件用命令行移动到对应的目录下：

   ```
   sudo mv HackedClang.xcplugin `xcode-select -print-path`/../PlugIns/Xcode3Core.ideplugin/Contents/SharedSupport/Developer/Library/Xcode/Plug-ins
   sudo mv HackedBuildSystem.xcspec `xcode-select -print-path`/Platforms/iPhoneSimulator.platform/Developer/Library/Xcode/Specifications
   ```

   ​


然后重启 Xcode，点击 Target 的 Build Settings，修改 Compiler for C/C++/Objective-C 项为 Clang LLVM Trunk（不进行第1步中 hack Xcode 操作的话是不会有这个选项的）

![Complier.png](https://upload-images.jianshu.io/upload_images/698554-4ad812d07429b0e9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后修改 OTHER_CFLAGS 选项：
![OTHER_CFLAGS.png](https://upload-images.jianshu.io/upload_images/698554-e50bcd49a7356985.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

   ```
   -Xclang -load -Xclang /Users/Vernon/Desktop/MyPlugin.dylib -Xclang -add-plugin -Xclang MyPlugin
   ```

> 注意
>
> 1. 将 /Users/Vernon/Desktop/MyPlugin.dylib 修改为你生成的插件对应的目录
> 2. 如果编译中出现一大堆系统库的 symbol not found 错误的话，可以在上述命令的最后手动指定你的 SDK 目录：-isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator11.2.sdk



最后编译你的项目，然后快捷键 Command+9 跳到 Show the Report navigator，选中刚刚的编译报告，注意下图中每个文件右上角都有可以点击展开的按钮，展开后就能看到我们插件的输出了（下图4为对应输出）。Nice~
![查看结果](https://upload-images.jianshu.io/upload_images/698554-de614d7d87c219d6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### 结语

文章不长，只是这看似简单的过程也花了我一个多星期的业余时间，写下这个系列文章一是为了记录自己这钻研的过程，以后也可查询，二是希望如果有人能看到这篇拙文可以省下一点时间，更快的踏进 LLVM 和 Clang 的世界探索。

接下来会根据我的个人需求尝试给 Clang 添加自定义的 attribute，如果有所心得，会撰文分享，敬请期待~