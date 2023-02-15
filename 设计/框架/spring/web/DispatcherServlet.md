```
HttpServlet#service
    FrameworkServlet#processRequest
        DispatcherServlet#doService
            * DispatcherServlet#doDispatch
```

```
DispatcherServlet#doDispatch
    * DispatcherServlet#getHandler
        AbstractHandlerMapping#getHandler
            AbstractHandlerMethodMapping#getHandlerInternal
                AbstractHandlerMethodMapping#lookupHandlerMethod
            AbstractHandlerMethodMapping#getHandlerExecutionChain
    * DispatcherServlet#getHandlerAdapter
    * HandlerExecutionChain#applyPreHandle
    * AbstractHandlerMethodAdapter#handle
        RequestMappingHandlerAdapter#handleInternal
            RequestMappingHandlerAdapter#invokeHandlerMethod
```

## RequestMappingHandlerMapping 初始化

```
RequestMappingHandlerMapping#afterPropertiesSet
    AbstractHandlerMethodMapping#afterPropertiesSet
        AbstractHandlerMethodMapping#initHandlerMethods
            AbstractHandlerMethodMapping#processCandidateBean
                * AbstractHandlerMethodMapping#detectHandlerMethods
                    AbstractHandlerMethodMapping#registerHandlerMethod
        AbstractHandlerMethodMapping#handlerMethodsInitialized
```

## 查找目标方法

```
* DispatcherServlet#getHandler
    AbstractHandlerMapping#getHandler
        RequestMappingInfoHandlerMapping#getHandlerInternal
            AbstractHandlerMethodMapping#getHandlerInternal
                AbstractHandlerMethodMapping#lookupHandlerMethod
                    * AbstractHandlerMethodMapping#getMappingsByDirectPath
                    * AbstractHandlerMethodMapping#addMatchingMappings
                        RequestMappingInfoHandlerMapping#getMatchingMapping
                            RequestMappingInfo#getMatchingCondition
```

```java
private final AntPathMatcher antPathMatcher = new AntPathMatcher();

@GetMapping("/hello/**")
public String hello(HttpServletRequest request) {
    String path = (String) request.getAttribute(HandlerMapping.PATH_WITHIN_HANDLER_MAPPING_ATTRIBUTE);
    String pattern = (String) request.getAttribute(HandlerMapping.BEST_MATCHING_PATTERN_ATTRIBUTE);
    String name = antPathMatcher.extractPathWithinPattern(pattern, path);

    log.info("path: {}, pattern: {}, name: {}", path, pattern, name);

    return name;
}
```

## 参数格式转换

```
* AbstractHandlerMethodAdapter#handle
    RequestMappingHandlerAdapter#handleInternal
        RequestMappingHandlerAdapter#invokeHandlerMethod
            ServletInvocableHandlerMethod#invokeAndHandle
                InvocableHandlerMethod#invokeForRequest
                    InvocableHandlerMethod#getMethodArgumentValues
                        * HandlerMethodArgumentResolverComposite#resolveArgument
                            AbstractNamedValueMethodArgumentResolver#resolveArgument
                                RequestParamMethodArgumentResolver#resolveName
                                DefaultDataBinderFactory#createBinder
                                DataBinder#convertIfNecessary
```

```java
@GetMapping("/hi")
public String hi(@RequestParam("date") @DateTimeFormat(pattern="yyyy-MM-dd") Date date) {
    log.info("date: {}", date);
    SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
    return dateFormat.format(date);
}
```

## 请求 Header 处理

```
* AbstractHandlerMethodAdapter#handle
    RequestMappingHandlerAdapter#handleInternal
        RequestMappingHandlerAdapter#invokeHandlerMethod
            ServletInvocableHandlerMethod#invokeAndHandle
                InvocableHandlerMethod#invokeForRequest
                    InvocableHandlerMethod#getMethodArgumentValues
                        HandlerMethodArgumentResolverComposite#resolveArgument
                            HandlerMethodArgumentResolverComposite#getArgumentResolver
                            RequestHeaderMapMethodArgumentResolver#resolveArgument
```

```java
// curl http://localhost:8088/header -H "key:v1" -H "key:v2"

@GetMapping("/header")
public String header(@RequestHeader() MultiValueMap map) {
    log.info("map: {}", map);
    return map.toString();
}
```

## 响应 Header 处理

```
DispatcherServlet#doDispatch
    * AbstractHandlerMethodAdapter#handle
        RequestMappingHandlerAdapter#handleInternal
            RequestMappingHandlerAdapter#invokeHandlerMethod
                ServletInvocableHandlerMethod#invokeAndHandle
                    InvocableHandlerMethod#invokeForRequest
                    * HandlerMethodReturnValueHandlerComposite#handleReturnValue
                        HandlerMethodReturnValueHandlerComposite#selectHandler
                        RequestResponseBodyMethodProcessor#handleReturnValue
                            AbstractMessageConverterMethodProcessor#writeWithMessageConverters
                                AbstractMessageConverterMethodProcessor#getAcceptableMediaTypes
                                AbstractMessageConverterMethodProcessor#getProducibleMediaTypes
                                MediaType#sortBySpecificityAndQuality
                                AbstractHttpMessageConverter#write
```

```java
// curl http://localhost:8088/header -H "key:v" -H "Accept:application/json" -i

@GetMapping(value = "/header", produces = {"application/json"})
public String header(@RequestHeader() HttpHeaders map, HttpServletResponse response) {
    log.info("map: {}", map);

    response.addHeader("k", "v");

    // 不能生效
    response.addHeader(HttpHeaders.CONTENT_TYPE, "application/json");
    return "ok";
}
```

## Body 读取

```
RequestMappingHandlerAdapter#handleInternal
    RequestMappingHandlerAdapter#invokeHandlerMethod
        ServletInvocableHandlerMethod#invokeAndHandle
            InvocableHandlerMethod#invokeForRequest
                InvocableHandlerMethod#getMethodArgumentValues
                    HandlerMethodArgumentResolverComposite#resolveArgument
                        RequestResponseBodyMethodProcessor#resolveArgument
                            * RequestResponseBodyMethodProcessor#readWithMessageConverters
                                RequestResponseBodyMethodProcessor#readWithMessageConverters
                                    * RequestResponseBodyAdviceChain#afterBodyRead
```

```java
@ControllerAdvice
public class PrintRequestBodyAdviceAdapter extends RequestBodyAdviceAdapter {
    @Override
    public boolean supports(MethodParameter methodParameter, Type targetType, Class<? extends HttpMessageConverter<?>> converterType) {
        return true;
    }

    @Override
    public Object afterBodyRead(Object body, HttpInputMessage inputMessage, MethodParameter parameter, Type targetType, Class<? extends HttpMessageConverter<?>> converterType) {
        System.out.println("request body:" + body);
        return super.afterBodyRead(body, inputMessage, parameter, targetType, converterType);
    }
}
```