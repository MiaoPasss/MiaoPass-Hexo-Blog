---
title: MVC和MVVM
date: 2026-03-04 17:46:46
categories: Design Paradigm
tags: 
- Observer Pattern
---

# MVC
* Model：纯数据容器（如：PlayerState），不包含 UI 或游戏逻辑。
* View：负责渲染数据（如：UI 文本、生命值条），通常直接监听（观察） Model 的变化。
* Controller：处理用户输入（如：点击按钮）并修改 Model。

特点：View 和 Model 之间仍有直接关联（View 需要知道 Model 的存在以进行监听），这使得 View 难以独立测试。

# MVP
* Model：纯数据，不包含业务逻辑，不依赖其他层。
* View：被动视图，不直接访问 Model。它只负责将用户操作传给 Presenter，并提供更新 UI 的公共接口。
* Presenter：核心中间层。它监听 Model 的变化，格式化数据后手动调用 View 的接口更新 UI；同时监听 View 的事件来修改 Model。

特点：View 和 Model 完全解耦。由于 View 非常“薄”，你可以轻松编写 Presenter 的单元测试。

# MVC 和 MVP 的区别
MVC（模型-视图-控制器）和MVP（模型-视图-表示器）的核心区别在于View与Model的解耦程度和通信方式。
MVC中View可直接访问Model，耦合度较高；而MVP通过接口将View与Model完全隔绝，所有交互由Presenter中转，使代码更易于测试和维护。

## View 与 Model 的关系
* MVC： View 可以直接读取 Model 数据（View 依赖 Model），View 中可能包含部分业务逻辑。
* MVP： View 与 Model 彻底解耦。View 只能通过接口与 Presenter 通信，无法感知 Model。
## 交互逻辑
* MVC： Controller 接收用户请求，更新 Model，View 监听 Model 变化更新界面（有时 C 直接更新 V）。
* MVP： Presenter 作为中介，不仅处理 UI 事件，还负责从 Model 获取数据并通过接口更新 View。View 是被动接收渲染。
## 测试性
* MVC： 较难进行单元测试，因为 UI 逻辑与数据逻辑存在耦合。
* MVP： 极易测试，Presenter 逻辑与具体 UI 实现分离，可通过 Mock 接口进行纯逻辑测试。

# MVVM
MVVM 是从 WPF 等前端技术引入的，核心在于 数据绑定 (Data Binding)。 

* Model（模型）：纯数据，不包含业务逻辑，不依赖其他层。
* View（视图）：UI 组件，通过数据绑定 (Data Binding)直接映射 ViewModel 的属性。
* ViewModel（视图模型）：它是 View 的抽象。它将 Model 的数据转换为 View 可以直接使用的格式（如将 float 转换为 string），并通过属性变更通知（如 OnPropertyChanged）自动驱动 View 更新。

特点：双向/单向数据绑定，View 不知道 ViewModel 的存在
优点：自动化程度最高。Model 改变时，View 会通过绑定自动更新，无需像 MVP 那样手动写代码控制。只需“订阅”数据变化，实现了极高的解耦。

# Unity 中的 UI-Model 设计架构

## UGUI：以 MVP 为核心的实现
UGUI 本身不提供自动化的数据绑定，因此最契合的模式是 MVP (Model-View-Presenter)。通过 Presenter 手动协调数据与界面的同步。
* Model: 纯 C# 类或 ScriptableObject。定义 Action 或 UnityEvent，在数据变化时发出通知。
* View: 继承 MonoBehaviour 的 UI 脚本。包含对 Text、Slider 等组件的引用，并提供简单的公共方法（如 UpdateHealthBar(int value)）供外部调用。
* Presenter: 用于连接/同步 Model 和 View 的中间层。
    * 监听 Model: 订阅 Model 的事件，当数据改变时，调用 View 的更新方法。
    * 监听 UI: 接收来自 View 的 UI 事件（如 Button.onClick），处理逻辑后修改 Model。

## UI Toolkit：以 MVVM 为核心的实现
UI Toolkit（尤其是 Unity 6 以后）原生支持 数据绑定 (Data Binding)，这使得实现 MVVM 变得非常自然且高效。
* Model: 基础数据结构，C#原生类型变量。
* View: 使用 UXML（结构）和 USS（样式）文件定义。在 UI Builder 中，你可以直接通过 Attributes 检查器 将 UI 元素的属性（如 Label 的 Text）绑定到数据源路径。
* ViewModel: 负责“翻译”数据。
    * 实现通知: 属性通常需要实现 INotifyPropertyChanged 接口或使用特定属性标签（如 [ObservableProperty]），以便在值变化时自动触发 UI 更新。
    * 命令绑定: 使用 ICommand 或 RelayCommand 绑定按钮点击等行为，无需在脚本中手动查找 VisualElement。
* 代码解耦: View 和 ViewModel 之间不需要显式的引用关系，完全通过绑定路径 (Binding Path) 链接。