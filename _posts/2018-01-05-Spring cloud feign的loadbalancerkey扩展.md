---
layout: post
title:  "Spring Cloud Feign的loadbalancerkey扩展"
date:   2018-01-05
author:       "CaiJiahe"
header-img:   "img/tag-bg.jpg"
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - Spring
    - Spring Cloud
    - Feign
    - 负载均衡
---

## 0x01 netflix-loadbalancer

在微服务的水平扩展和微服务的状态性上的权衡，考虑微服务有状态，在客户端做静态hash。但是随之而来的问题就是ribbon balancer并不支持这种静态hash的策略选择。代码如下：
```java
	// spring-cloud-netflix-core org.springframework.cloud.netflix.feign.ribbon.LoadBalancerFeignClient
	@Override
	public Response execute(Request request, Request.Options options) throws IOException {
		try {
			URI asUri = URI.create(request.url());
			String clientName = asUri.getHost();
			URI uriWithoutHost = cleanUrl(request.url(), clientName);
			FeignLoadBalancer.RibbonRequest ribbonRequest = new FeignLoadBalancer.RibbonRequest(
					this.delegate, request, uriWithoutHost);

			IClientConfig requestConfig = getClientConfig(options, clientName);
			return lbClient(clientName).executeWithLoadBalancer(ribbonRequest,
					requestConfig).toResponse();
		}
		catch (ClientException e) {
			IOException io = findIOException(e);
			if (io != null) {
				throw io;
			}
			throw new RuntimeException(e);
		}
	}
	
	// ribbon-loadbalancer com.netflix.client.AbstractLoadBalancerAwareClient
	public T executeWithLoadBalancer(final S request, final IClientConfig requestConfig) throws ClientException {
        RequestSpecificRetryHandler handler = getRequestSpecificRetryHandler(request, requestConfig);
        LoadBalancerCommand<T> command = LoadBalancerCommand.<T>builder()
                .withLoadBalancerContext(this)
                .withRetryHandler(handler)
                .withLoadBalancerURI(request.getUri())
                .build();
		
		...
	}
	
	public class LoadBalancerCommand<T> {
		
		public static class Builder<T> {
			
			private Object              loadBalancerKey;
			...
		}
	}
	
```
从上面可以看到ribbon-loadbalancer包并没有设置loadBalancerKey。<br>
<br>
[netflix/ribbon add loadBalancerKey when building a LoadBalancerCommand #309](https://github.com/Netflix/ribbon/pull/309)<br>
上面的merge里提供了customizeLoadBalancerCommandBuilder这个方法，可以使用子类重写这个方法，来手动的设置Loadbalancerkey。<br>
netflix-loadbalancer-2.2.4.jar及以上版本。

## 0x02 spring-cloud-netflix-core
虽然设置loadbalancerkey的问题解决了，但是如何从feign的声明式客户端把自定义的key值传入呢？<br>
[spring-cloud/spring-cloud-netflix public FeignLoadBalancer.RibbonRequest and FeignLoadBalancer.RibbonRe… #2536](https://github.com/spring-cloud/spring-cloud-netflix/pull/2536)<br>
上面的链接里把RibbonRequest修改为可重写的，所以我们可以重写这个request，在子类中设置loadbalancerkey，这样就可以将自定义的key值传入到ILoadBalancer的chooseServer中，这样就可以使用ribbon实现客户端自定义hash了。<br>
这个代码目前还没在release的版本中，静静等待。