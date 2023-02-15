```
SpringApplication#createApplicationContext
    AnnotationConfigServletWebServerApplicationContext#Constructor
        AnnotatedBeanDefinitionReader#Constructor
            AnnotationConfigUtils#registerAnnotationConfigProcessors
                AnnotationConfigUtils#registerPostProcessor
                    * GenericApplicationContext#registerBeanDefinition
```

```
SpringApplication#prepareContext
    SpringApplication#load
        SpringApplication#createBeanDefinitionLoader
            BeanDefinitionLoader#load
                AnnotatedBeanDefinitionReader#register
                    AnnotatedBeanDefinitionReader#registerBean
                        AnnotatedBeanDefinitionReader#doRegisterBean
                            BeanDefinitionReaderUtils#registerBeanDefinition
                                * GenericApplicationContext#registerBeanDefinition
```