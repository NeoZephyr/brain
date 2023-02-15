```java
@PostMapping(path = "/form")
public String form(@RequestParam("p1") String p1, @RequestParam("p2") String p2) {
    return "hello, p1: " + p1 + ", p2: " + p2;
}
```

```java
RestTemplate template = new RestTemplate();
MultiValueMap<String, Object> paramMap = new LinkedMultiValueMap<>();

paramMap.add("p1", "001");
paramMap.add("p2", "002");

String url = "http://localhost:8088/form";
String result = template.postForObject(url, paramMap, String.class);
System.out.println(result);
```

在发送请求的时候，遍历当前支持的所有编解码器，如果找到合适的编解码器，就使用它来完成 Body 的转化。当使用的 Body 是 MultiValueMap 才能使用表单来提交

## URL 编码

```java
RestTemplate template = new RestTemplate();
String url = "http://localhost:8088/param?a=100";
UriComponentsBuilder builder = UriComponentsBuilder.fromHttpUrl(url);
builder.queryParam("b", "开发 001");
URI uri = builder.build().encode().toUri();

HttpEntity<?> entity = new HttpEntity<>(null);
ResponseEntity<String> result = template.exchange(uri, HttpMethod.GET, entity, String.class);
System.out.println(result.getBody());
```