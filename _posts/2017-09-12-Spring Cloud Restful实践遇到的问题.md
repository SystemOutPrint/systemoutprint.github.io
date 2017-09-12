---
layout: post
title:  "Spring Cloud Restful实践遇到的问题"
date:   2017-09-12
categories: Spring
---

## 0x01 问题

```java
@FeignClient(name = "kof-matchqueue", fallback = HystrixClientFallback.class)
public interface MatchQueueFeignClient {

	@RequestMapping("/matchqueue")
	public boolean beginMatch(@RequestParam("roleId") Long roleId, @RequestParam("serviceId") String serviceId);
	
	@DeleteMapping("/matchqueue")
	public boolean cancelMatch(@RequestParam("roleId") Long roleId, @RequestParam("serviceId") String serviceId);

	...
	
}
```
这段代码会导致如下异常

		org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'producer': Unsatisfied dependency expressed through field 'matchQueueFeignClient': Error creating bean with name 'game.gameserver.common.spring.feign.MatchQueueFeignClient': FactoryBean threw exception on object creation; nested exception is java.lang.IllegalStateException: Method cancelMatch not annotated with HTTP method type (ex. GET, POST); nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'game.gameserver.common.spring.feign.MatchQueueFeignClient': FactoryBean threw exception on object creation; nested exception is java.lang.IllegalStateException: Method cancelMatch not annotated with HTTP method type (ex. GET, POST)
		
## 0x02 原因
在FeignClient里，不能使用@GetMapping、@DeleteMapping这种注解，要使用@RequestMapping(value = "/matchqueue", method = RequestMethod.GET)这种。