---
layout: post
title: CodeSmith Templates for MySQL — Multi-Table Generation
category: CodeSmith
date: 2016-10-04
tags:
- dotnet
---

CodeSmith is a capable code generator on .NET. I used to target SQL Server with a set of templates for Model/DAL/BLL. After switching to MySQL, I found fewer resources, so I adapted templates and noted the process here.

Workflow: copy the template folder into CodeSmith’s template path, open CodeSmith, right‑click `Main.cst` → Execute → select the MySQL connection → choose tables → Generate.

Problem: the original `Main.cst` used `SchemaExplorer.TableSchema` so you could only pick a single table. Solution: change it to `SchemaExplorer.TableSchemaCollection` and loop.

Snippet:

```csharp
<%@ Property Name="SourceTables" Type="SchemaExplorer.TableSchemaCollection" %>
<%@ Register Name="SE" Template="CreatSingleTable.cst" %>

<% foreach(TableSchema ts in SourceTables) {
     SE s = new SE();
     s.SourceTable = ts;
     s.RootNamespace = RootNamespace;
     s.OutputDirectory = OutputDirectory;
     s.Render(this.Response);
} %>
```

This generates Model/DAL/BLL for all selected tables in one go.

Field comments: the default MySQL provider `SchemaExplorer.MySQLSchemaProvider.dll` doesn’t populate `column.Description`. Replace the provider with a patched version to read column comments:

How‑to (Chinese): http://www.cnblogs.com/LonelyShadow/p/4147743.html

Template repo: https://github.com/liguobao/CodeSmith-for-MySQL-Template

Notes:

- The template trims the first three chars of table names (prefix like `tbl_` recommended), or adjust the template.
- Backup and replace `SchemaExplorer.MySQLSchemaProvider.dll` to enable column comments.

