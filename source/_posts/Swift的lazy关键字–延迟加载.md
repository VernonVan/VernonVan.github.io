---
title: Swift的lazy关键字–延迟加载
date: 2016-12-28 16:29:02
tags:
- iOS
- Swift
categories:
- ruanpapa -- 技术贴
---
### 定义
lazy属性就是初始值直到第一次使用的时候才执行计算的属性，这对小内存的手机所产生的性能上的优化是相当可观的。
**注意：lazy属性必须是变量（var修饰符），因为常量属性（let修饰符）必须在初始化之前就有值，所以常量属性不能定义为lazy。**

### Objective-C中的延迟加载
Objective-C并没有在语法上支持延迟加载，通常是由程序员自己手动实现的。
示例如下：
``` swift
@property (nonatomic, strong) NSArray *names;

- (NSArray *)names {
    if (!_names) {
        _names = [[NSArray alloc] init];
        NSLog(@"只在首次访问输出");
    }
    return _names;
}
```
说明：在初始化对象后，_names 是 nil。只有当首次访问 names 属性时 getter 方法会被调用，并检查如果还没有初始化的话，就进行赋值。可以想见，控制台打印的“只在首次访问输出”的确只会输出一次。我们之后再多次访问这个属性的话，因为 _names已经有值，因此将直接返回。
分析：getter方法和下划线语法对初学者并不是那么的友好，同时属性以及对应的getter方法空间上隔得比较远，代码逻辑不直观。

### Swift的延迟加载
Swift中则可以通过**lazy**关键字简单地实现相同功能，比如上述示例代码在Swift中实现的话：
``` swift
lazy var names: NSArray = {
	let names = NSArray()
	print("只在首次访问输出")
	return names
}()
```
分析：相比起Objective-C中的实现，现在的lazy是在是简单的多了，而且更加的直观。除了上述直接在属性后面定义闭包调用的方法以外，还可以使用实例方法（func）和类方法（class func）来为lazy属性初始化添加必要的逻辑。

### 使用场景
延迟加载主要有以下两个使用的场景：
1. 属性的初始值依赖于其他的属性值，只有其他的属性值有值之后才能得出该属性的值。
2. 属性的初始值需要大量的计算。

### 高级用法
在Swift标准库中还有一组lazy方法，可以配合map、filter这类接受闭包并进行运行的方法一起，让整个行为变成延时进行的。以下是map函数的用法（示例取自喵神的[Swifter](http://swifter.tips/lazy/)）:
``` Swift
let data = 1...3
let result = data.lazy.map {
    (i: Int) -> Int in
    print("正在处理 \(i)")
    return i * 2
}

print("准备访问结果")
for i in result {
    print("操作后结果为 \(i)")
}

print("操作完毕")
```
``` swift
// 准备访问结果
// 正在处理 1
// 操作后结果为 2
// 正在处理 2
// 操作后结果为 4
// 正在处理 3
// 操作后结果为 6
// 操作完毕
```

> 对于那些不需要完全运行，可能提前退出的情况，使用 lazy 来进行性能优化效果会非常有效。

参考链接：
1. 喵神：[LAZY 修饰符和 LAZY 方法](http://swifter.tips/lazy/)
2. Swiftist：[Swift中的延迟加载](http://swiftist.org/topics/129)
3. [Apple developer](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Properties.html)
