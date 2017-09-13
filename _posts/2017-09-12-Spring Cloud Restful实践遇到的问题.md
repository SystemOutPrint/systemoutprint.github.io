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

打码(feign.Contract)
```java

protected MethodMetadata parseAndValidateMetadata(Class<?> targetType, Method method) {
  MethodMetadata data = new MethodMetadata();
  data.returnType(Types.resolve(targetType, targetType, method.getGenericReturnType()));
  data.configKey(Feign.configKey(targetType, method));

  if(targetType.getInterfaces().length == 1) {
	processAnnotationOnClass(data, targetType.getInterfaces()[0]);
  }
  processAnnotationOnClass(data, targetType);

  // 在这检查每个方法的注解
  for (Annotation methodAnnotation : method.getAnnotations()) {
	processAnnotationOnMethod(data, methodAnnotation, method);
  }
  
  // 如果没设置data.template().method()就抛异常，因为Spring4.3新加的那些注解没有被支持，所以会抛上述异常。
  checkState(data.template().method() != null,
			 "Method %s not annotated with HTTP method type (ex. GET, POST)",
			 method.getName());
  Class<?>[] parameterTypes = method.getParameterTypes();

  Annotation[][] parameterAnnotations = method.getParameterAnnotations();
  int count = parameterAnnotations.length;
  for (int i = 0; i < count; i++) {
	boolean isHttpAnnotation = false;
	if (parameterAnnotations[i] != null) {
	  isHttpAnnotation = processAnnotationsOnParameter(data, parameterAnnotations[i], i);
	}
	if (parameterTypes[i] == URI.class) {
	  data.urlIndex(i);
	} else if (!isHttpAnnotation) {
	  checkState(data.formParams().isEmpty(),
				 "Body parameters cannot be used with form parameters.");
	  checkState(data.bodyIndex() == null, "Method has too many Body parameters: %s", method);
	  data.bodyIndex(i);
	  data.bodyType(Types.resolve(targetType, targetType, method.getGenericParameterTypes()[i]));
	}
  }

  if (data.headerMapIndex() != null) {
	checkState(Map.class.isAssignableFrom(parameterTypes[data.headerMapIndex()]),
			"HeaderMap parameter must be a Map: %s", parameterTypes[data.headerMapIndex()]);
  }

  if (data.queryMapIndex() != null) {
	checkState(Map.class.isAssignableFrom(parameterTypes[data.queryMapIndex()]),
			"QueryMap parameter must be a Map: %s", parameterTypes[data.queryMapIndex()]);
  }

  return data;
}

protected void processAnnotationOnMethod(MethodMetadata data,
			Annotation methodAnnotation, Method method) {
	// 这里因为注解不是RequestMapping，所以直接返回，没有设置data.template().method()
	if (!(methodAnnotation instanceof RequestMapping)) {
		return;
	}

	RequestMapping methodMapping = findMergedAnnotation(method, RequestMapping.class);
	// HTTP Method
	RequestMethod[] methods = methodMapping.method();
	if (methods.length == 0) {
		methods = new RequestMethod[] { RequestMethod.GET };
	}
	checkOne(method, methods, "method");
	data.template().method(methods[0].name());

	// path
	checkAtMostOne(method, methodMapping.value(), "value");
	if (methodMapping.value().length > 0) {
		String pathValue = emptyToNull(methodMapping.value()[0]);
		if (pathValue != null) {
			pathValue = resolve(pathValue);
			// Append path from @RequestMapping if value is present on method
			if (!pathValue.startsWith("/")
					&& !data.template().toString().endsWith("/")) {
				pathValue = "/" + pathValue;
			}
			data.template().append(pathValue);
		}
	}

	// produces
	parseProduces(data, method, methodMapping);

	// consumes
	parseConsumes(data, method, methodMapping);

	// headers
	parseHeaders(data, method, methodMapping);

	data.indexToExpander(new LinkedHashMap<Integer, Param.Expander>());
}
```