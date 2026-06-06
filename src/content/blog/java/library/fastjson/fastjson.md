---
title: "fastjson"
description: "fastjson的学习笔记，实现序列化与反序列化"
date: "2026-06-04T12:47:11.157Z"
draft: false
showHeroImage: false
tags:
  - java
categories:
  - Java常用类库
series: 
  - 编程路线
comments: true
sidebar:
  enable: true
  toc: true
  relatedPosts: true
---

## 常用方法

### 序列化和反序列化

```java
// 序列化：对象变json字符串
UserEntity userEntity = JSON.parseObject(strJson, UserEntity.class);
// 反序列化：json字符串变对象
UserEntity userEntity = JSON.parseObject(strJson, UserEntity.class);
```

### 配置序列化字段

```java
// 不被序列化
@JSONField(name="amount", serialize=false)
private Double amount;
// 序列化格式
@JSONField(name="createTime", format="dd/MM/yyyy", ordinal = 3)
private Date createTime;

@JsonProperty("top_p")
private Double topP = 1d;
@JsonProperty("max_tokens")
private Integer maxTokens = 2048;

```

对象的属性上添加 `@JSONField`、`@JsonProperty` 都可以改变序列化字段的名字。同时还可以扩展是否被序列化和格式化

### 排除序列化字段

```java
UserEntity userEntity = UserEntity.builder()
        .amount(100D)
        .userName("xfg")
        .password("abc000")
        .createTime(new Date())
        .build();

SimplePropertyPreFilter filter = new SimplePropertyPreFilter();
Collections.addAll(filter.getExcludes(), "password");
log.info(JSON.toJSONString(userEntity, filter));
```

因为有些时候不是你能修改被序列化的对象，如你引入了别人的 JAR 之后需要对某个类进行序列化，但因为有些对象不能被序列化或者不要序列化。那么这个时候就可以通过 filter 过滤的方式进行处理。

### json2map

```java
@Test
public void test_map2json() {
    Map<String, Object> map = new HashMap<>();
    map.put("key1", "xfg");
    map.put("key2", 123);
    map.put("key3", false);
    log.info(JSON.toJSONString(map));
}

@Test
public void test_json2map() {
    String jsonString = "{\"key1\":\"xfg\",\"key2\":123,\"key3\":false}";
    Map<String, Object> map = JSON.parseObject(jsonString, Map.class);
    for (Map.Entry<String, Object> entry : map.entrySet()) {
        log.info("{} : {}", entry.getKey(), entry.getValue());
    }
}
```

有些时候我们接收的对象就是个 Map 那么你可以使用 fastjson 来对对象进行 map 的转换或者序列化

### toString处理

```java
// ToString2Bean.java
@Test
public void testToString2Bean() throws Exception {
    UserEntity userEntity = UserEntity.builder()
            .amount(100D)
            .userName("xfg")
            .password("abc000")
            .createTime(new Date())
            .build();
    log.info(userEntity.toString());
    log.info(JSON.toJSONString(ToString2Bean.toObject(userEntity.toString(), UserEntity.class)));
}

public static <T> T toObject(String str, Class<T> clazz) throws Exception {
    // 创建一个新的对象
    T obj = clazz.getDeclaredConstructor().newInstance();
    // 获取类对象
    Class<?> objClass = obj.getClass();
    // 解析字符串
    String[] fields = str.substring(str.indexOf("{") + 1, str.indexOf("}")).split(", ");
    // 遍历成员变量
    for (String field : fields) {
	    // 获取成员变量名和值
	    String[] parts = field.split("=");
	    // 获取成员变量对象
	    Field objField = objClass.getDeclaredField(parts[0].trim());
	    // 设置成员变量可以访问
	    objField.setAccessible(true);
	    // 设置成员变量的值
	    objField.set(obj, convertValue(objField.getType(), parts[1].trim()));
	    // 设置成员变量不可访问
	    objField.setAccessible(false);
    }
    return obj;
}

```