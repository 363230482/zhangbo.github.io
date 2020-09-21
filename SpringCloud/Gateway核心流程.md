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
com.netflix.loadbalancer.ZoneAwareLoadBalancer#chooseServer

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
