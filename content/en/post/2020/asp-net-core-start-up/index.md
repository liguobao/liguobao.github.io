---
layout: post
title: ASP.NET Core Hosting — Startup Options
category: dotnet core
date: 2016-10-04
tags:
- dotnet core
---
# ASP.NET Core Hosting — Startup Options

In classic ASP.NET, apps depended on IIS to start. IIS registered an ISAPI filter and launched `w3wp` to run the ASP.NET runtime. Pre-.NET Core, Windows + IIS was the default. Although Linux options like jexus (via mono) existed, they weren’t as robust as Microsoft’s native stack.

With .NET Core, ASP.NET Core became cross-platform, so its hosting model was redesigned.

## 1) Kestrel and IIS (Http Platform Handler)

ASP.NET Core rewrote the runtime. It runs like a console app and ships with a high-performance I/O server: Kestrel. Kestrel handles HTTP but doesn’t provide IIS-like process management and protection. If you still host behind IIS, you need a “bridge” — historically the Http Platform Handler — configured in `web.config` to start your ASP.NET Core app and forward requests.

Example:

```xml
<system.webServer>
  <handlers>
    <add name="httpPlatformHandler" path="*" verb="*"
         modules="httpPlatformHandler" resourceType="Unspecified"/>
  </handlers>
  <httpPlatform processPath="WebApp.exe" arguments="" 
                stdoutLogEnabled="false" startupTimeLimit="3600"/>
</system.webServer>
```

In this setup, IIS accepts the HTTP request, forwards it to your app (e.g., `WebApp.exe`), your app boots Kestrel, and the request flows through ASP.NET Core. IIS acts as a simple reverse proxy. Note: newer IIS integration modules supersede Http Platform Handler.

## 2) Main()

Like other .NET apps, ASP.NET Core starts from `static void Main()`. Example:

```csharp
public static void Main(string[] args)
{
    var host = new WebHostBuilder()
        .UseServer("Microsoft.AspNetCore.Server.Kestrel")
        .UseContentRoot(Directory.GetCurrentDirectory())
        .UseDefaultConfiguration(args)
        .UseIISPlatformHandlerUrl()
        .UseStartup<Startup>()
        .Build();

    host.Run();
}
```

Key calls:

- `UseServer(...)`: choose the server (e.g., Kestrel). You can implement your own if it conforms to `IServerFactory`.
- `UseContentRoot(...)`: set the working directory (defaults to the app’s folder).
- `UseDefaultConfiguration(args)`: provide hosting config via args (app key, environment, server factory location, content root). These can also come from `appsettings.json`.

## 3) Startup

`UseStartup<Startup>()` designates the Startup class, typically with:

```csharp
public class Startup
{
    public IConfigurationRoot Configuration { get; }

    public Startup(IHostingEnvironment env)
    {
        var builder = new ConfigurationBuilder()
            .SetBasePath(env.ContentRootPath)
            .AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
            .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true)
            .AddEnvironmentVariables();
        Configuration = builder.Build();
    }

    public void ConfigureServices(IServiceCollection services)
    {
        services.AddEntityFramework()
                .AddDbContext<BlogsContext>(o => o.UseSqlServer(Configuration["Data1:DefaultConnection:ConnectionString"]));

        services.AddMvc();
    }

    public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
    {
        loggerFactory.AddConsole(Configuration.GetSection("Logging"));
        loggerFactory.AddDebug();

        app.UseStaticFiles();
        app.UseMvcWithDefaultRoute();
    }
}
```

- `ConfigureServices` registers services (EF, MVC, etc.).
- `Configure` wires middleware and their order.

## 4) Build and Run

- `Build()`: constructs the hosting service, combines services/middleware/content root/app name/configuration, and initializes the host engine.
- `Run()`: starts the host and hooks `CancelKeyPress` so you can stop it with `Ctrl+C`.

Original (Chinese): ASP.NET Core 的启动方式 (Hosting)
https://dotblogs.com.tw/aspnetshare/2016/03/28/20160327

