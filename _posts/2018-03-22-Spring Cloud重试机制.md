---
layout: post
title:  "Spring Cloud重试机制"
date:   2018-03-22
author:       "CaiJiahe"
header-img:   "img/tag-bg.jpg"
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - Spring
    - Spring Cloud
---

>> 某些情况下我们无法设计出幂等的API接口，所以我们不能执行fail-retry这种策略。

## 0x01 问题
我没有开启ribbon，但是仍然有请求可以重试。

## 0x02 查看文档
看文档Spring Cloud提供了一个`spring.cloud.loadbalancer.retry.enabled`参数用来禁用loadbalancer的retry策略，但是在实际操作的时候，设置该值为false时，问题解决。

## 0x03 源码分析
为了了解如何这个参数如何禁用的我去翻看了下源码。
```java
// spring-cloud-commons o.s.c.client.loadbalancer
@ConfigurationProperties("spring.cloud.loadbalancer.retry")
public class LoadBalancerRetryProperties {
    private boolean enabled = true;
    
    public boolean isEnabled() {
        return enabled;
    }

    ...
    
}

// spring-cloud-openfeign o.s.c.openfeign.ribbon
public class RetryableFeignLoadBalancer extends FeignLoadBalancer implements ServiceInstanceChooser {

    private final LoadBalancedRetryPolicyFactory loadBalancedRetryPolicyFactory;
    private final LoadBalancedBackOffPolicyFactory loadBalancedBackOffPolicyFactory;
    private final LoadBalancedRetryListenerFactory loadBalancedRetryListenerFactory;
    
    @Override
    public RibbonResponse execute(final RibbonRequest request, IClientConfig configOverride)
            throws IOException {
        final Request.Options options;
        if (configOverride != null) {
            RibbonProperties ribbon = RibbonProperties.from(configOverride);
            options = new Request.Options(
                    ribbon.connectTimeout(this.connectTimeout),
                    ribbon.readTimeout(this.readTimeout));
        }
        else {
            options = new Request.Options(this.connectTimeout, this.readTimeout);
        }
        final LoadBalancedRetryPolicy retryPolicy = loadBalancedRetryPolicyFactory.create(this.getClientName(), this);
        RetryTemplate retryTemplate = new RetryTemplate();
        BackOffPolicy backOffPolicy = loadBalancedBackOffPolicyFactory.createBackOffPolicy(this.getClientName());
        retryTemplate.setBackOffPolicy(backOffPolicy == null ? new NoBackOffPolicy() : backOffPolicy);
        RetryListener[] retryListeners = this.loadBalancedRetryListenerFactory.createRetryListeners(this.getClientName());
        if (retryListeners != null && retryListeners.length != 0) {
            retryTemplate.setListeners(retryListeners);
        }
        retryTemplate.setRetryPolicy(retryPolicy == null ? new NeverRetryPolicy()
                : new FeignRetryPolicy(request.toHttpRequest(), retryPolicy, this, this.getClientName()));
        return retryTemplate.execute(new RetryCallback<RibbonResponse, IOException>() {
            @Override
            public RibbonResponse doWithRetry(RetryContext retryContext) throws IOException {
                Request feignRequest = null;
                //on retries the policy will choose the server and set it in the context
                //extract the server and update the request being made
                if (retryContext instanceof LoadBalancedRetryContext) {
                    ServiceInstance service = ((LoadBalancedRetryContext) retryContext).getServiceInstance();
                    if (service != null) {
                        feignRequest = ((RibbonRequest) request.replaceUri(reconstructURIWithServer(new Server(service.getHost(), service.getPort()), request.getUri()))).toRequest();
                    }
                }
                if (feignRequest == null) {
                    feignRequest = request.toRequest();
                }
                Response response = request.client().execute(feignRequest, options);
                if (retryPolicy.retryableStatusCode(response.status())) {
                    byte[] byteArray = StreamUtils.copyToByteArray(response.body().asInputStream());
                    response.close();
                    throw new RibbonResponseStatusCodeException(RetryableFeignLoadBalancer.this.clientName, response,
                            byteArray, request.getUri());
                }
                return new RibbonResponse(request.getUri(), response);
            }
        }, new RibbonRecoveryCallback<RibbonResponse, Response>() {
            @Override
            protected RibbonResponse createResponse(Response response, URI uri) {
                return new RibbonResponse(uri, response);
            }
        });
    }

    @Override
    public ServiceInstance choose(String serviceId) {
        return new RibbonLoadBalancerClient.RibbonServer(serviceId,
                this.getLoadBalancer().chooseServer(serviceId));
    }
    
    ...
}

```
可以看到`spring.cloud.loadbalancer.retry.enabled`默认值是`true`。在`RetryableFeignLoadBalancer`中发送http请求的时候，根据是否开启重试策略和`LoadBalancedRetryPolicy`去构造一个`InterceptorRetryPolicy`。

