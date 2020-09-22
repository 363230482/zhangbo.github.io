# SpringCloudGateway 核心流程

## DispatcherHandler 核心方法
```
public Mono<Void> handle(ServerWebExchange exchange) {
    // handlerMappings 默认有4个：routerFunctionMapping,requestMappingHandlerMapping,routePredicateHandlerMapping,resourceHandlerMapping(SimpleUrlHandlerMapping)
    // 其中requestMappingHandlerMapping为类似MVC的普通controller转发，routePredicateHandlerMapping为gateway网关的handler
    if (this.handlerMappings == null) {
        return createNotFoundError();
    }
    return Flux.fromIterable(this.handlerMappings)
            .concatMap(mapping -> mapping.getHandler(exchange))
            .next()
            .switchIfEmpty(createNotFoundError())
            .flatMap(handler -> invokeHandler(exchange, handler))
            .flatMap(result -> handleResult(exchange, result));
}
```

## RoutePredicateHandlerMapping 查找handler及本次请求对应的route
```
// 通过配置的route，循环查找对应此次请求的route
org.springframework.cloud.gateway.handler.RoutePredicateHandlerMapping#lookupRoute
org.springframework.cloud.gateway.handler.predicate.GatewayPredicate#test
// 通过配置的predicates，找到对应的route，然后返回 FilteringWebHandler
org.springframework.cloud.gateway.handler.RoutePredicateHandlerMapping#getHandlerInternal
```

## FilteringWebHandler 处理本次请求(Filter链)
```
org.springframework.web.reactive.handler.AbstractHandlerMapping#getHandler
org.springframework.web.reactive.DispatcherHandler#invokeHandler
org.springframework.web.reactive.result.SimpleHandlerAdapter#handle
org.springframework.cloud.gateway.handler.FilteringWebHandler#handle
// 开始执行filter
org.springframework.cloud.gateway.handler.FilteringWebHandler.DefaultGatewayFilterChain#filter

RemoveCachedBodyFilter      // doFinally中释放DataBuffer
AdaptCachedBodyGlobalFilter
NettyWriteResponseFilter    // 在响应完成后，从netty的响应流中读取响应内容，并写入到ServerHttpResponse
ForwardPathFilter           // 如果route的scheme为forward，则设置request path，否则啥也不做(一般route为lb://xxx-server,所以scheme为lb)
StripPrefixGatewayFilterFactory // 配置文件中定义,根据配置的StripPrefix=n，去掉uri前面的n个部分(以/分割)
PreserveHostHeaderGatewayFilterFactory // 配置文件中定义,仅在exchange attributes中设置preserveHostHeader=true的标志,保留request header中的host属性
RouteToRequestUrlFilter     // 将原始的uri(http://127.0.0.1:10010/api/v1/goods/1)转换为lb://goods-server/api/v1/goods/1 格式,并放在exchange attr(gatewayRequestUrl)中
LoadBalancerClientFilter    // 通过org.springframework.cloud.gateway.filter.LoadBalancerClientFilter#choose选择合适的服务方,并将uri替换为对应服务的uri(http://127.0.0.1:10030/api/v1/goods/1)
WebsocketRoutingFilter      // 如果当前请求为ws或wss的websocket请求，则使用该filter，普通HTTP等其他请求，直接放过
NettyRoutingFilter          // 发起对应服务的远程调用,这里会用到preserveHostHeader用于设置host
ForwardRoutingFilter        // 如果本次请求为forward，则转发
```

## LoadBalance 
```
# loadbalance
org.springframework.cloud.netflix.ribbon.RibbonLoadBalancerClient#choose(java.lang.String serviceId)
org.springframework.cloud.netflix.ribbon.RibbonLoadBalancerClient#choose(java.lang.String, java.lang.Object)
org.springframework.cloud.netflix.ribbon.RibbonLoadBalancerClient#getLoadBalancer
org.springframework.cloud.netflix.ribbon.SpringClientFactory#getLoadBalancer
org.springframework.cloud.netflix.ribbon.SpringClientFactory#getInstance
org.springframework.cloud.netflix.ribbon.RibbonLoadBalancerClient#getServer(com.netflix.loadbalancer.ILoadBalancer, java.lang.Object)
com.netflix.loadbalancer.ZoneAwareLoadBalancer#chooseServer
com.netflix.loadbalancer.PredicateBasedRule#choose
com.netflix.loadbalancer.AbstractServerPredicate#chooseRoundRobinAfterFiltering(java.util.List<com.netflix.loadbalancer.Server>, java.lang.Object)
com.netflix.loadbalancer.AbstractServerPredicate#incrementAndGetModulo 
com.netflix.loadbalancer.DynamicServerListLoadBalancer
// ===== incrementAndGetModulo ===== //
private int incrementAndGetModulo(int modulo) {
    for (;;) {
        int current = nextIndex.get();
        int next = (current + 1) % modulo;
        if (nextIndex.compareAndSet(current, next) && current < modulo)
            return current;
    }
}
// ===== incrementAndGetModulo ===== //

// 将原始请求URL转换为真正的服务的地址
org.springframework.cloud.netflix.ribbon.RibbonLoadBalancerClient#reconstructURI
com.netflix.loadbalancer.LoadBalancerContext#reconstructURIWithServer



DynamicServerListLoadBalancer:{NFLoadBalancer:name=goods-server,current list of Servers=[10.7.0.17:10030],Load balancer stats=Zone stats: {unknown=[Zone:unknown;	Instance count:1;	Active connections count: 0;	Circuit breaker tripped count: 0;	Active connections per server: 0.0;]
},Server stats: [[Server:10.7.0.17:10030;	Zone:UNKNOWN;	Total Requests:0;	Successive connection failure:0;	Total blackout seconds:0;	Last connection made:Thu Jan 01 08:00:00 CST 1970;	First connection made: Thu Jan 01 08:00:00 CST 1970;	Active Connections:0;	total failure count in last (1000) msecs:0;	average resp time:0.0;	90 percentile resp time:0.0;	95 percentile resp time:0.0;	min resp time:0.0;	max resp time:0.0;	stddev resp time:0.0]
]}ServerList:com.alibaba.cloud.nacos.ribbon.NacosServerList@2225db6f
```

