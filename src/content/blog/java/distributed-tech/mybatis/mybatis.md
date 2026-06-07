---
title: "Mybatis"
description: "Mybatis的学习笔记，介绍了mybatis的核心对象，插件开发，事务等"
date: "2026-06-01T12:47:11.157Z"
draft: false
showHeroImage: false
tags:
  - 事务
  - 插件开发
categories:
  - Java分布式技术
series: 
  - 编程路线
comments: true
sidebar:
  enable: true
  toc: true
  relatedPosts: true
---
## 案例背景

- **雇员表** `employee`：存储员工基本信息与职级
- **薪资表** `employee_salary`：每名员工当前薪酬（1v1）
- **调薪记录表** `employee_salary_adjust`：每次晋升/普调写一条记录（1vN）

通过这三张表，覆盖：插入、批量插入、修改、查询、事务、插件加解密等核心操作。

## 领域模型（结合 DDD

MyBatis 操作放在 **infrastructure（基础设施层）**，业务模型放在 **domain（领域层）**，两层之间通过 **仓储接口** 解耦。

### 值对象（Value Object）

> 当实体的某个字段有**明确范围**时，提取为值对象（枚举），与具体业务绑定，不共用。
> 

```java
public enum EmployeePostVO {
    T1("T-1", "初级工程师"),
    T2("T-2", "初级工程师"),
    T3("T-3", "中级工程师"),
    T4("T-4", "中级工程师"),
    T5("T-5", "高级工程师"),
    T6("T-6", "高级工程师"),
    T7("T-7", "架构师"),
    T8("T-8", "架构师");

    private final String code;
    private final String desc;
}
```

### 实体对象（Entity）

> 对数据库表的抽象，通常 1:1，复杂场景可以 1:N。
> 

```java
@Data @Builder @AllArgsConstructor @NoArgsConstructor
public class EmployeeEntity {
    private EmployeePostVO employeeLevel;
    private EmployeePostVO employeeTitle;
}

@Data @Builder @AllArgsConstructor @NoArgsConstructor
public class EmployeeSalaryAdjustEntity {
    private BigDecimal adjustTotalAmount;  // 总额调薪
    private BigDecimal adjustBaseAmount;   // 基础调薪
    private BigDecimal adjustMeritAmount;  // 绩效调薪
}
```

### 聚合对象（Aggregate）

> 封装多个实体/值对象，代表一类业务的聚合，通常作为 Service 入参。
> 

```java
@Data @Builder @AllArgsConstructor @NoArgsConstructor
public class AdjustSalaryApplyOrderAggregate {
    private String employeeNumber;                        // 雇员编号
    private String orderId;                               // 调薪单号
    private EmployeeEntity employeeEntity;                // 雇员实体
    private EmployeeSalaryAdjustEntity employeeSalaryAdjustEntity; // 调薪实体
}
```

### 仓储接口（Repository）

> DDD 依赖倒置：domain 层定义接口，infrastructure 层实现。  
好处：天然隔离 PO（数据库持久化对象），外层无法乱引用。
> 

```java
// domain 层定义
public interface ISalaryAdjustRepository {
    String adjustSalary(AdjustSalaryApplyOrderAggregate aggregate);
}
```

### 服务接口（Service）

```java
public interface ISalaryAdjustApplyService {
    String execSalaryAdjust(AdjustSalaryApplyOrderAggregate aggregate);
}
```

> 复杂场景必须结合**设计模式**，避免所有逻辑堆在实现类里。
> 

## 配置文件

MyBatis 配置统一放在 `xfg-dev-tech-app` 模块下，便于管理和上线后提取。

典型配置项（`application.yml`）：

```yaml
mybatis:
  mapper-locations:classpath:/mapper/*.xml
  configuration:
    map-underscore-to-camel-case:true  # 开启下划线转驼峰
```

---

## 功能实现

### 插入 & 批量插入

**Mapper 接口**（infrastructure 层）：

```java
@Mapper
public interface IEmployeeDAO {
    void insert(EmployeePO employee);
    void insertList(List<EmployeePO> list);
    void update(EmployeePO employeePO);
    EmployeePO queryEmployeeByEmployNumber(String employNumber);
}
```

**XML 映射**（`employee_mapper.xml`）：

