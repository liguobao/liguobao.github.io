---
layout: post
title: 手把手教你写dotnet core(入门篇)
category: dotnet core
date: 2018-05-29
tags:
- dotnet core
- docker

---
# dotnet core(入门篇)

## 开发环境准备

dotnet core最低开发环境要求就是一个[.NET SDK](https://www.microsoft.com/net/learn/get-started/),在这里可以下载的到最新版本的SDK,各个平台都有.

理论上有了SDK什么事都能做了.

安装SDK的步骤参考上面的连接就OK,这部分我们跳过.

简单讲一下不同操作系统的开发工具选择.

- Windows平台下首选Visual Studio 2017,安装的时候选择 .NET Core部分即可,安装下来估计占用磁盘空间5G,同时会帮你装好SDK的,好用,很好用.

- MacOS/Linux平台选择 SDK + Visual Studio Code + Debug插件 + Nuget插件,很不错,完全生产级别

- 备选方案 Jetbrains家的rider,暂时没用过,resharper一直好评如潮

今天我这边主要是是以VS Code + Debug插件 + 命令行来进来讲解.

装好dotnet core SDK之后,打开命令行界面,输入dotnet看看.

- Windows中为CMD或者Powershell,MacOS/Linux为终端

```log

Usage: dotnet [options]
Usage: dotnet [path-to-application]

Options:
  -h|--help            Display help.
  --version         Display version.

path-to-application:
```

再输入dotnet --version查看一下当前版本,我这边显示2.1.4.

输入dotnet help

```log
.NET Command Line Tools (2.1.4)
Usage: dotnet [runtime-options] [path-to-application]
Usage: dotnet [sdk-options] [command] [arguments] [command-options]

path-to-application:
  The path to an application .dll file to execute.

SDK commands:
  new              Initialize .NET projects.
  restore          Restore dependencies specified in the .NET project.
  run              Compiles and immediately executes a .NET project.
  build            Builds a .NET project.
  publish          Publishes a .NET project for deployment (including the runtime).
  test             Runs unit tests using the test runner specified in the project.
  pack             Creates a NuGet package.
  migrate          Migrates a project.json based project to a msbuild based project.
  clean            Clean build output(s).
  sln              Modify solution (SLN) files.
  add              Add reference to the project.
  remove           Remove reference from the project.
  list             List reference in the project.
  nuget            Provides additional NuGet commands.
  msbuild          Runs Microsoft Build Engine (MSBuild).
  vstest           Runs Microsoft Test Execution Command Line Tool.

Common options:
  -v|--verbosity        Set the verbosity level of the command. Allowed values are q[uiet], m[inimal], n[ormal], d[etailed], and diag[nostic].
  -h|--help             Show help.
```

有类似的这些信息,说明我们的SDK安装以及完成了.

Visual Studio 和Visual Studio Code的安装就不多说了.

## 创建 dotnet core程序

我这边只有SDK + VS Code环境,创建程序直接使用命令行了.

dotnet core SDK中已经有很多现成的APP模板,我们直接使用dotnet new命令就可以创建对应的程序.

命令行输入 " dotnet new ", 显示如下:

```log
Usage: new [options]

Options:
  -h, --help          Displays help for this command.
  -l, --list          Lists templates containing the specified name. If no name is specified, lists all templates.
  -n, --name          The name for the output being created. If no name is specified, the name of the current directory is used.
  -o, --output        Location to place the generated output.
  -i, --install       Installs a source or a template pack.
  -u, --uninstall     Uninstalls a source or a template pack.
  --type              Filters templates based on available types. Predefined values are "project", "item" or "other".
  --force             Forces content to be generated even if it would change existing files.
  -lang, --language   Specifies the language of the template to create.


Templates                                         Short Name       Language          Tags
--------------------------------------------------------------------------------------------------------
Console Application                               console          [C#], F#, VB      Common/Console
Class library                                     classlib         [C#], F#, VB      Common/Library
Unit Test Project                                 mstest           [C#], F#, VB      Test/MSTest
xUnit Test Project                                xunit            [C#], F#, VB      Test/xUnit
ASP.NET Core Empty                                web              [C#], F#          Web/Empty
ASP.NET Core Web App (Model-View-Controller)      mvc              [C#], F#          Web/MVC
ASP.NET Core Web App                              razor            [C#]              Web/MVC/Razor Pages
ASP.NET Core with Angular                         angular          [C#]              Web/MVC/SPA
ASP.NET Core with React.js                        react            [C#]              Web/MVC/SPA
ASP.NET Core with React.js and Redux              reactredux       [C#]              Web/MVC/SPA
ASP.NET Core Web API                              webapi           [C#], F#          Web/WebAPI
global.json file                                  globaljson                         Config
NuGet Config                                      nugetconfig                        Config
Web Config                                        webconfig                          Config
Solution File                                     sln                                Solution
Razor Page                                        page                               Web/ASP.NET
MVC ViewImports                                   viewimports                        Web/ASP.NET
MVC ViewStart                                     viewstart                          Web/ASP.NET

Examples:
    dotnet new mvc --auth Individual
    dotnet new viewimports --namespace
    dotnet new --help
```

既然是手把手教程,肯定从最原始的Console Application 开始咯,在命令行中输入命令"dotnet new console -n FirstApplication",创建一个名为FirstApplication的命令行程序

```sh
dotnet new console -n FirstApplication;
cd FirstApplication;
ls FirstApplication;
#✗ cd FirstApplication
# FirstApplication git:(master) ✗ ls
# FirstApplication.csproj  Program.cs  obj/
```

我们切换到FirstApplication文件中,可以看到现在已经有三个文件.简单讲解一下:

- FirstApplication.csproj .csproj为项目构建文件(C Sharp Project”),对应maven中的pom.xml或者是gradle中的build.gradle

- Program.cs 为程序的主入口, 有一个静态的Main方法

- obj用于存放编译过程中生成的中间临时文件,一般不用管

我们使用VS Code打开这个文件夹看看.

首次在VS Code中打开带有.csproj文件的文件夹,VS Code会提示是否需要安装相关插件,直接选择是即可.

我们这里要用到的插件主要是"C# for Visual Studio Code (powered by OmniSharp)",直接在插件仓库搜C#基本就能看到.

直接看下Program.cs的代码:

![代码1](http://qiniu.house2048.cn/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-05-29%20%E4%B8%8B%E5%8D%8810.51.02.png)

一句话输出"Hello World!"...

我们试着运行一下看看.

有两种方式:

- 直接在对应项目文件夹位置的命令行中执行dotner run;

- VS Code debug启动

### dotnet run

"VS Code-查看-集成终端"可以直接调出终端,并且切到当前项目文件路径,执行 dotnet run输出如下:

```log
➜  FirstApplication git:(master) ✗ dotnet run
Hello World!
```

### VS Code debug

VS Code左侧切到debug(一只虫子的图标),点击调试旁边的绿色按钮开始启动.

![debug](http://qiniu.house2048.cn/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-05-29%20%E4%B8%8B%E5%8D%8810.57.46.png)

终端输出:

![终端](http://qiniu.house2048.cn/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-05-29%20%E4%B8%8B%E5%8D%8810.59.51.png)

调试控制台输出:

![控制台](http://qiniu.house2048.cn/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-05-29%20%E4%B8%8B%E5%8D%8811.00.51.png)

都说了Debug了,我们简单也做个debug断点调试.

点击代码文件左侧黑色边栏,鼠标左键单击在第8,9行,对应位置出现断点(小红点),
如下图:
![断点](http://qiniu.house2048.cn/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-05-29%20%E4%B8%8B%E5%8D%8811.02.09.png)

再次Debug运行程序.

第8行位置出现黄色条纹,程序处于debug默认等待下一步操作.

![debug](http://qiniu.house2048.cn/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-05-29%20%E4%B8%8B%E5%8D%8811.04.41.png)

左侧可查看相关变量当前值,正上方有debug相关操作(F5继续,F10单步跳过,F11单步调试...)

F5按一下,黄色条纹往下走一步到第9行(上一步也下了断点).此时尚未输出任何的信息.

F5下一步执行Console.WriteLine("Hello World!");输出"Hello World!"

### 写个累加程序试试水

```C#
using System;

namespace FirstApplication
{
    class Program
    {
        static void Main(string[] args)
        {
            var sum = 0;
            for (var i = 0; i < 10; i++)
            {
                sum = sum + i;
                Console.WriteLine($"当前i值为:{i},sum:{sum}");
            }
        }
    }
}

/*
Loaded '/usr/local/share/dotnet/shared/Microsoft.NETCore.App/2.0.5/System.Runtime.Extensions.dll'. Skipped loading symbols. Module is optimized and the debugger option 'Just My Code' is enabled.
当前i值为:0,sum:0
当前i值为:1,sum:1
当前i值为:2,sum:3
当前i值为:3,sum:6
当前i值为:4,sum:10
当前i值为:5,sum:15
当前i值为:6,sum:21
当前i值为:7,sum:28
当前i值为:8,sum:36
当前i值为:9,sum:45
The program '[32086] FirstApplication.dll' has exited with code 0 (0x0).
*/
```

在循环里面打个断点看看i的值和sum的值.

![i+sum](http://qiniu.house2048.cn/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-05-29%20%E4%B8%8B%E5%8D%8811.17.06.png)

鼠标移动到对应变量上.
![鼠标移上去](http://qiniu.house2048.cn/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202018-05-29%20%E4%B8%8B%E5%8D%8811.17.49.png)

到这里,第一个dotnet core程序基本已经完成了,本教程结束....

## 骗你的,这里还有

还记得我们上面看到的FirstApplication.csproj吗?

我们直接在VS Code中打开看看.(VS用户随便找个文件编辑器打开)

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>netcoreapp2.0</TargetFramework>
  </PropertyGroup>

</Project>

```

这里就指定了SDK版本和程序版本,没有其他多余的东西了.

暂时没什么看的,我们找个web项目的来看看.

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>netcoreapp2.0</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <Folder Include="wwwroot\"/>
  </ItemGroup>
  <ItemGroup>
    <PackageReference Include="Dapper" Version="1.50.4"/>
    <PackageReference Include="Microsoft.ApplicationInsights.AspNetCore" Version="2.2.1"/>
    <PackageReference Include="Microsoft.AspNetCore" Version="2.0.2"/>
    <PackageReference Include="Microsoft.AspNetCore.Mvc" Version="2.0.3"/>
    <PackageReference Include="Microsoft.AspNetCore.StaticFiles" Version="2.0.2"/>
    <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="2.0.2"/>
    <PackageReference Include="Microsoft.Extensions.Logging.Debug" Version="2.0.1"/>
    <PackageReference Include="Microsoft.VisualStudio.Web.BrowserLink" Version="2.0.2"/>
    <PackageReference Include="Microsoft.VisualStudio.Web.CodeGeneration.Design" Version="2.0.3"/>
    <PackageReference Include="Newtonsoft.Json" Version="11.0.2"/>
    <PackageReference Include="StackExchange.Redis.StrongName" Version="1.2.6"/>
    <PackageReference Include="System.Diagnostics.Tools" Version="4.3.0"/>
    <PackageReference Include="System.Text.Encoding.CodePages" Version="4.4.0"/>
    <PackageReference Include="System.ValueTuple" Version="*"/>
    <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="2.1.0-rc1-final"/>
    <PackageReference Include="Pomelo.AspNetCore.TimedJob" Version="2.0.0-rtm-10046"/>
    <PackageReference Include="MySqlConnector" Version="0.40.3"/>
    <PackageReference Include="MongoDB.Driver.Core" Version="2.7.0-beta0001"/>
    <PackageReference Include="MongoDB.Driver" Version="2.7.0-beta0001"/>
  </ItemGroup>
</Project>
```

这边能看的东西就很多了.

- Project Sdk="Microsoft.NET.Sdk.Web" SDk为Web

- Folder Include="wwwroot\" 包含 wwwroot静态文件

- PackageReference Include="Dapper" Version="1.50.4" 引用了Dapper程序包(一个ORM框架)

- PackageReference Include="Microsoft.AspNetCore.Mvc" Version="2.0.3" 引用了MVC框架

- PackageReference Include="Newtonsoft.Json" Version="11.0.2" 引用了Newtonsoft.Json Json库

这里我们先看看,具体内容在下一讲asp.net core 入门我们会详细讲解.

全文完....
