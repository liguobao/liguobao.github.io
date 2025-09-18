---
layout: post
title: A Hands-On Guide to Writing a Crawler with .NET Core
category: asp.net core
date: 2016-12-04 00:00:00
tags:
- asp.net core
- crawler
---
# A Hands-On Guide to Writing a Crawler with .NET Core

## Preface

After migrating 58HouseSearch to .NET Core, I launched another side project, Dy2018Crawler, to crawl movie listings from dy2018. This post outlines the approach for building a crawler with .NET Core.

## Setup (.NET Core)

Install the .NET Core SDK (cross‑platform). With SDK installed, any editor works. For convenience, the VS .NET Core templates are fine to start with.

## Anatomy of a Crawler

### Analyze the page

Identify where the data lives in the HTML (ids, classes, attributes). For dy2018’s homepage, movie items live inside `div.co_content222`, with details in `a` elements.

Goal: find the `div.co_content222`, then extract all `a` links from within.

### Code

I use AngleSharp for HTML parsing in .NET.

- Project: https://anglesharp.github.io/
- NuGet: Install-Package AngleSharp

#### Fetch movie list

```csharp
private static HtmlParser htmlParser = new HtmlParser();
private ConcurrentDictionary<string, MovieInfo> _cdMovieInfo = new ConcurrentDictionary<string, MovieInfo>();

private void AddToHotMovieList()
{
  Task.Factory.StartNew(() =>
  {
    var htmlDoc = HTTPHelper.GetHTMLByURL("http://www.dy2018.com/");
    var dom = htmlParser.Parse(htmlDoc);
    var lstDivInfo = dom.QuerySelectorAll("div.co_content222");
    if (lstDivInfo != null)
    {
      foreach (var divInfo in lstDivInfo.Take(3))
      {
        divInfo.QuerySelectorAll("a").Where(a => a.GetAttribute("href").Contains("/i/")).ToList().ForEach(a =>
        {
          var onlineURL = "http://www.dy2018.com" + a.GetAttribute("href");
          // ... add to dictionary, etc.
        });
      }
    }
  });
}
```

#### Fetch movie details

```csharp
private MovieInfo FillMovieInfoFormWeb(AngleSharp.Dom.IElement a, string onlineURL)
{
  var movieHTML = HTTPHelper.GetHTMLByURL(onlineURL);
  var movieDoc = htmlParser.Parse(movieHTML);
  var zoom = movieDoc.GetElementById("Zoom");
  var lstDownLoadURL = movieDoc.QuerySelectorAll("[bgcolor='#fdfddf']");
  var updatetime = movieDoc.QuerySelector("span.updatetime");
  var pubDate = DateTime.Now;
  if (updatetime != null && !string.IsNullOrEmpty(updatetime.InnerHtml))
  {
    DateTime.TryParse(updatetime.InnerHtml.Replace("发布时间：", ""), out pubDate);
  }

  var movieInfo = new MovieInfo
  {
    MovieName = a.InnerHtml.Replace("<font color=\"#0c9000\">","").Replace("<font color=\"\t#0c9000\">","").Replace("</font>", ""),
    Dy2018OnlineUrl = onlineURL,
    MovieIntro = zoom != null ? WebUtility.HtmlEncode(zoom.InnerHtml) : "暂无介绍...",
    XunLeiDownLoadURLList = lstDownLoadURL?.Select(d => d.FirstElementChild.InnerHtml).ToList(),
    PubDate = pubDate,
  };
  return movieInfo;
}
```

### HTTPHelper

dy2018 uses GB2312. .NET Core needs `System.Text.Encoding.CodePages` and `Encoding.RegisterProvider(CodePagesEncodingProvider.Instance)` to support it.

```csharp
public static string GetHTMLByURL(string url)
{
  try
  {
    var wRequest = System.Net.WebRequest.Create(url);
    wRequest.ContentType = "text/html; charset=gb2312";
    wRequest.Method = "get";
    var wResp = wRequest.GetResponseAsync().Result;
    using (var reader = new StreamReader(wResp.GetResponseStream(), Encoding.GetEncoding("GB2312")))
    {
      return reader.ReadToEnd();
    }
  }
  catch { return string.Empty; }
}
```

### Scheduled jobs

Use Pomelo.AspNetCore.TimedJob for scheduled tasks.

- NuGet: Pomelo.AspNetCore.TimedJob

Register in Startup:

```csharp
services.AddTimedJob();
app.UseTimedJob();
```

Define a job:

```csharp
public class AutoGetMovieListJob : Job
{
  [Invoke(Begin = "2016-11-29 22:10", Interval = 1000 * 3600 * 3, SkipWhileExecuting = true)]
  public void Run()
  {
    // logic
  }
}
```

## Publish

Adjust `project.json` (for older tooling) — add `runtimes`, comment out `scripts` (if Node/Bower not present), and remove `type` under `Microsoft.NETCore.App`.

Build & publish:

```sh
dotnet restore
dotnet publish -r ubuntu.14.04-x64 -c Release -o "C:\\code\\website\\Dy2018Crawler"
```

Code: https://github.com/liguobao/Dy2018Crawler

Live: http://codelover.win/

