开启hystrix后，由于hystrix默认是线程隔离的，导致RequestContextHolder无法获取到当前的request对象。  

#解决方案一
设置hystrix的隔离类型为`SEMAPHORE`
```properties
hystrix.command.default.execution.isolation.strategy=SEMAPHORE
```

#解决方案二
自定义`HystrixConcurrencyStrategy`策略  
## 2.1实现策略类
```java
public class RequestContextHystrixConcurrencyStrategy extends HystrixConcurrencyStrategy {
    @Override
    public <T> Callable<T> wrapCallable(Callable<T> callable) {
        return new RequestAttributeAwareCallable<>(callable, RequestContextHolder.getRequestAttributes());
    }

    static class RequestAttributeAwareCallable<T> implements Callable<T> {

        private final Callable<T> delegate;
        private final RequestAttributes requestAttributes;

        public RequestAttributeAwareCallable(Callable<T> callable, RequestAttributes requestAttributes) {
            this.delegate = callable;
            this.requestAttributes = requestAttributes;
        }

        @Override
        public T call() throws Exception {
            try {
                // 设置requestAttributes到当前的请求上下文
                RequestContextHolder.setRequestAttributes(requestAttributes);
                return delegate.call();
            } finally {
                RequestContextHolder.resetRequestAttributes();
            }
        }
    }
}
```

### 2.2注册策略类
在工程的classpath下引入hystrix-plugins.properties配置文件，hystrix会默认读取hystrix-plugins.properties配置文件并生效自定义的plugins：  
```properties
hystrix.plugin.HystrixConcurrencyStrategy.implementation=com.xx.config.RequestContextHystrixConcurrencyStrategy
```

### 2.3完成
此时，就可以获取到request了。
```java
@Component
@Slf4j
public class FeignAccessTokenRequestInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate requestTemplate) {
        // 请求MediaType
        requestTemplate.header("Content-Type", MediaType.APPLICATION_JSON_UTF8.toString());
        requestTemplate.header("Accept", MediaType.APPLICATION_JSON_UTF8.toString());

        try {
            HttpServletRequest request = ((ServletRequestAttributes)RequestContextHolder.currentRequestAttributes()).getRequest();
            if (null != request) {
                // X-Auth-Token
                requestTemplate.header("X-Auth-Token", request.getHeader("X-Auth-Token"));
                // Authorization
                requestTemplate.header("Authorization", request.getHeader("Authorization"));
            }
        } catch (Exception e) {
            log.warn("getRequest fail!", e);
        }
    }
}
```