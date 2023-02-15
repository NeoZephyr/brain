```
AbstractApplicationContext#refresh
    AbstractApplicationContext#finishBeanFactoryInitialization
        * DefaultListableBeanFactory#preInstantiateSingletons
            AbstractBeanFactory#getBean
                AbstractBeanFactory#doGetBean
                    DefaultSingletonBeanRegistry#getSingleton
                        AbstractAutowireCapableBeanFactory#createBean
                            AbstractAutowireCapableBeanFactory#doCreateBean
                                * AbstractAutowireCapableBeanFactory#createBeanInstance
                                AbstractAutowireCapableBeanFactory#populateBean
                                AbstractAutowireCapableBeanFactory#initializeBean
```

```
* AbstractAutowireCapableBeanFactory#createBeanInstance
    AbstractAutowireCapableBeanFactory#determineConstructorsFromBeanPostProcessors
        AbstractAutowireCapableBeanFactory#autowireConstructor
            ConstructorResolver#autowireConstructor
```

## 创建

AbstractAutowireCapableBeanFactory#createBeanInstance。主要包含两大基本步骤：寻找构造器和通过反射调用构造器创建实例

## 填充

```
AbstractAutowireCapableBeanFactory#populateBean
    AutowiredAnnotationBeanPostProcessor#postProcessProperties
        * AutowiredAnnotationBeanPostProcessor#findAutowiringMetadata
            InjectionMetadata#inject
                AutowiredFieldElement#inject
                    AutowiredFieldElement#resolveFieldValue
                        DefaultListableBeanFactory#resolveDependency
                            DefaultListableBeanFactory#doResolveDependency
                                * DefaultListableBeanFactory#findAutowireCandidates
                                * DefaultListableBeanFactory#determineAutowireCandidate
                                    DefaultListableBeanFactory#determinePrimaryCandidate
                                    DefaultListableBeanFactory#determineHighestPriorityCandidate
```

标记为 Autowired 的成员属性会使用到 AutowiredAnnotationBeanPostProcessor 来完成装配过程：
1. 寻找出所有需要依赖注入的字段和方法
2. 根据依赖信息寻找出依赖并完成注入

### 填充优先级

DefaultListableBeanFactory#findAutowireCandidates 寻找到对应类型的 Bean，然后设置给对应的属性。在查找 Bean 的时候，调用 determineAutowireCandidate 方法来选出优先级最高的依赖。优先级的决策是先根据 @Primary 来决策，其次是 @Priority 决策，最后是根据 Bean 名字的严格匹配来决策。如果最终无法确定 Bean，则返回 null

```java
@Repository
@Slf4j
public class RedisDataService implements DataService {
    @Override
    public void delete(int id) {
        log.info("redis delete");
    }
}

@Repository
@Slf4j
public class MySqlDataService implements DataService {
    @Override
    public void delete(int id) {
        log.info("mysql delete");
    }
}

@RestController
public class DataServiceController {

    @Autowired
    DataService redisDataService;
    
    @Autowired
    @Qualifier("mySqlDataService")
    DataService dataService;
}
```

### 收集填充

```
DefaultListableBeanFactory#resolveDependency
    DefaultListableBeanFactory#doResolveDependency
        DefaultListableBeanFactory#resolveMultipleBeans
            DefaultListableBeanFactory#findAutowireCandidates
```

1. 获取集合类型的元素类型
2. 根据元素类型，找出所有的 Bean
3. 将匹配的所有的 Bean 按目标类型进行转化

```java
@Bean
public Student senior() {
    return new Student("pain");
}

@Bean
public Student junior() {
    return new Student("hoop");
}

@Autowired
private List<Student> students;
```

```java
@Service
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class ServiceImpl {}

@RestController
public class HelloWorldController {
    @Autowired
    private ServiceImpl serviceImpl;
}
```

寻找到要自动注入的 Bean 后，通过反射设置给对应的 field。这个 field 的执行只发生了一次，所以后续就固定起来了，它并不会因为 ServiceImpl 标记了 SCOPE_PROTOTYPE 而改变

```java
// 方式一
@Autowired
private ApplicationContext applicationContext;

public ServiceImpl getServiceImpl() {
    return applicationContext.getBean(ServiceImpl.class);
}

// 方式二
@Lookup
public ServiceImpl getServiceImpl() {
	return null;
}
```

在解析 HelloWorldController 这个 Bean 时，当有方法标记了 Lookup，此时就会添加相应方法到属性 methodOverrides 里面去（此过程由 AutowiredAnnotationBeanPostProcessor#determineCandidateConstructors 完成）。当 hasMethodOverrides 为 true 时，则使用 CGLIB，参考 SimpleInstantiationStrategy#instantiate