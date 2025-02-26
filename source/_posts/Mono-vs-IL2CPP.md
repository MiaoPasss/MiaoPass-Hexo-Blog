---
title: Mono vs. IL2CPP
date: 2025-02-25 17:50:04
categories: C#编程
tags:
- Unity引擎
- 编译器
- 运行环境
---

# Common Language Runtime
公共语言运行时

VM和LR其实比较类似，有人说LR的创建就是为了对标VM。简单来说，就是一个程序运行所需要的环境，包括各种资源、各种操作等等。不同语言、不同操作系统所需要的运行时环境都不一样。举个例子，Windows上的可执行程序都被包装成了.exe格式，而这种.exe格式文件提供了一个程序从加载到运行所需要的所有资源和环境。

而CLR提供了：
1. 一个支持GC的虚拟机，该虚拟机有自己的一套指令集，即CIL（公共中间语言，COmmon Intermediate Language）。高级语言最终会转化成CIL，
2. 一种丰富的元数据表示，用来描述数据类型、字段、方法等。通过这些统一的描述方法来生成对应的程序。
3. 一种文件格式，一种专属的不于操作系统和硬件绑定的格式，即跨平台。
4. 一套类库，提供了垃圾回收、异常、泛型等基本功能，提供了字符串、数组、列表、字典等数据结构，提供了文件、网络、交互等操作系统功能。
5. 一系列规则，定制了在运行时如果查找引用其他文件、生命周期等一系列规则。

![](CLR.png "高级语言、CLR与CIL之间的关系") <br>

## Common Language Specification
CLR最大的优势就在于跨语言跨平台支持。目前微软已经为多种语言开发旅了基于CLR的编译器，包括C++、C#、Visual Basic、F#、Iron Python、 Iron Ruby和IL。还有一些大学、公司和机构为一些语言也开发了基于CLR的编译器，包括da、APL、Caml、COBOL、Eiffel、Forth、Fortran、Haskell、Lexicon、LISP、LOGO、Lua、Mercury、ML、Mondrian、Oberon、Pascal、Perl、PHP、Prolog、RPG、Scheme、Smaltak、Tcl/Tk等。
CLR为不同的编程语言提供了统一的运行平台，对于开发者来说，他们无需考虑平台运行问题，无论使用什么语言开发，最终都会编译成IL，供CLR运行。对于CLR来说，它并不知道也无需知道IL是从什么语言编译过来的。

但是这么多种各式各样的语言最终都要编译成IL，肯定需要一种规范，CLS就是用来规范语言的。CLS全称Common Language Specification，即公共语言规范。也就是说所有被CLR支持的高级语言都需要最少支持CLS所规定的功能集。只要高级语言最少支持了CLS之后，其它附加功能/特性可自行实现。

## Managed Code
C#中的托管代码是指由.NET运行时环境（CLR）管理和执行的代码。当我们使用C#编写的代码被编译后，它会被转换成中间语言（IL）代码，也称为托管代码。托管代码在运行时由CLR加载和执行，CLR负责内存管理、垃圾回收、安全性等任务，开发者无需过度关注资源的释放。其实可以从字面上理解，托管代码委托给CLR进行管理，开发者不管。
而至于非托管代码是指不受CLR管理的代码，通常是使用其他编程语言（如C++）编写的代码，比如操作系统代码、C#中的Socket、Stream等，这些代码无法通过CLR的GC完全释放占用的资源。一般来说，非托管的功能都被包装过了，比如当我们访问文件的时候，肯定不会直接使用操作系统的CreateFile函数，而是使用System.IO.File类。

托管代码具有以下特点：
自动内存管理：CLR负责分配和释放内存，开发人员无需手动管理内存。
垃圾回收：CLR会自动检测和回收不再使用的对象，减少内存泄漏的风险。
安全性：CLR提供了安全性机制，确保代码的执行不会对系统造成损害。
跨平台：托管代码可以在不同的操作系统上运行，只要有对应的CLR。

相对应的，非托管代码直接操作计算机的硬件和操作系统资源，需要手动管理内存和资源的分配和释放。非托管代码在性能方面可能更高效，但也更容易出现内存泄漏和安全问题。C#可以通过使用InteropServices命名空间中的功能与非托管代码进行交互，这样可以在C#中调用非托管代码的功能。

## FCL

The Framework Class Library or FCL provides the system functionality in the .NET Framework as it has various classes, data types, interfaces, etc. to build different types of applications including desktop applications, web applications, mobile applications. The Framework Class Library is integrated with the Common Language Runtime (CLR) and is used by all the .NET languages such as C#, F#, Visual Basic .NET, etc. 

### BCL vs. FCL
- The Base Class Library (BCL) is literally that, the base. It contains basic, fundamental types like System.String and System.DateTime.
- The Framework Class Library (FCL) is the wider library that contains the totality: ASP.NET（web application framework 对标 Node.js）, WinForms, the XML stack, ADO.NET and more. You could say that the FCL includes the BCL.

# C# Compilers

C#源文件通过编译器（如CSC.exe）编译成中间语言（IL）和元数据，生成.exe或.dll文件。IL是一种伪代码，独立于任何CPU，可以在任何装有.Net Framework的机器上运行‌

Common Intermediate Language (CIL), formerly called Microsoft Intermediate Language (MSIL) or Intermediate Language (IL) is the intermediate language binary instruction set defined within the Common Language Infrastructure (CLI) specification. CIL instructions are executed by a CIL-compatible runtime environment such as the Common Language Runtime. Languages which target the CLI compile to CIL. CIL is object-oriented, stack-based bytecode. Runtimes typically just-in-time compile CIL instructions into native code.

