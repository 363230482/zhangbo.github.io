# Feign 优化
1.默认使用HttpURLConnection发起请求，没有连接池，可使用OkHttp或HTTPClient等
```yaml
feign:
  okhttp:
    enabled: true
```

2.feign在第一次请求时，会初始化很多对象，导致超时，合理配置超时时间,连接数等
```yaml
feign:
  httpclient:
    connection-timeout: 2000
    max-connections: 20
```

3.开启HTTP请求压缩功能
```yaml
feign:
  compression:
    request:
      enabled: true
    response:
      enabled: true
      useGzipDecoder: true
```

4.开启hystrix
```yaml
feign:
  hystrix:
    enabled: true
```

5.合理设置hystrix相关的线程池参数
```yaml
hystrix.command.default.execution.isolation.strategy=SEMAPHORE  // 默认线程池隔离 THREAD
hystrix.command.threadpool.default.coreSize=10
hystrix.command.execution.isolation.thread.timeoutInMilliseconds=1000
```