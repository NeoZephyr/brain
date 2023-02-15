对于非 Spring Boot 程序，除了添加相关 AOP 依赖项外，还常常会使用 @EnableAspectJAutoProxy 来开启 AOP 功能。这个注解类引入 AspectJAutoProxyRegistrar，它通过实现 ImportBeanDefinitionRegistrar 的接口方法来完成 AOP 相关 Bean 的准备工作

```
implementation 'org.springframework.boot:spring-boot-starter-aop'
```

```
AspectJAutoProxyRegistrar#registerBeanDefinitions
    AopConfigUtils#registerAspectJAnnotationAutoProxyCreatorIfNecessary
        AopConfigUtils#registerAspectJAnnotationAutoProxyCreatorIfNecessary
            AopConfigUtils#registerOrEscalateApcAsRequired
                DefaultListableBeanFactory#registerBeanDefinition
```

```
AbstractAutowireCapableBeanFactory#createBean
    AbstractAutowireCapableBeanFactory#doCreateBean
        AbstractAutowireCapableBeanFactory#initializeBean
            AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsAfterInitialization
                AbstractAutoProxyCreator#postProcessAfterInitialization
                    AbstractAutoProxyCreator#wrapIfNecessary
                        AbstractAutoProxyCreator#createProxy
```

## 内部调用

```java
@Aspect
@Component
public class AopConfig {
    
    @Around("execution(* com.pain.flame.punk.service.FooService.bar())")
    public void performance(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();
        joinPoint.proceed();
        long end = System.currentTimeMillis();
        System.out.printf("Pay method cost %s(ms)\n", (end - start));
    }
}
```

### 方式一
```java
@Service
public class FooService {
    @Lazy
    @Autowired
    private FooService fooService;

    public void foo() {
        fooService.bar();
    }
    
    public void bar() {}
}
```


### 方式二

从 AopContext 获取当前的 Proxy。这种方法需要在 @EnableAspectJAutoProxy 里加一个配置项 exposeProxy = true，表示将代理对象放入到 ThreadLocal

```java
@Service
public class FooService {

    public void foo() {
        ((FooService)AopContext.currentProxy()).bar();
    }

    public void bar() {}
}
```

## 属性为空

```java
@Aspect
@Component
public class AopConfig {

    @Around("execution(* com.pain.flame.punk.service.UserService.pay())")
    public void performance(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();
        joinPoint.proceed();
        long end = System.currentTimeMillis();
        System.out.printf("Pay method cost %s(ms)\n", (end - start));
    }
}

@Service
public class UserService {

    public final User admin = new User("jack");

    @Lazy
    @Autowired
    private UserService userService;

    public void pay() throws InterruptedException {}
}

@SpringBootApplication
public class PunkApplication implements CommandLineRunner {

    @Autowired
    UserService userService;

    public static void main(String[] args) {
        SpringApplication.run(PunkApplication.class, args);
    }

    @Override
    public void run(String... args) throws Exception {
        // 空指针错误
        System.out.printf("=== run command: %s\n", userService.admin.getName());
    }
}
```

```
AbstractAutoProxyCreator#createProxy
    DefaultAopProxyFactory#createAopProxy
        CglibAopProxy#getProxy
            CglibAopProxy#createEnhancer
            CglibAopProxy#getCallbacks
            ObjenesisCglibAopProxy#createProxyClassAndInstance
                SpringObjenesis#newInstance
```

Cglib 最后使用了 JDK 的 ReflectionFactory.newConstructorForSerialization 完成了代理对象的实例化。这种方式创建出来的对象是不会初始化类成员变量的

解决办法：增加 getAdmin 方法，获取 admin

```java
@Service
public class UserService {

    public final User admin = new User("jack");

    @Lazy
    @Autowired
    private UserService userService;

    public void pay() throws InterruptedException {}

    public User getAdmin() {
        return admin;
    }
}
```

创建代理类后，会调用 setCallbacks 来设置拦截后需要注入的代码。callbacks 中会存在一种服务于 AOP 的 DynamicAdvisedInterceptor，它的接口是 MethodInterceptor，实现了拦截方法 intercept。当代理类方法被调用，会被 Spring 拦截，从而进入此 intercept，并在此方法中获取被代理的原始对象。而在原始对象中，类属性是被实例化过且存在的。因此代理类是可以通过方法拦截获取被代理对象实例的属性

## 增强顺序

```
AbstractAutoProxyCreator#postProcessAfterInitialization
    AbstractAutoProxyCreator#wrapIfNecessary
        AbstractAdvisorAutoProxyCreator#getAdvicesAndAdvisorsForBean
            AbstractAdvisorAutoProxyCreator#findEligibleAdvisors
                AnnotationAwareAspectJAutoProxyCreator#findCandidateAdvisors
                    BeanFactoryAspectJAdvisorsBuilder#buildAspectJAdvisors
                AspectJAwareAdvisorAutoProxyCreator#sortAdvisors
```

```
BeanFactoryAspectJAdvisorsBuilder#buildAspectJAdvisors
    ReflectiveAspectJAdvisorFactory#getAdvisorMethods
```

同一个切面中，不同类型的增强方法被调用的顺序依次为 Around.class、Before.class、After.class、AfterReturning.class、AfterThrowing.class

在同一个切面配置中，如果存在多个相同类型的增强，那么其执行优先级是按照该增强的方法名排序