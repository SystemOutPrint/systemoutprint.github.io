---
layout: post
title:  "Spring Data Redis 源码分析"
date:   2018-03-13
author:       "CaiJiahe"
header-img:   "img/tag-bg.jpg"
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - Redis
---

>> 因为最近在使用WebFlux开发，在集成Redis的时候使用了Spring Data Redis 2.0提供的Reactive接口。
>> 发现了几个问题，顺手看了下源码，然后提了几个PR。<br>
>> [DATAREDIS-779 ReactiveValueOperations.set[ifPresent|ifAbsent](…) does not return a value if value was not set](https://github.com/spring-projects/spring-data-redis/pull/317)<br>
>> [DATAREDIS-782 Add support for SET key value NX EX max-lock-time](https://github.com/spring-projects/spring-data-redis/pull/320)<br>
>> [DATAREDIS-783 Fix typo in LettuceReactiveRedisConnection](https://github.com/spring-projects/spring-data-redis/pull/321)<br>
>> [DATAREDIS-784 Add support for reactive [INCR | INCRBY | INCRBYFLOAT | DECR | DECRBY]](https://github.com/spring-projects/spring-data-redis/pull/322)

## 0x01 简介
这个框架是Spring对Redis的胶水框架，集成了Jedis和Lettuce这两个java的redis客户端。整个项目的命令核心接口都在`org.springframework.data.redis.connection`目录下，分为命令式接口和响应式接口。

## 0x02 阻塞式API结构

阻塞式连接接口:<br>
```java
public interface RedisCommands extends RedisKeyCommands, RedisStringCommands, RedisListCommands, RedisSetCommands,
		RedisZSetCommands, RedisHashCommands, RedisTxCommands, RedisPubSubCommands, RedisConnectionCommands,
		RedisServerCommands, RedisScriptingCommands, RedisGeoCommands, RedisHyperLogLogCommands
	
	Object execute(String command, byte[]... args);
	
}

public interface RedisConnection extends RedisCommands {

	default RedisGeoCommands geoCommands() {
		return this;
	}

	default RedisHashCommands hashCommands() {
		return this;
	}

	default RedisHyperLogLogCommands hyperLogLogCommands() {
		return this;
	}

	default RedisKeyCommands keyCommands() {
		return this;
	}

	default RedisListCommands listCommands() {
		return this;
	}

	default RedisSetCommands setCommands() {
		return this;
	}
	
	...
}
```

这些Command的接口实现大同小异，无非是通过调用jedis或lettuce连接来调用这些命令，但是对事务和pipeline的支持过于粗糙，在每个方法里面去做特殊判断，Orz。

## 0x03 命令式API结构
Reactive命令接口:<br>
```java
public interface ReactiveRedisConnection extends Closeable {

	@Override
	void close();

	/**
	 * Get {@link ReactiveKeyCommands}.
	 *
	 * @return never {@literal null}.
	 */
	ReactiveKeyCommands keyCommands();

	/**
	 * Get {@link ReactiveStringCommands}.
	 *
	 * @return never {@literal null}.
	 */
	ReactiveStringCommands stringCommands();

	/**
	 * Get {@link ReactiveNumberCommands}
	 *
	 * @return never {@literal null}.
	 */
	ReactiveNumberCommands numberCommands();

	/**
	 * Get {@link ReactiveListCommands}.
	 *
	 * @return never {@literal null}.
	 */
	ReactiveListCommands listCommands();

	/**
	 * Get {@link ReactiveSetCommands}.
	 *
	 * @return never {@literal null}.
	 */
	ReactiveSetCommands setCommands();

	/**
	 * Get {@link ReactiveZSetCommands}.
	 *
	 * @return never {@literal null}.
	 */
	ReactiveZSetCommands zSetCommands();

	...
}
```
同阻塞式接口类似，也是在调用lettuce提供的响应式接口。

## 0x04 封装
无论是命令式还是响应式的接口，Commands方法对用户都不是很友好，所以Spring Data Redis封装了一层Operations接口，用来适配Commands接口里的对应方法，方便使用者使用。

## 0x05 问题
目前Spring Data Redis 2.0还是存在一些问题的，比如Reactive命令接口支持不完善，没有对事务、pipeline的支持。<br>
我尝试支持事务，发现一些命令的返回是`Mono<Boolean>`，但是大多数的redis命令返回不止二值，有可能是多值，比如典型的`setIfAbsent()`，返回值可能是`OK`、`nil`、`QUEUED`，在阻塞式接口中使用`null`表示`QUEUED`，但是如果在响应式接口中返回`Mono.empty()`，事件就无法发射出去，导致后续操作的失败，这种方案被否。转而想用Mono.error来实现，但是这种方法会有歧义，这个需要再去考虑考虑。