```xml
<!-- 单条插入 -->
<insert id="insert" parameterType="cn.bugstack.xfg.dev.tech.infrastructure.po.EmployeePO">
    INSERT INTO employee(employee_number, employee_name, employee_level, employee_title, create_time, update_time)
    VALUES(#{employeeNumber}, #{employeeName}, #{employeeLevel}, #{employeeTitle}, now(), now())
</insert>

<!-- 批量插入（foreach） -->
<insert id="insertList" parameterType="java.util.List">
    INSERT INTO employee(employee_number, employee_name, employee_level, employee_title, create_time, update_time)
    VALUES
    <foreach collection="list" item="item" separator=",">
        (#{item.employeeNumber}, #{item.employeeName}, #{item.employeeLevel}, #{item.employeeTitle}, now(), now())
    </foreach>
</insert>
```

> 批量插入使用 `<foreach>` 标签，`separator=","` 拼接多组值，性能远优于循环单条插入。
> 

---

### 事务

### 事务隔离级别（isolation）

| **级别** | **说明** | **问题** |
| --- | --- | --- |
| `DEFAULT` | 使用数据库默认（MySQL = REPEATABLE_READ） | — |
| `READ_UNCOMMITTED` | 可读未提交数据 | 脏读、不可重复读、幻读 |
| `READ_COMMITTED` | 只读已提交数据 | 不可重复读、幻读 |
| `REPEATABLE_READ` | 同一事务多次读结果一致 | 幻读 |
| `SERIALIZABLE` | 最严格，完全串行 | 并发性能差 |

### 事务传播行为（propagation）

| **行为** | **说明** |
| --- | --- |
| `REQUIRED`（默认） | 有事务就加入，否则新建 |
| `SUPPORTS` | 有事务就加入，否则非事务执行 |
| `MANDATORY` | 必须在事务中，否则抛异常 |
| `REQUIRES_NEW` | 始终新建事务，挂起当前事务 |
| `NOT_SUPPORTED` | 非事务执行，挂起当前事务 |
| `NEVER` | 非事务执行，有事务则抛异常 |
| `NESTED` | 嵌套事务，外部回滚则一起回滚 |

### 注解事务（声明式）

```java
@Transactional(
    rollbackFor = Exception.class,
    timeout = 350,
    propagation = Propagation.REQUIRED,
    isolation = Isolation.DEFAULT
)
public String adjustSalary(AdjustSalaryApplyOrderAggregate aggregate) {
    // 1. 更新岗位 employeeDAO.update(...)
    // 2. 更新薪酬 employeeSalaryDAO.update(...)
    // 3. 写入调薪流水 employeeSalaryAdjustDAO.insert(...)
    return orderId;
}
```

> 注意：放在 **repository 实现类**中，而非 service 层。
> 

### 编程事务（TransactionTemplate）

更细粒度控制，可以根据业务结果手动回滚（不依赖异常）。

**配置 Bean：**

```java
@Bean
public TransactionTemplate transactionTemplate(PlatformTransactionManager manager) {
    TransactionTemplate template = new TransactionTemplate(manager);
    template.setIsolationLevel(TransactionDefinition.ISOLATION_DEFAULT);
    template.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
    return template;
}
```

**使用：**

```java
@Resource
private TransactionTemplate transactionTemplate;

public void insertEmployeeInfo(EmployeeInfoEntity entity) {
    transactionTemplate.execute(new TransactionCallbackWithoutResult() {
        @Override
        protected void doInTransactionWithoutResult(TransactionStatus status) {
            try {
                employeeDAO.insert(...);
                employeeSalaryDAO.insert(...);
            } catch (Exception e) {
                status.setRollbackOnly(); // 手动标记回滚
                e.printStackTrace();
            }
        }
    });
}
```

> 编程事务 vs 注解事务：编程事务更灵活，可以在任意位置手动回滚，适合复杂业务逻辑。
> 

---

### MyBatis 插件开发（数据加解密）

MyBatis 插件基于**拦截器**机制，常见用途：字段加解密、分库分表路由、SQL 日志打印。

### 插件核心结构

