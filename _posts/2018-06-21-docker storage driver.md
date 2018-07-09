---
layout: post
title:  "docker storage driver"
date:   2018-06-19
categories: docker
---

## 0x01 写在前面
搞PAAS的一般会超卖各种能力，存储能力就是通过存储引擎来做的超卖。机制类似于Linux的COW，但是效率普遍没有直接写Data Volumn高。所以在一些写入比较多的场景，建议直接挂载数据卷。<br>

下面是docker支持的所有存储引擎:
```go
  // List of drivers that should be used in an order
  priority = "btrfs,zfs,overlay2,aufs,overlay,devicemapper,vfs"
```

## 0x02 device mapper(Linux kernel)
device mapper有两种模式，`loop-lvm`和`direct-lvm`，其根本思想是使用Linux提供的thin provisioning来"超分配"资源，以达到在一台有限资源的物理机上跑多个container的目的。两种模式都支持运行时增加容量，loop-lvm可以使用device_tool.go(生产环境禁用)，也可以使用操作系统工具来做。direct-lvm只能操作系统命令来做<br>
* loop-lvm: 通过loopback设备分配资源，不适用生产环境。
* direct-lvm: 直接使用块设备分配资源，适用生产环境。

device mapper会在`/var/lib/docker/`目录下创建`devicemapper`目录。在这个目录中分为`metadata`和`mnt`目录，metadata中存储了镜像、容器、快照和device mapper的配置的元信息。mnt目录是每个容器层的挂载点(镜像层的挂载点是空的目录)。<br>
device mapper通过快照来实现分层信息。这样做的好处是节省资源，对于镜像存储，可以存一份镜像供多个容器使用，对于硬盘存储，因为采用了采用了`用时分配`，每个块大小64K，所以存储也相对较省(但是在对于写负载较大的容器，可能效率相对其他存储驱动会低，建议采用data volumn来做)。对于内存占用，因为快照在运行时采用COW，一些修改操作只会影响容器层，不会影响镜像层，所以运行时也可以共享大部分的内存。<br>
因为device mapper使用块设备来存储，所以可以同时修改多个块设备，而不需要像联合文件系统那样，对大文件的少许修改也会触发对整个文件的COW。同时device mapper也可以被操作系统的工具去备份，比如直接复制/var/lib/docker/devicemapper/目录。<br>
device mapper的缺点是占内存较大，因为修改要先将block加载到内存。


## 0x03 btrfs(Linux kernel)
btrfs是下一代的COW文件系统，docker的multi layer利用了btrfs subvolumn的snapshot来做的。在storage driver中，只有images会被存成subvolumn，container都会存为该subvolumn的snapshot。btrfs可以动态的增加空间(chunk)。
因为snapshot和subvolumn指向同一个block，所以读取速度差不多，写入速度也差不多，更新速度开销也比较小，btrfs适用于读写大文件，对于小文件的读写可能会导致性能降低。<br>

```
# 设备添加
$ sudo btrfs device add /dev/svdh /var/lib/docker

## balance文件系统
$ sudo btrfs filesystem balance /var/lib/docker
```

性能问题：
* btrfs不支持缓存页共享
* 大量的小文件IO可能会导致磁盘空间不足。
* 因为COW事务，可能导致顺序写的性能降低至50%。
* btrfs可能导致内存碎片化。
* 可以对SSD优化。
* 可能需要使用crontab来经常的balance文件系统。

拓展阅读
* [新一代 Linux 文件系统 btrfs 简介](https://www.ibm.com/developerworks/cn/linux/l-cn-btrfs/)