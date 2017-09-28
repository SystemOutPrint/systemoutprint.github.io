---
layout: post
title:  "Spring Feign实现"
date:   2017-09-27
categories: Spring
---

## 0x01 简介
Feign是一个声明式的WebService客户端。使用Feign能让编写WebService客户端更加简单，它的使用方法是定义一个接口，然后在接口上添加注解，同时也支持JAX-RS标准的注解。Feign也支持可插拔式的编码器和解码器。SpringCloud对Feign进行了封装，使其支持SpringMVC标准注解和HttpMessageConverters。Feign可以与Eureka和Ribbon组合使用以支持负载均衡。

## 0x02 demo

``` java
@SpringBootApplication(exclude={DataSourceAutoConfiguration.class, HibernateJpaAutoConfiguration.class})
@EnableDiscoveryClient
@EnableFeignClients
@EnableCircuitBreaker
public class SwitcherApplication {
	...
}


@FeignClient(name = "kof-matchqueue", fallback = HystrixClientFallback.class)
public interface MatchQueueFeignClient {

	@RequestMapping(value = "/match-queue", method = RequestMethod.GET)
	public boolean beginMatch(@RequestParam("roleId") Long roleId);
	
	@RequestMapping(value = "/match-queue", method = RequestMethod.DELETE)
	public boolean cancelMatch(@RequestParam("roleId") Long roleId);

	@Component
	static class HystrixClientFallback implements MatchQueueFeignClient {
		
		private static final Logger LOGGER = LoggerFactory.getLogger(HystrixClientFallback.class);

		@Override
		public boolean beginMatch(Long roleId) {
			LOGGER.error("---断路器---匹配失败！！！ roleId " + roleId);
			return false;
		}
		
		@Override
		public boolean cancelMatch(Long roleId) {
			LOGGER.error("---断路器---取消匹配失败！！！ roleId " + roleId);
			return false;
		}
	}

}
```

## 0x03 分析

### 注册过程

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients {
	...
}
```
这里import个FeignClientRegistrar来注册自定义的bean，看下对Import注解的处理过程。

```java
private void loadBeanDefinitionsForConfigurationClass(ConfigurationClass configClass,
			TrackedConditionEvaluator trackedConditionEvaluator) {

	if (trackedConditionEvaluator.shouldSkip(configClass)) {
		String beanName = configClass.getBeanName();
		if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
			this.registry.removeBeanDefinition(beanName);
		}
		this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
		return;
	}

	if (configClass.isImported()) {
		registerBeanDefinitionForImportedConfigurationClass(configClass);
	}
	for (BeanMethod beanMethod : configClass.getBeanMethods()) {
		loadBeanDefinitionsForBeanMethod(beanMethod);
	}
	loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
	// 这个地方准备处理Configuration中Import进来的ImportBeanDefinitionRegistrar方法
	loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
}

// 最后会调到FeignClientsRegistrar类这个方法中。
public void registerFeignClients(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
	ClassPathScanningCandidateComponentProvider scanner = getScanner();
	scanner.setResourceLoader(this.resourceLoader);

	Set<String> basePackages;

	Map<String, Object> attrs = metadata
			.getAnnotationAttributes(EnableFeignClients.class.getName());
	AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(
			FeignClient.class);
	final Class<?>[] clients = attrs == null ? null
			: (Class<?>[]) attrs.get("clients");
	if (clients == null || clients.length == 0) {
		scanner.addIncludeFilter(annotationTypeFilter);
		basePackages = getBasePackages(metadata);
	}
	else {
		final Set<String> clientClasses = new HashSet<>();
		basePackages = new HashSet<>();
		for (Class<?> clazz : clients) {
			basePackages.add(ClassUtils.getPackageName(clazz));
			clientClasses.add(clazz.getCanonicalName());
		}
		AbstractClassTestingTypeFilter filter = new AbstractClassTestingTypeFilter() {
			@Override
			protected boolean match(ClassMetadata metadata) {
				String cleaned = metadata.getClassName().replaceAll("\\$", ".");
				return clientClasses.contains(cleaned);
			}
		};
		scanner.addIncludeFilter(
				new AllTypeFilter(Arrays.asList(filter, annotationTypeFilter)));
	}

	for (String basePackage : basePackages) {
		Set<BeanDefinition> candidateComponents = scanner
				.findCandidateComponents(basePackage);
		for (BeanDefinition candidateComponent : candidateComponents) {
			if (candidateComponent instanceof AnnotatedBeanDefinition) {
				// verify annotated class is an interface
				AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
				AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
				Assert.isTrue(annotationMetadata.isInterface(),
						"@FeignClient can only be specified on an interface");

				Map<String, Object> attributes = annotationMetadata
						.getAnnotationAttributes(
								FeignClient.class.getCanonicalName());

				String name = getClientName(attributes);
				registerClientConfiguration(registry, name,
						attributes.get("configuration"));

				registerFeignClient(registry, annotationMetadata, attributes);
			}
		}
	}
}

```

registerFeignClients方法中会以标有@EnableFeignClients注解类所在的位置为根目录，使用自定义的ClassPathScanningCandidateComponentProvider进行递归查找，过滤所有的@FeignClient的BeanDefinition，分别创建FeignClientFactoryBean和FeignClientSpecification两种bean注册到DefaultListableBeanFactory中。

### 注入过程

```java
// FeignClientFactoryBean.java
@Override
public Object getObject() throws Exception {
	FeignContext context = applicationContext.getBean(FeignContext.class);
	Feign.Builder builder = feign(context);

	if (!StringUtils.hasText(this.url)) {
		String url;
		if (!this.name.startsWith("http")) {
			url = "http://" + this.name;
		}
		else {
			url = this.name;
		}
		url += cleanPath();
		return loadBalance(builder, context, new HardCodedTarget<>(this.type,
				this.name, url));
	}
	if (StringUtils.hasText(this.url) && !this.url.startsWith("http")) {
		this.url = "http://" + this.url;
	}
	String url = this.url + cleanPath();
	Client client = getOptional(context, Client.class);
	if (client != null) {
		if (client instanceof LoadBalancerFeignClient) {
			// not lod balancing because we have a url,
			// but ribbon is on the classpath, so unwrap
			client = ((LoadBalancerFeignClient)client).getDelegate();
		}
		builder.client(client);
	}
	Targeter targeter = get(context, Targeter.class);
	return targeter.target(this, builder, context, new HardCodedTarget<>(
			this.type, this.name, url));
}

