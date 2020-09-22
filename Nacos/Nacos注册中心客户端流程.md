
# Nacos注册中心客户端流程

## 启动时注册服务
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

## 后台定时更新 serviceList
```
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