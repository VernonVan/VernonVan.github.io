---
title: iOS开发相关命令整理
tags:
- iOS
- 命令
categories:
- ruanpapa--技术贴
---



1. 查看本机安装的 SDK：

   ```
   xcodebuild -showsdks
   ```

![xcodebuild](http://upload-images.jianshu.io/upload_images/698554-8767d163320b30c5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



2. 查看 .a 以及 .framework 支持的架构：

   .framework 文件：

   ```
   lipo -info xxx.framework/xxxxFramework
   ```

   .a 文件： 

   ```
   lipo -info xxx.a
   ```

   ![framew架构](https://upload-images.jianshu.io/upload_images/698554-e7004756972e1ccf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




3. 暂停正在执行的 make 命令：

   暂停：```Command+Z ```

   继续执行：```fg %1```



4. 查看上一个命令是否执行成功，返回值为 0 则表示执行成功：

   ```echo $? ```