---
title: "connectionpool"
description: "connnectionpool的学习笔记，比较了主流数据库连接池的性能"
date: "2026-06-03T12:47:11.157Z"
draft: false
showHeroImage: false
tags:
  - 连接池
  - 索引
  - apache bench
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

## **为什么需要连接池？**

### **没有连接池时发生了什么？**

每次执行 SQL，程序都要做以下事情：

```
应用程序
   │
   ├─ 1. 建立 TCP 连接（三次握手）
   ├─ 2. MySQL 认证（用户名/密码校验）
   ├─ 3. 执行 SQL
   ├─ 4. 返回结果
   └─ 5. 关闭连接（四次挥手）
```

**问题**：步骤 1、2、5 是纯粹的"开销"，在高并发下每个请求都重复这个过程，极其浪费。

### **连接池的核心思想**

> **预先创建一批连接，用完归还，循环复用。**
> 

```
应用程序
   │
   ├─ 1. 从池中借一个现成的连接  ← 几乎无开销
   ├─ 2. 执行 SQL
   └─ 3. 把连接还回池中         ← 不真正关闭
```

---

## **主流连接池对比**

Java 生态中有四种主流连接池：

| **连接池** | **发布方** | **目前状态** | **SpringBoot 默认？** |
| --- | --- | --- | --- |
| **c3p0** | 第三方 | 老牌，更新缓慢 | 否 |
| **DBCP** | Apache | 稳定，性能一般 | 否 |
| **Druid** | 阿里巴巴 | 活跃，功能丰富（监控、SQL防火墙） | 否 |
| **HikariCP** | 第三方 | 极致性能，代码精简 | ✅ 是（Spring Boot 2.x+） |

### **压测数据（插入1万条，20并发，数据库已有100万数据）**

| **方案** | **总耗时** | **50% 响应** | **80% 响应** | **90% 响应** |
| --- | --- | --- | --- | --- |
| 无连接池 | 88.990 s | 155 ms | 223 ms | 291 ms |
| c3p0 | 24.228 s | 39 ms | 61 ms | 75 ms |
| DBCP | 33.656 s | 60 ms | 86 ms | 103 ms |
| Druid | 25.971 s | 45 ms | 64 ms | 75 ms |
| **HikariCP** | **25.002 s** | **43 ms** | **64 ms** | **76 ms** |

**结论**：

- 使用连接池 vs 不使用，性能相差 **3~4 倍**
- HikariCP 和 c3p0 性能接近并领先，DBCP 相对较弱
- 生产环境优先选 **HikariCP**（SpringBoot 默认）或 **Druid**（需要监控）

## **SpringBoot 中配置连接池**

### **HikariCP（推荐，默认内置）**

`pom.xml`：

```xml
<!-- spring-boot-starter-jdbc 已内置 HikariCP，无需额外引入 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <scope>runtime</scope>
</dependency>
```

`application.yml`：

```yaml
spring:
  datasource:
    driver-class-name:com.mysql.cj.jdbc.Driver
    url:jdbc:mysql://localhost:3306/your_db?useSSL=false&serverTimezone=Asia/Shanghai
    username:root
    password:your_password
    # HikariCP 连接池配置
    hikari:
      minimum-idle:5           # 最小空闲连接数
      maximum-pool-size:20     # 最大连接数
      idle-timeout:600000      # 连接空闲超时（毫秒），超时则关闭
      max-lifetime:1800000     # 连接最大生命周期（毫秒），防止数据库主动断开
      connection-timeout:30000 # 获取连接的等待超时时间
      connection-test-query:SELECT 1  # 连接有效性测试 SQL
```

### **Druid（阿里巴巴，带监控）**

