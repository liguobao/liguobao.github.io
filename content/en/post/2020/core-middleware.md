---
layout: post
title: ASP.NET Core Middleware — Concepts, Order, and Internals
category: dotnet core
date: 2016-10-04
tags:
- dotnet core
---

In classic ASP.NET, you might have touched HttpModule/HttpHandler for certain cross‑cutting needs. ASP.NET Core replaces them with Middleware and embraces a pipeline model — you’ll use middleware a lot.

## What is middleware?

In `Startup.Configure` you compose the pipeline:

```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
{
  loggerFactory.AddConsole(Configuration.GetSection("Logging"));
  loggerFactory.AddDebug();

  app.UseStaticFiles();

  app.UseMvc(routes =>
  {
    routes.MapRoute("default", "{controller=Home}/{action=Index}/{id?}");
  });
}
```

Each `Use...` adds a middleware. `UseStaticFiles` serves static files; `UseMvc` enables MVC routing. Remove `UseMvc` and controller routes won’t work.

## Differences from HttpModule

HttpModule used a fixed set of lifecycle events; you had to choose carefully where to plug in. Middleware is just code you write that receives `HttpContext`, can act, and then choose to call the next middleware.

## Flow

The engine calls the first middleware’s `Invoke(HttpContext)`, which can do work then call the next middleware, and so on. On the way back up the stack, you can also perform post‑processing.

## Write a middleware

```csharp
public class SampleMiddleware
{
  private readonly RequestDelegate _next;
  public SampleMiddleware(RequestDelegate next) => _next = next;

  public async Task Invoke(HttpContext context)
  {
    if (string.IsNullOrEmpty(context.User.Identity.Name))
    {
      context.Response.Redirect("/NoName.html");
      return;
    }
    await _next(context);
  }
}
```

Register it:

```csharp
public static class MiddlewareExtensions
{
  public static IApplicationBuilder UseSampleMiddleware(this IApplicationBuilder builder)
    => builder.UseMiddleware<SampleMiddleware>();
}

// Startup.Configure
app.UseStaticFiles();
app.UseSampleMiddleware();
app.UseMvc(...);
```

## Order matters

Place `UseSampleMiddleware` before `UseMvc` so redirects happen before MVC routing. If you put it before `UseStaticFiles` and try to redirect to a static page, it will fail because static file serving hasn’t been added yet.

## Internals (high level)

`RequestDelegate` represents a middleware delegate. `IApplicationBuilder.Use(...)` chains delegates into a stack; `UseMiddleware<T>` validates your type (has a single `Invoke(HttpContext)`) and builds the delegate. When the host starts, the pipeline is built once (Build/Run) and reused per request.

