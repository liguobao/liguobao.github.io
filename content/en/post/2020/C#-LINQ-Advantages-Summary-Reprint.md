---
layout: post
title: C# LINQ Advantages Summary (Reprint)
category: dotnet core
date: 2016-10-04
tags:
- dotnet core
---
# LINQ Advantages Summary (Reprint)

Original link: http://www.cnblogs.com/c-jquery-linq-sql-net-problem/archive/2011/01/15/LINQ_Merit.html

I've been reading a book on LINQ called 'Essential LINQ', and here I share it with everyone.

Since a deep summary of LINQ requires a lot of space, I'll divide it into several parts here.

(*I read the English version of 'Essential LINQ', please forgive me if some terms cannot be translated into proper Chinese explanations)

Advantages of LINQ:

LINQ basically has the following seven advantages, let me illustrate them one by one:

## Integrated: The so-called Integrated (integration), LINQ embodies integration from the following aspects:

(1): Integrate query syntax into languages like C#(VB), making it a syntax. This way, it can support the same as other syntax in C#:

Statement highlighting, type checking, allowing debugger debugging

(2): Integrate and encapsulate the previous complex pre-query work, allowing developers to focus on the query.

(3): The integrated syntax is clearer and more readable.

```csharp
Comparison:
//Original format
SqlConnection sqlConn = new SqlConnection(connectionString);>
sqlConn.Open();
SqlCommand command = new SqlCommand();
command.Connection = sqlConn;
command.CommandText = "Select * From Customer";
SqlDataReader dataReader = command.ExecuteReader(CommandBehavior.CloseConnection);
//LINQ format
NORTHWNDDataContext dc = new NORTHWNDDataContext();
var query = from c in dc.Customers
            select c;
```

## Unitive: The so-called Unitive (unification) means using unified query syntax for any type of external and internal data sources (object collections, xml, database data).

The benefits of using a unified query language are as follows:

You don't have to spend a lot of effort learning unfamiliar data sources, you can quickly and simply use LINQ syntax to query them.

Since unified syntax is used, code maintenance becomes simpler.

The following code embodies LINQ's unification:

```csharp
//Data source: object collection
var query = from c in GetCustomers()
            select c;

//Data source: SQL
var query1 = from c in dc.Customers
             select c;
//Data source: XML
var query2 = from c in customers.Descendants("Customer")
             select c;
```

## Extensible: The so-called Extensible (extensibility) refers to the following 2 aspects:

(1). Extension of queryable data sources. LINQ provides a LINQ provider model, you can create or provide providers for LINQ to support more data sources.

(2). Extensible query methods. Developers can rewrite and extend query methods for LINQ according to their own needs.

Here are some third-party LINQ providers:

LINQ Extender, LINQ to JavaScript, LINQ to JSON, LINQ to MySQL, LINQ to Flickr, LINQ to Google

## Declarative: The so-called Declarative (declarative), simply put, means that developers only tell the program what to do, and the program judges how to do it.

The advantages of Declarative programming are reflected in the following 2 points:

(1). Improve development speed. Because developers don't need to write a lot of code to concretize execution steps, just tell the program what to do.

(2). Improve code optimization space. Because developers don't interfere with the specific steps of program execution, this provides more space for the compiler to optimize the code.

For example, in SQL, LINQ-generated SQL statements are often better than SQL statements written by developers with average SQL skills.

Compare Declarative programming with Imperative programming:

```csharp
//Declarative programming
List<List<int>> lists = new List<List<int>> { new List<int> { 1, 2, 3 }, new List<int> { 4, 5 } };
var query = from list in lists
            from num in list
            where num % 3 == 0
            orderby num descending
            select num;

//Imperative programming
List<int> list1 = new List<int>();
list1.Add(1);
list1.Add(2);
list1.Add(3);
List<int> list2 = new List<int>();
list2.Add(4);
list2.Add(5);
List<List<int>> lists1 = new List<List<int>>();
lists1.Add(list1);
lists1.Add(list2);

List<int> newList = new List<int>();
foreach (var item in lists1)
      foreach (var num in item)
        if (num % 3 == 0)
            newList.Add(num);
newList.Reverse();

```

## Hierarchical: The so-called Hierarchical (hierarchical) refers to abstracting data in an object-oriented way.

SQL is a relational database, it describes data and data relationships in a relational way, but our programs are designed to be object-oriented, so the database data we get in the program are often rectangular grid (planar display data). But LINQ converts relational to object-to-object description of data through the so-called O-R Mapping method.

The benefits are: Developers can directly operate data in an object way, and for developers accustomed to object-oriented, the object-oriented model is easier to understand.

## Composable: The so-called Composable (composable) means that LINQ can split a complex query into multiple simple queries.

The results returned by LINQ are all based on the interface: IEnumerable<T>, so you can continue querying the query results, and LINQ has the characteristic of deferred execution, so splitting execution will not affect efficiency.

The advantages are:

(1). Convenient debugging. Split complex queries into simple queries, then debug them one by one.

(2). Easy code maintenance. Splitting the code can make the code easier to understand.

The following code embodies composability:

```csharp
//The following code embodies Composable
List<List<int>> lists = new List<List<int>> { new List<int> { 1, 2, 3 }, new List<int> { 4, 5 } };

var query1 = from list in lists
             from num in list
             select num;

var query2 = from num in query1
             where num % 3 == 0
             select num;

var query3 = from num in query2
             orderby num descending
             select num;
```

## Transformative: The so-called Transformative (transformative) means that LINQ can convert the content of one data source to other data sources.

Convenient for users to do data migration.

The following code embodies the transformation feature:

```csharp
//Convert relational data to XML type
	var query = new XElement("Orders",
            from c in dc.Customers
            where c.City == "Paris"
            select new XElement("Order",
                new XAttribute("Address", c.Address)));
```

The above are the main advantages of LINQ, I'm glad to share here with everyone. If there are any shortcomings, please supplement and correct, thank you for visiting the hut.

//2011/1/28 Supplement (LINQ TO SQL)

In terms of LINQ TO SQL, if you use LINQ TO SQL, you can effectively prevent SQL injection, LINQ TO SQL will treat the injected code as useless parameters.

http://www.cnblogs.com/c-jquery-linq-sql-net-problem/archive/2011/01/15/LINQ_Merit.html
