Request 的 getInputStream 和 getReader 只能使用一次，getInputStream、getReader、getParameter 方法互斥

## 重复读

HttpServletRequestWrapper + Filter 解决输入流不能重复度问题

```java
public class RequestWrapper extends HttpServletRequestWrapper {

    // 存储输入流数据
    private final byte[] body;

    public RequestWrapper(HttpServletRequest request) {
        super(request);
        body = RequestParseUtil
            .getBodyString(request)
            .getBytes(Charset.defaultCharset());
    }

    @Override
    public BufferedReader getReader() throws IOException {
        return new BufferedReader(new InputStreamReader(getInputStream()));
    }

    @Override
    public ServletInputStream getInputStream() throws IOException {
        ByteArrayInputStream inputStream = new ByteArrayInputStream(body);

        return new ServletInputStream() {
            @Override
            public boolean isFinished() {
                return false;
            }

            @Override
            public boolean isReady() {
                return false;
            }

            @Override
            public void setReadListener(ReadListener listener) {}

            @Override
            public int read() throws IOException {
                return inputStream.read();
            }
        };
    }
}
```

```java
public class RequestParseUtil {

    public static boolean isJson(HttpServletRequest request) {

        if (request.getContentType() != null) {
            return request.getContentType()
                .equals(MediaType.APPLICATION_JSON_VALUE) ||
                request.getContentType()
                .equals(MediaType.APPLICATION_JSON_UTF8_VALUE);
        }

        return false;
    }

    public static String getBodyString(final ServletRequest request) {
        try {
            return inputStream2String(request.getInputStream());
        } catch (IOException ex) {
            throw new RuntimeException();
        }
    }

    private static String inputStream2String(InputStream inputStream) {
        StringBuilder sb = new StringBuilder();
        BufferedReader reader = null;

        try {
            reader = new BufferedReader(
                new InputStreamReader(
                    inputStream,
                    Charset.defaultCharset()));
            String line;

            while ((line = reader.readLine()) != null) {
                sb.append(line);
            }
        } catch (IOException ex) {
            throw new RuntimeException();
        }

        return sb.toString();
    }
}
```

```java
@WebFilter(urlPatterns = "/*", filterName = "requestWrapperFilter")
public class RequestWrapperFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {}

    @Override
    public void doFilter(ServletRequest request,
                         ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        ServletRequest requestWrapper =
            new RequestWrapper((HttpServletRequest) request);
        chain.doFilter(requestWrapper, response);
    }

    @Override
    public void destroy() {}
}
```

```
@ServletComponentScan
```