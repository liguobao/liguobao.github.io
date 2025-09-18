---
layout: post
title: C# — 58.com Branded Apartments Crawler
category: dotnet core
date: 2016-10-04
tags:
- dotnet core
---

Found a crawler tutorial on Zhihu over a weekend — “Amap API + Python to solve renting”. It hit my pain point as I was looking for a new place, so I jumped in right away. The tutorial on Shiyanlou has step-by-step instructions.

The project breaks down into two steps:

1) Use Python to crawl data and generate a data file
2) Import the data file, display listings on a map, select your workplace, and auto-calc routes and commute time

After trying the tutorial, I felt it was too rough for practical use, and it’s Beijing-only while I’m in Shanghai. You can tweak the Python data source and the front-end JS, but it still felt clunky. So I decided to build my own.

First, the original Python (excerpt):

```python
# -*- coding:utf-8 -*-
from bs4 import BeautifulSoup
from urlparse import urljoin
import requests
import csv

url = "http://bj.58.com/pinpaigongyu/pn/{page}/?minprice=2000_4000"

page = 0
csv_file = open("rent.csv","wb") 
csv_writer = csv.writer(csv_file, delimiter=',')

while True:
    page += 1
    response = requests.get(url.format(page=page))
    html = BeautifulSoup(response.text)
    house_list = html.select(".list > li")
    if not house_list:
        break
    for house in house_list:
        house_title = house.select("h2")[0].string.encode("utf8")
        house_url = urljoin(url, house.select("a")[0]["href"])
        house_info_list = house_title.split()
        if "公寓" in house_info_list[1] or "青年社区" in house_info_list[1]:
            house_location = house_info_list[0]
        else:
            house_location = house_info_list[1]
        house_money = house.select(".money")[0].select("b")[0].string.encode("utf8")
        csv_writer.writerow([house_title, house_location, house_money, house_url])

csv_file.close()
```

It scrapes `http://bj.58.com/pinpaigongyu/pn/{page}/?minprice=2000_4000`, then writes to CSV for downstream use. Each listing is an `li` element:

```html
<li>
  <a href="/pinpaigongyu/..." tongji_label="listclick">
    <div class="des">
      <h2>【合租】菊园新区 柳湖景庭 3室次卧</h2>
    </div>
    <div class="money"><span><b>1100</b>元/月</span></div>
  </a>
</li>
```

While the Python extracts from `li`, I noticed the key info actually lives in the `a[tongji_label="listclick"]` element. Rather than regex, I used HtmlAgilityPack in .NET for HTML parsing/manipulation.

Install via NuGet:

```powershell
Install-Package HtmlAgilityPack
```

Controller snippet (core logic abbreviated):

```csharp
public ActionResult Get58CityRoomData(int costFrom, int costTo, string cnName)
{
    // ... validate params ...
    var lstHouse = new List<HouseInfo>();
    string tempURL = "http://" + cnName + ".58.com/pinpaigongyu//pn/{0}/?minprice=" + costFrom + "_" + costTo;
    Uri uri = new Uri(tempURL);
    var htmlResult = HTTPHelper.GetHTMLByURL(string.Format(tempURL, 1));
    HtmlDocument htmlDoc = new HtmlDocument();
    htmlDoc.LoadHtml(htmlResult);
    var countNodes = htmlDoc.DocumentNode.SelectSingleNode(".//span[contains(@class,'list')]");
    int pageCount = 10;
    if (countNodes != null && countNodes.HasChildNodes)
    {
        pageCount = Convert.ToInt32(countNodes.ChildNodes[0].InnerText) / 20;
        if(pageCount==0) { return Json(new { IsSuccess = false, Error = "No results in this price range." }); }
    }
    for (int pageIndex = 1; pageIndex <= pageCount; pageIndex++)
    {
        htmlResult = HTTPHelper.GetHTMLByURL(string.Format(tempURL, pageIndex));
        htmlDoc.LoadHtml(htmlResult);
        var roomList = htmlDoc.DocumentNode.SelectNodes(".//a[contains(@tongji_label,'listclick')]");
        foreach (var room in roomList)
        {
            var houseTitle = room.SelectSingleNode(".//h2").InnerHtml;
            var houseURL = uri.Host + room.Attributes["href"].Value;
            var house_info_list = houseTitle.Split(' ');
            var house_location = (house_info_list[1].Contains("公寓") || house_info_list[1].Contains("青年社区"))
               ? house_info_list[0] : house_info_list[1];
            var money = room.SelectSingleNode(".//b").InnerHtml;
            lstHouse.Add(new HouseInfo { HouseTitle = houseTitle, HouseLocation = house_location, HouseURL = houseURL, Money = money });
        }
    }
    return Json(new { IsSuccess = true, HouseInfos = lstHouse });
}
```

Two key points:

- The first page contains a hidden total count like `<span class="listsum"><em>1813</em>条结果</span>`, so pages = total/20.
- Target the `a` with `tongji_label="listclick"`; inside it, `h2` has the title/location, and `b` inside `.money` has the price.

That’s the backend part. The frontend (Amap/GAODE integration) will be for another day — time to play games with my partner :-)

Source code: https://github.com/liguobao/58HouseSearch

Live: 58 Apartment Map Search (China): https://woyaozufang.live

