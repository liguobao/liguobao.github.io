---
layout: post
title: 可能是一份没什么用的爬虫代理IP指南
category: ipproxy
date: 2016-10-04 00:00:00
tags:
- Java
- 开源
---
##写在前面

做爬虫的小伙伴一般都绕不过代理IP这个问题.

PS:如果还没遇到被封IP的场景，要不就是你量太小人家懒得理你，要不就是人家压根不在乎...

爬虫用户自己是没有能力维护一系列的代理服务器和代理IP的，这个成本实在有点高了。
所以公用代理服务器应运而生,现在几大云服务商家都提供代理IP服务，一般论个买...

同时网上也有很多代理IP共享网站，会把一些免费的代理IP放出来给大家用。
大家都是做爬虫的，那么，是不是可以先把代理IP网站的数据爬一遍？
所以可以看到不少的爬代理IP的爬虫，如[突破反爬虫的利器——开源IP代理池](https://zhuanlan.zhihu.com/p/23928595)之类的项目。
这些项目都能达到抓取代理IP数据的目的，很多时候也够用了。

然而在使用过程中我们发现了一些问题：
1. 网站公布的代理IP不一定是可用的。可能代理服务器挂了，可能IP无效了...等等之类的。
2. 代理IP是部分可用的。某代理IP可用代理访问百度，但是代理访问谷歌的时候就GG了。
3. 代理连通性是好的，但是已经被访问站识别为代理IP返回验证码或者辣鸡数据。


## proxyipcenter的解决方案

出于以上种种的原因，按耐不住的dalao(@[virjar](https://www.zhihu.com/people/virjar))开始自己动手撸了。

于是，[proxyipcenter](http://git.oschina.net/virjar/proxyipcenter)出来了。

这个项目做了什么东西呢？
dalao是这样写的：
>
>DungProxy是一个代理IP服务,他包括一个代理IP资源server端和一系列适配中心IP资源得客户端。
>
>server负责代理IP资源的收集维护。
>
>client则是一系列方便用户使用得API,他屏蔽了代理IP下载、代理IP选取、IP绑定、IP切换等比较复杂逻辑。用户只需要引入client即可方便使用代理IP服务

简单来说：

server是一个代理IP爬虫，同时负责简单的数据清洗。

client本质是一个代理IP池，获取数据源之后，基于需要代理访问的网站在本地维护一个代理IP池供爬虫使用。

以下内容转载自[http://git.oschina.net/virjar/proxyipcenter](http://git.oschina.net/virjar/proxyipcenter/edit/master/doc/server/deploy/README.md),
并已获得相关授权。(PS:dalao写的太好了，我都懒得重新写一遍了...问过dalao之后直接转载过来算了。)

## proxyipcenter server

DrungProxy的代理IP都是从互联网收集,他是架设在一个高度不可用的资源上面的服务。server会负责对这些资源进行清洗、校验、打分,最终输出可以被客户端使用的IP资源。IP资源从入库到最终判定可用生命流程如下
1. IP抓取
    server监听了很多代理IP网站,这些网站包括国内外十几家,有意思的是drungProxy的IP爬虫是一系列网站模版。五六行配置即可实现一个简单的网站模版,然后我们有一个上层调度模块将会负责调起模版进行数据抓取。
2. IP消重
需要消重的原因是程序运行到一定时间之后,大量IP都是数据库里面已经存在的了,这个时候如果在数据库进行消重逻辑将会导致大量数据库读写,实际上我们的服务器是一个1块钱的腾讯云(曾经是),看起来是撑不住这么大的请求的(平均每天可以有10K量级)。最后在入库前设置了一个bloomFilter消重模块,能够高效的检测资源是否被入库过。
3. 位置信息完善
这个逻辑不大,通过taobaoIp接口获取地址信息,完善IP资源元数据。[taobaoIP](http://ip.taobao.com/)
4. IP验证
IP验证分为好几个步骤。我们的IP总资源有80W,检验一个IP是否可用一般来说需要20秒左右的时间,因为代理IP本身响应比较慢,我们会把超时时间设置得比较长。所以可以计算一下80W数据走一轮将要消耗得时间,即使在多线程并行环境下时间也是很多的。为了在一定资源下完成校验,我们设计了如下步骤
- 端口开启校验,在进行可用性校验前,首先需要检查IP端口是否开启。调研发现大量资源其实端口都不通,所以专门设计一个任务验证端口是否开启,端口开启验证超时时间为5秒。由于大多数资源端口都没有开启,所以大部分资源的校验时间下降到5秒了。
- 可用性校验,进行可用性校验的需要先进行端口开启校验,系统中端口开启的资源大概3W,所以校验可用性的总资源有3W左右。可用性校验存在如下问题,很多代理IP其实不是代理网站,想他发送请求最终不是我们预期的数据,比如他返回给我们一个代理IP认证网页。所以我们不能根据是否能够请求到数据来判定IP是否可用。我们的做法是在公网放置一个API接口,然后控制代理IP访问我们自己的接口,如果能够拿到符合我们接口的预期数据,那么认为IP可用。
- domain可用该校验,可用性校验通过之后IP还不是真正可用,悲伤的发现代理IP是和域名相关的。所以同一个IP在不同域名下表现可能不一样。所以我们维护了一个域名IP池,这里面存储各个域名下可用IP
5. IP分发
IP分发是根据客户请求分配可用IP。分发逻辑现在还没有完全完善,但是已经实现了最迫切和有校的分发方案。分发逻辑设计是:先尝试查询domainIP池,再根据其他请求参数做条件匹配,再查询系统可用IP,再随机选择可用填充。四个步骤如果有一个步骤得到的IP超过请求参数期待数目,则不进行接下来的动作。

### IP验证模型
再IP验证的时候,我们设计了一个模型用来确定哪些IP应该优先验证。模型描述如下:长期可用IP检测频率低,长期不可用IP检测评率低。不稳定IP和刚加入的IP检测频率高。我们使用优先队列来实现这个逻辑,所有IP根据分值放在不同优先队列中,每次校验的时候再不同优先队列中拿出一定资源进行校验(不同优先级拿出的资源数目不一样,高优先级的对象拿出更多资源),对于同一个优先队列,我们根据最后验证时间排序。使上次更新时间最久的资源被优先选择。
### 分发去重
分发资源的时候,设计去重问题,也就是根据相同条件,每次分发得到的IP很大可能会重复。为了规避这个问题,每次分发都会相应的下发一个资源签名,他会记录分发过的IP。在下次请求的时候,客户端需要带上这个签名,服务器会根据签名过滤,同时会重新对新分发的IP资源做再次签名.

## server部署
server端使用java编写,使用maven管理项目,使用mysql作为数据库。相关技术包括springMVC,spring,tomcat,mybatis,guava,fastjson,httpclient等。
运行server的方式很简单
1. 在项目根目录执行maven命令(需要提前安装maven,maven安装方式略)```mvn install -Dmaven.test.skip=true```
2. 在server目录执行maven命令 ```mvn tomcat7:run```

### server配置
直接运行项目使用的是我们的默认数据库,同时使用的是默认配置。实际上server存在一些配置用来设置运行参数。合理的运行参数能够合理使用机器资源以及达到更好的运行效果。
项目主要有两个配置文件需要配置:
1. mysql.properties 用来配置数据库信息
2. config.properties 配置其他启动参数,主要需要关注里面几个url地址,还有 system.thread.*的参数项。system.thread*用于指定某一种类型的任务执行的线程数,如果数据小于1,则这个模块不会启动。但是如果这个模块接收到了任务请求,那么他会转发到其他服务器上面(也就是上面的两个forward相关的url,没办法服务器都是腊鸡服务器  )

其他的应该没有了把,哦对了,项目存在多个profile,也就是resources.local,resources.beta,resources.prod等。他们叫做profile,是maven里面的概念,默认是resources.local生效的。如果想使用其他profile下面的配置,则增加 -Pprofile参数,如运行server ```mvn -Pskyee clean tomcat7:run```

### server接口事例
[http://115.159.40.202:8080/proxyipcenter/av?usedSign=&checkUrl=http%3A%2F%2Ffree-proxy-list.net%2F&domain=free-proxy-list.net&num=10](http://115.159.40.202:8080/proxyipcenter/av?usedSign=&checkUrl=http%3A%2F%2Ffree-proxy-list.net%2F&domain=free-proxy-list.net&num=10)
````
{
     "data": {
         "data": [
             {
                 "id": 257,
                 "ip": "203.192.12.148",
                 "proxyIp": "203.192.12.149",
                 "port": 80,
                 "ipValue": 3418360980,
                 "country": "中国",
                 "area": "华北",
                 "region": "北京市",
                 "city": "北京市",
                 "isp": "",
                 "countryId": "CN",
                 "areaId": "100000",
                 "regionId": "110000",
                 "cityId": "110100",
                 "ispId": "-1",
                 "transperent": 2,
                 "speed": 104,
                 "type": 1,
                 "connectionScore": 1310,
                 "availbelScore": 8,
                 "connectionScoreDate": 1475641264000,
                 "availbelScoreDate": 1475646860000,
                 "createtime": 1473840886000,
                 "lostheader": false
             },
             {
                 "id": 654,
                 "ip": "120.55.245.47",
                 "proxyIp": "112.124.119.21",
                 "port": 80,
                 "ipValue": 2016933167,
                 "country": "中国",
                 "area": "华东",
                 "region": "浙江省",
                 "city": "杭州市",
                 "isp": "阿里云",
                 "countryId": "CN",
                 "areaId": "300000",
                 "regionId": "330000",
                 "cityId": "330100",
                 "ispId": "1000323",
                 "transperent": 2,
                 "speed": 83,
                 "type": 1,
                 "connectionScore": 1429,
                 "availbelScore": 2,
                 "connectionScoreDate": 1475659905000,
                 "availbelScoreDate": 1475630273000,
                 "createtime": 1473840884000,
                 "lostheader": false
             },
             {
                 "id": 2489,
                 "ip": "124.193.33.233",
                 "proxyIp": "124.193.33.233",
                 "port": 3128,
                 "ipValue": 2093031913,
                 "country": "中国",
                 "area": "华北",
                 "region": "北京市",
                 "city": "北京市",
                 "isp": "鹏博士",
                 "countryId": "CN",
                 "areaId": "100000",
                 "regionId": "110000",
                 "cityId": "110100",
                 "ispId": "1000143",
                 "transperent": 2,
                 "speed": 3390,
                 "type": 1,
                 "connectionScore": 310,
                 "availbelScore": 2,
                 "connectionScoreDate": 1475657685000,
                 "availbelScoreDate": 1475661878000,
                 "createtime": 1473839334000,
                 "lostheader": false
             },
             {
                 "id": 5004,
                 "ip": "203.192.12.146",
                 "proxyIp": "203.192.12.149",
                 "port": 80,
                 "ipValue": 3418360978,
                 "country": "中国",
                 "area": "华北",
                 "region": "北京市",
                 "city": "北京市",
                 "isp": "",
                 "countryId": "CN",
                 "areaId": "100000",
                 "regionId": "110000",
                 "cityId": "110100",
                 "ispId": "-1",
                 "transperent": 2,
                 "speed": 161,
                 "type": 1,
                 "connectionScore": 1291,
                 "availbelScore": 10,
                 "connectionScoreDate": 1475638336000,
                 "availbelScoreDate": 1475636727000,
                 "createtime": 1473840882000,
                 "lostheader": false
             },
             {
                 "id": 5421,
                 "ip": "221.237.155.64",
                 "proxyIp": "221.237.155.64",
                 "port": 9797,
                 "ipValue": 3723336512,
                 "country": "中国",
                 "area": "西南",
                 "region": "四川省",
                 "city": "成都市",
                 "isp": "电信",
                 "countryId": "CN",
                 "areaId": "500000",
                 "regionId": "510000",
                 "cityId": "510100",
                 "ispId": "100017",
                 "transperent": 2,
                 "speed": 3238,
                 "type": 1,
                 "connectionScore": 119,
                 "availbelScore": -1,
                 "connectionScoreDate": 1475611973000,
                 "availbelScoreDate": 1475629954000,
                 "createtime": 1473840773000,
                 "lostheader": false
             },
             {
                 "id": 8722,
                 "ip": "58.243.0.162",
                 "proxyIp": "58.243.0.162",
                 "port": 9999,
                 "ipValue": 989003938,
                 "country": "中国",
                 "area": "华东",
                 "region": "安徽省",
                 "city": "安庆市",
                 "isp": "联通",
                 "countryId": "CN",
                 "areaId": "300000",
                 "regionId": "340000",
                 "cityId": "340800",
                 "ispId": "100026",
                 "transperent": 2,
                 "speed": 5143,
                 "type": 1,
                 "connectionScore": 154,
                 "availbelScore": -3,
                 "connectionScoreDate": 1475665673000,
                 "availbelScoreDate": 1475614147000,
                 "createtime": 1473839836000,
                 "lostheader": false
             },
             {
                 "id": 11698,
                 "ip": "218.7.170.190",
                 "proxyIp": "218.7.170.190",
                 "port": 3128,
                 "ipValue": 3657935550,
                 "country": "中国",
                 "area": "东北",
                 "region": "黑龙江省",
                 "city": "绥化市",
                 "isp": "联通",
                 "countryId": "CN",
                 "areaId": "200000",
                 "regionId": "230000",
                 "cityId": "231200",
                 "ispId": "100026",
                 "transperent": 2,
                 "speed": 3145,
                 "type": 1,
                 "connectionScore": 317,
                 "availbelScore": -1,
                 "connectionScoreDate": 1475642001000,
                 "availbelScoreDate": 1475524810000,
                 "createtime": 1473839128000,
                 "lostheader": false
             },
             {
                 "id": 13318,
                 "ip": "220.249.185.178",
                 "proxyIp": "220.249.185.178",
                 "port": 9999,
                 "ipValue": 3707353522,
                 "country": "中国",
                 "area": "华东",
                 "region": "福建省",
                 "city": "福州市",
                 "isp": "联通",
                 "countryId": "CN",
                 "areaId": "300000",
                 "regionId": "350000",
                 "cityId": "350100",
                 "ispId": "100026",
                 "transperent": 2,
                 "speed": 5094,
                 "type": 1,
                 "connectionScore": 129,
                 "availbelScore": -1,
                 "connectionScoreDate": 1475615670000,
                 "availbelScoreDate": 1475585178000,
                 "createtime": 1473840539000,
                 "lostheader": false
             },
             {
                 "id": 57033,
                 "ip": "210.245.25.228",
                 "proxyIp": "210.245.25.228",
                 "port": 3128,
                 "ipValue": 3539278308,
                 "country": "越南",
                 "area": "",
                 "region": "",
                 "city": "",
                 "isp": "",
                 "countryId": "VN",
                 "areaId": "",
                 "regionId": "",
                 "cityId": "",
                 "ispId": "",
                 "transperent": 2,
                 "speed": 1024,
                 "type": 1,
                 "connectionScore": 488,
                 "availbelScore": 36,
                 "connectionScoreDate": 1475635386000,
                 "availbelScoreDate": 1475630473000,
                 "createtime": 1473836572000,
                 "lostheader": false
             },
             {
                 "id": 124334,
                 "ip": "60.194.72.253",
                 "proxyIp": "60.194.72.253",
                 "port": 3128,
                 "ipValue": 1019365629,
                 "country": "中国",
                 "area": "华北",
                 "region": "北京市",
                 "city": "北京市",
                 "isp": "鹏博士",
                 "countryId": "CN",
                 "areaId": "100000",
                 "regionId": "110000",
                 "cityId": "110100",
                 "ispId": "1000143",
                 "transperent": 2,
                 "speed": 2366,
                 "type": 1,
                 "connectionScore": 610,
                 "availbelScore": 16,
                 "connectionScoreDate": 1475643516000,
                 "availbelScoreDate": 1475631080000,
                 "createtime": 1473839561000,
                 "lostheader": false
             }
         ],
         "num": 10,
         "sign": "9999#C99+999#9B99B99999##Y9999+9999999999999999999999t9999s99999999s9999999999999999999999999999#99999999999999GB999999999G9999s9s99999#9999999999Y9+999##99999999+99999999999999+999999999999B999+Y9999G9+99999999999YB99999999999999999999999+99Y999999999B9999G999s99G999999999#99999#9Y999s999999999#B99999999999999999999+999999Y9999999Y9999999999999Y9999Y999999999999999"
     },
     "status": true
 }
````




## proxyipcenter client
### [client 设计文档](http://git.oschina.net/virjar/proxyipcenter/blob/master/doc/client/design/README.md)
### [client 使用文档](http://git.oschina.net/virjar/proxyipcenter/blob/master/doc/client/userGuide/README.md)

client运行原理
这里讲述IP池的设计相关,如果您仅仅是为了使用dunproxy-client,则不必关心本文内容

client就是一个代理IP池的实现,IP池的设计基于两个点:

1. 代理IP都是不稳定的,不可靠的,需要一个机制来切换IP,尽可能使用高质量IP。

2. IP和环境关系很大,同一个IP在不同的机器下访问不同的目标网站,其可用性表现都是不一样的

## 使用IpPoolAPI获取IP资源

### 获取IP
获取一个IP的方式是这样的 ``IpPool.getInstance().bind(domain, accessUrl);`` 
- 第一个参数是域名,可以传递null,传递null提取accessUrl schema里面的host
- 第二个参数是你当前需要访问的url, 可以为null,为null时domain不能为空

曾经有一个绑定用户的功能,使得同一个账户每次获取的IP相同,后来觉得是过度设计,因为对于抓取场景,切换IP是很普通的需求,而且貌似没有多少server会检查常用IP。就算有也是小众需求,本框架不必支持

### 记录IP使用
每当使用IP的时候,需要记录一次IP使用,也就是将IP实例的使用次数加一,用于打分机制计算IP的使用分值,评估IP可用性
方式如下:``com.virjar.dungproxy.client.model.AvProxy.recordUsage``

### 记录IP使用失败
每当IP使用失败的时候,需要记录IP使用失败,也就是将IP实例的失败次数加一,用于打分机制计算IP的使用分值,评估IP可用性
方式如下:``com.virjar.dungproxy.client.model.AvProxy.recordFailed()``

### IP下线
IP下线很简单,拿到IP实例,这样调用``com.virjar.dungproxy.client.model.AvProxy.offline()``。
一般情况不建议这么做,因为IP池会自动检查IP是否应该下线,IP池可以定制各种策略。当时,有些时候IP池的检查机制比较缓慢,而上层业务可以明确知道本IP不可用,这个时候可以使用本API强制下线


### 销毁IP池实例
IP池是单例的,同时里面维护了两个任务线程,在业务完成的时候,需要销毁IP池才能终止内部线程。同时也会执行一些收尾工作,如将可用IP dump。
销毁方式是:``com.virjar.dungproxy.client.ippool.IpPool.destroy``


### 其他
IP池有其他很多扩展点,但是目前接口没有开发完成,待后续完善




## TODO 几个主流语言的客户端

1. .NET/.NET core client (辣鸡楼主还没动手...)
2. Python client(另外一个大佬好像忘了这事了...)
3. Golang client(可能还是辣鸡楼主动手...)








