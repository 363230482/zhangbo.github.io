# SpringMVC请求流程
```
org.springframework.web.servlet.DispatcherServlet#doDispatch
//1. 查找handler
org.springframework.web.servlet.DispatcherServlet#getHandler
    org.springframework.web.servlet.handler.AbstractHandlerMapping#getHandler
    org.springframework.web.servlet.mvc.method.RequestMappingInfoHandlerMapping#getHandlerInternal
    org.springframework.web.servlet.handler.AbstractHandlerMethodMapping#getHandlerInternal
        //1.1 解析请求路径
        org.springframework.web.util.UrlPathHelper#getLookupPathForRequest(javax.servlet.http.HttpServletRequest)
        // 根据路径查找handler
        org.springframework.web.servlet.handler.AbstractHandlerMethodMapping#lookupHandlerMethod
            //1.1.1 直接通过url在urlLookup map中get
            org.springframework.web.servlet.handler.AbstractHandlerMethodMapping.MappingRegistry#getMappingsByUrl
            org.springframework.web.servlet.handler.AbstractHandlerMethodMapping.MappingRegistry#urlLookup#get(urlPath)
            //1.1.2 轮询所有的RequestMappingInfo
            org.springframework.web.servlet.handler.AbstractHandlerMethodMapping#addMatchingMappings
            org.springframework.web.servlet.mvc.method.RequestMappingInfoHandlerMapping#getMatchingMapping
            org.springframework.web.servlet.mvc.method.RequestMappingInfo#getMatchingCondition
            //1.1.3 抛出HttpRequestMethodNotSupportedException/HttpMediaTypeNotSupportedException异常，否则返回null
            org.springframework.web.servlet.handler.AbstractHandlerMethodMapping#handleNoMatch

        //1.2 根据查找到的handlerMethod,add对应的interceptor到HandlerExecutionChain
    org.springframework.web.servlet.handler.AbstractHandlerMapping#getHandlerExecutionChain

//  if (mappedHandler == null) 处理404后直接return;
org.springframework.web.servlet.DispatcherServlet#noHandlerFound

//2. interceptor#preHandle,如返回false，直接执行afterCompletion，然后在DispatcherServlet#doDispatch方法中直接return;
org.springframework.web.servlet.HandlerExecutionChain#applyPreHandle
//3. 执行handle
org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter#handle
    org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter#handleInternal
    org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter#invokeHandlerMethod
    org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod#invokeAndHandle
    org.springframework.web.method.support.InvocableHandlerMethod#invokeForRequest
        //3.1 解析参数
        org.springframework.web.method.support.InvocableHandlerMethod#getMethodArgumentValues
        org.springframework.web.method.support.HandlerMethodArgumentResolverComposite#resolveArgument
            //3.1.1 解析普通name value及path variable
            org.springframework.web.method.annotation.AbstractNamedValueMethodArgumentResolver#resolveArgument
            //3.1.2 request body
            org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor#resolveArgument
            org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor#readWithMessageConverters
            org.springframework.web.servlet.mvc.method.annotation.AbstractMessageConverterMethodArgumentResolver#readWithMessageConverters(org.springframework.http.HttpInputMessage, org.springframework.core.MethodParameter, java.lang.reflect.Type)
            // 
            org.springframework.web.servlet.mvc.method.annotation.RequestBodyAdvice#beforeBodyRead
            org.springframework.http.converter.json.AbstractJackson2HttpMessageConverter#read
            org.springframework.web.servlet.mvc.method.annotation.RequestBodyAdvice#afterBodyRead
        //3.2 反射调用controller
        org.springframework.web.method.support.InvocableHandlerMethod#doInvoke
        sun.reflect.MethodAccessor.invoke
    //3.3 处理返回值，ResponseBody等
    org.springframework.web.method.support.HandlerMethodReturnValueHandlerComposite#handleReturnValue
    org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor#handleReturnValue
        org.springframework.web.servlet.mvc.method.annotation.AbstractMessageConverterMethodProcessor#writeWithMessageConverters(T, org.springframework.core.MethodParameter, org.springframework.http.server.ServletServerHttpRequest, org.springframework.http.server.ServletServerHttpResponse)
        org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyAdviceChain#beforeBodyWrite
        org.springframework.http.converter.AbstractGenericHttpMessageConverter#write
        org.springframework.http.converter.json.AbstractJackson2HttpMessageConverter#writeInternal
        

org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter#getModelAndView
//4. interceptor#postHandle
org.springframework.web.servlet.HandlerExecutionChain#applyPostHandle
//5. resolve exception or render view
org.springframework.web.servlet.DispatcherServlet#processDispatchResult
    //5.1 resolve exception
    ExceptionHandler也是通过ServletInvocableHandlerMethod#invokeAndHandle，然后经过message converter等返回view
    org.springframework.web.servlet.DispatcherServlet#processHandlerException
        org.springframework.web.servlet.handler.HandlerExceptionResolverComposite#resolveException
        org.springframework.web.servlet.handler.AbstractHandlerExceptionResolver#resolveException
        org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver#doResolveHandlerMethodException
    //5.2 render view
    org.springframework.web.servlet.DispatcherServlet#render

//6. interceptor#afterCompletion
org.springframework.web.servlet.HandlerExecutionChain#triggerAfterCompletion
// 最终异常处理
org.apache.catalina.core.StandardWrapperValve#exception
```


## 查找 HandlerMethod
```
protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {
    List<Match> matches = new ArrayList<>();
    // 先直接根据url在map中查找，可能会有多个而request method不同
    List<T> directPathMatches = this.mappingRegistry.getMappingsByUrl(lookupPath);
    if (directPathMatches != null) {
        // 根据request method，produce，consume，header，path pattern等判断过滤
        addMatchingMappings(directPathMatches, matches, request);
    }
    if (matches.isEmpty()) {
        // 如果上一步没有找到，则在全部requestMapping中查找
        addMatchingMappings(this.mappingRegistry.getMappings().keySet(), matches, request);
    }

    if (!matches.isEmpty()) {
        Match bestMatch = matches.get(0);
        if (matches.size() > 1) {
            Comparator<Match> comparator = new MatchComparator(getMappingComparator(request));
            matches.sort(comparator);
            bestMatch = matches.get(0);
            if (logger.isTraceEnabled()) {
                logger.trace(matches.size() + " matching mappings: " + matches);
            }
            if (CorsUtils.isPreFlightRequest(request)) {
                return PREFLIGHT_AMBIGUOUS_MATCH;
            }
            Match secondBestMatch = matches.get(1);
            if (comparator.compare(bestMatch, secondBestMatch) == 0) {
                Method m1 = bestMatch.handlerMethod.getMethod();
                Method m2 = secondBestMatch.handlerMethod.getMethod();
                String uri = request.getRequestURI();
                throw new IllegalStateException(
                        "Ambiguous handler methods mapped for '" + uri + "': {" + m1 + ", " + m2 + "}");
            }
        }
        request.setAttribute(BEST_MATCHING_HANDLER_ATTRIBUTE, bestMatch.handlerMethod);
        handleMatch(bestMatch.mapping, lookupPath, request);
        return bestMatch.handlerMethod;
    }
    else {
        return handleNoMatch(this.mappingRegistry.getMappings().keySet(), lookupPath, request);
    }
}

```