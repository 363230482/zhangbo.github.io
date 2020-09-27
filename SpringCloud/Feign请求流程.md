# Feign 请求流程

## 整体流程
```
// 开启hystrix： feign.hystrix.enabled=true
feign.hystrix.HystrixInvocationHandler#invoke
// new HystrixCommand()进行HystrixCommand初始化流程
com.netflix.hystrix.HystrixCommand#execute
com.netflix.hystrix.HystrixCommand#queue
com.netflix.hystrix.AbstractCommand#toObservable
// request cache:默认true,但cacheKey为null，一样不会缓存
// 配置全局项 hystrix.command.default.requestCache.enabled=false
// 配置单个接口 hystrix.command.UserApi#get(Long).requestCache.enabled=false
com.netflix.hystrix.AbstractCommand#isRequestCachingEnabled
// hystrix监控
com.netflix.hystrix.AbstractCommand#applyHystrixSemantics
// 熔断判断
com.netflix.hystrix.HystrixCircuitBreaker#allowRequest
com.netflix.hystrix.AbstractCommand#executeCommandAndObserve
// 处理fallback
com.netflix.hystrix.AbstractCommand#handleThreadPoolRejectionViaFallback/handleTimeoutViaFallback/handleBadRequestByEmittingError/handleFailureViaFallback
// 选择执行隔离策略：THREAD OR SEMAPHORE：hystrix异步线程中执行
com.netflix.hystrix.AbstractCommand#executeCommandWithSpecifiedIsolation
com.netflix.hystrix.AbstractCommand#getUserExecutionObservable
com.netflix.hystrix.HystrixCommand#getExecutionObservable
// 开始执行真正的command
com.netflix.hystrix.HystrixCommand#run
// HystrixCommand.run：SynchronousMethodHandler为每个api method对应一个feign.hystrix.HystrixInvocationHandler#dispatch Map<Method, MethodHandler>
feign.SynchronousMethodHandler#invoke
// 转换请求参数为 RequestTemplate
feign.ReflectiveFeign.BuildTemplateByResolvingArgs#create
feign.SynchronousMethodHandler#executeAndDecode
// 自定义的RequestInterceptor执行,并转换RequestTemplate为feign.Request
feign.SynchronousMethodHandler#targetRequest
feign.RequestInterceptor#apply
feign.Target.HardCodedTarget#apply
org.springframework.cloud.openfeign.ribbon.LoadBalancerFeignClient#execute
com.netflix.client.AbstractLoadBalancerAwareClient#executeWithLoadBalancer(S, com.netflix.client.config.IClientConfig)
com.netflix.loadbalancer.reactive.LoadBalancerCommand#submit

// feign 负载均衡
com.netflix.loadbalancer.reactive.LoadBalancerCommand#submit
com.netflix.loadbalancer.reactive.LoadBalancerCommand#selectServer
com.netflix.loadbalancer.LoadBalancerContext#getServerFromLoadBalancer
com.netflix.loadbalancer.ZoneAwareLoadBalancer#chooseServer
com.netflix.loadbalancer.BaseLoadBalancer#chooseServer
com.netflix.loadbalancer.PredicateBasedRule#choose
com.netflix.loadbalancer.AbstractServerPredicate#chooseRoundRobinAfterFiltering(java.util.List<com.netflix.loadbalancer.Server>, java.lang.Object)
com.netflix.loadbalancer.AbstractServerPredicate#incrementAndGetModulo

// 
com.netflix.loadbalancer.reactive.LoadBalancerCommand#submit
com.netflix.loadbalancer.LoadBalancerContext#reconstructURIWithServer
org.springframework.cloud.openfeign.ribbon.FeignLoadBalancer#execute
feign.Client.Default#execute
// 默认使用 HttpURLConnection，可配置OKhttp等:feign.okhttp.enabled=true / feign.httpclient.enabled=true
feign.Client.Default#convertAndSend 
feign.Client.Default#convertResponse

```

### fallback流程
```
feign.hystrix.HystrixInvocationHandler#invoke
com.netflix.hystrix.HystrixCommand#execute
com.netflix.hystrix.HystrixCommand#queue
com.netflix.hystrix.AbstractCommand#toObservable
com.netflix.hystrix.AbstractCommand#applyHystrixSemantics
com.netflix.hystrix.AbstractCommand#executeCommandAndObserve 
// 多分支exception处理
com.netflix.hystrix.AbstractCommand#handleShortCircuitViaFallback/com.netflix.hystrix.AbstractCommand#handleSemaphoreRejectionViaFallback
com.netflix.hystrix.AbstractCommand#getFallbackOrThrowException
com.netflix.hystrix.AbstractCommand#getFallbackObservable
com.netflix.hystrix.HystrixCommand#getFallback

// 隔离策略：THREAD OR SEMAPHORE 配置项:hystrix.command.default.execution.isolation.strategy
com.netflix.hystrix.AbstractCommand#executeCommandAndObserve
com.netflix.hystrix.AbstractCommand#executeCommandWithSpecifiedIsolation

```

