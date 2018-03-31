---
layout: post
title:  "Spring Data Redis的Json对象解析器"
date:   2018-03-30
author:       "CaiJiahe"
header-img:   "img/tag-bg.jpg"
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - Spring
    - Spring Data
---

## 0x01 问题
在kotlin中使用Spring Data Redis的时候如下配置无法解析对象。
```kotlin
@Bean
open fun playerTemplate(factory: ReactiveRedisConnectionFactory): ReactiveRedisTemplate<String, Player> {
	val keySerializer = StringRedisSerializer()
	val valueSerializer = Jackson2JsonRedisSerializer(Player::class.java)
	val ctx = RedisSerializationContext.newSerializationContext<String, Player>()
			.key(keySerializer).value(valueSerializer).hashKey(keySerializer).hashValue(valueSerializer).build()
	return ReactiveRedisTemplate(factory, ctx)
}
```

## 0x02 解决方案
将配置修改如下
```kotlin
@Bean
open fun playerTemplate(factory: ReactiveRedisConnectionFactory): ReactiveRedisTemplate<String, Player> {
	val keySerializer = StringRedisSerializer()
	val valueSerializer = Jackson2JsonRedisSerializer(Player::class.java)
	valueSerializer.setObjectMapper(ObjectMapper().registerModule(KotlinModule()))
	val ctx = RedisSerializationContext.newSerializationContext<String, Player>()
			.key(keySerializer).value(valueSerializer).hashKey(keySerializer).hashValue(valueSerializer).build()
	return ReactiveRedisTemplate(factory, ctx)
}
```

```xml
<dependency>
	<groupId>com.fasterxml.jackson.module</groupId>
	<artifactId>jackson-module-kotlin</artifactId>
</dependency>
```
解决方案就是设置`valueSerializer`中的`objectMapper`为注册过`KotlinModule`的ObjectMapper。

## 0x03 Module
`Module`是jaskson databind为了支持新的数据类型增加的接口，将Module注册到ObjectMapper可以实现修改ObjectMapper默认的序列化和反序列化的行。上面就用到了`jackson-module-kotlin`提供的KotlinModule。