protected <T> T loadBalance(Feign.Builder builder, FeignContext context,
			HardCodedTarget<T> target) {
	Client client = getOptional(context, Client.class);
	if (client != null) {
		builder.client(client);
		Targeter targeter = get(context, Targeter.class);
		return targeter.target(this, builder, context, target);
	}

	throw new IllegalStateException(
			"No Feign Client for loadBalancing defined. Did you forget to include spring-cloud-starter-ribbon?");
}
```
调用targerer的targer方法，内层判断如果builder是HystrixFeign.Builder类型，则返回builder.build().newInstance()。

```java
// Feign.build
public Feign build() {
  SynchronousMethodHandler.Factory synchronousMethodHandlerFactory =
	  new SynchronousMethodHandler.Factory(client, retryer, requestInterceptors, logger,
										   logLevel, decode404);
  ParseHandlersByName handlersByName =
	  new ParseHandlersByName(contract, options, encoder, decoder,
							  errorDecoder, synchronousMethodHandlerFactory);
  return new ReflectiveFeign(handlersByName, invocationHandlerFactory);
}

// ReflectiveFeign.newInstance
public <T> T newInstance(Target<T> target) {
    Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
    Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
    List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();

    for (Method method : target.type().getMethods()) {
      if (method.getDeclaringClass() == Object.class) {
        continue;
      } else if(Util.isDefault(method)) {
        DefaultMethodHandler handler = new DefaultMethodHandler(method);
        defaultMethodHandlers.add(handler);
        methodToHandler.put(method, handler);
      } else {
        methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
      }
    }
    InvocationHandler handler = factory.create(target, methodToHandler);
    T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(), new Class<?>[]{target.type()}, handler);

    for(DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
      defaultMethodHandler.bindTo(proxy);
    }
    return proxy;
}
```
build中创建了synchronousMethodHandlerFactory，在newInstance的targetToHandlerByName.apply中，会调用这个工厂的create方法，根据FeignClient的配置创建一个SynchronousMethodHandler。<br>
在接下来的过程中会根据这个handler使用InvocationHandlerFactory创建一个InvocationHandler，然后使用jvm代理创建好FeignClient接口的代理类。

### 使用过程
```java
// SynchronousMethodHandler.invoke
@Override
public Object invoke(Object[] argv) throws Throwable {
    RequestTemplate template = buildTemplateFromArgs.create(argv);
    Retryer retryer = this.retryer.clone();
    while (true) {
      try {
        return executeAndDecode(template);
      } catch (RetryableException e) {
        retryer.continueOrPropagate(e);
        if (logLevel != Logger.Level.NONE) {
          logger.logRetry(metadata.configKey(), logLevel);
        }
        continue;
      }
    }
}
```
先根据参数创建好RequestTemplate，然后准备一个Retryer，接下来就开始执行请求，如果有异常就重试。这个client可就厉害了，看一下子类，发现有Default、LoadBalancerFeignClient和TraceFeignClient。分别是默认的，加了@LoadBalanced注解的，和使用sleuth后的client。
默认的没啥好说的，先看下TraceFeignClient的execute方法。
```java
@Override
public Response execute(Request request, Request.Options options) throws IOException {
	String spanName = getSpanName(request);
	Span span = getTracer().createSpan(spanName);
	if (log.isDebugEnabled()) {
		log.debug("Created new Feign span " + span);
	}
	try {
		AtomicReference<Request> feignRequest = new AtomicReference<>(request);
		spanInjector().inject(span, new FeignRequestTextMap(feignRequest));
		span.logEvent(Span.CLIENT_SEND);
		addRequestTags(request);
		Request modifiedRequest = feignRequest.get();
		if (log.isDebugEnabled()) {
			log.debug("The modified request equals " + modifiedRequest);
		}
		Response response = this.delegate.execute(modifiedRequest, options);
		logCr();
		return response;
	} catch (RuntimeException | IOException e) {
		logCr();
		logError(e);
		throw e;
	} finally {
		closeSpan(span);
	}
}
```
封装了一下原来的Client，对delegate前后增加统计时间的操作。这个client使用aop来实现这层代理。
```java
@Aspect
class TraceFeignAspect {

	private final BeanFactory beanFactory;

	TraceFeignAspect(BeanFactory beanFactory) {
		this.beanFactory = beanFactory;
	}

	@Around("execution (* feign.Client.*(..)) && !within(is(FinalType))")
	public Object feignClientWasCalled(final ProceedingJoinPoint pjp) throws Throwable {
		Object[] args = pjp.getArgs();
		Request request = (Request) args[0];
		Request.Options options = (Request.Options) args[1];
		Object bean = pjp.getTarget();
		if (!(bean instanceof TraceFeignClient)) {
			return new TraceFeignClient(this.beanFactory, (Client) bean).execute(request, options);
		}
		return pjp.proceed();
	}
}
```
下面看下LoadBalancerFeignClient的execute。
```java
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
```
在executeWithLoadBalancer方法中就会使用ribbon来选择合适的服务来发送请求。