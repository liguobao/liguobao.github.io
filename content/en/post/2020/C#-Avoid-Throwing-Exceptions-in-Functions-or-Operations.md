---
layout: post
title: C# Avoid Throwing Exceptions in Functions or Operations
category: dotnet
date: 2016-10-04
tags:
- dotnet core
- dotnet
---
# C# Avoid Throwing Exceptions in Functions or Operations

## Introduction

In some scenarios, your function or operation needs to operate on a sequence of objects, and an exception is thrown during the process. At this time, if there is no data such as status records, we do not know how much data has been processed, nor do we know what strategy to use to roll back, so we cannot return to the previous state.

Let's look at the following code:

```csharp
var allEmp = FindAllEmployees();
allEmp.ForEach(e => e.MonthlySalary *=1.05M);
```

This code looks fine. But one day, when this program runs, an exception is thrown. The location where the exception is thrown may be unknown, causing some employees to get a raise, while others do not. The result is that except for manually checking the data, we have no way to recover the lost state.

This way of modifying elements leads to the above problem. This code does not follow the "strong exception safety guarantee" rule. In other words, when an error occurs at runtime, we cannot know what happened and what did not happen.

### Principle: If we can guarantee that the program's state will not change when the method cannot complete, such problems will not occur.

We have several methods to achieve this requirement, but each method has its own advantages and risks.

## Throwing Exceptions in Functions/Operations

Obviously, not all methods will encounter such problems (exceptions leading to state loss). Many times we just check the elements in the sequence, access them without modifying them. We don't need to be too careful about this kind of behavior. Now let's go back to the beginning, for the above scenario (giving each employee a 5% raise), if we want to follow the "strong exception safety guarantee" principle, how should we modify this method?

#### First exception: Exception when getting data

In the above example, even if the FindAllEmployees() function throws an exception, causing us to fail to correctly give employees a raise. Although this situation does not cause problems with our data, the people who should get a raise did not get it, which is a frustrating thing.

#### Solution: Rewrite the operation method given earlier with lambda expression (i.e. FindAllEmployees() method), so that it never throws an exception.

Many times, before we start modifying data, it is not very difficult to validate the legitimacy of the data and exclude erroneous data (if allowed). We can take this approach to achieve our goal. But here, we must strictly handle the operation method so that it can meet the needs in all situations.

#### Second exception: Exception in lambda expression when operating data

Also in the above example, if we throw an exception when executing the raise operation, raising the salary of those employees who have already resigned, causing the program to interrupt and lose state. In this case, filtering out resigned employees before executing the raise operation is a correct approach.

#### Solution: Validate and filter before operating data

Such as:

allEmp.Where(emp=>emp.Active).ForEach(e => e.MonthlySalary *=1.05M);

#### Third exception: Exception when executing operation

Sometimes, we cannot guarantee whether an exception will be thrown when processing. At this time, we must adopt some more expensive processing methods.

#### Solution: Create a copy to try the operation, and execute the real operation after the copy is correct

When writing this kind of code, we should consider the handling plan after throwing an exception. This means that our operation should first execute on the original data copy, and then replace the original data only after the operation succeeds.

Such as:

```C#
var updatas = (from e in allEmp 
              select new Emp
              {
                  EmpID=e.EmpID,
                  ......
                  MonthlySalary =e.MonthlySalary *=1.05M
              }).ToList();

allEmp = updatas;
```

But such modifications also cause other problems: the amount of code increases, and generating copies also consumes a lot of resources. Such an approach also has a benefit, when we encounter exceptions when operating on copy data, we have ample "space" to handle these data.

In practice, this means that we let the query expression return a new sequence instead of modifying the elements in the original sequence. In this way, we try to complete all operations at the same time, even if it fails, it will not affect the original state of our program.
