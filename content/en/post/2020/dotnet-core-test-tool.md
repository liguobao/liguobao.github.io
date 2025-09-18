---
layout: post
title: .NET Core Unit Test Coverage with coverlet + ReportGenerator
category: dotnet core
date: 2019-04-08
tags:
- dotnet core
- UnitTest
- ReportGenerator
- coverlet
---

# Test Coverage with coverlet + ReportGenerator

Need coverage for .NET Core? Use coverlet to collect, and ReportGenerator to render HTML.

## coverlet

Install either as a global tool or as an MSBuild package:

```sh
dotnet tool install --global coverlet.console
```

or add to your test project:

```xml
<PackageReference Include="coverlet.msbuild" Version="2.5.0" />
```

Then run tests with coverage:

```sh
dotnet test \
  /p:CollectCoverage=true \
  /p:CoverletOutput='./results/' \
  /p:CoverletOutputFormat=opencover
```

Outputs `coverage.opencover.xml` under `results/`.

## ReportGenerator

Render human‑readable reports from XML:

```sh
dotnet tool install --global dotnet-reportgenerator-globaltool
reportgenerator '-reports:UnitTests/results/*.xml' '-targetdir:UnitTests/results'
```

Open `UnitTests/results/index.htm` to view the coverage report.