### 转换请求参数为 RequestTemplate
```
feign.SynchronousMethodHandler#invoke
// 转换请求参数为 RequestTemplate
feign.ReflectiveFeign.BuildTemplateByResolvingArgs#create
feign.ReflectiveFeign.BuildTemplateByResolvingArgs#expandElements
org.springframework.cloud.openfeign.support.SpringMvcContract.ConvertingExpanderFactory#getExpander
org.springframework.core.convert.support.GenericConversionService#convert(java.lang.Object, org.springframework.core.convert.TypeDescriptor, org.springframework.core.convert.TypeDescriptor)
// 设置参数到 RequestTemplate
feign.ReflectiveFeign.BuildTemplateByResolvingArgs#resolve

org.springframework.cloud.openfeign.support.SpringEncoder#encode
```

## 未启用hystrix
```
feign.ReflectiveFeign.FeignInvocationHandler#invoke
feign.SynchronousMethodHandler#invoke
feign.SynchronousMethodHandler#executeAndDecode
// 自定义的RequestInterceptor执行,并转换RequestTemplate为feign.Request
feign.SynchronousMethodHandler#targetRequest
feign.RequestInterceptor#apply
feign.Target.HardCodedTarget#apply
org.springframework.cloud.openfeign.ribbon.LoadBalancerFeignClient#execute
com.netflix.client.AbstractLoadBalancerAwareClient#executeWithLoadBalancer(S, com.netflix.client.config.IClientConfig)

// 提交任务，loadbalance
com.netflix.loadbalancer.reactive.LoadBalancerCommand#submit
com.netflix.loadbalancer.reactive.LoadBalancerCommand#selectServer
com.netflix.loadbalancer.LoadBalancerContext#getServerFromLoadBalancer
com.netflix.loadbalancer.ZoneAwareLoadBalancer#chooseServer
com.netflix.loadbalancer.BaseLoadBalancer#chooseServer
com.netflix.loadbalancer.PredicateBasedRule#choose
com.netflix.loadbalancer.AbstractServerPredicate#chooseRoundRobinAfterFiltering(java.util.List<com.netflix.loadbalancer.Server>, java.lang.Object)
com.netflix.loadbalancer.AbstractServerPredicate#incrementAndGetModulo
// 重新构建uri
org.springframework.cloud.openfeign.ribbon.FeignLoadBalancer#reconstructURIWithServer
com.netflix.loadbalancer.LoadBalancerContext#reconstructURIWithServer
// 最终执行request
org.springframework.cloud.openfeign.ribbon.FeignLoadBalancer#execute
// 默认是 HttpURLConnection,可以引用 feign-httpclient / feign-okhttp 依赖包,并配置 feign.okhttp.enabled=true 或 feign.httpclient.enabled=true启用对应的client
feign.Client.Default#execute

```

## 启动时初始化流程
```
// 启动流程
org.springframework.beans.factory.support.DefaultListableBeanFactory#addCandidateEntry
org.springframework.beans.factory.config.DependencyDescriptor#resolveCandidate
org.springframework.beans.factory.support.AbstractBeanFactory#getBean(java.lang.String)
org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#getObjectForBeanInstance
org.springframework.beans.factory.support.AbstractBeanFactory#getObjectForBeanInstance
org.springframework.beans.factory.support.FactoryBeanRegistrySupport#getObjectFromFactoryBean
org.springframework.beans.factory.support.FactoryBeanRegistrySupport#doGetObjectFromFactoryBean
org.springframework.cloud.openfeign.FeignClientFactoryBean#getObject
org.springframework.cloud.openfeign.FeignClientFactoryBean#getTarget
org.springframework.cloud.openfeign.FeignClientFactoryBean#loadBalance
org.springframework.cloud.openfeign.HystrixTargeter#targetWithFallbackFactory
feign.hystrix.HystrixFeign.Builder#target(feign.Target<T>, feign.hystrix.FallbackFactory<? extends T>)
feign.ReflectiveFeign#newInstance
feign.InvocationHandlerFactory#create
feign.hystrix.HystrixInvocationHandler#new

// HystrixCommand初始化流程
threadPool, 默认以服务为单位隔离，corePoolSize=10,maxPoolSize=10,workQueue=SynchronousQueue
com.netflix.hystrix.HystrixThreadPoolProperties#default_coreSize=10 // 配置项:hystrix.command.threadpool.default.coreSize
com.netflix.hystrix.HystrixThreadPoolProperties#default_maximumSize=10
com.netflix.hystrix.HystrixThreadPoolProperties#default_maxQueueSize=-1
com.netflix.hystrix.strategy.concurrency.HystrixConcurrencyStrategy#getBlockingQueue // 选择任务队列:<=0时为SynchronousQueue,大于0时 LinkedBlockingQueue

```

