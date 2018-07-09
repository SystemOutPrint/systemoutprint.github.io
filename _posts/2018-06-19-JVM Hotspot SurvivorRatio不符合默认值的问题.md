---
layout: post
title:  "JVM Hotspot SurvivorRatio不符合默认值的问题"
date:   2018-06-19
categories: gc
---

## 0x01 问题
使用JMC查看运行在Windows上的jvm进程的时候发现，使用默认参数启动的进程，survivor和eden的比例不符合默认值8。
```
Heap Configuration:
   MinHeapFreeRatio         = 0
   MaxHeapFreeRatio         = 100
   MaxHeapSize              = 2132803584 (2034.0MB)
   NewSize                  = 44564480 (42.5MB)
   MaxNewSize               = 710934528 (678.0MB)
   OldSize                  = 89653248 (85.5MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
PS Young Generation
Eden Space:
   capacity = 30932992 (29.5MB)
   used     = 19601520 (18.693466186523438MB)
   free     = 11331472 (10.806533813476562MB)
   63.36768198821504% used
From Space:
   capacity = 1048576 (1.0MB)
   used     = 753664 (0.71875MB)
   free     = 294912 (0.28125MB)
   71.875% used
To Space:
   capacity = 1048576 (1.0MB)
   used     = 0 (0.0MB)
   free     = 1048576 (1.0MB)
   0.0% used
PS Old Generation
   capacity = 124256256 (118.5MB)
   used     = 119121224 (113.60285186767578MB)
   free     = 5135032 (4.897148132324219MB)
   95.86738554234243% used
```
可以看到eden和survivor的比例变成了31.5，并不符合默认值8。

## 0x02 PS修正SurvivorRatio
PS在启动的时候判断如果没有设置默认值，那么会对InitialSurvivorRatio进行修正，变成6。代码如下：
```C
void Arguments::set_parallel_gc_flags() {
  ...

  // If InitialSurvivorRatio or MinSurvivorRatio were not specified, but the
  // SurvivorRatio has been set, reset their default values to SurvivorRatio +
  // 2.  By doing this we make SurvivorRatio also work for Parallel Scavenger.
  // See CR 6362902 for details.
  if (!FLAG_IS_DEFAULT(SurvivorRatio)) {
    if (FLAG_IS_DEFAULT(InitialSurvivorRatio)) {
       FLAG_SET_DEFAULT(InitialSurvivorRatio, SurvivorRatio + 2);
    }
    if (FLAG_IS_DEFAULT(MinSurvivorRatio)) {
      FLAG_SET_DEFAULT(MinSurvivorRatio, SurvivorRatio + 2);
    }
  }

  ...
}
```
但是这个也不符和31.5的比例，这个还要再细细斟酌，不过通过这个，我们可以明白，JVM参数在特定模式下，表现会不一致，所以在研究某个JVM参数应用的时候，一定要结合着实际情况来(垃圾回收器，server还是client模式)。

## 0x03 UseAdaptiveSizePolicy参数
再继续研究，发现`UseAdaptiveSizePolicy`参数的开启后，并行收集器会自动选择年轻代区大小和相应的Survivor区比例，以达到目标系统规定的最低相应时间或者收集频率等。所以SurvivorRatio设置的就没有用了，导致出现SurvivorRatio出现31.5:1。所以在设置的时候`SurvivorRation`和`UseAdaptiveSizePolicy`不应同时设置。

## 0x04 扩展
项目打算采用G1GC(备选CMS)，JDK11中新提供的ZGC虽然很诱惑人(GC停顿时间可以控制在10ms以下)，但是本着追求稳定的服务器架构的思路，还是忍痛放弃了。
