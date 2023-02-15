## 添加方式

### @Bean 方式

```java
@Bean
public FilterRegistrationBean<TimeFilter> filterFilterRegistrationBean(){
    FilterRegistrationBean<TimeFilter> registrationBean = new FilterRegistrationBean<>();
    registrationBean.setFilter(new TimeFilter());
    registrationBean.addUrlPatterns("/*");
    registrationBean.setOrder(1);
    return registrationBean;
}
```

### @Component 方式

```java
@Component
@Order(1)
public class TimeFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        long start = System.currentTimeMillis();
        chain.doFilter(request, response);
        long end = System.currentTimeMillis();
        System.out.println("cost(ms):" + (end - start));
    }
}
```

### @WebFilter 方式

```java
@WebFilter
public class TimeFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        long start = System.currentTimeMillis();
        chain.doFilter(request, response);
        long end = System.currentTimeMillis();
        System.out.println("cost(ms):" + (end - start));
    }
}

@ServletComponentScan
@SpringBootApplication
public class Application implements CommandLineRunner {}
```

过滤器被 @WebFilter 修饰后，TimeCostFilter 只会被包装为 FilterRegistrationBean，而 TimeCostFilter 自身，只会作为一个 InnerBean 被实例化，这意味着 TimeCostFilter 实例并不会作为 Bean 注册到 Spring 容器

#### 注册 BeanDefinition

```
AbstractApplicationContext#refresh
    AbstractApplicationContext#invokeBeanFactoryPostProcessors
        PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors
            * ServletComponentRegisteringPostProcessor#postProcessBeanFactory
                ServletComponentRegisteringPostProcessor#scanPackage
                    ServletComponentHandler#handle
                        * WebFilterHandler#doHandle
                            GenericApplicationContext#registerBeanDefinition
```

#### 创建 FilterRegistrationBean

```
AbstractApplicationContext#refresh
    * ServletWebServerApplicationContext#onRefresh
        ServletWebServerApplicationContext#createWebServer
            TomcatServletWebServerFactory#getWebServer
                TomcatServletWebServerFactory#getTomcatWebServer
                    TomcatWebServer#initialize
                        * Tomcat#start
                            StandardServer#startInternal
                                StandardService#startInternal
                                    StandardEngine#startInternal
                                        StandardHost#startInternal
                                            StandardContext#startInternal
                                                TomcatStarter#onStartup
```

```
TomcatStarter#onStartup
    ServletWebServerApplicationContext#selfInitialize
        ServletWebServerApplicationContext#getServletContextInitializerBeans
            ServletContextInitializerBeans#addServletContextInitializerBeans
                * ServletContextInitializerBeans#getOrderedBeansOfType
                    AbstractBeanFactory#getBean
                        AbstractAutowireCapableBeanFactory#createBean
                            AbstractAutowireCapableBeanFactory#populateBean
                                AbstractAutowireCapableBeanFactory#applyPropertyValues
                                    BeanDefinitionValueResolver#resolveValueIfNecessary
                                        * BeanDefinitionValueResolver#resolveInnerBean
                                            AbstractAutowireCapableBeanFactory#createBean
```

```java
@Qualifier("com.pain.flame.bean.TimeFilter")
FilterRegistrationBean timeFilter;
```

## ApplicationFilterChain

```
StandardWrapperValve#invoke
    ApplicationFilterFactory#createFilterChain
        * ApplicationFilterChain#doFilter
            ApplicationFilterChain#internalDoFilter
                TimeFilter#doFilter
                    * ApplicationFilterChain#doFilter
                        ApplicationFilterChain#internalDoFilter
                            HttpServlet#service
                                FrameworkServlet#service
                                    HttpServlet#service
                                        FrameworkServlet#processRequest
                                            DispatcherServlet#doService
                                                DispatcherServlet#doDispatch
```