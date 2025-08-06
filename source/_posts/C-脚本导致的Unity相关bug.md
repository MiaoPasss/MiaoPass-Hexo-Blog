---
title: C#脚本导致的Unity相关bug
date: 2025-02-14 13:14:58
categories: Game Engine
tags: 
- Unity
---

---

# 问题：属性(Property)和访问器(Accessor)

如果在当前属性中直接添加自定义get和set访问器，在Unity引擎中访问/修改此属性时游戏会闪退。

```cs
public class TestClass : MonoBehaviour
{
    public int TestProperty {
        get { return TestProperty;}
        set {
            TestProperty = value;
        }
    }
}
```

## 解决方案

1. 直接使用默认访问器

```cs
public class StartPage : MonoBehaviour
{
    public int TestProperty {
        get;
        set;
    }
}
```

2. 创建一个新属性专门用于访问

```cs
public class StartPage : MonoBehaviour
{
    private int testProperty;

    public int TestProperty {
        get { return testProperty;}
        set {
            testProperty = value;
        }
    }
}
```

---

# 问题：Start函数中使用不当loop循环

如果在start函数中使用while循环处理资源加载进度条，Unity引擎会在importing asset弹窗中卡死

```cs
public class TestClass : MonoBehaviour
{
    [SerializeField] private Progressbar loadingBar;
    [SerializeField] private AssetLabelReference gameImageAssets;
    
    void Start() {
        AsyncOperationHandle asyncHandle = Addressables.LoadAssetsAsync<Sprite>(gameImageAssets, _ => {});
        asyncHandle.Completed += EnterGame;
        StartCoroutine(GameLoadingCoroutine(asyncHandle));
    }

    private IEnumerator GameLoadingCoroutine(AsyncOperationHandle handle) {
        // yield return Addressables.LoadAssetsAsync<Sprite>(gameImageAssets, _ => {});

        while (!handle.IsDone && handle.PercentComplete < 1f)
            loadingBar.SetProgressPercent(handle.PercentComplete);
    }

    private void EnterGame() {
        // Disable Loading bar
        // Enter gameplay
    }
}
```

## 解决方案

使用协程(Coroutine)将while操作与Unity生命周期同步

```cs
public class TestClass : MonoBehaviour
{
    [SerializeField] private Progressbar loadingBar;
    [SerializeField] private AssetLabelReference gameImageAssets;
    
    void Start() {
        AsyncOperationHandle asyncHandle = Addressables.LoadAssetsAsync<Sprite>(gameImageAssets, _ => {});
        StartCoroutine(GameLoadingCoroutine(asyncHandle));
    }

    private IEnumerator GameLoadingCoroutine(AsyncOperationHandle handle) {
        while (!handle.IsDone && handle.PercentComplete < 1f) {
            loadingBar.SetProgressPercent(handle.PercentComplete);
            yield return null;
        }

        if (handle.Status == AsyncOperationStatus.Succeeded)
            EnterGame();
        else
            Debug.LogError($"Failed to load game asset sprites: {handle.Status}");
    }

    private void EnterGame() {
        // Disable Loading bar
        // Enter gameplay
    }
}
```

---

# 问题：在Awake/Start函数中修改GameObject位置

在Awake/Start函数中GameObject的position component尚未初始化完成，此时修改位置将不会起作用

```cs
private void Start() {
    var newPos = transform.localPosition;
    newPos.y += 100;
    
    transform.localPosition = newPos;
}
```

## 解决方案

1. 在Start函数中等待一帧后再修改位置，此时GameObject的position component可使用

```cs
private IEnumerator Start() {
    yield return null;
    
    var newPos = transform.localPosition;
    newPos.y += 100;
    
    transform.localPosition = newPos;
}
```

2. 在Awake/Start函数之后的函数进行修改位置操作

---

# 问题：使用dotween移动layout group中的游戏对象会影响其他游戏对象

## 举例：一个老虎机有5个转轴Reel[] reels，移动reels[2]会将reels[0]和reels[1]的position清零

---