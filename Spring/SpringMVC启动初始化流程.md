# SpringMVC启动时的初始化流程

```
org.springframework.web.servlet.handler.AbstractHandlerMethodMapping#afterPropertiesSet
org.springframework.web.servlet.handler.AbstractHandlerMethodMapping#initHandlerMethods
protected void initHandlerMethods() {
    for (String beanName : getCandidateBeanNames()) {
        if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
            processCandidateBean(beanName);
        }
    }
    handlerMethodsInitialized(getHandlerMethods());
}
org.springframework.web.servlet.handler.AbstractHandlerMethodMapping#processCandidateBean
// 将BeanFactory中所有的bean，判断isHandler
// (AnnotatedElementUtils.hasAnnotation(beanType, Controller.class) || AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class))
org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping#isHandler
// 
org.springframework.web.servlet.handler.AbstractHandlerMethodMapping#detectHandlerMethods
org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping#getMappingForMethod
// create RequestMappingInfo
org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping#createRequestMappingInfo(java.lang.reflect.AnnotatedElement)
org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping#createRequestMappingInfo(org.springframework.web.bind.annotation.RequestMapping, org.springframework.web.servlet.mvc.condition.RequestCondition<?>)

// register
org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping#registerHandlerMethod
org.springframework.web.servlet.handler.AbstractHandlerMethodMapping#registerHandlerMethod
org.springframework.web.servlet.handler.AbstractHandlerMethodMapping.MappingRegistry#register
// create handlerMethod
org.springframework.web.servlet.handler.AbstractHandlerMethodMapping#createHandlerMethod
// validate,同一个请求路径，不能有两个method
org.springframework.web.servlet.handler.AbstractHandlerMethodMapping.MappingRegistry#validateMethodMapping
// real register
org.springframework.web.servlet.handler.AbstractHandlerMethodMapping.MappingRegistry#mappingLookup.put(mapping, handlerMethod);
org.springframework.web.servlet.handler.AbstractHandlerMethodMapping.MappingRegistry#urlLookup.add(url, mapping);
org.springframework.web.servlet.handler.AbstractHandlerMethodMapping.MappingRegistry#corsLookup.put(handlerMethod, corsConfig);
org.springframework.web.servlet.handler.AbstractHandlerMethodMapping.MappingRegistry#registry.put(mapping, new MappingRegistration<>(mapping, handlerMethod, directUrls, name));

```

```
// 初始化 exception handler
org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver#afterPropertiesSet
org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver#initExceptionHandlerAdviceCache
org.springframework.web.method.ControllerAdviceBean#findAnnotatedBeans
```