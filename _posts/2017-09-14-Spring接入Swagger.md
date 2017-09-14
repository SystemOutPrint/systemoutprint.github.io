---
layout: post
title:  "Spring接入Swagger"
date:   2017-09-14
categories: Spring
---

## 0x01 pom
```xml
<dependency>
	<groupId>io.springfox</groupId>
	<artifactId>springfox-swagger2</artifactId>
	<version>2.7.0</version>
</dependency>
<dependency>
	<groupId>io.springfox</groupId>
	<artifactId>springfox-swagger-ui</artifactId>
	<version>2.7.0</version>
</dependency>
```

## 0x02 Spring Configuration

``` java
@Configuration
@EnableSwagger2
public class Swagger2 {
	@Bean
	public Docket createRestApi() {
		return new Docket(DocumentationType.SWAGGER_2)
				.apiInfo(apiInfo())
				.select()
				.apis(RequestHandlerSelectors.basePackage("com.kof.matchqueue")) // 扫描该包下的所有注解
				.paths(PathSelectors.any())
				.build();
	}

	private ApiInfo apiInfo() {
		return new ApiInfoBuilder()
				.title("kof-匹配队列")
				.description("测试版匹配队列，功能等策划慢慢扩展去吧~去吧~去吧！")
				.contact("蔡佳何")
				.version("1.0.0")
				.build();
	}
}
```

## 0x03 demo
```java
@ApiOperation(value = "开始匹配", notes = "", consumes="application/x-www-form-urlencoded")
@ApiImplicitParams({
	 @ApiImplicitParam(name = "roleId", paramType = "query", value = "角色id", required = true, dataType = "Long"),
	 @ApiImplicitParam(name = "serviceId", paramType = "query", value = "该角色gs服务id", required = true, dataType = "String")
})
@RequestMapping(value = "/matchqueue", method = RequestMethod.GET)
public boolean beginMatch(@RequestParam("roleId") Long roleId, @RequestParam("serviceId") String serviceId) {
	return matchService.beginMatch(roleId, serviceId);
}
```

## 0x04 恶心之处
如果在ApiImplicitParam中不配置paramType，默认是body，然后就会找不到参数。

## 0x05 Swagger注解大全
* @Api：用在controller上，对controller进行注释；
* @ApiOperation：用在API方法上，对该API做注释，说明API的作用；
* @ApiImplicitParams：用来包含API的一组参数注解，可以简单的理解为参数注解的集合声明；
* @ApiImplicitParam：用在@ApiImplicitParams注解中，也可以单独使用，说明一个请求参数的各个方面。
	* paramType：参数所放置的地方，包含query、header、path、body以及form，最常用的是前四个。
	* name：参数名；
	* dataType：参数类型，可以是基础数据类型，也可以是一个class；
	* required：参数是否必须传；
	* value：参数的注释，说明参数的意义；
	* defaultValue：参数的默认值；
* @ApiResponses：通常用来包含接口的一组响应注解，可以简单的理解为响应注解的集合声明；
* @ApiResponse：用在@ApiResponses中，一般用于表达一个错误的响应信息