```java
public class InterceptorRetryPolicy implements RetryPolicy {

    private HttpRequest request;
    private LoadBalancedRetryPolicy policy;
    private ServiceInstanceChooser serviceInstanceChooser;
    private String serviceName;

    ...

    @Override
    public boolean canRetry(RetryContext context) {
        LoadBalancedRetryContext lbContext = (LoadBalancedRetryContext)context;
        if(lbContext.getRetryCount() == 0  && lbContext.getServiceInstance() == null) {
            //We haven't even tried to make the request yet so return true so we do
            lbContext.setServiceInstance(serviceInstanceChooser.choose(serviceName));
            return true;
        }
        return policy.canRetryNextServer(lbContext);
    }
    
    ...
}

public interface LoadBalancedRetryPolicy {

    public boolean canRetrySameServer(LoadBalancedRetryContext context);

    public boolean canRetryNextServer(LoadBalancedRetryContext context);

    public abstract void close(LoadBalancedRetryContext context);

    public abstract void registerThrowable(LoadBalancedRetryContext context, Throwable throwable);

    public boolean retryableStatusCode(int statusCode);
}

```
可以看到重试是否可以重试实际上回调了`LoadBalancedRetryPolicy`的方法，而`LoadBalancedRetryPolicy`目前在openfeign中默认的策略是`RibbonLoadBalancedRetryPolicy`。
```java
public class RibbonLoadBalancedRetryPolicy implements LoadBalancedRetryPolicy {

    public static final IClientConfigKey<String> RETRYABLE_STATUS_CODES = new CommonClientConfigKey<String>("retryableStatusCodes") {};
    private static final Log log = LogFactory.getLog(RibbonLoadBalancedRetryPolicy.class);
    private int sameServerCount = 0;
    private int nextServerCount = 0;
    private String serviceId;
    private RibbonLoadBalancerContext lbContext;
    private ServiceInstanceChooser loadBalanceChooser;
    List<Integer> retryableStatusCodes = new ArrayList<>();
    
    ...

    public boolean canRetry(LoadBalancedRetryContext context) {
        HttpMethod method = context.getRequest().getMethod();
        return HttpMethod.GET == method || lbContext.isOkToRetryOnAllOperations();
    }

    @Override
    public boolean canRetrySameServer(LoadBalancedRetryContext context) {
        return sameServerCount < lbContext.getRetryHandler().getMaxRetriesOnSameServer() && canRetry(context);
    }

    @Override
    public boolean canRetryNextServer(LoadBalancedRetryContext context) {
        //this will be called after a failure occurs and we increment the counter
        //so we check that the count is less than or equals to too make sure
        //we try the next server the right number of times
        return nextServerCount <= lbContext.getRetryHandler().getMaxRetriesOnNextServer() && canRetry(context);
    }

    @Override
    public void registerThrowable(LoadBalancedRetryContext context, Throwable throwable) {
        //if this is a circuit tripping exception then notify the load balancer
        if (lbContext.getRetryHandler().isCircuitTrippingException(throwable)) {
            updateServerInstanceStats(context);
        }
        
        //Check if we need to ask the load balancer for a new server.
        //Do this before we increment the counters because the first call to this method
        //is not a retry it is just an initial failure.
        if(!canRetrySameServer(context)  && canRetryNextServer(context)) {
            context.setServiceInstance(loadBalanceChooser.choose(serviceId));
        }
        //This method is called regardless of whether we are retrying or making the first request.
        //Since we do not count the initial request in the retry count we don't reset the counter
        //until we actually equal the same server count limit.  This will allow us to make the initial
        //request plus the right number of retries.
        if(sameServerCount >= lbContext.getRetryHandler().getMaxRetriesOnSameServer() && canRetry(context)) {
            //reset same server since we are moving to a new server
            sameServerCount = 0;
            nextServerCount++;
            if(!canRetryNextServer(context)) {
                context.setExhaustedOnly();
            }
        } else {
            sameServerCount++;
        }

    }
    
    private void updateServerInstanceStats(LoadBalancedRetryContext context) {
        ServiceInstance serviceInstance = context.getServiceInstance();
        if (serviceInstance instanceof RibbonServer) {
            Server lbServer = ((RibbonServer)serviceInstance).getServer();
            ServerStats serverStats = lbContext.getServerStats(lbServer);
            serverStats.incrementSuccessiveConnectionFailureCount();
            serverStats.addToFailureCount();                    
            LOGGER.debug(lbServer.getHostPort() + " RetryCount: " + context.getRetryCount() 
                + " Successive Failures: " + serverStats.getSuccessiveConnectionFailureCount() 
                + " CirtuitBreakerTripped:" + serverStats.isCircuitBreakerTripped());
        }
    }

    @Override
    public boolean retryableStatusCode(int statusCode) {
        return retryableStatusCodes.contains(statusCode);
    }
}
```
`canRetry`方法中判断如果HttpMethod`是GET`则直接重试。如果`不是GET`，要去判断Ribbon的参数`isOkToRetryOnAllOperations`，是否允许所有操作重试(这个我想跟Restful的API规范有关系)。
所以结论就来了，如果是GET方法的重试，直接开启`spring.cloud.loadbalancer.retry.enabled`参数即可，
如果我们想开启非GET方法的重试，还需要开启Ribbon的`isOkToRetryOnAllOperations`。

