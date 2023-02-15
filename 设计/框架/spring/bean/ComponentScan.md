```
AbstractApplicationContext#refresh
    PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors
        PostProcessorRegistrationDelegate#invokeBeanDefinitionRegistryPostProcessors
            ConfigurationClassPostProcessor#postProcessBeanDefinitionRegistry
                ConfigurationClassPostProcessor#processConfigBeanDefinitions
                    ConfigurationClassParser#parse
                        ConfigurationClassParser#doProcessConfigurationClass
                            ComponentScanAnnotationParser#parse
```

当 basePackages 为空时，扫描的包会是 declaringClass 所在的包

```java
Set<String> basePackages = new LinkedHashSet<>();
String[] basePackagesArray = componentScan.getStringArray("basePackages");

for (String pkg : basePackagesArray) {
    String[] tokenized = StringUtils.tokenizeToStringArray(
        this.environment.resolvePlaceholders(pkg),
        ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
    Collections.addAll(basePackages, tokenized);
}

for (Class<?> clazz : componentScan.getClassArray("basePackageClasses")) {
    basePackages.add(ClassUtils.getPackageName(clazz));
}

if (basePackages.isEmpty()) {
    basePackages.add(ClassUtils.getPackageName(declaringClass));
}
```

一旦显式指定其它包，原来的默认扫描包就被忽略了

```java
@ComponentScan("com.blue.wiki.controller")
@ComponentScans(value = { @ComponentScan(value = "com.blue.wiki.controller") })
```