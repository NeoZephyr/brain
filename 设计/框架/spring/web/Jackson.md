## 注解

```java
// 序列化只包含非空
@JsonInclude(JsonInclude.Include.NON_NULL)

// 忽略
@JsonIgnoreProperties({"code", "id"})
class Coupon {
	@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "GMT+8")
	Date birthday;

    @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "hh:mm:ss")
    Date joinDate;

    String code;
    
    Long id;
    
    // 忽略
    @JsonIgnore
    String name;
}
```

## 序列化

```java
public class CouponSerializer extends JsonSerializer<Coupon> {

    @Override
    public void serialize(
        Coupon coupon,
        JsonGenerator generator,
        SerializerProvider serializers) throws IOException {

        // 开始序列化
        generator.writeStartObject();

        generator.writeStringField("id", String.valueOf(coupon.getId()));
        generator.writeStringField("name", coupon.getName());
        generator.writeStringField(
            "joinDate",
            new SimpleDateFormat("HH:mm:ss").format(coupon.getJoinDate()));

        // 结束序列化
        generator.writeEndObject();
    }
}
```

```java
@JsonSerialize(using = CouponSerializer.class)
class Coupon {
	Date birthday;

    @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "hh:mm:ss")
    Date joinDate;

    String code;
    Long id;

    // 忽略
    @JsonIgnore
    String name;
}
```

```java
@Configuration
public class ObjectMapperConfig {

    @Bean
    @Primary
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();

        // 忽略 json 字符串中不识别的字段
        mapper.configure(
            DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES,
            false
        );

        return mapper;
    }
}
```

## 反序列化

```java
public class DateDeserializer extends JsonDeserializer<Date> {

    private static final String[] pattern = new String[] {
        "yyyy-MM-dd HH:mm:ss",
        "yyyy/MM/dd"
    };

    @Override
    public Date deserialize(JsonParser jsonParser, DeserializationContext context) throws IOException, JsonProcessingException {

        Date destDate = null;
        String srcDate = jsonParser.getText();

        if (StringUtils.isNotEmpty(srcDate)) {
            try {
                long longDate = Long.parseLong(srcDate.trim());
                destDate = new Date(longDate);
            } catch (NumberFormatException pe) {
                try {
                    destDate = DateUtils.parseDate(
                        srcDate,
                        DateJacksonConverter.pattern);
                } catch (ParseException ex) {}
            }
        }

        return destDate;
    }

    @Override
    public Class<?> handledType() {
        return Date.class;
    }
}
```

### 局部配置

```java
@JsonDeserialize(using = DateDeserializer.class)
Date birthday;
```

### 全局配置

```java
@Configuration
public class DateConverterConfig {

    @Bean
    public DateDeserializer dateDeserializer() {
        return new DateDeserializer();
    }

    @Bean
    public Jackson2ObjectMapperFactoryBean jackson2ObjectMapperFactoryBean(
        @Autowired DateDeserializer dateDeserializer) {
        Jackson2ObjectMapperFactoryBean jackson2ObjectMapperFactoryBean =
            new Jackson2ObjectMapperFactoryBean();
        jackson2ObjectMapperFactoryBean.setDeserializers(dateDeserializer);
        return jackson2ObjectMapperFactoryBean;
    }
}
```