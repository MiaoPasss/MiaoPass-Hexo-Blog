---
title: C#中的unsafe、fixed和checked关键字
date: 2025-02-11 15:19:51
categories: Programming
tags: 
- Memory Management
---

# Stack vs. Heap

In C#, primitive types such as int, double, and bool are all structs. Arrays of int are allocated on heap as int structs, only pointer to them will be allocated on stack.
To make a fixed size array be stored on stack, you need to use the *stackalloc* keyword, either with *unsafe* block or *Span* type.

```cs unsafe关键词示例
public void unsafe foo(int length)
{
    int* bar = stackalloc int [length];
}
```

```cs type Span示例
public void foo(int length)
{
    Span<int> bar = stackalloc int [length];
}
```

# unsafe

unsafe关键字用于声明不安全的代码块。在C#中，默认情况下，代码是安全的，这意味着它遵循.NET的安全规则，包括对内存的访问控制。使用unsafe关键字可以告诉编译器，你了解并信任这段代码，即使它可能违反安全规则。

使用unsafe关键字需要满足一些条件：

1. 你的项目必须被标记为允许不安全代码（通过在项目的属性中设置Allow Unsafe Code）。
2. 你的代码必须在unsafe代码块中。
3. 你必须使用fixed关键字来固定内存块。

# fixed

fixed 关键字在 C# 中主要用于固定内存地址，通常与不安全代码（unsafe）一起使用。当你在不安全的代码中直接访问内存时，使用 fixed 关键字可以确保内存地址在程序运行期间保持不变。

使用 fixed 关键字的主要原因是：在垃圾回收过程中，垃圾回收器可能会移动内存中的对象。如果一个指针指向一个对象，而该对象在垃圾回收过程中被移动，那么该指针就会变得无效。通过使用 fixed 关键字，你可以告诉垃圾回收器不要移动这个对象，从而确保指针始终指向有效的内存地址。


```cs fixed关键字示例
unsafe class Example  
{  
    int[] array = new int[10];  
    fixed int* ptr = stackalloc int[] { 1, 2, 3 };  
  
    void Method()  
    {  
        int* p = ptr; // 这里的p指向一个固定的内存地址  
        for (int i = 0; i < array.Length; i++)  
        {  
            *(p + i) = array[i]; // 将数组的值赋给固定的内存地址  
        }  
    }  
}
```

在这个例子中，我们创建了一个固定大小的数组 ptr，并在方法 Method 中使用它来修改另一个数组 array 的值。因为 ptr 是用 fixed 关键字声明的，所以它指向的内存地址在 Method 执行期间是固定的，不会发生位移

# checked

checked关键字用于在算术运算中控制溢出检查。默认情况下，当一个整数运算结果超出了该类型的表示范围时，会抛出System.OverflowException异常。使用checked关键字可以强制执行溢出检查，并在发生溢出时抛出异常。

```cs checked关键字示例
class Program  
{  
    static unsafe void Main(string[] args)  
    {  
        int maxValue = int.MaxValue;  
        int* ptr = stackalloc int[] { maxValue }; // 创建一个固定大小的数组  
        fixed (int* p = ptr) // 使用fixed关键字固定内存地址  
        {  
            *(p + 1) = 0; // 尝试访问超出数组范围的内存，这会导致未定义的行为（除非使用unsafe代码）  
        }  
        Console.WriteLine(ptr[1]); // 这将输出0，因为我们在不安全的代码中修改了内存  
    }  
}
uint a = uint.MaxValue;
 
unchecked
{
    Console.WriteLine(a + 3);  // output: 2
}
 
try
{
    checked
    {
        Console.WriteLine(a + 3);
    }
}
catch (OverflowException e)
{
    Console.WriteLine(e.Message);  // output: Arithmetic operation resulted in an overflow.
}
```

> 输出:
2
Arithmetic operation resulted in an overflow.

如果没有checked，那么输出的就是2，不会抛出异常也不会提示结果实际上已经超出范围了。导致程序发生一些不可预估的问题；

在这个例子中，我们创建了一个固定大小的数组，并在一个fixed代码块中修改了数组外的内存。因为我们使用了unsafe和fixed关键字，所以这是合法的。但请注意，试图访问数组外的内存是一种未定义的行为，可能会导致程序崩溃或其他不可预测的结果。

---

# Credits

https://blog.csdn.net/qq_31418645/article/details/135245645

---