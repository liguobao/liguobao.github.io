---
layout: post
title: C#为匿名类型定义局部函数
category: dotnet
date: 2016-10-04
tags:
- dotnet
---

可能是因为我们都习惯于明确定义，一般而言,日常我们很少使用匿名对象。
然而对于实现那些短时间存在的、并不在应用程序逻辑中起支配地位的类型，使用匿名对象就是一个不错的选择了另一个方面，可能也是因为匿名类型的生命周期无法跨越包含改类型的方法，导致很多人觉得匿名对象并不好用，因为其无法在多个方法之间传递。

这样说并不准确。我们完全可以为匿名类型编写泛型方法。不过若是如此，我们便不能在泛型类型方法中处理任何特殊的元素或者编写任何的专门逻辑。
下面我们就来写一个简单的示例，示例功能：返回集合中与待查找对象相等的所有元素。

```csharp
static IEnumerable<T> FindValue(IEnumerable<T> enumerable, T value)
{
    foreach (T element in enumerable)
    {
        if (element.Equals(value))
            yield return element;
    }
}
```

这个方法是可以配合匿名类型使用的，但是这个方法本质上其实就是一个泛型方法，并不了解匿名类型的信息。


```csharp

```



```csharp

```


```csharp

```