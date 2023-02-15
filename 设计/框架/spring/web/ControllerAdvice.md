WebMvcConfigurationSupport 中的 handlerExceptionResolver 实例化并注册了一个 ExceptionHandlerExceptionResolver 的实例，而所有被 @ControllerAdvice 注解修饰的异常处理器，都会在 ExceptionHandlerExceptionResolver 实例化的时候自动扫描并装载在其类成员变量 exceptionHandlerAdviceCache 中

## Filter 使用 ExceptionHandler

```java
@RestControllerAdvice
public class NotAllowExceptionHandler {

    @ExceptionHandler(NotAllowException.class)
    @ResponseBody
    public String handle() {
        return "{\"resultCode\": 403}";
    }
}

public class TimeFilter implements Filter {

    @Autowired
    @Qualifier("handlerExceptionResolver")
    private HandlerExceptionResolver resolver;

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        long start = System.currentTimeMillis();

        HttpServletRequest httpServletRequest = (HttpServletRequest) request;
        HttpServletResponse httpServletResponse = (HttpServletResponse) response;
        String token = httpServletRequest.getHeader("token");

        if ("hello".equals(token)) {
            resolver.resolveException(httpServletRequest, httpServletResponse, null, new NotAllowException());
            return;
        }

        chain.doFilter(request, response);
        long end = System.currentTimeMillis();
        System.out.println("执行时间(ms):" + (end - start));
    }
}
```

## NotFound 处理

```java
void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    mappedHandler = getHandler(processedRequest);
    
    if (mappedHandler == null) {
        noHandlerFound(processedRequest, response);
        return;
    }
}

void noHandlerFound(HttpServletRequest request, HttpServletResponse response) throws Exception {
    if (pageNotFoundLogger.isWarnEnabled()) {
        pageNotFoundLogger.warn("No mapping for " + request.getMethod() + " " + getRequestUri(request));
    }
    if (this.throwExceptionIfNoHandlerFound) {
        throw new NoHandlerFoundException(request.getMethod(), getRequestUri(request),
                                          new ServletServerHttpRequest(request).getHeaders());
    }
    else {
        response.sendError(HttpServletResponse.SC_NOT_FOUND);
    }
}
```

DispatcherServlet 中的 doDispatch 中，首先调用 getHandler 获取当前请求的处理器，如果获取不到，则调用 noHandlerFound：如果 throwExceptionIfNoHandlerFound 属性为 true，则直接抛出 NoHandlerFoundException 异常，反之则会进一步获取到对应的请求处理器执行，并将执行结果返回给客户端

在 WebMvcAutoConfiguration 类中，默认添加的两个 ResourceHandler，一个用来处理请求路径 `/webjars/*` ，另一个是 `/**`。因此，即便当前请求没有定义任何对应的请求处理器，getHandler 也一定会获取到一个 Handler 来处理当前请求。这就导致程序不可能走到 noHandlerFound，也就不会抛出 NoHandlerFoundException 异常，也无法被后续的异常处理器进一步处理

```yaml
spring:
    web:
        resources:
            add-mappings: false

mvc:
    throw-exception-if-no-handler-found: true
```

```java
@RestControllerAdvice
public class ExceptionHandler {

    @ResponseStatus(HttpStatus.NOT_FOUND)
    @ExceptionHandler(NoHandlerFoundException.class)
    @ResponseBody
    public String handle404() {
        return "{\"resultCode\": 404}";
    }
}
```