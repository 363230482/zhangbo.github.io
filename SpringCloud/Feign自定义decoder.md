
## feign client
```java
// 指定对应的配置类，还可以指定fallback，decode404等
@FeignClient(name = "notification-center", contextId = "sms-client", configuration = CenterConfiguration.class)
public interface SmsClient {
    
}
```

## feign 配置
```java
@RequiredArgsConstructor
public class CenterConfiguration {

    private final ObjectFactory<HttpMessageConverters> messageConverters;

    @Bean
    public Decoder decoder() {
        return new CenterDecoder(new OptionalDecoder(
                new ResponseEntityDecoder(new SpringDecoder(this.messageConverters))));
    }

    @Bean
    public ErrorDecoder errorDecoder(){
        return new CenterErrorDecoder();
    }
}
```

## decoder
```java
// decode实现在feign client中，返回的响应参数只包含业务参数，不需要公共的Result结构
public class CenterDecoder implements Decoder {

    final Decoder delegate;

    public CenterDecoder(Decoder delegate){
        Objects.requireNonNull(delegate, "Decoder must not be null. ");
        this.delegate = delegate;
    }

    @Override
    public Object decode(Response response, Type type) throws IOException{
        Object result = delegate.decode(response, TypeUtils.parameterize(Result.class, type));
        if(result instanceof Result){
            Result<?> httpResult = ((Result<?>) result);
            return httpResult.getData();
        }
        return result;
    }

}
```

## error decoder
```java
public class CenterErrorDecoder implements ErrorDecoder {

    @Override
    public Exception decode(String methodKey, Response response) {
        // 自定义错误码
        if (HttpStatus.I_AM_A_TEAPOT.value() == response.status()) {
            InputStream is = null;
            try {
                is = response.body().asInputStream();
            } catch (IOException e) {
                e.printStackTrace();
            }
            ObjectMapper objectMapper = new ObjectMapper();
            Result<?> result = null;
            try {
                result = objectMapper.readValue(is, Result.class);
            } catch (IOException e) {
                e.printStackTrace();
            }
            assert result != null;
            return new NotificationCenterException(result.getMessage());
        }
        return errorStatus(methodKey, response);
    }
}
```
