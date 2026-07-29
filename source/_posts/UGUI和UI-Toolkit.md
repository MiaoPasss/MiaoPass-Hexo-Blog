---
title: UGUI和UI Toolkit
date: 2026-07-24 17:03:05
categories: 
- Game Engine
- Unity
---

# 核心架构差异

**UGUI：每个 UI 元素是一个 GameObject**
* UI 就是场景里的 GameObject，挂 RectTransform + Canvas Renderer + 各种组件（Image、Text、Button…）。
* 一个复杂界面 = 成百上千个 GameObject，纳入正常的场景层级、Prefab 系统。
* 优点：概念统一，和场景其它对象一致，容易理解、容易挂脚本。
* 缺点：GameObject/Component 开销大，元素多了内存和遍历成本高。

**UI Toolkit：一棵轻量的 VisualElement 树**
* UI 是一棵 VisualElement 树，不是 GameObject，是纯 C# 轻量对象。
* 结构（UXML）、样式（USS）、逻辑（C#）三者分离，像 Web 的 HTML/CSS/JS。
* 优点：元素轻量，海量元素也省内存；结构/样式/逻辑解耦。
* 缺点：不在场景层级里，不能直接给单个元素挂 MonoBehaviour，思路要转变。

# 渲染机制

UGUI的渲染单元是Canvas，按 Canvas 合批
* 合批：依赖 Canvas 内元素的层级/材质，容易被打断，需手动优化（拆 Canvas、图集）
* 重建：一个元素变化可能触发整个 Canvas rebuild（著名的性能坑）

UI Toolkit的渲染单元是统一的 UIR（UI Renderer）网格系统
* 合批：通过引擎自动生成网格并合批，通常 drawcall 更少
* 重建：局部脏标记，只重建变化部分

# 更新UI样式
UGUI的样式序列化在每个GameObject/组件字段上
* 复用：改一处只影响一个游戏对象，批量改要靠 Prefab 或脚本
* 热重载：组件字段改了即时可见（Inspector）
* 逻辑分离：样式和结构混在场景/预制体里

UI Toolkit的样式存保存于独立的 USS 文本文件中
* 复用：一个选择器（class）改一处，所有匹配元素全局生效
* 热重载：存盘热重载，且可运行时动态换 USS
* 逻辑分离：样式（USS）/结构（UXML）/逻辑（C#）三文件分离

UI Toolkit 让"改样式"变成改一个集中的、可选择器批量作用、可热重载、可运行时切换的文本文件，且和 C# 逻辑彻底分离

样式和结构可以脱离 C# 迭代，但数据源和逻辑仍在 C# 里:
* 纯视觉（颜色、间距、字体、布局、:hover 效果）→ 改 USS 即可，不碰 C#。
* 改结构（增删元素、换控件）→ 改 UXML，通常也不用重编译 C#（如果不新增需要 C# 引用的元素）。
* 改数据绑定关系：如果绑定写在 UXML 里，改绑定路径也不用重编译；但如果绑定写在 C# 里（SetBinding(...)），那就要重编译。
* 改 ViewModel / [CreateProperty] 属性 / 业务逻辑 → 必须重编译 C#。

---

# 数据绑定
UGUI 根本没有内置数据绑定系统。你在 UGUI 里做的所有"数据显示到 UI"都是手写命令式代码。
UI Toolkit（Unity 6 起）则提供了声明式的数据绑定。所以优势不是"绑定做得更好"，而是"从无到有"。

## UGUI 的写法（命令式，手动同步）
```cs 在Player类中手动引用healthSlider和实现更新
public class Player : MonoBehaviour
{
    public int health = 100;
    public int maxHealth = 100;

    public Slider healthSlider;   // 拖引用
    public Text healthLabel;      // 拖引用

    public void TakeDamage(int dmg)
    {
        health -= dmg;
        RefreshUI();   // 每次改数据都得记得手动刷新
    }

    void RefreshUI()
    {
        healthSlider.value = (float)health / maxHealth;
        healthLabel.text  = $"{health}/{maxHealth}";
    }
}
```

* 数据和 UI 强耦合：Player 逻辑类里塞了 Slider/Text 引用。
* 容易漏刷新：任何地方改了 health 都得记得调 RefreshUI()，漏一处就显示错。
* UI 引用靠拖拽：换了 UI 结构容易丢引用、报 NullReference。

