## BeanFactory vs ApplicationContext

BeanFactory 初始化容器时，并未实例化 Bean，直到第一次访问某个 Bean 时才实例化目标 Bean
ApplicationContext 在初始化上下文时就实例化所有单实例的 Bean

ApplicationContext 会利用反射机制自动识别出配置文件中定义的 BeanPostProcessor, InstantiationAwareBeanPostProcessor, BeanFactoryPostProcessor，并自动将它们注册到应用上下文中

## 生命周期

### prepareRefresh

1. 启动时间 startupDate
2. 状态标识 closed(false)，active(true)
3. 初始化 PropertySources initPropertySources()
4. 检验 Environment 中必须属性
5. 初始化事件监听器集合
6. 初始化早起 spring 事件集合

### obtainFreshBeanFactory

刷新 Spring 应用上下文底层 BeanFactory - refreshBeanFactory

1. 销毁或关闭 BeanFactory，如果已存在的话
2. 创建 BeanFactory，createBeanFactory()
3. 设置 BeanFactory id
4. 设置是否允许 BeanDefinition 重复定义
5. 设置是否允许循环引用
6. 加载 BeanDefinition
7. 关联新建 BeanFactory 到 Spring 应用上下文

返回 Spring 应用上下文底层 BeanFactory - getBeanFactory

### prepareBeanFactory

1. 关联 ClassLoader
2. 设置 Bean 表达式处理器
3. 添加 PropertyEditorRegistrar 实现 - ResourceEditorRegistrar
4. 添加 Aware 回调接口 BeanPostProcessor 实现 - ApplicationContextAwareProcessor
5. 忽略 Aware 回调接口作为依赖注入接口
6. 注册 ResolvableDependency 对象 - BeanFactory, ResourceLoader, ApplicationEventPublisher 以及 ApplicationContext
7. 注册 ApplicationListenerDetector 对象
8. 注册 LoadTimeWeaverAwareProcessor 对象
9. 注册单例对象 - Environment, Java System Properties 以及 OS 环境变量

### BeanFactory 后置处理

```
AbstractApplicationContext#postProcessBeanFactory(ConfigurableListableBeanFactory)

AbstractApplicationContext#invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory)
```

### registerBeanPostProcessors

1. 注册 PriorityOrdered 类型的 BeanFactoryProcessor Beans
2. 注册 Ordered 类型的 BeanPostProcessor Beans
3. 注册普通 BeanPostProcessor Beans
4. 注册 MergedBeanDefinitionPostProcessor Beans
5. 注册 ApplicationListenerDetector 对象

### initMessageSource

### initApplicationEventMulticaster

### onRefresh

### registerListeners

1. 添加当前应用上下文所关联的 ApplicationListener 对象
2. 添加 BeanFactory 所注册 ApplicationListener Beans
3. 广播早期 Spring 事件

### finishBeanFactoryInitialization

1. BeanFactory 关联 ConversionService Bean，如果存在
2. 添加 StringValueResolver 对象
3. 依赖查找 LoadTimeWeaverAware Bean
4. BeanFactory 临时 ClassLoader 设置为 null
5. BeanFactory 冻结配置
6. BeanFactory 初始化非延迟单例 Beans

### finishRefresh

1. 清除 ResourceLoader 缓存 - clearResourceCaches()
2. 初始化 LifecycleProcessor 对象 - initLifecycleProcessor()
3. 调用 LifecycleProcessor#onRefresh 方法
4. 发布 Spring 应用上下文已刷新事件 - ContextRefreshedEvent
5. 向 MBeanServer 托管 Live Beans

### start

1. 启动 LifecycleProcessor。依赖查找 Lifecycle Beans，启动 Lifecycle Beans
2. 发布 Spring 应用上下文已启动事件 - ContextStartedEvent

### stop

1. 停止 LifecycleProcessor。依赖查找 Lifecycle Beans，停止 Lifecycle Beans
2. 发布 Spring 应用上下文已停止事件 - ContextStoppedEvent

### close

1. 状态标识：active(false)、closed(true)
2. Live Beans JMX 撤销托管 LiveBeansView.unregisterApplicationContext(ConfigurableApplicationContext)
3. 发布 Spring 应用上下文已关闭事件 ContextClosedEvent
4. 关闭 LifecycleProcessor。依赖查找 Lifecycle Beans，停止 Lifecycle Beans
5. 销毁 Spring Beans
6. 关闭 BeanFactory
7. 回调 onClose()
8. Shutdown Hook 线程