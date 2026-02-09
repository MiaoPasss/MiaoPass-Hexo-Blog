---
title: HybridCLR简述
date: 2026-02-09 17:31:56
categories: Programming
tags: 
- Hotfix
- C#
---

HybridCLR扩充了IL2CPP的代码，使它由纯AOT Runtime变成“AOT+Interpreter“混合Runtime，进而原生支持动态加载Assembly，使得基于IL2CPP Backend打包的游戏不仅能在Android平台，也能在iOS、Consoles等限制了JIT的平台上高效地以AOT+interpreter混合模式执行。

通过 "Differential Hybrid dll" 技术，可以对 AOT dll 实现任意增删改，会智能地让*被修改或者新增的类和方法*以 Interpreter 模式运行，但 *未被修改的类* 以AOT方式运行，从而使*热更新的游戏逻辑*的运行性能基本达到原生AOT的水平。

---

# 基础原理
CLR，即Common Language Runtime，中文叫公共语言运行时，是让.NET程序执行所需的外部服务的集合，是.NET平台的核心和最重要的组件，类似于Java的JVM。

IL2CPP是Unity开发的跨平台CLR解决方案，诞生它的一个关键原因是Unity需要跨平台运行。但一些平台如iOS，这种禁止JIT并导致依赖JIT的官方CLR虚拟机无法运行，而是必须使用AOT技术将Mananged程序提前转化为目标平台的静态原生程序后再运行。而Mono虽然也支持AOT，但性能较差以及跨平台支持不佳。The IL2CPP backend converts MSIL (Microsoft Intermediate Language) code (for example, C# code in scripts) into C++ code, then uses the C++ code to create a native binary file (for example, .exe, .apk, or .xap) for your chosen platform.

IL2CPP方案包含一套AOT运行时以及一套DLL到C++代码及元数据的转换工具，使得原始的C#开发的代码最终能在iOS这样的平台运行起来。因为 IL2CPP 生成的 C++ 代码不是普通的 C++，它本质上还是在模拟 C# 的行为。很多 C# 特性在 C++ 里根本不存在，必须有额外代码来支撑。IL2CPP 只是把计算逻辑翻译成了 C++，但 C# 作为托管语言的那些"托管服务"，必须由运行时来提供。生成的 C++ 代码到处都在调用 il2cpp_xxx() 这类运行时 API，离开运行时根本跑不起来。

IL2CPP是一个纯静态的AOT运行时，不支持运行时加载DLL，因此不支持热更新；不像Mono有Hybrid mode execution，可支持动态加载DLL。

目前Unity平台的主流热更新方案xLua、ILRuntime之类都是引入一个第三方VM（Virtual Machine），在VM中解释执行代码，来实现热更新。这里我们只分析使用C#为开发语言的热更新方案。这些热更新方案的VM与IL2CPP是独立的，意味着它们的元数据系统是不相通的，在热更新里新增一个类型是无法被IL2CPP所识别的（例如，通过System.Activator.CreateInstance是不可能创建出这个热更新类型的实例），这种看起来像，但实际上又不是的伪CLR虚拟机，在与IL2CPP这种复杂的CLR运行时交互时，会产生极大量的兼容性问题，另外还有严重的性能问题。

HybridCLR 对 IL2CPP运行时进行扩充，添加Interpreter模块，将它由AOT运行时改造为“AOT + interpreter”双引擎的混合运行时，进而实现Mono hybrid mode execution这样的机制。这样一来就能彻底支持热更新，并且兼容性极佳。对开发者来说，除了解释模式运行的部分执行得比较慢，其他方面跟标准的运行时没有区别，完美支持在iOS这种禁止JIT的平台上以解释模式无缝地运行动态加载的DLL。

# 与其他热更新方案对比
HybridCLR是原生的C#热更新方案。通俗地说，IL2CPP相当于Mono的AOT模块，HybridCLR相当于Mono的Interpreter模块，两者合一成为完整Mono。HybridCLR使得IL2CPP变成一个全功能的Runtime，原生（即通过System.Reflection.Assembly.Load）支持动态加载DLL，从而支持iOS平台的热更新。

正因为HybridCLR是原生Runtime级别实现，热更新部分的类型与主工程AOT部分类型是完全等价并且无缝统一的。可以随意调用、继承、反射或多线程，不需要生成代码或者写适配器。

其他热更新方案则是独立VM，与IL2CPP的关系本质上相当于Mono中嵌入Lua的关系。因此类型系统不统一，为了让热更新类型能够继承AOT部分类型，需要写适配器，并且解释器中的类型不能为主工程的类型系统所识别。特性不完整、开发麻烦、运行效率低下。

