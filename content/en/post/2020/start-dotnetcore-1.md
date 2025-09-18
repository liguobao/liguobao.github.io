---
layout: post
title: .NET Core Hands-on (Part 1 Getting Started)
category: dotnet core
date: 2018-05-29
tags:
- dotnet core
- docker
---
# .NET Core (Getting Started)

## Environment

Install the .NET SDK: https://www.microsoft.com/net/learn/get-started

- Windows: Visual Studio 2017 (select .NET Core workload)
- macOS/Linux: SDK + VS Code (+ C# extension)

Verify:

```sh
dotnet --version
dotnet help
```

## Create a project

List templates: `dotnet new`

Create and run a console app:

```sh
dotnet new console -n FirstApplication
cd FirstApplication
dotnet run
```

Open in VS Code, install the C# extension, and debug (set breakpoints, F5/F10/F11).

Example loop:

```csharp
for (var i = 0; i < 10; i++)
{
    sum += i;
    Console.WriteLine($"i:{i}, sum:{sum}");
}
```

## About .csproj

Console:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>netcoreapp2.0</TargetFramework>
  </PropertyGroup>
</Project>
```

Web projects use `Microsoft.NET.Sdk.Web` and `PackageReference` entries for dependencies (e.g., ASP.NET Core, Dapper, Newtonsoft.Json). More details in the next part.

