---
title: Clang 之旅—[翻译]添加自定义的 attribute
date: 2018-03-22 11:05:35
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

这是 Clang 之旅系列的第二篇，自己想要完成的需求是：在编译阶段检查某个方法的参数与返回值的类型相同，如果类型不一致的话能抛出编译错误的提示。需要接触到 Clang 中关于 attribute 处理的代码，所以这篇先来翻译[官方文档](https://clang.llvm.org/docs/InternalsManual.html#how-to-add-an-attribute)中添加自定义的 attribute 这一节，不得不说，虽然 Clang 的文档可以说是很标杆了，但是总有一种看了后面忘了前面的感觉，可能是 Clang 比较庞大，涉及专有词汇比较多的原因，所以我会偏向意译多一点，试图用更加易懂的表达组织语言，也是加深自己的记忆吧。



## 怎样添加 attribute

attribute 是一种可以附加到程序结构中的数据形式，允许开发人员传递信息给编译器来实现各种需求。例如，attribute 可以用来改变在程序构造时生成的代码，或者用来提供额外的信息给静态分析。本文档讲解如何添加一个自定义的 attribute 到 Clang 中。现有 attribute 列表的文档可以在[这里](https://clang.llvm.org/docs/AttributeReference.html)找到。



### attribute 基础知识

Clang 中的 attribute 涉及到三个阶段：解析 attribute 、从已解析的 attribute 转换成语法树上的 attribute、对 attribute 进行处理。

attribute 的解析可以采用多种语法形式，例如 GNU、C++ 11 和 Microsoft 形式，还由 attribute 提供的其他信息来确定。最终，解析好的 attribute 用一个 `AttributeList` 对象来表示。这些解析好的 attribute 会链成一个 attribute 链，加到声明或者定义上。attribtue 的解析是由 Clang 自动完成的，除了那些关键字 attribute。关键字的解析和 `AttributeList` 对象的生成必须由我们手动完成。

最后，`Sema::ProcessDeclAttributeList()` 带着 `Decl` 类型和 `AttributeList` 类型的参数被调用，此时解析好的 attribute 就会被转化成语法树上的 attribute。这个处理依赖于 attribute 的属性定义和语义要求。最后的结果就是语法树上的 attribute 对象可以从 `Decl` 对象获取到，也就是通过调用 `Decl::getAttr<T>()` 来获取。

语法树上的 attribute 的结构同样也受到 Attr.td 文件中的定义所限制。这个定义会自动生成 attribute 的实现所用到的功能，包括生成 `clang::Attr` 的子类、解析器所用到的信息和某些 attribute 自动进行的语义分析等等。



### include/clang/Basic/Attr.td

添加新的 attribute 到 Clang 的第一个步骤就是把其定义添加到 [include/clang/Basic/Attr.td](http://llvm.org/viewvc/llvm-project/cfe/trunk/include/clang/Basic/Attr.td?view=markup)。这个定义必须从 `Attr` 或者其子类继承。大多数 attribute 会直接从 `InheritableAttr` 继承，`InheritableAttr` 指定了这个 attribute 可以通过它所关联的 `Decl` 稍后进行重声明。如果这个 attribute 是作用于类型而不是声明，那么这种 attribute 应该从 `TypeAttr` 派生，并且通常不会被赋予 AST 表示（注意本文档并不讲解生成类型所用的 attribute）。一个继承于 `IgnoredAttr` 的 attribute 会被解析，但是会在被使用的时候产生一个 "被忽略的属性" 的警告，这种处理方法在某个属性支持别的前端而不支持 Clang 的情况下是很有用的。

这个定义能指定 attribute 的一些关键部分，比如 attribute 的名字、attribute 支持的拼写、attribute 的参数等等。`Attr` 类型中的大多数成员变量都不需要派生定义，缺省的就足够了。但是，每个 attribute 都需要至少指定 拼写列表、subject 列表和文档列表。



##### 拼写

所有 attribute 都需要指定一个拼写列表，表示拼写 attribute 的方式。比如某个 attribute 可能会包含关键字拼写， C++11 拼写和 GNU 拼写。空的拼写列表也是允许的并且可能对隐式创建的 attribute 有用。以下是支持的拼写的表格：

| 拼写     | 描述                                                         |
| :------- | :----------------------------------------------------------- |
| GNU      | 用 GNU 风格 `__attribute__((attr))` 语法和位置拼写           |
| CXX11    | 用 C++ 风格 `[[attr]]` 语法拼写。如果该 attribute 是由 Clang 所使用的，那么应该设置命名空间为 `"clang"` |
| Declspec | 用 Microsoft 风格 ` __declspec(attr)` 语法拼写               |
| Keyword  | 这个 attribute 用关键字的方式拼写，并且需要自定义解析        |
| GCC      | 指定两种拼写：首先是 GNU 风格拼写；然后是 C++ 风格拼写，命名空间为 `gnu`。只能为支持 GCC 的 attribute 指定这个拼写。 |
| Pragma   | attribute 用 `#pragma` 的形式拼写，并且需要在预处理器中执行自定义的处理。如果该 attribute 是由 Clang 所使用的，那么应该设置命名空间为 `"clang"`。需要注意这个拼写并不能被用于声明语句中。 |



##### Subjects

每个 attribute 都有一个或者多个 subject。如果 attribute 被使用到了一个不在 subject 列表上的 subject，就会自动显示诊断信息。 这个信息是警告还是错误是由 attribute 中的 `SubjectList` 决定的，默认的是警告。显示给用户的诊断信息将根据 subject 列表自动确定，但是也可以在 `SubjectList` 中指定自定义诊断参数。不符合 subject 列表导致的诊断信息要么是 `diag::warn_attribute_wrong_decl_type`，要么是 `diag::err_attribute_wrong_decl_type`。具体参数的枚举值可以从 [include/clang/Sema/AttributeList.h](http://llvm.org/viewvc/llvm-project/cfe/trunk/include/clang/Sema/AttributeList.h?view=markup) 找到。如果先前未使用的 `Decl` 节点被添加到 `SubjectList` 中，则可能需要更新用于自动确定 [utils/TableGen/ClangAttrEmitter.cpp](http://llvm.org/viewvc/llvm-project/cfe/trunk/utils/TableGen/ClangAttrEmitter.cpp?view=markup) 中的诊断参数的逻辑。

所有在 SubjectList 中的 subject 要么是在 `DeclNodes.td` 中定义的 Decl 节点，要么就是在 `StmtNodes.td` 中定义的 statement 节点。不过，可以生成 `SubsetSubject` 对象来创建更加复杂的 subject。每个这样的对象都有一个它所属的基本对象（必须是一个 Decl 或 Stmt 节点，而不是一个 SubsetSubject 节点），还有一些自定义代码在确定某个 attribute 是否属于该对象时被调用。例如，一个 `NonBitField` SubsetSubject 关联到 `FieldDecl` 类，同时会测试给定的 FieldDecl 是否是一个位字段。当在 SubjectList 中指定了一个 SubsetSubject 时必须同时提供一个自定义的诊断信息参数。

attribute 的 subject 列表会在 `HasCustomParsing` 设为 `1` 的情况下自动进行诊断检查。



##### 文档

所有的 attribute 都必须具有某种形式的文档。文档是通过每天运行的服务器端进程在公共服务器上生成的。通常来说，attribute 的文档是在 [include/clang/Basic/AttrDocs.td](http://llvm.org/viewvc/llvm-project/cfe/trunk/include/clang/Basic/AttdDocs.td?view=markup) 中单独定义的，以文档属性命名。

如果 attribute 不是通用的，或者是隐式创建的没有对应拼写的 attribuet，则可以将文档列表变量设置为 `Undocumented`。否则，该 attribute 应将其文档添加到 AttrDocs.td。

文档属性是从 `Documentation` tablegen 类型继承而来的，所有的派生类型都必须创建一个文档类别和设置文档本身内容。此外，它还可以为 attribute 指定一个自定义的标题，否则会选择默认的标题。

现在有四种预先定义好的文档类别：`DocCatFunction` 对应函数的 attribute，`DocCatVariable` 对应到变量的 attribute，`DocCatType` 对应类型的 attribute，`DocCatStmt` 对应声明的 attribute。自定义文档类别应该用于具有类似功能的 attribute 组。自定义类别非常适合用来为组中的 attribute 提供概述信息。

文档内容（包括 attribute 的内容或者类别的内容）是用 reStructuredText（RST）格式写的。

在编写该 attribute 的文档之后，应该对其在本地对其进行测试，以确保在服务器上生成文档不会有问题。本地测试需要重新构建 clang-tblgen。要生成 attribute 文档，请执行以下命令：

```
clang-tblgen -gen-attr-docs -I /path/to/clang/include /path/to/clang/include/clang/Basic/Attr.td -o /path/to/clang/docs/AttributeReference.rst
```

在本地进行测试时，不要对 `AttributeReference.rst` 提交更改。该文件是由服务器自动生成的，并且对该文件所做的任何更改都将被覆盖。



##### 参数

attribute 可以选择指定可以传递给 attribute 的参数列表。attribute 的参数指定 attribute 的解析形式和语义形式。例如，如果 `Args` 是 `[StringArgument<"Arg1">, IntArgument<"Arg2">]`，那么 `__attribute__((myattribute("Hello", 3)))` 就是一个合法的使用方式；这个 attribute 在解析时要求有两个参数：一个 string 类型一个 integer 类型。

每个参数都有个名字和一个用来指定这个参数是否为可选的标志。参数关联的 C++ 类型由参数定义类型确定。如果现有参数类型不足，则可以创建新类型，但需要修改 [utils/TableGen/ClangAttrEmitter.cpp](http://llvm.org/viewvc/llvm-project/cfe/trunk/utils/TableGen/ClangAttrEmitter.cpp?view=markup) 才能正确支持该新类型。



##### 其他属性

`Attr` 的定义还具有其他变量来控制 attribute 的行为。其中有很多是用于特殊用途的，超出了本文档的范围，但有一些还是值得提上一嘴的。

如果 attribute 的解析形式更加复杂或者和语义形式不同，则可以将 `HasCustomParsing` 变量设置为 `1`，并且可以针对特殊情况修改 [Parser::ParseGNUAttributeArgs()](http://llvm.org/viewvc/llvm-project/cfe/trunk/lib/Parse/ParseDecl.cpp?view=markup) 中的解析代码。请注意，这仅适用于具有 GNU 拼写的 attribute；__declspec 拼写的 attribute 现在是忽略这个标志的，并由 `Parser::ParseMicrosoftDeclSpec`  负责解析。 

请注意，把 `HasCustomParsing` 设置为 `1` 将不再使用通用的 attribute 处理逻辑，需要额外的处理来确保该  attribute 能使用。

如果该 attribute 不通过模板声明实例化，则将 `Clone` 成员变量设置为 0。默认情况下，所有的 attribute 都将通过模板进行实例化。

不需要 AST 节点的 attribute 应该将 `ASTNode` 变量设置为 0 以避免污染 AST。请注意，从 `TypeAttr` 或 `IgnoredAttr` 继承的类都不会自动生成 AST 节点。所有其他属性默认会生成一个 AST 节点。该 AST 节点是 attribute 的语义表示。

`LangOpts` 变量指定了 attribute 所需的语言选项列表。例如，所有的 CUDA-specific 的 attribuet 都将 `LangOpts` 字段指定为 `[CUDA]`，并且当 CUDA 语言选项未启用时，会发出“attribute ignored”的警告诊断。由于语言选项不是自动生成的节点，因此必须手动创建新的语言选项，并应指定 `LangOptions` 类所使用的拼写。

可以基于 attribute 的拼写列表为该 attribute 生成自定义的存取器。例如，如果某个 attribute 有两种不同的拼写：'foo' 和 'bar'，则可以创建访问器：`[Accessor<"isFoo", [GNU<"Foo">]>, Accessor<"isBar",[GNU<"Bar">]>]`。这些存取器将在该 attribute 的语义形式上生成，不接受任何参数并返回一个布尔值。

不需要自定义语义分析的 attribute 应该将 `SemaHandler` 变量设为 `0`。请注意，任何从 `IgnoredAttr` 继承的 attribute 都不会自动进行语义处理。所有其他 attribute 都使用默认的语义处理。没有语义处理的 attribute 都不会有解析好的 attribute `Kind` 枚举器。

指定 Target 的 attribute 可能会与不同 Target 的 attribute 共用一个拼写。例如，ARM 和 msp430 Target 都有一个拼写为 `GNU<"interrupt">` 的 attribute，但各自有不同的解析方式和语义要求。为了支持这个特性，继承自 `TargetSpecificAttribute` 的 attribute 可以指定 `ParseKind` 变量。这个变量在共用拼写的所有参数之间应该是相同的，并且对应于解析 attribute 的 `Kind` 的枚举器。这允许 attribute 共用一种解析类型，但具有不同的语义属性。例如，`AttributeList::AT_Interrupt` 是共用的解析类型，但 ARMInterruptAttr 和 MSP430InterruptAttr 是各自的语义属性。

默认情况下，当声明为 merging attribute 时，该 attributes 不会被复制。但是，如果在此合并阶段中可以复制某个 attribute，那么将 `DuplicatesAllowedWhileMerging` 变量设置为 `1`，该 attribute 就会被合并。

默认情况下，attribute 的参数在上下文中被解析。如果应该在上下文中解析 attribute 的参数（类似于解析 `sizeof` 表达式的参数的方式），请将 `ParseArgumentsAsUnevaluated` 设置为 `1`。



### 样板代码

声明 attribute 的所有的语义处理都在文件 [lib/Sema/SemaDeclAttr.cpp](http://llvm.org/viewvc/llvm-project/cfe/trunk/lib/Sema/SemaDeclAttr.cpp?view=markup) 中，并且通常都从 `ProcessDeclAttribute()` 函数开始。如果这个 attribute 是一个“简单的” attribute，也就是说这个 attribute 除了自动生成的内容之外不需要自定义的语义处理，那么就添加 `handleSimpleAttribute<YourAttr>(S, D, Attr);` 函数到 switch 语句中。否则，编写一个新的 `handleYourAttr()` 函数，并将其添加到 switch 语句中。不要直接在 `case` 语句中实现处理逻辑。

除非 attribute 的定义中另有规定，否则将自动处理解析 attribute 的常见语义检查，包括诊断不属于给定 `Decl` 的解析的 attribute、确保传递正确的最小数量的参数等等。

如果 attribute 要加上额外的警告，那么在 [include/clang/Basic/DiagnosticGroups.td](http://llvm.org/viewvc/llvm-project/cfe/trunk/include/clang/Basic/DiagnosticGroups.td?view=markup) 文件中定义一个 `DiagGroup`。如果只有一个诊断信息的话，直接在 [DiagnosticSemaKinds.td](http://llvm.org/viewvc/llvm-project/cfe/trunk/include/clang/Basic/DiagnosticSemaKinds.td?view=markup) 文件中使用 `InGroup<DiagGroup<"your-attribute">>` 也是可以的。

所有为你自定义的 attribute 所生成的诊断信息，包括自动生成的（比如 subject 和参数个数），都应该有一个对应的测试用例。



### 语义处理

大多数 attribute 被实现为对编译器有一定的影响。例如，修改生成代码的方式，或为分析过程添加额外的语义检查等，将 attribute 的定义和转换添加到该 attribute 的语义表示中，剩下的就是实现 attribute 的自定义逻辑。

可以使用 `hasAttr<T>()` 方法来查询 `clang::Decl` 对象中是否有 attribute。可以使用 `getAttr<T>` 来获取一个指向 attribute 的指针。 

