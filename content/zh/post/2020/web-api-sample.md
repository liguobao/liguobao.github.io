---
layout: post
title: 后端API入门学习指北
category: API
date: 2018-06-18
tags:
- RESTful API
- Spring Boot
- ASP.NET Core
- PHP
- Python
- Node.js
---

# 后端API入门学习指北

了解一下一下概念.

## RESTful API标准]

所有的API都遵循[RESTful API标准].

建议大家都简单了解一下HTTP协议和RESTful API相关资料.

1. [阮一峰:理解RESTful架构](http://www.ruanyifeng.com/blog/2011/09/restful.html?spm=a2c4e.11153940.blogcont533536.9.8aa6538f8as8Dc)

2. [阮一峰:RESTful API 设计指南](http://www.ruanyifeng.com/blog/2014/05/restful_api.html)

3. [RESTful API指南](https://github.com/aisuhua/restful-api-design-references)

## 依赖注入 DI

1. [浅谈依赖注入](http://www.cnblogs.com/yangecnu/p/Introduce-Dependency-Injection.html)

2. [阮一峰:软件架构入门](http://www.ruanyifeng.com/blog/2016/09/software-architecture.html)

## Java版

- JDK版本:1.8 +

- 集成开发环境: IDEA [https://www.jetbrains.com/idea/](https://www.jetbrains.com/idea/)

- 数据库:MySQL 5.7+

- 内存数据库:Redis

- 数据库访问框架: mybatis + groovy脚本(PS:如果自己熟悉JPA也可以用)

- 构建工具: maven(自己熟悉gradle的话也可以用)

Java框架直接上Spring Boot + Spring MVC.

### 资料链接

1. [IBM:Spring 框架简介](https://www.ibm.com/developerworks/cn/java/wa-spring1/index.html)

2. [IBM:Maven 让事情变得简单](https://www.ibm.com/developerworks/cn/java/j-maven/index.html)

3. [Spring MVC快速入门教程](https://www.tianmaying.com/tutorial/spring-mvc-quickstart)

4. [IBM:Spring Boot 基础](https://www.ibm.com/developerworks/cn/java/j-spring-boot-basics-perry/index.html)

5. [Spring Boot——开发新一代Spring Java应用](https://www.tianmaying.com/tutorial/spring-boot-overview)

6. [Building an Application with Spring Boot](https://spring.io/guides/gs/spring-boot/)

7. [MyBatis入门实例：整合Spring MVC与MyBatis开发问答网站](https://www.tianmaying.com/tutorial/mybatis-spring)

8. [mybatis 官网](https://www.mybatis.org/mybatis-3/zh/index.html)

### Java入门目标

使用Spring boot 搭建Web API,通过Web API对数据增删查改.

## C#版

- .NET版本: dotnet core 2.0

- 集成开发环境: Visual Studio Code + dotnet core SDK 或者 Visual Studio 2017(推荐使用  Visual Studio Code)

- 数据库:MySQL 5.7+

- 内存数据库:Redis

- 数据库访问框架: Dapper

dotnet core 直接使用dotnet core mvc框架即可,依赖注入直接使用原生框架.

### 入门资料链接

1. [手把手教你写dotnet core(入门篇)](https://zhuanlan.zhihu.com/p/37460329)

2. [手把手教你ASP.NET Core](https://zhuanlan.zhihu.com/p/37464924)

3. [微软:NET Core 教程](https://docs.microsoft.com/zh-cn/dotnet/core/tutorials/)

4. [ASP.NET Core 中文文档 第一章 入门](http://www.cnblogs.com/dotNETCoreSG/p/aspnetcore-1-getting_started.html)

5. [Dapper 使用教程](http://www.cnblogs.com/Sinte-Beuve/p/4231053.html)

6. [Dapper Github](https://github.com/StackExchange/Dapper)

### C#入门目标

使用ASP.NET Core搭建Web API,通过Web API对数据增删查改.

## Python版

- Python版本:3.6.5

- 集成开发环境: Visual Studio Code + Python debug插件 或者 pycharm

- 数据库:MySQL 5.7+

- 内存数据库:Redis

- 数据库访问框架: sqlalchemy

Python使用flask框架搭建Web API

### 入门到放弃资料

1. [知乎-李辉:Hello, Flask!](https://zhuanlan.zhihu.com/flask)

2. [廖雪峰:Python教程](https://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000)

3. [菜鸟教程:Python3基础](http://www.runoob.com/python3/python3-tutorial.html)

4. [SQLAlchemy ORM教程](https://www.jianshu.com/p/0d234e14b5d3)

5. [实验楼:SQLAlchemy 基础教程](https://www.shiyanlou.com/courses/724)

6. [知乎-猪了个去:SQLAlchemy入门和进阶](https://zhuanlan.zhihu.com/p/27400862)

### Python入门目标

使用Python flask搭建Web API,通过Web API对数据增删查改.

## PHP版本

真有人选择这个?拖出去打死算了吧...

- PHP版本: 7.1 +

- 集成开发环境:  Visual Studio Code + PHP debug插件 + nginx + php-fpm 

- 数据库:MySQL 5.7+

- 内存数据库:Redis

- 数据库访问框架: 忘了,回头补

- 构建工具:composer

### 入门到拍黄片

1. [Laravel-简洁、优雅的PHP开发框架(PHP Web Framework)](https://docs.golaravel.com/docs/5.6/installation/)

2. [laravel 中文教程](https://lvwenhan.com/laravel/432.html)

### 拍黄片入门目标

使用laravel 搭建Web API,通过Web API对数据增删查改.

## node.js 版

- node.js版本:9.0+

- 集成开发环境: Visual Studio Code

- 数据库:MySQL 5.7+

- 内存数据库:Redis

- 数据库访问框架: sequelize 或者orm2

- 构建工具:npm

### node.js入门资料链接

1. [Express:基于 Node.js 平台，快速、开放、极简的 web 开发框架。](http://www.expressjs.com.cn/)

2. [菜鸟教程:Node.js Express 框架](http://www.runoob.com/nodejs/nodejs-express-framework.html)

3. [sequelizejs](http://docs.sequelizejs.com/)

4. [Sequelize 中文手册](https://haibinpark.gitbooks.io/sequelize-zh-manual/content/)

### node.js入门目标

使用Express 搭建Web API,通过Web API对数据增删查改.

























没了,纯粹占行用的...


拜.
