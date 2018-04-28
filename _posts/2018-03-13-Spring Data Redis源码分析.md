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

## 0x01 Auto-Configuration
这个框架是Spring对Redis的胶水框架，集成了Jedis和Lettuce这两个java的redis客户端。整个项目的命令核心接口都在`org.springframework.data.redis.connection`目录下，分为响应式接口和响应式接口。
```java
// org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration
@Configuration
@ConditionalOnClass(RedisOperations.class)
@EnableConfigurationProperties(RedisProperties.class)
@Import({ LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class })
public class RedisAutoConfiguration {
	...
}

@Configuration
@ConditionalOnClass(RedisClient.class)
class LettuceConnectionConfiguration extends RedisConnectionConfiguration {
	
	@Bean
	@ConditionalOnMissingBean(RedisConnectionFactory.class)
	public LettuceConnectionFactory redisConnectionFactory(
			ClientResources clientResources) throws UnknownHostException {
		LettuceClientConfiguration clientConfig = getLettuceClientConfiguration(
				clientResources, this.properties.getLettuce().getPool());
		return createLettuceConnectionFactory(clientConfig);
	}

	private LettuceConnectionFactory createLettuceConnectionFactory(
			LettuceClientConfiguration clientConfiguration) {
		if (getSentinelConfig() != null) {
			return new LettuceConnectionFactory(getSentinelConfig(), clientConfiguration);
		}
		if (getClusterConfiguration() != null) {
			return new LettuceConnectionFactory(getClusterConfiguration(),
					clientConfiguration);
		}
		return new LettuceConnectionFactory(getStandaloneConfig(), clientConfiguration);
	}
	
	protected final RedisClusterConfiguration getClusterConfiguration() {
		if (this.clusterConfiguration != null) {
			return this.clusterConfiguration;
		}
		if (this.properties.getCluster() == null) {
			return null;
		}
		RedisProperties.Cluster clusterProperties = this.properties.getCluster();
		RedisClusterConfiguration config = new RedisClusterConfiguration(
				clusterProperties.getNodes());
		if (clusterProperties.getMaxRedirects() != null) {
			config.setMaxRedirects(clusterProperties.getMaxRedirects());
		}
		if (this.properties.getPassword() != null) {
			config.setPassword(RedisPassword.of(this.properties.getPassword()));
		}
		return config;
	}
	
	...
}
```
`RedisAutoConfiguration`判断是否有Lettuce的依赖，然后解析`spring.redis`的`ConfigurationProperties`创建好配置Bean或者通过`ObjectProvider`来获取各种配置bean。

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

Spring Data Redis基于命令模式，这些Command的接口实现大同小异，无非是通过调用jedis或lettuce连接中的`execute`方法来调用这些命令，实现在standalone和cluster不同环境中的调用。但是对阻塞式接口的事务和pipeline的支持过于粗糙，需要在每个方法里面去做特殊判断，Orz。

## 0x03 响应式API结构
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
无论是响应式还是阻塞式的接口，Commands方法对用户都不是很友好，所以Spring Data Redis封装了一层Operations接口，用来适配Commands接口里的对应方法(ShortCut)，方便使用者使用。

## 0x05 问题
目前Spring Data Redis 2.0还是存在一些问题的，比如Reactive命令接口支持不完善，没有对事务、pipeline的支持。<br>
我尝试支持事务，发现一些命令的返回是`Mono<Boolean>`，但是大多数的redis命令返回不止二值，有可能是多值，比如典型的`setIfAbsent()`，返回值可能是`OK`、`nil`、`QUEUED`，在阻塞式接口中使用`null`表示`QUEUED`，但是如果在响应式接口中返回`Mono.empty()`，事件就无法发射出去，导致后续操作的失败，这种方案被否。转而想用Mono.error来实现，但是这种方法会有歧义，这个需要再去考虑考虑。