## 其他
在配置文件中，通过配置predicates或者filter的名称前缀，找到对应的Predicates及Filter实现
```
// 处理predicate和filter的名称前缀
org.springframework.cloud.gateway.support.NameUtils#normalizeFilterFactoryName
org.springframework.cloud.gateway.support.NameUtils#normalizeRoutePredicateName
```

```
// springcloud中，每个服务对应一个独立的context，使有的bean在独立的子context中
org.springframework.cloud.context.named.NamedContextFactory

org.springframework.cloud.context.named.NamedContextFactory#getContext(serviceId)
org.springframework.cloud.context.named.NamedContextFactory#createContext
org.springframework.context.annotation.AnnotationConfigApplicationContext#register(class com.alibaba.cloud.nacos.ribbon.NacosRibbonClientConfiguration)

org.springframework.beans.factory.support.DefaultListableBeanFactory#beanDefinitionNames
```

## 更新serverList 流程
```
# com.netflix.loadbalancer.BaseLoadBalancer,该实例在第一次请求到达时，初始化服务对应的context时初始化
// ping各个服务：默认间隔时间 30s，默认不ping：com.netflix.loadbalancer.DummyPing
com.netflix.loadbalancer.BaseLoadBalancer#setupPingTask

// BaseLoadBalancer获取serverList和nacos获取server是异步的，BaseLoadBalancer直接获取nacos的本地缓存，nacos有定时任务自动更新本地缓存的server

// 注册中心获取serverList
com.netflix.loadbalancer.DynamicServerListLoadBalancer#DynamicServerListLoadBalancer(com.netflix.client.config.IClientConfig, com.netflix.loadbalancer.IRule, com.netflix.loadbalancer.IPing, com.netflix.loadbalancer.ServerList<T>, com.netflix.loadbalancer.ServerListFilter<T>, com.netflix.loadbalancer.ServerListUpdater)
com.netflix.loadbalancer.DynamicServerListLoadBalancer#restOfInit
com.netflix.loadbalancer.DynamicServerListLoadBalancer#updateListOfServers
// nacos从注册中心获取
com.alibaba.cloud.nacos.ribbon.NacosServerList#getUpdatedListOfServers
com.alibaba.cloud.nacos.ribbon.NacosServerList#getServers
com.alibaba.nacos.client.naming.NacosNamingService#selectInstances(java.lang.String, java.lang.String, boolean)
// 设置到BaseLoadBalancer
com.netflix.loadbalancer.DynamicServerListLoadBalancer#updateAllServerList
com.netflix.loadbalancer.BaseLoadBalancer#setServersList
// 设置 allServerList的同时，直接设置 upServerList //
545 allServerList = allServers;
546 if (canSkipPing()) {
547     for (Server s : allServerList) {
548         s.setAlive(true);
549     }
550     upServerList = allServerList;
551 }

// 定时更新 allServerList 流程
com.netflix.loadbalancer.DynamicServerListLoadBalancer#restOfInit
com.netflix.loadbalancer.DynamicServerListLoadBalancer#enableAndInitLearnNewServersFeature
// 默认延迟1s,间隔30s更新一次(com.netflix.client.config.CommonClientConfigKey#ServerListRefreshInterval 配置项)
com.netflix.loadbalancer.PollingServerListUpdater#start
com.netflix.loadbalancer.ServerListUpdater.UpdateAction#doUpdate

// nacos更新server流程
com.alibaba.nacos.client.naming.core.HostReactor#getServiceInfo
com.alibaba.nacos.client.naming.core.HostReactor#scheduleUpdateIfAbsent
com.alibaba.nacos.client.naming.core.HostReactor#addTask
com.alibaba.nacos.client.naming.core.HostReactor.UpdateTask#run // 默认每10s更新一次
com.alibaba.nacos.client.naming.core.HostReactor#updateServiceNow
com.alibaba.nacos.client.naming.core.HostReactor#processServiceJSON // 处理nacos响应，刷新本地内存缓存及磁盘缓存
serviceInfoMap.put(key, serviceInfo)
com.alibaba.nacos.client.naming.cache.DiskCache#write

```

```
nacos启动注册服务流程
org.springframework.cloud.client.serviceregistry.AbstractAutoServiceRegistration#onApplicationEvent
org.springframework.cloud.client.serviceregistry.AbstractAutoServiceRegistration#bind
org.springframework.cloud.client.serviceregistry.AbstractAutoServiceRegistration#start
com.alibaba.cloud.nacos.registry.NacosAutoServiceRegistration#register
com.alibaba.cloud.nacos.registry.NacosServiceRegistry#register
com.alibaba.nacos.client.naming.NacosNamingService#registerInstance(java.lang.String, java.lang.String, com.alibaba.nacos.api.naming.pojo.Instance)
com.alibaba.nacos.client.naming.beat.BeatReactor#addBeatInfo // 发送心跳，默认间隔5s
com.alibaba.nacos.client.naming.net.NamingProxy#registerService
com.alibaba.nacos.client.naming.net.NamingProxy#reqAPI(java.lang.String, java.util.Map<java.lang.String,java.lang.String>, java.lang.String)
```