```java
@Intercepts({
    // 拦截写操作（insert/update）
    @Signature(type = Executor.class, method = "update",
               args = {MappedStatement.class, Object.class}),
    // 拦截读操作（query）
    @Signature(type = Executor.class, method = "query",
               args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class})
})
public class FieldEncryptionAndDecryptionMybatisPlugin implements Interceptor {

    private static final String KEY = "1898794876567654"; // AES 密钥（16位）
    private static final String IV  = "1233214566547891"; // 偏移量（16位）

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        Object[] args = invocation.getArgs();
        MappedStatement ms = (MappedStatement) args[0];
        Object parameter = args[1];
        String sqlId = ms.getId();

        // ① 写操作：加密 employeeName 字段
        if (parameter != null && (sqlId.contains("insert") || sqlId.contains("update"))) {
            String columnName = "employeeName";
            if (parameter instanceof Map) {
                // 批量插入场景
                List<Object> list = (List<Object>) ((Map<?, ?>) parameter).get("list");
                for (Object obj : list) {
                    if (hasField(obj, columnName)) {
                        String val = BeanUtils.getProperty(obj, columnName);
                        BeanUtils.setProperty(obj, columnName, encrypt(val));
                    }
                }
            } else {
                // 单条插入场景
                if (hasField(parameter, columnName)) {
                    String val = BeanUtils.getProperty(parameter, columnName);
                    BeanUtils.setProperty(parameter, columnName, encrypt(val));
                }
            }
        }

        Object result = invocation.proceed(); // 执行原始方法

        // ② 读操作：解密 employeeName 字段
        if (result != null && sqlId.contains("query")) {
            String columnName = "employeeName";
            if (result instanceof List) {
                for (Object obj : (List<Object>) result) {
                    if (!hasField(obj, columnName)) continue;
                    String val = BeanUtils.getProperty(obj, columnName);
                    if (StringUtils.isBlank(val)) continue;
                    BeanUtils.setProperty(obj, columnName, decrypt(val));
                }
            }
        }

        return result;
    }

    // AES/CBC/PKCS5Padding 加密
    public String encrypt(String content) throws Exception {
        Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");
        SecretKeySpec keySpec = new SecretKeySpec(KEY.getBytes(), "AES");
        IvParameterSpec ivSpec = new IvParameterSpec(IV.getBytes());
        cipher.init(Cipher.ENCRYPT_MODE, keySpec, ivSpec);
        return Base64.getEncoder().encodeToString(cipher.doFinal(content.getBytes()));
    }

    // AES/CBC/PKCS5Padding 解密
    public String decrypt(String content) throws Exception {
        Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");
        SecretKeySpec keySpec = new SecretKeySpec(KEY.getBytes(), "AES");
        IvParameterSpec ivSpec = new IvParameterSpec(IV.getBytes());
        cipher.init(Cipher.DECRYPT_MODE, keySpec, ivSpec);
        byte[] encrypted = Base64.getDecoder().decode(content);
        return new String(cipher.doFinal(encrypted));
    }

    // 反射判断对象是否有指定字段（向上遍历父类）
    public boolean hasField(Object obj, String fieldName) {
        Class<?> clazz = obj.getClass();
        while (clazz != null) {
            try {
                clazz.getDeclaredField(fieldName);
                return true;
            } catch (NoSuchFieldException e) {
                clazz = clazz.getSuperclass();
            }
        }
        return false;
    }
}
```

### 注册插件到 Spring

```java
@Bean
public FieldEncryptionAndDecryptionMybatisPlugin encryptPlugin() {
    return new FieldEncryptionAndDecryptionMybatisPlugin();
}
```

或在 `mybatis-config.xml` 中：

```xml
<plugins>
    <plugin interceptor="cn.bugstack.xfg.dev.tech.plugin.FieldEncryptionAndDecryptionMybatisPlugin"/>
</plugins>
```

---

## 知识点总结

### MyBatis 核心对象

| **对象** | **说明** |
| --- | --- |
| `SqlSessionFactory` | MyBatis 核心工厂，全局唯一 |
| `SqlSession` | 每次数据库操作的会话 |
| `MappedStatement` | 每个 SQL 语句的映射描述对象（插件中获取 sqlId） |
| `Executor` | SQL 执行器，插件通常拦截此层 |
| `Mapper` | 接口代理，底层由 JDK 动态代理生成实现类 |

### DDD 分层与 MyBatis 位置

```
app（应用层）
  └─ domain（领域层）
       ├─ model（值对象、实体、聚合）
       ├─ repository（仓储接口定义）
       └─ service（业务服务接口）
  └─ infrastructure（基础设施层）
       ├─ dao（Mapper 接口，直接操作数据库）
       ├─ po（数据库持久化对象）
       ├─ repository（仓储实现，含事务注解）
       └─ plugin（MyBatis 插件）
```

> **核心原则**：PO 只在 infrastructure 内部流转，不暴露给 domain 或 app 层。
> 

### 常见 XML 标签速记

| **标签** | **用途** |
| --- | --- |
| `<insert>` | 插入语句 |
| `<update>` | 更新语句 |
| `<select>` | 查询语句 |
| `<foreach>` | 遍历集合（批量插入/IN 查询） |
| `<if>` | 条件判断（动态 SQL） |
| `<where>` | 自动处理 WHERE 和 AND |
| `<set>` | 自动处理 SET 和逗号（动态 UPDATE） |
| `<resultMap>` | 自定义结果映射 |