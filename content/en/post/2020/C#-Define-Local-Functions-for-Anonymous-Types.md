---
layout: post
title: C# Define Local Functions for Anonymous Types
category: dotnet
date: 2016-10-04
tags:
- dotnet
---

It may be because we are all accustomed to explicit definitions. Generally speaking, in daily life, we rarely use anonymous objects.

However, for implementing those short-lived types that do not dominate the application logic, using anonymous objects is a good choice. On the other hand, it may also be because the life cycle of anonymous types cannot span the method containing the type, which makes many people feel that anonymous objects are not easy to use because they cannot be passed between multiple methods.

This is not accurate. We can completely write generic methods for anonymous types. However, if so, we cannot handle any special elements or write any special logic in the generic type method.

Below we will write a simple example, the example function: return all elements in the collection that are equal to the object to be found.

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

This method can be used with anonymous types, but this method is essentially a generic method and does not understand the information of anonymous types.

```csharp

```

```csharp

```

```csharp

```