```
// hystrix异步线程开始
schedule:166, HystrixContextScheduler$ThreadPoolWorker (com.netflix.hystrix.strategy.concurrency)
schedule:106, HystrixContextScheduler$HystrixContextSchedulerWorker (com.netflix.hystrix.strategy.concurrency)
call:50, OperatorSubscribeOn (rx.internal.operators)
call:30, OperatorSubscribeOn (rx.internal.operators)
call:48, OnSubscribeLift (rx.internal.operators)
call:30, OnSubscribeLift (rx.internal.operators)
unsafeSubscribe:10327, Observable (rx)
call:41, OnSubscribeDoOnEach (rx.internal.operators)
call:30, OnSubscribeDoOnEach (rx.internal.operators)
unsafeSubscribe:10327, Observable (rx)
call:41, OnSubscribeDoOnEach (rx.internal.operators)
call:30, OnSubscribeDoOnEach (rx.internal.operators)
call:48, OnSubscribeLift (rx.internal.operators)
call:30, OnSubscribeLift (rx.internal.operators)
unsafeSubscribe:10327, Observable (rx)
call:41, OnSubscribeDoOnEach (rx.internal.operators)
call:30, OnSubscribeDoOnEach (rx.internal.operators)
unsafeSubscribe:10327, Observable (rx)
call:41, OnSubscribeDoOnEach (rx.internal.operators)
call:30, OnSubscribeDoOnEach (rx.internal.operators)
unsafeSubscribe:10327, Observable (rx)
call:41, OnSubscribeDoOnEach (rx.internal.operators)
call:30, OnSubscribeDoOnEach (rx.internal.operators)
call:48, OnSubscribeLift (rx.internal.operators)
call:30, OnSubscribeLift (rx.internal.operators)
unsafeSubscribe:10327, Observable (rx)
call:51, OnSubscribeDefer (rx.internal.operators)
call:35, OnSubscribeDefer (rx.internal.operators)
unsafeSubscribe:10327, Observable (rx)
call:48, OnSubscribeMap (rx.internal.operators)
call:33, OnSubscribeMap (rx.internal.operators)
unsafeSubscribe:10327, Observable (rx)
call:41, OnSubscribeDoOnEach (rx.internal.operators)
call:30, OnSubscribeDoOnEach (rx.internal.operators)
call:48, OnSubscribeLift (rx.internal.operators)
call:30, OnSubscribeLift (rx.internal.operators)
unsafeSubscribe:10327, Observable (rx)
call:41, OnSubscribeDoOnEach (rx.internal.operators)
call:30, OnSubscribeDoOnEach (rx.internal.operators)
unsafeSubscribe:10327, Observable (rx)
call:51, OnSubscribeDefer (rx.internal.operators)
call:35, OnSubscribeDefer (rx.internal.operators)
call:48, OnSubscribeLift (rx.internal.operators)
call:30, OnSubscribeLift (rx.internal.operators)
subscribe:10423, Observable (rx)
subscribe:10390, Observable (rx)
toFuture:51, BlockingOperatorToFuture (rx.internal.operators)
toFuture:410, BlockingObservable (rx.observables)
queue:378, HystrixCommand (com.netflix.hystrix)
execute:344, HystrixCommand (com.netflix.hystrix)
invoke:170, HystrixInvocationHandler (feign.hystrix)
get:-1, $Proxy88 (com.sun.proxy)
getByUserId:42, GoodsController (com.zb.demo.goods.controller)


rx.internal.operators.OperatorSubscribeOn#call
com.netflix.hystrix.strategy.concurrency.HystrixContextScheduler.HystrixContextSchedulerWorker#schedule(rx.functions.Action0)
com.netflix.hystrix.strategy.concurrency.HystrixContextScheduler.ThreadPoolWorker#schedule(rx.functions.Action0)
rx.internal.schedulers.ScheduledAction#new
rx.internal.schedulers.ScheduledAction#run
com.netflix.hystrix.strategy.concurrency.HystrixContexSchedulerAction#call
com.netflix.hystrix.AbstractCommand#executeCommandWithSpecifiedIsolation ## rx.functions.Func0#call

```