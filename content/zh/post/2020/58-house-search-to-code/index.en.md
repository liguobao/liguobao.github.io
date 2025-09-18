---
layout: post
title: Porting the 58.com Rental Map to ASP.NET Core
category: dotnet core
date: 2016-10-04
tags:
- asp.net core
- 58City
---

Steps to migrate the original ASP.NET MVC project to ASP.NET Core:

1) Install VS2015 + .NET Core SDK + VS2015 Tooling (Preview 2 at the time); fix common installer issues (0x80072f8a) per linked guide.

2) Create a new empty ASP.NET Core app; add NuGet packages (AngleSharp, Newtonsoft.Json, etc.).

3) Copy Controllers/Views; in Core, static files live under `wwwroot`, so move assets accordingly.

4) Configure Startup:

```csharp
public void ConfigureServices(IServiceCollection services) => services.AddMvc();
public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory lf)
{
  lf.AddConsole(); if (env.IsDevelopment()) app.UseDeveloperExceptionPage();
  app.UseStaticFiles();
  app.UseMvc(routes => {
    routes.MapRoute(name: "default", template: "{controller=House}/{action=Index}/{id?}");
  });
}
```

5) Rewrite `GetHTMLByURL` using `WebRequest` (Core changed HttpWebRequest usage) and proper encodings.

The rest is adapting namespaces/usings and resolving API differences.

