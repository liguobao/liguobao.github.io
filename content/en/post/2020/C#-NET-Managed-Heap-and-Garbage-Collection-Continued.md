---
layout: post
title: C#.NET Managed Heap and Garbage Collection (Continued)
category: dotnet
date: 2016-10-04
tags:
- dotnet
---
# Managed Heap and Garbage Collection (Continued)

## Large Objects

CLR divides objects into large objects and small objects. Currently, objects of 85000 bytes or larger are considered large objects. CLR treats large objects differently.

1. Large objects are not allocated in the address space of small objects, but in other places in the process address space.

2. The current version of GC does not "compress" large objects because moving them in memory is too costly. But this may cause address space fragmentation between large objects in the process, leading to OutOfMemoryException. Future versions of CLR may compress large objects.

3. Large objects are always generation 2, never generation 0 or generation 1. So only create large objects for resources that need to live long. Allocating short-lived large objects will cause generation 2 to be recycled more frequently, losing performance. Large objects are generally large strings (XML/JSON) or byte arrays used for I/O operations (reading bytes from files/network into buffers for processing).

## Garbage Collection Modes

When CLR starts, it selects a GC mode, and the mode in the process will not change before.

There are two basic GC modes.

### Workstation

- This mode optimizes GC for client applications. The delay caused by GC is very low, and the application thread suspension time is short, avoiding user anxiety. In this mode, GC assumes that other applications running on the machine do not consume too much CPU resources.

### Server

- This mode optimizes GC for server applications. What is optimized is mainly throughput and resource utilization. GC assumes that no other applications are running on the machine (whether client or server applications), and assumes that all CPUs on the machine can be used to assist in completing GC. In this mode, the managed heap is split into several regions, one for each CPU. When starting garbage collection, the garbage collector runs a special thread on each CPU; each thread recycles its own region concurrently with other threads. For server applications where worker threads behave consistently, concurrent recycling can work well. This feature requires the application to run on a multi-CPU computer, so that threads can truly work simultaneously, thereby gaining performance improvement.

Applications run in "workstation" GC mode by default. Server applications hosting CLR (such as ASP.NET) can request CLR to load server GC. But if the application runs on a single-processor computer, CLR always uses "workstation" GC mode.

Standalone applications can create a configuration file to tell CLR to use the server recycler. The configuration file needs to add the gcServer element for the application. Below is an example configuration file:

```XML
<configuration>
       <runtime>
             <gcServer enabled="true">
      </runtime>
</configuration>
```

You can use the read-only Boolean property IsServerGC of the GCSettings class to get whether CLR is in "server" GC mode.

In addition to these two modes, GC also supports two sub-modes: concurrent (default) or non-concurrent.

In concurrent mode, the garbage collector has an additional background thread that can mark objects concurrently while the application is running. While the program is running, the garbage collector runs a normal priority background thread to find unreachable objects. After finding them, the garbage collector suspends all threads again, determines whether to "compress" memory. If it decides to compress, memory is compressed, root references are corrected, and application threads resume running. This garbage collection takes less time than usual because the unreachable object collection has been constructed. But the garbage collector may also decide not to compress memory; in fact, the garbage collector tends not to compress. The more available memory, the less likely the garbage collector is to compress the heap; this helps enhance performance, but increases the program's working set. Using concurrent garbage collector, applications usually consume more memory than using non-concurrent garbage collector.
