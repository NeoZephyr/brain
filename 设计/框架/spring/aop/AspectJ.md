采用编译期织入和类装载期织入

开启 aspectj 自动代理

```xml
<aop:aspectj-autoproxy proxy-target-class="true" />
```

```java
@EnableAspectJAutoProxy(proxyTargetClass = true)
```
