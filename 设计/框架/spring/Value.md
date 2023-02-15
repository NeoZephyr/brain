```
DefaultListableBeanFactory#doResolveDependency
    QualifierAnnotationAutowireCandidateResolver#getSuggestedValue
        QualifierAnnotationAutowireCandidateResolver#findValue
            QualifierAnnotationAutowireCandidateResolver#extractValue
                AbstractBeanFactory#resolveEmbeddedValue
```

1. 获取 @Value 
2. 解析 @Value 的字符串值。解析结果可能是一个字符串，也可能是一个对象
3. 将解析结果转化为要装配的对象的类型