`pom.xml`：

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.2.18</version>
</dependency>
```

`application.yml`：

```yaml
spring:
  datasource:
    type:com.alibaba.druid.pool.DruidDataSource
    driver-class-name:com.mysql.cj.jdbc.Driver
    url:jdbc:mysql://localhost:3306/your_db?useSSL=false&serverTimezone=Asia/Shanghai
    username:root
    password:your_password
    druid:
      initial-size:5          # 初始连接数
      min-idle:5              # 最小空闲连接数
      max-active:20           # 最大连接数
      max-wait:60000          # 获取连接最大等待时间（毫秒）
      time-between-eviction-runs-millis:60000   # 检测空闲连接的间隔
      min-evictable-idle-time-millis:300000      # 连接最小空闲时间
      validation-query:SELECT 1
      test-while-idle:true    # 连接空闲时是否做有效性检测
      test-on-borrow:false
      test-on-return:false
      # 开启 Druid 监控（可选）
      stat-view-servlet:
        enabled:true
        url-pattern:/druid/*
        login-username:admin
        login-password:admin123
      filters:stat,wall       # stat=监控统计，wall=SQL防火墙
```

### **c3p0**

`pom.xml`：

```xml
<dependency>
    <groupId>com.mchange</groupId>
    <artifactId>c3p0</artifactId>
    <version>0.9.5.5</version>
</dependency>
```

`application.yml`：

```yaml
spring:
  datasource:
    type:com.mchange.v2.c3p0.ComboPooledDataSource
    driver-class-name:com.mysql.cj.jdbc.Driver
    url:jdbc:mysql://localhost:3306/your_db
    username:root
    password:your_password
```

`c3p0-config.xml`（放在 resources 目录）：

```xml
<c3p0-config>
    <default-config>
        <property name="initialPoolSize">5</property>
        <property name="minPoolSize">5</property>
        <property name="maxPoolSize">20</property>
        <property name="maxIdleTime">600</property>
        <property name="acquireIncrement">3</property>
    </default-config>
</c3p0-config>
```

---

## **关键参数说明**

| **参数名（HikariCP）** | **含义** | **建议值** |
| --- | --- | --- |
| `minimumIdle` | 最小空闲连接数（低峰期保持的连接） | 与 `maximumPoolSize` 相同，避免频繁创建销毁 |
| `maximumPoolSize` | 最大连接数 | CPU 核数 × 2，或压测决定（通常 10~20） |
| `connectionTimeout` | 等待连接的超时时间 | 30000 ms（30秒） |
| `idleTimeout` | 连接空闲多久后被回收 | 600000 ms（10分钟） |
| `maxLifetime` | 连接最长存活时间 | 1800000 ms（30分钟），**需小于数据库 wait_timeout** |

> ⚠️ **重要**：`maxLifetime` 必须小于 MySQL 的 `wait_timeout`（默认8小时）。否则 MySQL 主动断开连接但连接池不知情，会出现 **"Connection is closed"** 错误。
> 

## **索引对性能的影响（压测数据）**

连接池只是性能的一方面，索引同样关键。

### **更新操作对比（100万数据，20并发，更新200条）**

| **场景** | **耗时** | **原因** |
| --- | --- | --- |
| 无索引字段 WHERE | 24小时+ | 全表扫描 + 行锁扩大为表锁 |
| 有索引但区分度低（如 userId，1000人产生100万数据） | 24小时+ | 锁住大量行，并发争用严重 |
| 有索引且区分度高（如 orderId，全局唯一） | **0.432 秒** | 精准锁定一行，互不影响 |

### **5.2 查询操作对比（100万数据，20并发，查询5000次）**

| **场景** | **耗时** | **50%** | **80%** |
| --- | --- | --- | --- |
| 无索引 | 6小时+ | 7s | 9s |
| 有索引，区分度低 | 8.343 s | 13ms | 20ms |
| 有索引，区分度高 | **2.051 s** | **7ms** | **10ms** |
| 联合索引：高区分度在前 | 2.168 s | 7ms | 11ms |
| 联合索引：低区分度在前 | 3.279 s | 11ms | 17ms |

### **5.3 索引使用原则总结**

```
✅ 对高频 WHERE 条件字段建索引
✅ 联合索引中，区分度高的字段放前面（最左前缀原则）
✅ 优先用唯一性高的字段（如 orderId）作为更新/查询条件
❌ 区分度极低的字段（如性别 0/1）不值得建索引
❌ 不要对无索引字段做批量更新，会锁全表
```

---


## **压测工具 ApacheBench 快速入门**

ApacheBench（ab）是 Apache 自带的 HTTP 压测工具，用于模拟并发请求。

```
# 基本语法
ab -c <并发数> -n <总请求数> <URL>

# 示例：20个并发，发送1000次请求
ab -c 20 -n 1000 http://127.0.0.1:8091/api/mysql/insert

# 关键输出指标说明
# Requests per second    每秒处理请求数（越大越好）
# Time per request       平均每个请求耗时（越小越好）
# Percentage of the requests served within a certain time（50%/80%/90% 响应时间）

$ docker run --rm -it jordi/ab -c 20 -n 10000 http://172.23.206.112:8091/api/mysql/insert

```

---

这个测试数据库部署在云服务器现在暂时做不了，我提供的http接口在本机，云服务器访问不了我本机网络，需要内网穿透，后面再搞搞！！！

## **总结**

```
**高性能数据库访问 = 合适的连接池 + 正确的索引**

连接池选型：
  SpringBoot 默认           → HikariCP  （极速、简洁）
  需要监控/SQL防火墙         → Druid     （功能丰富）
  老项目维护                 → c3p0/DBCP

关键参数：
  maximumPoolSize  → 根据并发压测决定，不是越大越好
  maxLifetime      → 必须 < MySQL wait_timeout

索引原则：
  更新/查询条件字段务必建索引
  联合索引：高区分度字段在前
  区分度低的字段（性别等）不要建索引
```