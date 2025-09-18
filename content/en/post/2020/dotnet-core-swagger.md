---
layout: post
title: .NET Core Web API — Generate Swagger Docs with Swashbuckle
category: linux
date: 2019-07-06
tags:
- dotnet core
- swagger
---
# Swagger for ASP.NET Core Web API

Swagger/OpenAPI has become the de facto way to document REST APIs. Instead of maintaining a YAML/JSON manually, let Swashbuckle generate docs from your controllers.

## Getting Started

- Microsoft tutorial: https://docs.microsoft.com/aspnet/core/tutorials/getting-started-with-swashbuckle
- Repo: https://github.com/domaindrivendev/Swashbuckle.AspNetCore

Install packages:

```sh
dotnet add package Swashbuckle.AspNetCore --version 4.0.1
dotnet add package Swashbuckle.AspNetCore.Annotations --version 4.0.1
```

Startup configuration:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_2);
    services.AddSwaggerGen(c =>
    {
        c.SwaggerDoc("v1", new Info { Title = "XX-404 API", Version = "v1" });
        c.EnableAnnotations();
    });
}

public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    app.UseSwagger();
    app.UseSwaggerUI(c =>
    {
        c.RoutePrefix = "docs";
        c.SwaggerEndpoint("/swagger/v1/swagger.json", "Zhihu-404 API");
    });
    app.UseMvc();
}
```

Example controllers (abbrev.):

```csharp
[Route("api/v1/health")]
[ApiController]
public class HealthController : ControllerBase
{
    [HttpGet]
    public ActionResult<IEnumerable<string>> Get() => new [] { "value1", "value2" };
}
```

```csharp
[ApiController]
public class ExtendDataController : ControllerBase
{
    [HttpPost("api/v1/extend-data")]
    [SwaggerResponse(200, "", typeof(int))]
    public ActionResult AddExtendData([FromBody, SwaggerParameter("raw data")] List<DBExtendData> extendData)
    {
        if (extendData == null || extendData.Any(a => a.Id == default))
            throw new Exception("Invalid input; check each Id");
        var count = 10; // bulk insert
        return Ok(new { code = 0, data = count });
    }
}
```

Open `http://localhost:5000/docs` to view the generated UI.

