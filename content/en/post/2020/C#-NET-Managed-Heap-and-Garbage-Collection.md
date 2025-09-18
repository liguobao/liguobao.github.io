---
layout: post
title: C#.NET Managed Heap and Garbage Collection
category: dotnet
date: 2016-10-04
tags:
- dotnet core
---
# .NET Managed Heap and Garbage Collection

## Basics of Managed Heap

Brief description: Every program needs to use resources of one kind or another, including files, memory buffers, screen space, network connections..... In fact, in an object-oriented environment, each type represents a resource available to the program. To use these resources, memory must be allocated for the type representing the resource.

The following are the steps required to access a resource:

1. Call the IL instruction newobj to allocate memory for the type representing the resource. (C# new operator)
2. Initialize the memory to set the initial state of the resource. (Generally refers to the constructor)
3. Access the type's members to use the resource. (Use member variables, methods, properties, etc.)
4. Destroy the resource's state for cleanup. (Dispose???)
5. Free the memory. (GC)

## Allocating Resources from the Managed Heap

CLR requires all objects to be allocated from the managed heap.

When the process initializes, CLR allocates an address space region as the managed heap. CLR also maintains a pointer, let's call it NextObjPtr, which points to the allocation position of the next object in the heap. At the beginning, NextObjPtr is set to the base address of the address space region.

When a region is filled with non-garbage objects, CLR allocates more regions.

This process repeats until the entire process address space is filled. So, application memory is limited by the process's virtual address space.

32-bit processes can allocate up to 1.5GB, 64-bit processes can allocate up to 8T.

Note: Related materials on process memory size

[Memory Support and Windows Operating Systems](https://msdn.microsoft.com/zh-cn/library/windows/hardware/Dn613959(v=vs.85).aspx)

[Process Address Space](https://msdn.microsoft.com/zh-cn/library/ms189334.aspx)

[Maximum memory available for C/C++ programs in 32-bit mode](http://blog.csdn.net/yusiguyuan/article/details/12405799)

## The new operator in C# causes CLR to perform the following operations:

1. Calculate the number of bytes required for the type's fields (and fields inherited from base types).

2. Add the bytes required for object overhead. Each object has two overhead fields: type object pointer and sync block index. For 32-bit applications, these two fields each require 32 bits, so each object needs to add 8 bytes. For 64-bit applications, these two fields each require 64 bits, so each object adds 16 bytes.

3. CLR checks if there are enough bytes in the region to allocate the object. If the managed heap has enough available space, the object is placed at the address pointed to by the NetxObjPtr pointer, and the bytes allocated for the object are zeroed. Then the type's constructor is called (passing NextObjPtr as the this parameter), the new operator returns the object reference. Just before returning the object reference, the value of the NextObjPtr pointer is increased by the number of bytes occupied by the object to get a new value, which is the address where the next object will be placed in the managed heap. As shown in the figure:

![tup](http://r.photo.store.qq.com/psb?/4d3e65a5-4593-42bc-88f9-7bbb2e647ebe/i3rlSCPAcnT9pL0El0BptPIBpuvnxHpBw9Nkp*UqIjw!/o/dJMAAAAAAAAA&ek=1&kp=1&pt=0&bo=LwKNAC8CjQADACU!&su=1199793361&sce=0-12-12&rf=2-9)

### Garbage Collection Algorithm

#### CLR uses reference tracking algorithm.

Reference tracking algorithm only cares about reference type variables, because only this type of variable can reference objects on the heap;

Value type variables directly contain value type instances. Reference type variables can be used in many situations, including static and instance fields of classes, or parameters and local variables of methods. Here we call all reference type variables roots.

When CLR starts GC, it first suspends all threads. (This prevents threads from accessing objects and changing their state during CLR inspection.) Then CLR enters the GC marking phase. In this phase, CLR traverses all objects in the heap, setting a bit in the sync block index field to 0. This indicates that all objects should be deleted. Then, CLR checks all active roots to see which objects they reference. This is why CLR's GC is called reference tracking GC. If a root contains null, CLR ignores this root and continues to check the next root.

The figure below shows a heap containing several objects.

![图片1](http://r.photo.store.qq.com/psb?/4d3e65a5-4593-42bc-88f9-7bbb2e647ebe/eVBVeXGrNAfoWfyRgl4aC2RRSGgiDpmbrocv4lTSJMA!/o/dJIAAAAAAAAA&ek=1&kp=1&pt=0&bo=gAIFAYACBQEDACU!&su=1176931729&sce=0-12-12&rf=2-9)

The application's roots directly reference objects A, C, D, F. All objects have been marked. When marking object D, GC finds that this object contains a field referencing object H, causing object H to also be marked. The marking process continues until all roots of the application have been checked.

After checking, the objects in the heap are either marked or unmarked. Marked objects cannot be garbage collected because at least one root is referencing them. We say that such objects are reachable because the application can reach them through variables referencing them. Unmarked objects are unreachable because there is no root in the application that can make the object accessible again.

After CLR knows which objects can survive and which can be deleted, it enters the GC compression (similar to defragmentation) phase. In the compression phase, CLR "moves" the marked objects in the heap to organize all surviving objects to occupy contiguous memory.

The benefits of doing this are:

1. All surviving objects are next to each other in memory, restoring the "locality" of references, reducing the application's working set, thereby improving performance when accessing these objects in the future;

2. After defragmentation, the available space is also contiguous, liberating the entire address space segment, allowing other things to reside.

After moving objects in memory, there is a problem that needs to be solved urgently. The roots referencing surviving objects now reference the original location of the object in memory, not the moved location. When suspended threads resume execution, they will access the old memory location, causing memory corruption. This is obviously intolerable, so as part of the compression phase, CLR also subtracts the number of bytes the referenced object is offset in memory from each root. This ensures that each root still references the same object as before, only the object's position in memory has changed.

As shown in the figure:

![123](http://r.photo.store.qq.com/psb?/4d3e65a5-4593-42bc-88f9-7bbb2e647ebe/FyP2yk1O6kMsq3.u4e4x3qrAxpwbajgSHOd4QHTJOhE!/o/dJIAAAAAAAAA&ek=1&kp=1&pt=0&bo=TQI*AU0CPwEDACU!&su=1202148209&sce=0-12-12&rf=2-9)

## Generations: Improving Performance (To be continued)

CLR's GC is a generational garbage collector, it makes the following assumptions about your code:

1. The newer the object, the shorter the lifetime.

2. The older the object, the longer the lifetime.

3. Recycling part of the heap is faster than recycling the entire heap.

Extensive research shows that these assumptions hold for most applications today, and they affect the implementation of the garbage collector. Here we will explain how generations work.

The managed heap does not include objects when initialized. Objects added to the heap become generation 0 objects. Simply put, generation 0 objects are those newly constructed objects that the garbage collector has never checked. As shown in the figure, a newly started application has allocated 5 objects (A to E). After a while, C and E become unreachable.

![23](http://r.photo.store.qq.com/psb?/4d3e65a5-4593-42bc-88f9-7bbb2e647ebe/77WJus7lssJpEJ2RZREQoNx.5CL31HLdboJbAgCqS0E!/o/dJMAAAAAAAAA&ek=1&kp=1&pt=0&bo=tQIVAbUCFQEDACU!&su=172682065&sce=0-12-12&rf=2-9)

CLR initializes generation 0 objects by selecting a budget capacity. If allocating a new object causes generation 0 to exceed the budget, a GC must be started. Assuming objects A to E just fill generation 0's space, allocating object F must start GC. After GC, surviving objects become generation 1 objects. As shown in the figure:

![123](http://r.photo.store.qq.com/psb?/4d3e65a5-4593-42bc-88f9-7bbb2e647ebe/GEDzaV4pNFNQUuDwl2EQrv*eD9Sk9OJCzx5SpRRI2fk!/o/dGUBAAAAAAAA&ek=1&kp=1&pt=0&bo=OAL5ADgC.QADACU!&su=1155276897&sce=0-12-12&rf=2-9)

After one GC, generation 0 does not contain any objects. As before, new objects will be allocated to generation 0. Newly allocated objects F to K all go to generation 0.

![234](http://r.photo.store.qq.com/psb?/4d3e65a5-4593-42bc-88f9-7bbb2e647ebe/Op0QokzBTNYCFR6zzm2tpc2V7U70IsIJTeWrd0UAUb0!/o/dGUBAAAAAAAA&ek=1&kp=1&pt=0&bo=yAJeAcgCXgEDACU!&su=1124261217&sce=0-12-12&rf=2-9)

Then, the program continues to run, B, H, J become unreachable, their memory will be recycled at some point.

Assume that allocating new object L will cause generation 0 to exceed the budget, causing GC to start.

When starting garbage collection, the garbage collector must decide which generations to check. As mentioned earlier, CLR initializes by selecting a budget for generation 0 objects. In fact, it must also select a budget for generation 1.

When starting a garbage collection, the garbage collector also checks how much memory generation 1 occupies. In this example, since generation 1 occupies much less memory than the budget, the garbage collector decides to check only generation 0 objects. Recalling the first assumption made by the generational garbage collector: the newer the object, the shorter the lifetime. Therefore, generation 0 contains more garbage, and more memory can be recycled. By ignoring objects in generation 1, the garbage collection speed is accelerated.

Obviously, ignoring objects in generation 1 can improve the performance of the garbage collector. But what has a greater performance boost is that now there is no need to traverse every object in the managed heap. If a root or object references an object in an older generation, the garbage collector can ignore all references inside the old object, and can construct the reachable object graph in a shorter time. Of course, if the fields of old objects may also reference new objects. To ensure that updated fields of old objects are checked, the garbage collector uses a mechanism inside the JIT compiler. This mechanism sets a corresponding flag when the reference field of an object changes. In this way, the garbage collector knows which old objects (if any) have been written to since the last garbage collection. Only old objects whose fields have changed need to be checked to see if they reference any new objects in generation 0.

The generational garbage collector also assumes that the older the object, the longer it lives. That is, generation 1 objects are likely to continue to be reachable in the application. If the garbage collector checks the objects in generation 1, it is likely to find little garbage, and as a result, little memory can be recycled. Therefore, garbage collecting generation 1 is likely to be a waste of time. If there is really garbage in generation 1, the garbage will stay there. As shown in the figure:

![2345](http://r.photo.store.qq.com/psb?/4d3e65a5-4593-42bc-88f9-7bbb2e647ebe/Do.yRCBJEnaOfZaUOdxj4II9*pX2BEcX2QmIG6NQPBE!/o/dGUBAAAAAAAA&ek=1&kp=1&pt=0&bo=qAI5AagCOQEDACU!&su=187009937&sce=0-12-12&rf=2-9)

The program continues to run, continues to allocate objects to generation 0, and at the same time the program stops using some objects in generation 1.

As shown in the figure:

![edf](http://r.photo.store.qq.com/psb?/4d3e65a5-4593-42bc-88f9-7bbb2e647ebe/YEqIM16xFsSgXdvEzgrerLnKw7fEItnrSqEzlaYnUfE!/o/dGUBAAAAAAAA&ek=1&kp=1&pt=0&bo=egJPAXoCTwEDACU!&su=1118118497&sce=0-12-12&rf=2-9)

Allocating object P causes generation 0 to exceed the budget, starting GC. All objects in generation 1 still occupy less than the budget, the garbage collector decides again to recycle only generation 0. Ignore garbage objects in generation 1. As shown in the figure:

![2345](http://r.photo.store.qq.com/psb?/4d3e65a5-4593-42bc-88f9-7bbb2e647ebe/EcdSNU5AatqRERWtVdlJ7LiIPHHXe8.mklN.0hHDK9U!/o/dJQAAAAAAAAA&ek=1&kp=1&pt=0&bo=aAIxAWgCMQEDACU!&su=1214124305&sce=0-12-12&rf=2-9)

The program continues to run, assuming that the growth of generation 1 causes all its objects to occupy the full budget. At this time, the application allocates objects P to S, making the generation 0 objects reach their budget total. As shown in the figure:

![43](http://r.photo.store.qq.com/psb?/4d3e65a5-4593-42bc-88f9-7bbb2e647ebe/6dB68RIUYrqMZ4p0VIY3REJZPg.g3ybkZFIazJ3h.CQ!/o/dJIAAAAAAAAA&ek=1&kp=1&pt=0&bo=jwIiAY8CIgEDACU!&su=177976657&sce=0-12-12&rf=2-9)

At this time, the application is ready to allocate object T, since generation 1 is full, GC must start. But this time the garbage collector finds that generation 1 occupies too much memory, using up the budget. Since the previous GC on generation 0, there may already be many unreachable objects in generation 1. So this time the garbage collector decides to check all objects in generation 1 and generation 0. After garbage collection of both generations, the heap looks like the figure:

![123](http://r.photo.store.qq.com/psb?/4d3e65a5-4593-42bc-88f9-7bbb2e647ebe/bxdZDsZi2Y6FSDWs7RXNPkkJK8dCzMD.cfnjwNY2Mjs!/o/dJIAAAAAAAAA&ek=1&kp=1&pt=0&bo=tgI2AbYCNgEDACU!&su=197762641&sce=0-12-12&rf=2-9)

The managed heap only supports three generations: generation 0, generation 1, and generation 2.

When CLR initializes, it selects a budget for each generation.

However, CLR's garbage collection is self-adjusting.

This means that the garbage collector learns about the program's behavior during the garbage collection process.

For example: Assume that the application constructs many objects, but each object has a very short lifetime.

In this case, garbage collection of generation 0 will recycle a large amount of memory. In fact, all objects in generation 0 may be recycled.

If the garbage collector finds that after recycling generation 0, few objects survive, it may reduce the budget for generation 0. The reduction in allocated space means that garbage collection will occur more frequently, but the garbage collector does less each time, which reduces the process's working set.

On the other hand, if the garbage collector recycles generation 0 and finds that many objects survive, little memory can be recycled, it will increase the budget for generation 0.

The same heuristic algorithm adjusts the budget for generation 1 and generation 2.

Quoted from: "CLR VIA C# - Chapter 21"

[Automatic Memory Management](https://msdn.microsoft.com/zh-cn/library/vstudio/f144e03t(v=vs.100).aspx)

[Fundamentals of Garbage Collection](https://msdn.microsoft.com/zh-cn/library/vstudio/ee787088(v=vs.100).aspx)

[Generations](https://msdn.microsoft.com/zh-cn/library/vstudio/ee787088(v=vs.100).aspx#generations)
