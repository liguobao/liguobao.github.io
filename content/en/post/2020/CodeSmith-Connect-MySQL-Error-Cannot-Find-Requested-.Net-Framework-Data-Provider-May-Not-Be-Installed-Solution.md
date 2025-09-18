---
layout: post
title: CodeSmith Connect MySQL Database Error "Can't Find .NET Framework Data Provider"
category: MySQL
date: 2016-10-04
tags:
- dotnet
---

1. Download mysql-connector-net installation

[mysql-connector-net](https://dev.mysql.com/downloads/connector/net/6.9.html)

2. After mysql-connector-net installation is complete, go to the corresponding installation directory and copy the corresponding MySQL .NET dll to CodeSmith's bin directory and SchemaProviders directory.

The general DLL directory is:

C:\\Program Files (x86)\\MySQL\\MySQL Connector Net 6.9.8\\Assemblies\\v4.0

3. Restart CodeSmith to take effect

<br>
<br>

Other solutions:
<br>
[Solution for codesmith unable to connect to Mysql](http://blog.csdn.net/joke01/article/details/9469515)

[Solution for codesmith6.5 connecting to Mysql prompting "Cannot find the requested .Net Framework Data Provider. It may not be installed."](http://www.cnblogs.com/tim190/archive/2013/01/18/2866161.html)
