---
layout: post
title: C# Join and LINQ Join Basics
category: dotnet
date: 2016-10-04
tags:
- dotnet core
---
# Join

Quick notes on inner/left/right/outer joins — focusing on LINQ’s `Join` and `Enumerable.Join`.

`Enumerable.Join` signature:

```csharp
public static IEnumerable<TResult> Join<TOuter, TInner, TKey, TResult>(
    this IEnumerable<TOuter> outer,
    IEnumerable<TInner> inner,
    Func<TOuter, TKey> outerKeySelector,
    Func<TInner, TKey> innerKeySelector,
    Func<TOuter, TInner, TResult> resultSelector)
```

Where:

- `outer`/`inner`: sequences to join
- `outerKeySelector`/`innerKeySelector`: key selectors
- `resultSelector`: projector for the joined pair

MSDN-style example:

```csharp
var query = people.Join(
    pets,
    person => person,
    pet => pet.Owner,
    (person, pet) => new { OwnerName = person.Name, Pet = pet.Name });
foreach (var x in query)
    Console.WriteLine($"{x.OwnerName} - {x.Pet}");
```

LINQ query syntax equivalent (inner join):

```csharp
var query = from person in people
            join pet in pets on person equals pet.Owner
            select new { OwnerName = person.Name, Pet = pet.Name };
```

Left join via `GroupJoin`:

```csharp
var queryGroup = from person in people
                 join pet in pets on person equals pet.Owner into ps
                 select new { OwnerName = person.Name, Pet = ps };
```

References:
- https://msdn.microsoft.com/zh-cn/library/bb311040.aspx
- https://msdn.microsoft.com/zh-cn/library/bb534675(v=vs.110).aspx

