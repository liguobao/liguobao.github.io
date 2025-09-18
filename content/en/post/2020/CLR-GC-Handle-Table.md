---
layout: post
title: CLR GC Handle Table — Monitoring and Controlling Object Lifetime
category: dotnet
date: 2016-10-04
tags:
- dotnet core
- dotnet
---

The CLR provides each AppDomain with a GC Handle Table that lets applications monitor or explicitly control object lifetimes. The table is empty when the AppDomain is created.

Each entry contains:

- A reference to an object on the managed heap
- A flag describing how to monitor/control that object

You typically interact with this via `System.Runtime.InteropServices.GCHandle` (e.g., `GCHandleType.Weak`, `WeakTrackResurrection`, `Normal`, `Pinned`). Use cases include pinning objects for interop or creating weak references without allocating `WeakReference`.

Example (pinning):

```csharp
var bytes = new byte[1024];
var handle = GCHandle.Alloc(bytes, GCHandleType.Pinned);
try
{
    IntPtr ptr = handle.AddrOfPinnedObject();
    // pass ptr to native API
}
finally
{
    handle.Free();
}
```

Example (weak):

```csharp
var target = new object();
var handle = GCHandle.Alloc(target, GCHandleType.Weak);
target = null;
GC.Collect();
var alive = handle.Target != null; // may be null after collection
handle.Free();
```

Prefer high-level `WeakReference` unless you specifically need `GCHandle` semantics (e.g., pinning or resurrection tracking). Always `Free()` to avoid leaks.

