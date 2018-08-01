---
layout: post
title:  "projectreactor"
date:   2018-08-01
categories: 响应式编程
---

## 0x01 背景
projectreactor是一个实现了reactive-stream的第三代响应式编程框架。

## 0x02 retry
```java
val retry = Retry.anyOf<IllegalMonitorStateException>(IllegalMonitorStateException::class.java)
            .retryMax(5000)
            .randomBackoff(Duration.ofMillis(100), Duration.ofMillis(300))

Mono.just(true)
	.doOnNext {
		System.out.println("first...")
	}
    .doOnNext {
        System.out.println("second...")
        throw Exceptions.propagate(IllegalMonitorStateException())
    }
    .retryWhen(retry)
    .subscribe()
```
先定义了一个Retry，监听IllegalMonitorStateException，最大重试次数5000(默认是1)，随机退避时间100ms~300ms。<br>
输出是first...和second...交替5000次。