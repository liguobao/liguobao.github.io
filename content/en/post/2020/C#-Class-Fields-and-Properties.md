---
layout: post
title: C# Class Fields and Properties
category: dotnet
date: 2016-10-04
tags:
- dotnet core
---
# C# Class Fields and Properties

## Fields

- Fields represent read-only or readable/writable data values.
- Fields can be static, which are considered part of the type state.
- Fields can also be instance (non-static), which are considered part of the object state. 
- It is strongly recommended to declare fields as private to prevent the type or object's state from being corrupted by external code.

## Properties

Properties allow setting or querying the logical state of a type or object using simple, field-style syntax, while ensuring the state is not corrupted.
Properties that act on types are called static properties, and those that act on objects are called instance properties.
Properties can have no parameters or multiple parameters (rare, but common in collection classes).

```csharp
	using System;
	
	public sealed class SomeType
	{                            //  1  
	// Nested class  
	   private class SomeNestedType { }                //  2  
	
	   // Constant, read­only, and static read/write field   
	   private const Int32 c_SomeConstant = 1;            //  3     
	
	   private readonly String m_SomeReadOnlyField = "2";     //  4    
	
	   private static Int32 s_SomeReadWriteField = 3;      //  5  
	
	   // Type constructor  
	   static SomeType() { }                                  //  6  
	
	   // Instance constructors  
	   public SomeType(Int32 x) { }                           //  7  
	
	   public SomeType() { }                                  //  8 
	
	   // Instance and static methods  
	   private String InstanceMethod() { return null; }       // 9   
	
	   public static void Main() { }                        // 10 
	
	   // Instance property  
	   public Int32 SomeProp
	   {                                // 11      
	       get { return 0; }                                // 12      
	       set { }                                          // 13  
	   }
	
	   // Instance parameterful property (indexer) 
	   public Int32 this[String s]
	   {                          // 14       
	       get { return 0; }                                // 15        
	       set { }                                          // 16  
	   }
	
	   // Instance event  
	   public event EventHandler SomeEvent;                  // 17  
	}
```

![enter image description here](https://kekaeq-ch3301.files.1drv.com/y3meEWWaK-o9SNfr_fT71Xr3YrPqO1LIswWqMvlHyUxWeH8P0PtXsQlfRkDnGshlMJIy2gPsxNet14efOPOuX-dHmZhCTg8PXyELJR9tnayye4LeEQ6F997b8pSI84wBR6nmmOF8IAr92oKWk36-f8alkEj9TrDQbiMoKGQwO5MTFY/20150105.jpg?psid=1)

![enter image description here](https://kekaeq-ch3301.files.1drv.com/y3mN8FseIEbAaQfT8ynEIc4nYsOUK0p0IN7Hp39imVSZXNQXlSfAYmvaC-9bDM3Hq6rPCV1XgrgvoST8wXejwARXaDmXptZpb_nyWwUzWK1rzaJ6fSsYfnP0icRKUclaVxnlltOJSQiuTFO7_fCfabmv0AsrgL5sLo6GdVJmpd-L10/QQ%E5%9B%BE%E7%89%8720151003165909.jpg?psid=1)