## 0x04 RetryLoadBalancerInterceptor
```java
// spring-cloud-commons o.s.c.client.loadbalancer
public class RetryLoadBalancerInterceptor implements ClientHttpRequestInterceptor {

    ...

    @Override
    public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
                                        final ClientHttpRequestExecution execution) throws IOException {
        final URI originalUri = request.getURI();
        final String serviceName = originalUri.getHost();
        Assert.state(serviceName != null, "Request URI does not contain a valid hostname: " + originalUri);
        final LoadBalancedRetryPolicy retryPolicy = lbRetryPolicyFactory.create(serviceName,
                loadBalancer);
        RetryTemplate template = this.retryTemplate == null ? new RetryTemplate() : this.retryTemplate;
        BackOffPolicy backOffPolicy = backOffPolicyFactory.createBackOffPolicy(serviceName);
        template.setBackOffPolicy(backOffPolicy == null ? new NoBackOffPolicy() : backOffPolicy);
        template.setThrowLastExceptionOnExhausted(true);
        RetryListener[] retryListeners = this.retryListenerFactory.createRetryListeners(serviceName);
               if (retryListeners != null && retryListeners.length != 0) {
                   template.setListeners(retryListeners);
               }
        template.setRetryPolicy(
                !lbProperties.isEnabled() || retryPolicy == null ? new NeverRetryPolicy()
                        : new InterceptorRetryPolicy(request, retryPolicy, loadBalancer,
                        serviceName));
        return template
                .execute(new RetryCallback<ClientHttpResponse, IOException>() {
                    @Override
                    public ClientHttpResponse doWithRetry(RetryContext context)
                            throws IOException {
                        ServiceInstance serviceInstance = null;
                        if (context instanceof LoadBalancedRetryContext) {
                            LoadBalancedRetryContext lbContext = (LoadBalancedRetryContext) context;
                            serviceInstance = lbContext.getServiceInstance();
                        }
                        if (serviceInstance == null) {
                            serviceInstance = loadBalancer.choose(serviceName);
                        }
                        ClientHttpResponse response = RetryLoadBalancerInterceptor.this.loadBalancer.execute(
                                serviceName, serviceInstance,
                                requestFactory.createRequest(request, body, execution));
                        int statusCode = response.getRawStatusCode();
                        if (retryPolicy != null && retryPolicy.retryableStatusCode(statusCode)) {
                            throw new ClientHttpResponseStatusCodeException(serviceName, response);
                        }
                        return response;
                    }
                }, new RibbonRecoveryCallback<ClientHttpResponse, ClientHttpResponse>() {
                    @Override
                    protected ClientHttpResponse createResponse(ClientHttpResponse response, URI uri) {
                        return response;
                    }
                });
    }
}
```
拦截器`RetryLoadBalancerInterceptor`与`openfeign`的`FeignLoadBalancer`的`execute`和逻辑差不多。