---
layout: post
title: 能动手绝不多说：开源评论系统remark42上手指南
categories: 
- remark
date: 2020-08-06
tags:
  - 博客
  - 开源
  - 评论系统
  - 多说
  - remark42
---


## 前言

写博客嘛，

谁不喜欢自己倒腾一下呢。

从自建系统到 Github Page，

从 [Jekyll](http://jekyllcn.com/) 到 [Hexo](https://hexo.io/zh-cn/index.html)，

年轻的时候谁不喜欢多折腾折腾呢。

年纪稍稍长了一下之后， 最后我自己还是选了 Hexo 直接做静态博客生成，

结合一下 Gitlab CI 推代码之后自动构建之后更新到自己的服务器了。

后来又基于“多说”直接支持了博客内容评论，

再后来，多说倒下了，评论功能就一直没有维护了。

前阵子因为某些需求，对市面上部分评论系统做了一次调研，

这时候发现其实大家都做了好多轮子了，近期空出来了终于可以再玩玩了。

## 开源评论系统调研

调研前提：

- 开源，协议友好，可商用
- 项目代码“正常”（实在不太想看 Ruby/PHP），维护积极
- 部署简单，二开方便

下面直接扔了上次的调研结果出来。

### vkuznecovas/mouthful

- Mouthful is a self-hosted alternative to Disqus https://mouthful.dizzy.zone/
- Golang 编写，支持 MySQL、PG、SQLIte，支持评论审核
- 维护状态：2018 年之后没看到更新了
  Features
- Multiple database support(sqlite, mysql, postgres, dynamodb)
- Moderation with an administration panel
- Server side caching to prevent excessive database calls
- Rate limiting
- Honeypot feature, to prevent bots from posting comments
- Migrations from existing commenting engines(isso, disqus)
- Configuration - most of the features can be turned on or off, as well as customized to your preferences.
- Admin login through third parties such as facebook and twitter, and 35 more.
- Dumping comments out, and importing an old dump.

### schn4ck/schnack

- Simple self-hosted node app for Disqus-like drop-in commenting on static websites https://schnack.cool/
- Node js 编写, SQLite, 没看到其他数据库的支持
- 维护状态：最后一次更新是 15 个月前
  Features
- Tiny! It takes only ~8 KB!!! to embed Schnack.
- Open source and self-hosted.
- Ad-free and Tracking-free. Schnack will not disturb your users.
- It's simpy to moderate, with a minimal and slick UI to allow/reject comments or trust/block users.
- webpush protocol to notify the site owner about new comments awaiting for moderation.
- Third party providers for authentication like Github, Twitter, Google and Facebook. Users are not required to register a new account on your system and you don't need to manage a user management system.

### adtac/commento

- A fast, bloat-free comments platform (Github mirror) https://commento.io
- golang 编写，支持 PG
- 维护状态：两个月还在更新，支持本地部署
- https://docs.commento.io/
  What features does Commento have?
  Commento comes with a lot of useful features out-of-the-box: rich text support, upvotes and downvotes, automatic spam detection, moderation tools, sticky comments, thread locking, OAuth login, email notifications, and more!

### posativ/isso

- a Disqus alternative https://posativ.org/isso/
- Python，SQLIte，引入了 ORM 组件但是暂时没看到其他数据库的支持
- Admin panel to moderate comments
- 维护状态：活跃，几天前还在更新

Isso – Ich schrei sonst – is a lightweight commenting server written in Python and JavaScript. It aims to be a drop-in replacement for Disqus.

### umputun/remark42

- comment engine https://remark42.com
- Golang + SQLIte + 支持 Admin
- 维护状态：活跃，几天前还在更新
  Remark42 is a self-hosted, lightweight, and simple (yet functional) comment engine, which doesn't spy on users. It can be embedded into blogs, articles or any other place where readers add comments.
- Social login via Google, Twitter, Facebook, GitHub and Yandex
- Login via email
- Optional anonymous access
- Multi-level nested comments with both tree and plain presentations
- Import from Disqus and WordPress
- Markdown support with friendly formatter toolbar
- Moderator can remove comments and block users
- Voting, pinning and verification system
- Sortable comments
- Images upload with drag-and-drop
- Extractor for recent comments, cross-post
- RSS for all comments and each post
- Telegram and email notifications
- Export data to json with automatic backups
- No external databases, everything embedded in a single data file
- Fully dockerized and can be deployed in a single command
- Self-contained executable can be deployed directly to Linux, Windows and MacOS
- Clean, lightweight and customizable UI with white and dark themes
- Multi-site mode from a single instance
- Integration with automatic ssl (direct and via nginx-le)
- Privacy focused

### jacobwb/hashover

- Free and Open Source PHP Comment System http://tildehash.com/?page=hashover
- PHP
- 维护状态：九个月前

## 开始实践

调研搞完之后，停了一阵子，最近把项目扔给同事做进一步 demo 搭建，

同事花了点时候搭了一下 demo 和看了代码，最后在 commento 和 remark42 中“犹豫了”

最后比较了代码结构和二次开发成本， 选择了 remark42

所以，我这边最后也使用 remark42 直接搭了自己的评论系统。

## 开始“搞事”

[remark42](https://github.com/umputun/remark42)

由于文档写得实在太好了，部署也没遇到什么奇怪问题，

部署服务端这一步真的跳过了，有奇怪的问题需要的朋友自行提问吧。

## Hexo next 集成 remark42

这一段是需要写写的。

看完了文档的朋友应该知道，

在某个页面集成评论只需要加下面这些代码。

### 页面插入 script

```php
<script>
  var remark_config = {
    host: "REMARK_URL", // hostname of remark server, same as REMARK_URL in backend config, e.g. "https://demo.remark42.com"
    site_id: "YOUR_SITE_ID",
    components: ["embed"], // optional param; which components to load. default to ["embed"]
    // to load all components define components as ['embed', 'last-comments', 'counter']
    // available component are:
    //     - 'embed': basic comments widget
    //     - 'last-comments': last comments widget, see `Last Comments` section below
    //     - 'counter': counter widget, see `Counter` section below
    url: "PAGE_URL", // optional param; if it isn't defined
    // `window.location.origin + window.location.pathname` will be used,
    //
    // Note that if you use query parameters as significant part of url
    // (the one that actually changes content on page)
    // you will have to configure url manually to keep query params, as
    // `window.location.origin + window.location.pathname` doesn't contain query params and
    // hash. For example default url for `https://example/com/example-post?id=1#hash`
    // would be `https://example/com/example-post`.
    //
    // The problem with query params is that they often contain useless params added by
    // various trackers (utm params) and doesn't have defined order, so Remark treats differently
    // all this examples:
    // https://example.com/?postid=1&date=2007-02-11
    // https://example.com/?date=2007-02-11&postid=1
    // https://example.com/?date=2007-02-11&postid=1&utm_source=google
    //
    // If you deal with query parameters make sure you pass only significant part of it
    // in well defined order
    max_shown_comments: 10, // optional param; if it isn't defined default value (15) will be used
    theme: "dark", // optional param; if it isn't defined default value ('light') will be used
    page_title: "Moving to Remark42", // optional param; if it isn't defined `document.title` will be used
    locale: "en", // set up locale and language, if it isn't defined default value ('en') will be used
  };

  (function (c) {
    for (var i = 0; i < c.length; i++) {
      var d = document,
        s = d.createElement("script");
      s.src = remark_config.host + "/web/" + c[i] + ".js";
      s.defer = true;
      (d.head || d.body).appendChild(s);
    }
  })(remark_config.components || ["embed"]);
</script>
```

### 页面插入评论框

```html
<div id="remark42"></div>
```

那么问题来了，在 hexo 的 next 主题里面怎么做呢？

答案是： 肯定是抄代码啊！！！

### hexo next 主题

首先知道（鬼知道啊），next 主题一般在项目 themes/next 路径，

themes/next/layout 这个文件夹存放了布局文件，其中\_layout.swig 是一个重要的全局布局文件。

所以，明显是修改\_layout.swig 引入上面的 script 代码啦。

看了一眼\_layout.swig 的代码, 底部基本是 include 引入同级的 swig 文件。

```js
  {% include '_scripts/boostrap.swig' %}

  {% include 'remark42.swig' %}

  {% include '_third-party/comments/index.swig' %}
  {% include '_third-party/search/index.swig' %}
```

明显，我们也可以加一个 swig，然后在上面引入一下。

### 新建 remark42.swig，贴入 script 代码

```php

<script>
  var remark_config = {
    host: "你部署的remark42 服务", // hostname of remark server, same as REMARK_URL in backend config, e.g. "https://demo.remark42.com"
    site_id: '你的站点Id,部署的时候指定的',
    components: ['embed'], // optional param; which components to load. default to ["embed"]
                           // to load all components define components as ['embed', 'last-comments', 'counter']
                           // available component are:
                           //     - 'embed': basic comments widget
                           //     - 'last-comments': last comments widget, see `Last Comments` section below
                           //     - 'counter': counter widget, see `Counter` section below
    url: window.location.origin + window.location.pathname, // optional param; if it isn't defined
                     // `window.location.origin + window.location.pathname` will be used,
                     //
                     // Note that if you use query parameters as significant part of url
                     // (the one that actually changes content on page)
                     // you will have to configure url manually to keep query params, as
                     // `window.location.origin + window.location.pathname` doesn't contain query params and
                     // hash. For example default url for `https://example/com/example-post?id=1#hash`
                     // would be `https://example/com/example-post`.
                     //
                     // The problem with query params is that they often contain useless params added by
                     // various trackers (utm params) and doesn't have defined order, so Remark treats differently
                     // all this examples:
                     // https://example.com/?postid=1&date=2007-02-11
                     // https://example.com/?date=2007-02-11&postid=1
                     // https://example.com/?date=2007-02-11&postid=1&utm_source=google
                     //
                     // If you deal with query parameters make sure you pass only significant part of it
                     // in well defined order
    max_shown_comments: 10, // optional param; if it isn't defined default value (15) will be used
    theme: 'dark', // optional param; if it isn't defined default value ('light') will be used
    page_title: 'Moving to Remark42', // optional param; if it isn't defined `document.title` will be used
    locale: 'en' // set up locale and language, if it isn't defined default value ('en') will be used
  };

  (function(c) {
    for(var i = 0; i < c.length; i++){
      var d = document, s = d.createElement('script');
      s.src = remark_config.host + '/web/' +c[i] +'.js';
      s.defer = true;
      (d.head || d.body).appendChild(s);
    }
  })(remark_config.components || ['embed']);
</script>

```

### layout.swig 添加 remark42.swig 引用

```php
  {% include '_scripts/boostrap.swig' %}

  {% include 'remark42.swig' %}

  {% include '_third-party/comments/index.swig' %}
  {% include '_third-party/search/index.swig' %}
```

搞完这个，全局脚本引用已经搞掂了。

下面就是每个文章页面需要新增 remark42 评论框了

### 明显，应该是 post.swig

观察一下 themes/next/layout 目录不难发现，

每个文章的的样式模板都在 post.swig，

明显评论框也应该直接在 post.swig。

打开一看，感觉应该加在 <div class="post-spread"> 下面

于是：


```php

<div id="posts" class="posts-expand">
  {{ post_template.render(page) }}

  <div class="post-spread">
    {% if theme.jiathis %} {% include '_partials/share/jiathis.swig' %} {%
    elseif theme.baidushare %} {% include '_partials/share/baidushare.swig' %}
    {% elseif theme.add_this_id %} {% include '_partials/share/add-this.swig' %}
    {% elseif theme.duoshuo_shortname and theme.duoshuo_share %} {% include
    '_partials/share/duoshuo_share.swig' %} {% endif %}
  </div>
  <div id="remark42"></div>
</div>
```

完事...

## 最后效果

![remark42](http://qiniu.house2048.cn/remark42-demo.png)

## 欢迎评论来玩！！