## UI Toolkit 的写法（声明式，自动同步）
```cs 数据源（纯数据，不知道 UI 的存在）
public class PlayerViewModel : INotifyBindablePropertyChanged
{
    int _health = 100;

    [CreateProperty]
    public int Health
    {
        get => _health;
        set
        {
            if (_health == value) return;
            _health = value;
            Notify(); // 只管喊一声"我变了"
        }
    }

    [CreateProperty]
    public float HealthRatio => _health / 100f; // 派生属性也能绑

    public event EventHandler<BindablePropertyChangedEventArgs> propertyChanged;
    private void Notify([CallerMemberName] string name = "") {
        propertyChanged?.Invoke(this, new BindablePropertyChangedEventArgs(name));
    }
}
```

```cs 绑定（一次性建立，之后自动同步）
var vm = new PlayerViewModel();
root.dataSource = vm;

var bar = root.Q<ProgressBar>("healthBar");
bar.SetBinding("value", new DataBinding
{
    dataSourcePath = new PropertyPath(nameof(PlayerViewModel.HealthRatio)),
    bindingMode = BindingMode.ToTarget
});

var label = root.Q<Label>("healthLabel");
label.SetBinding("text", new DataBinding
{
    dataSourcePath = new PropertyPath(nameof(PlayerViewModel.Health)),
    bindingMode = BindingMode.ToTarget
});
```

之后业务代码只写 vm.Health -= 10;，血条和文字自动更新。逻辑层再也不碰任何 UI 引用。
这些绑定也可以直接写在 UXML 里，做到 UI 结构和绑定关系纯声明式、和 C# 逻辑完全分离。

## UI Toolkit的数据绑定优势
1. 解耦：数据源（ViewModel）完全不知道 UI 存在，可单独测试、复用。
2. 声明式 + 少样板：绑定关系集中声明（甚至写在 UXML），不用到处写刷新代码，也不会漏刷新。
3. 双向绑定：BindingMode.TwoWay，输入框改了值自动回写数据源，UGUI InputField得手动写 onValueChanged 回调。
4. 无反射、低 GC：底层用 Unity.Properties（[CreateProperty]），不走反射，运行时开销小。
5. 派生属性/转换器：可以绑派生属性或加 converter 做格式转换（如 100 → "100/100"），不用在数据里存冗余字段。

UI Toolkit 真正的工作流优势在于三层各自独立迭代：
* USS (Styles) → 美术/UI 改，热重载，不碰 C#
* UXML (Structure) → 布局/绑定调整，多数不碰 C#
* C# (ViewModel) → 程序改数据源，需重编译

而 UGUI 里，样式（组件字段）、结构（Prefab 层级）、刷新逻辑（C# 里的 RefreshUI）纠缠在一起，想修改外观往往还是要回到 C# 手动加刷新代码。

## UGUI 用 INotifyPropertyChanged数据绑定
INotifyPropertyChanged 本身只是一个"我变了"的通知接口，它不会自动更新任何 UI。
要在 UGUI 上复现 UI Toolkit 的体验，你需要自己写一个绑定层：

```cs 手动写这样的桥接组件，每个属性一个
public class TextBinder : MonoBehaviour
{
    public Text target;
    INotifyPropertyChanged _source;
    string _propName;

    public void Bind(INotifyPropertyChanged src, string propName)
    {
        _source = src;
        _propName = propName;
        src.PropertyChanged += OnChanged;
        UpdateNow();
    }

    void OnChanged(object s, PropertyChangedEventArgs e)
    {
        if (e.PropertyName == _propName) UpdateNow();
    }

    void UpdateNow()
    {
        // 还得用反射或委托去取值
        var val = _source.GetType().GetProperty(_propName).GetValue(_source);
        target.text = val.ToString();
    }

    void OnDestroy() { if (_source != null) _source.PropertyChanged -= OnChanged; }
}
```

也就是说，用 INotifyPropertyChanged 你需要自己解决：
* 取值/赋值：要么反射（有 GC 和性能开销），要么每个属性写委托；UI Toolkit 用 Unity.Properties 免了这层。
* 绑定注册/注销：手动管理事件订阅，注意 OnDestroy 里退订，否则内存泄漏。
* 双向绑定：还得反向监听 UI 的 onValueChanged 再写回，自己防循环更新。
* 每种控件/属性都要写对应 binder。