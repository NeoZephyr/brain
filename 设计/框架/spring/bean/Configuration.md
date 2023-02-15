## 自动配置

```java
@Configuration
@EnableFoo
public class FooAutoConfiguration {}

public class FooConfiguration {

    @Bean
    public String foo() {
        return "foo";
    }
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(FooConfiguration.class)
public @interface EnableFoo {}
```

```properties
# spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.pain.flame.blink.configuration.FooAutoConfiguration
```

### Initializer

```java
@Order(Ordered.HIGHEST_PRECEDENCE)
public class FooContextInitializer<C extends ConfigurableApplicationContext> implements ApplicationContextInitializer<C> {
    @Override
    public void initialize(C applicationContext) {
        System.out.println("FooContextInitializer initialize: " + applicationContext.getId());
    }
}
```

```properties
org.springframework.context.ApplicationContextInitializer=\
com.pain.flame.blink.initializer.FooContextInitializer
```

## 编程配置

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
@Documented
@Conditional(OnSystemPropertyCondition.class)
public @interface ConditionalOnSystemProperty {

    String name();
    String value();
}

public class OnSystemPropertyCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        Map<String, Object> annotationAttributes = metadata.getAnnotationAttributes(ConditionalOnSystemProperty.class.getName());

        String propertyName = String.valueOf(annotationAttributes.get("name"));
        String propertyValue = String.valueOf(annotationAttributes.get("value"));

        String systemValue = System.getProperty(propertyName);

        return propertyValue.equals(systemValue);
    }
}
```

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(FooImportSelector.class)
public @interface EnableFoo {}

public class FooConfiguration {

    @Bean
    @ConditionalOnSystemProperty(name = "user.name", value = "pain")
    public String foo() {
        return "foo";
    }
}

public class FooImportSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        return new String[]{FooConfiguration.class.getName()};
    }
}

@EnableFoo
public class Application {

	public static void main(String[] args) {
		ConfigurableApplicationContext context = new SpringApplicationBuilder(Application.class)
				.web(WebApplicationType.NONE)
            	.profiles("prod")
				.run(args);
	}
}
```