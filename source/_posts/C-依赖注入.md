---
title: C# 依赖注入
date: 2025-08-28 16:30:54
categories: Programming
tags: 
- OOP
---

# No Dependency Injection
不使用依赖注入，必须在依赖方(Dependent Class)中主动创建或者获取被依赖方(Denpendency Class)

```cs
public class ClassA
{
    private readonly ClassB _classB;

    public ClassA()
    {
        _classB = new ClassB(); //主动创建对象B
    }

    public void Process()
    {
        Console.WriteLine("Class A start process");
        _classB.DoSomething();
        Console.WriteLine("Class A finish process");
    }
}

public class ClassB
{
    public void DoSomething()
    {
        // ClassB performs some logic
    }
}
```

# Denpendency Injection and IoC
使用依赖注入，不需要在依赖方的代码里主动创建或者获取被依赖方，反而，只需要在构造器参数里声明需要对象B的引用。

```cs ClassA.cs
public class ClassA
{
    private readonly IInterfaceB _b;

    public ClassA(IInterfaceB b)
    {
        _b = b;
    }

    public void Process()
    {
        Console.WriteLine("Class A start process");
        _b.DoSomething();
        Console.WriteLine("Class A finish process");
    }
}
```
在使用依赖注入时，更多时候，对于依赖更提倡使用接口，这样解耦了接口和实现：ClassA不需要知道ClassB的内部，只需要知道IClassB有个叫DoSomething的方法可以调用
并且，业务代码中不需要主动实例化对象，即无需这样主动调用构造函数 new ClassA(new ClassB())

```cs ClassB.cs
public class ClassB : IClassB
{
    public void DoSomething()
    {
        Console.WriteLine("class B is doing something ...");
    }
}
```

```cs IClassB.cs
public interface IClassB
{
    void DoSomething();
}
```

```cs Program.cs
internal class Program
{
    public static void Main(string[] args)
    {
        IHost host = Host.CreateDefaultBuilder(args)
                        .ConfigureServices((context, services) =>
                        {
                            services.AddSingleton<ClassA>();
                            services.AddSingleton<IInterfaceB, ClassB>();
                        }).Build();

        ClassA a = host.Services.GetRequiredService<ClassA>();
        a.Process();
    }
}
```
原本是在ClassA内部决定使用怎样的ClassB实例，使用了依赖注入设计后，这种控制关系（决定关系）变为由外部控制了，这就是所谓的“控制反转”(Inversion of Control)

* Host 以及它内部的 Services可以理解为 C# 提供的依赖注入系统
* 通过 GetRequiredService 可以获得对应的实例并执行业务逻辑（此处为ClassA实例）

# 使用依赖注入系统实例化对象
如果我们想要达到不需要手动实例化ClassA的效果，可以新建一个类并实现IHostedService接口

```cs DoSomethingHostedService.cs
public class DoSomethingHostedService : IHostedService
{
    private readonly ClassA _a;

    public DoSomethingHostedService(ClassA a)
    {
        _a = a;
    }

    public Task StartAsync(CancellationToken cancellationToken)
    {
        _a.Process();
        return Task.CompletedTask;
    }

    public Task StopAsync(CancellationToken cancellationToken)
    {
        return Task.CompletedTask;
    }
}
```

```cs 修改 Program.cs
internal class Program
{
    public static async Task Main(string[] args)
    {
        IHost host = Host.CreateDefaultBuilder(args)
                        .ConfigureServices((context, services) =>
                        {
                            services.AddSingleton<ClassA>();
                            services.AddSingleton<IInterfaceB, ClassB>();

                            services.AddHostedService<DoSomethingHostedService>();
                        }).Build();

        await host.RunAsync();
    }
}
```

* IHostedService 是一个特殊的接口，实现这个接口的类通过在Host中的services里注册后，可以在Host运行时自动实例化并执行
* services.AddHostedService<>() 用于注册要自动执行的类
* await host.RunAsync() 运行Host实例

---

# Credits:

https://zhuanlan.zhihu.com/p/592698341

---