- **Just In Time compiler**
即时编译。当程序运行时，IL通过CLR中的即时编译器(JIT)将CIL的byte code编译为目标平台的原生码(针对特定CPU的机械码)。JIT编译是在程序运行时进行的，确保了代码的可移植性和执行效率‌程序运行过程中。
- **Ahead Of Time compiler**
提前编译。程序运行之前，提前编译器(AOT)将C#源码直接编译为目标平台的原生码并且存储。这种方式通常用于生成本地应用程序，提高启动速度和性能‌。

将.exe或.dll文件中的CIL的byte code

## Unity compiler
Unity编译C#脚本的过程通常是自动进行的，当你在Unity编辑器中构建项目时（比如导出为执行文件或者打包为Android/iOS应用），Unity会编译所有C#脚本。
如果你需要在Unity编辑器外部编译C#代码，你可以使用Mono的mcs编译器或者.NET Core SDK。

以下是使用mcs编译器的基本命令行示例：
> mcs -out:YourGame.exe -recurse:*.cs

# Mono
Mono, the open source development platform based on the .NET Framework, helps developers to build cross-platform applications. Mono’s .NET implementation is based on the ECMA standards for C# and the Common Language Infrastructure. Mono was originally reimplementation of the .NET for linux. Today is much more.

## Unity Mono
The Mono scripting backend compiles code at runtime, with a technique called just-in-time compilation (JIT). Unity uses a fork of the open source Mono project.
Some platforms don’t support JIT compilation, so the Mono backend doesn’t work on every platform. Other platforms support JIT and Mono but not ahead-of-time compilation (AOT), and so can’t support the IL2CPP backend. When a platform can support both backends, Mono is the default.

# IL2CPP
The IL2CPP backend converts MSIL (Microsoft Intermediate Language) code (for example, C# code in scripts) into C++ code, then uses the C++ code to create a native binary file (for example, .exe, .apk, or .xap) for your chosen platform. This type of compilation, in which Unity compiles code specifically for a target platform when it builds the native binary, is called ahead-of-time (AOT) compilation.

## How IL2CPP works
1. The Roslyn C# compiler compiles your application’s C# code and any required package code to .NET DLLs (managed assemblies).
2. Unity applies managed bytecode stripping（代码裁剪）. This step can significantly reduce the size of a built application.
3. The IL2CPP backend converts all managed assemblies into standard C++ code.
4. The C++ compiler compiles the generated C++ code and the runtime part of IL2CPP with a native platform compiler.
5. Unity creates either an executable file or a DLL, depending on the platform you target.

# Mono vs. IL2CPP on Unity
Unity中C#代码的处理过程

1. 编写C#代码：
开发者在Unity编辑器中编写C#脚本，这些脚本通常用于实现游戏逻辑、控制角色行为、处理用户输入等。

2. 编译C#代码：
当开发者在Unity中保存C#脚本时，Unity会自动触发编译过程。Unity使用Mono或IL2CPP作为其脚本引擎。
Mono：在使用Mono时，Unity会将C#代码编译成CIL（Common Intermediate Language）。这个过程是在Unity编辑器中完成的，生成的CIL代码会被打包到Unity的可执行文件中。
IL2CPP：如果选择使用IL2CPP，Unity会将C#代码首先编译为CIL，然后将CIL代码转换为C++代码，最后再编译为本地机器代码。IL2CPP的主要优点是可以提高性能和安全性。

3. 生成的CIL代码：
生成的CIL代码会被打包到Unity的可执行文件中，通常是一个DLL文件。这个DLL文件包含了所有的游戏逻辑和功能。

4. 构建过程：
在构建游戏时，Unity会将所有的资源（如纹理、模型、音频等）和编译后的CIL代码打包成一个可执行文件（如EXE、APK、IPA等），具体取决于目标平台。
Unity的构建系统会处理所有的依赖关系，确保所有需要的资源和代码都包含在最终的构建中。

5. 运行时执行：
当用户运行构建的游戏时，Unity的运行时环境会加载可执行文件。
如果使用Mono，运行时会在需要时将CIL代码即时编译为本地机器代码（JIT编译）。如果使用IL2CPP，CIL代码已经在构建时转换为本地机器代码，因此可以直接执行。
运行时会管理内存、处理输入、渲染图形等，确保游戏的正常运行。

6. 总结
在Unity中，开发者编写的C#代码会经过编译过程，生成CIL代码，并在构建时打包到可执行文件中。Unity使用Mono或IL2CPP作为脚本引擎，分别通过JIT编译或AOT编译将CIL代码转换为本地机器代码。这个过程使得Unity能够在不同平台上运行相同的代码，同时也为开发者提供了灵活的开发环境。

## Cross-platform
Mono运行时编译器支持将IL代码转为对应平台原生码
IL可以在任何支持CLI，通用语言环境的平台上运行，IL的运行是依托于Mono运行时。

## IOS Platform
IOS不支持动态生成的代码具有执行权限，而通常jit就是运行过程中动态编译代码为机器码并缓存/执行（Mono运行时将IL编译成机械码），即封存了内存的可执行权限，变相的封锁了jit编译方式

---

# Credits

CLR简介：https://blog.csdn.net/weixin_42186870/article/details/119621977/
Mono简介：https://www.mono-project.com/docs/about-mono/
IL2CPP简介：https://docs.unity3d.com/6000.0/Documentation/Manual/scripting-backends-il2cpp.html
IOS平台代码热更：https://blog.csdn.net/qq_33060405/article/details/144314440

---