---
title: "在设计mysql数据库表时要注意的一些规范"
description: "mysql的学习笔记，规范的建表、查询、索引等操作"
date: "2026-06-02T12:47:11.157Z"
draft: false
showHeroImage: false
tags:
  - 索引
  - sql
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

## 为什么要遵守规范

规范不是限制，而是前人踩坑后总结出的最佳实践。不规范的库表设计会导致：

- **性能下降**：随机主键引发 InnoDB 页分裂、大量随机 I/O
- **索引失效**：隐式类型转换、字段上用函数、左模糊查询
- **数据错误**：NULL 参与聚合计算结果偏差、`count(列名)` 不统计 NULL
- **维护困难**：`select *` 在表结构变更后系统报错、字段无注释

## 建表规范

| **规范项** | **说明** |
| --- | --- |
| 命名风格 | 库名、表名、字段名全小写 + 下划线分隔，如 `user_order` |
| 命名长度 | 不超过 12 个字符（MySQL 支持最长 64 个） |
| 见名知意 | 用名词而非动词，如 `order_status` 而非 `get_status` |
| 存储引擎 | 必须使用 **InnoDB**（支持事务、行锁、高并发） |
| 字符集 | 推荐 **utf8mb4**（可存 emoji，兼容 emoji 字符） |
| 单表字段数 | 建议 **不超过 40 个**，过多应考虑垂直拆表 |

```sql
-- 正确示例
CREATE TABLE `user_order` (
  ...
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

## 字段规范

### 数值类型

| **场景** | **推荐类型** | **错误写法** | **原因** |
| --- | --- | --- | --- |
| 整数 | `INT` | `INT(4)` | 括号里的数字仅影响显示宽度，不影响存储范围，无意义 |
| 浮点/金额 | `DECIMAL(10,2)` | `FLOAT`、`DOUBLE` | 后者存在精度丢失问题，金融场景必须用 DECIMAL |
| 0~80 范围的状态 | `TINYINT` | `INT` | 节省存储空间 |
| 枚举值 | `TINYINT` | `ENUM` | ENUM 修改成本高，扩展性差 |

```sql
-- 金额字段
`unit_price`   DECIMAL(10,2) NOT NULL COMMENT '商品价格',
`total_amount` DECIMAL(10,2) NOT NULL COMMENT '支付金额',

-- 状态字段
`order_status` TINYINT(1) NOT NULL COMMENT '0创建 1完成 2掉单 3关单',
```

### 字符串类型

| **场景** | **推荐类型** | **说明** |
| --- | --- | --- |
| 手机号 | `VARCHAR(20)` | 不能用整型：可能含 +/- 区号，需要 LIKE 模糊查询 |
| 普通字符串 | `VARCHAR(N)` | 按实际最大长度定义 |
| IPV4 | `INT UNSIGNED` | 用 `INET_ATON`/`INET_NTOA` 函数互转，节省空间 |
| IPV6 | `VARBINARY(16)` | 用 `INET6_ATON`/`INET6_NTOA` 函数互转 |
| JSON 扩展数据 | `JSON` | MySQL 8.x 支持，可按字段路径查询 |

### 时间类型

| **场景** | **推荐类型** | **原因** |
| --- | --- | --- |
| 一般时间 | `DATETIME` | `TIMESTAMP` 只能表示到 2038 年 |
| 创建时间 | `DATETIME DEFAULT CURRENT_TIMESTAMP` | 自动填入当前时间 |
| 更新时间 | `DATETIME ON UPDATE CURRENT_TIMESTAMP` | 每次更新自动刷新 |

```sql
`update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
              ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
`create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
```

### 通用约束

- **所有字段必须 `NOT NULL`，并设默认值**
    - NULL 值导致每行占用额外存储空间
    - NULL 字段使索引统计和值比较更复杂
    - NULL 只能用 `IS NULL` / `IS NOT NULL` 查询，`!=`、`NOT IN` 均无法命中 NULL 行
- **所有字段必须有 `COMMENT` 注释**
- **是否类字段命名必须以 `is_` 开头**，如 `is_delete`

## 索引规范

### 主键设置

```sql
-- ✅ 正确：自增 bigint 主键
`id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '自增ID'

-- ❌ 错误：用业务字段（如 order_id）作主键
-- 随机性强的字段作主键 → InnoDB B+ 树页分裂 → 大量随机 I/O → 性能下降
```

INT 约 21 亿上限，BIGINT 约 9000 亿上限，大流量系统优先选 BIGINT。

### 索引数量控制

- 单表索引 **≤ 5 个**
- 单个索引的字段数 **≤ 5 个**
- 单条索引记录长度 **≤ 64 KB**
- 若已有 `idx(a, b)`，则 `idx(a)` 可以删除（前者可以覆盖后者的查询需求）

### 索引选择原则

**区分度（Selectivity）公式：**

```sql
-- 越接近 1 越适合建索引
SELECT COUNT(DISTINCT 字段) / COUNT(id) FROM 表名;
```

- `order_id`、`uuid` 这类唯一字段 → 非常适合建索引
- `status`、`type` 这类低区分度字段 → 单独建索引几乎无意义（反而降低写入性能）

### 覆盖索引（避免回表）

```sql
-- 组合索引 idx_sku_unit_price_total_amount(sku, unit_price, total_amount)
-- 以下查询直接命中索引，无需回表读取完整行数据
SELECT sku, unit_price, total_amount FROM user_order WHERE sku = 'SKU001';
```

**最左前缀原则**：组合索引中，区分度最高的字段放最左边。

## 索引失效的场景（重点）

| **失效原因** | **错误示例** | **正确写法** |
| --- | --- | --- |
| 字段上使用函数 | `WHERE LENGTH(name) = 10` | `WHERE name = 'xxxx'` |
| 字段参与表达式运算 | `WHERE user_id + 2 = 1002` | `WHERE user_id = 1000` |
| 隐式类型转换（字段类型与值类型不一致） | `WHERE user_id = 123`（user_id 是 varchar） | 保持类型一致 |
| 左模糊查询 | `WHERE name LIKE '%abc%'` | 加其他条件 或 改为 `LIKE 'abc%'` |
| 或条件（某些场景） | `WHERE a=1 OR b=2` | 改为 `UNION` + 各自索引 |

### 更新频繁字段不建立索引

索引维护有开销。频繁更新的字段（如每秒都在变化的计数器）不要建索引。

## SQL查询规范

### 查询规范

```sql
-- ❌ 禁止
SELECT * FROM user_order;

-- ✅ 正确：指定字段
SELECT user_name, sku, total_amount FROM user_order;
```

> `SELECT *` 会读取不需要的列，浪费网卡带宽；且表结构变动后 model 层未更新会报错。
> 

```sql
-- ❌ count(列名) 不统计 NULL 行
SELECT COUNT(user_name) FROM user_order;

-- ✅ count(*) 才是标准统计行数语法
SELECT COUNT(*) FROM user_order;
```

### INSERT 规范

```sql
-- ❌ 不指定字段名
INSERT INTO user_order VALUES(...);

-- ✅ 显式指定字段名
INSERT INTO user_order (user_name, sku, ...) VALUES (...);

-- ✅ 批量插入，但每次 VALUES 不超过 5000 个（否则主从同步延迟）
INSERT INTO user_order (user_name, sku) VALUES
  ('张三', 'SKU001'),
  ('李四', 'SKU002');
```

### 分页查询优化

```sql
-- ❌ 大偏移量 LIMIT 性能极差（全表扫描 + 丢弃前 N 行）
SELECT a, b, c FROM t1 LIMIT 10000, 20;

-- ✅ 利用主键过滤
SELECT a, b, c FROM t1 WHERE id > 10000 LIMIT 20;
```

### IN / UNION / OR

```sql
-- IN 列表不超过 500 个值
SELECT * FROM user_order WHERE user_id IN (/* 500 个以内 */);

-- UNION ALL（不去重）比 UNION（去重）性能更好
SELECT ... UNION ALL SELECT ...;

-- OR 可优化为 UNION
-- ❌
WHERE a = 1 OR b = 2
-- ✅
WHERE a = 1 UNION WHERE b = 2
```

### WHERE 子句规范

```sql
-- ❌ 等号两侧类型不一致（隐式转换导致索引失效）
WHERE user_id = 123        -- user_id 是 varchar 类型

-- ✅ 类型保持一致
WHERE user_id = '123'

-- ❌ 纯模糊查询无法用索引
WHERE name LIKE '%abc%'

-- ✅ 结合其他条件
WHERE order_status = 0 AND name LIKE 'abc%'
```

### 其他重要规范

| **规范** | **说明** |
| --- | --- |
| 单表行数控制 | 单表不超过 **500 万行**，文件不超过 2G，超过考虑分表 |
| 水平分表策略 | 业务表用**取模**；日志/报表类用**日期**分表 |
| ALTER TABLE | 超过 100 万行记录的表 ALTER，必须在**业务低峰期**执行（产生表锁） |
| TRUNCATE vs DELETE | TRUNCATE 更快但无事务、不触发 trigger，**生产代码中禁用 TRUNCATE** |
| 避免全表扫描 | 查询数据量不超过表总行数的 **25%**，否则可能不走索引 |
| 禁用 hint | 不要用 `force index`、`sql_no_cache`，相信 MySQL 优化器 |
| DDL 合并 | 同一张表的字段/索引变更，**合并成一条 DDL** 执行 |

## 规范化建表实例

```sql
CREATE TABLE `user_order` (
  -- 主键：bigint 自增，不用 order_id 做主键
  `id`              BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '自增ID',

  -- 字符串：varchar，含 NOT NULL
  `user_name`       VARCHAR(64)     NOT NULL COMMENT '用户姓名',
  `user_id`         VARCHAR(24)     NOT NULL COMMENT '用户编号',

  -- 手机号：varchar 而不是整型
  `user_mobile`     VARCHAR(20)     NOT NULL COMMENT '用户电话',

  `sku`             VARCHAR(64)     NOT NULL COMMENT '商品编号',
  `sku_name`        VARCHAR(128)    NOT NULL COMMENT '商品名称',
  `order_id`        VARCHAR(64)     NOT NULL COMMENT '订单ID',

  -- 整数：不写 INT(4)，直接 INT
  `quantity`        INT             NOT NULL DEFAULT '1' COMMENT '商品数量',

  -- 金额：DECIMAL，禁止 FLOAT/DOUBLE
  `unit_price`      DECIMAL(10,2)   NOT NULL COMMENT '商品价格',
  `discount_amount` DECIMAL(10,2)   NOT NULL COMMENT '折扣金额',
  `tax`             DECIMAL(4,2)    NOT NULL COMMENT '费率',
  `total_amount`    DECIMAL(10,2)   NOT NULL COMMENT '支付金额',

  -- 时间：DATETIME（支持 2038 年以后）
  `order_date`      DATETIME        NOT NULL COMMENT '订单日期',

  -- 状态：TINYINT，不用 ENUM
  `order_status`    TINYINT(1)      NOT NULL COMMENT '0创建 1完成 2掉单 3关单',

  -- 逻辑删除：is_ 前缀命名
  `is_delete`       TINYINT(1)      NOT NULL DEFAULT '0' COMMENT '0未删除 1已删除',

  -- UUID：分布式全局唯一，适合 binlog 同步 ES
  `uuid`            VARCHAR(128)    NOT NULL COMMENT '全局唯一标识',

  -- IPV4：INT UNSIGNED 存储，函数互转
  `ipv4`            INT UNSIGNED    NOT NULL DEFAULT '2130706433' COMMENT 'IPv4地址',

  -- IPV6：VARBINARY(16) 存储
  `ipv6`            VARBINARY(16)   NOT NULL COMMENT 'IPv6地址',

  -- JSON 扩展字段（MySQL 8.x）
  `ext_data`        JSON            NOT NULL COMMENT '扩展数据',

  -- 时间戳：自动维护
  `update_time`     DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP
                    ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `create_time`     DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',

  -- 索引设计
  PRIMARY KEY (`id`),
  UNIQUE KEY `uq_orderid`  (`order_id`),           -- 业务唯一键
  UNIQUE KEY `uq_uuid`     (`uuid`),               -- 全局唯一
  KEY `idx_order_date`     (`order_date`),         -- 单列索引（时间范围查询）
  KEY `idx_sku_unit_price_total_amount`             -- 组合索引（覆盖查询）
      (`sku`, `unit_price`, `total_amount`)

) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

这后面有点看不懂了，后面了解MySQL的底层再看

---

## SQL操作示例

### IP 地址存取

```
-- 插入（IPv4 整数存储）
INSERT INTO user_order (..., ipv4, ipv6, ...)
VALUES (..., INET_ATON('127.0.0.1'), INET6_ATON('2001:0db8:85a3:0000:0000:8a2e:0370:7334'), ...);

-- 查询（转换为可读格式）
SELECT user_name, INET_NTOA(ipv4), INET6_NTOA(ipv6)
FROM user_order;

-- 函数测试
SELECT INET_ATON('192.168.1.1');          -- 转整数
SELECT INET_NTOA(3232235777);             -- 转 IP 字符串
SELECT INET6_NTOA(INET6_ATON('::1'));     -- IPv6 互转验证
```

### JSON 字段查询

```
-- ->> 操作符读取 JSON 路径的值（去掉外层双引号）
SELECT
  user_name,
  ext_data->>'$.device'          AS device_info,
  ext_data->>'$.device.machine'  AS device_machine
FROM user_order;

-- 结果示例：
-- 小傅哥 | {"machine": "IPhone 14 Pro", "location": "shanghai"} | IPhone 14 Pro
```

### EXPLAIN 分析索引使用

```
-- 分析是否命中唯一索引
EXPLAIN SELECT user_name, sku FROM user_order
WHERE order_id = 'ORD002';
-- type = const，key = uq_orderid → 命中唯一索引 ✅

-- 分析组合索引覆盖查询
EXPLAIN SELECT sku, total_amount, order_date FROM user_order
WHERE total_amount > 10
  AND order_date BETWEEN '2023-08-09 00:00:00' AND '2023-08-09 23:59:59';
```

> **EXPLAIN 关键字段解读：**
> 
> - `type`：`const` > `ref` > `range` > `index` > `ALL`（ALL 为全表扫描，应避免）
> - `key`：实际使用的索引名称
> - `rows`：预估扫描行数，越小越好
> - `Extra`：`Using index` 表示覆盖索引（最优），`Using filesort` 表示需要排序

### FOR UPDATE（悲观锁）

```
-- 事务中锁定指定行，防止并发修改
START TRANSACTION;

SELECT user_name, sku, total_amount, order_status
FROM user_order
WHERE order_id = 'ORD002'
FOR UPDATE;   -- 其他事务无法修改此行，直到 COMMIT

-- 执行业务逻辑...

COMMIT;
```

> **注意**：`FOR UPDATE` 必须在索引列上使用，否则退化为**表锁**，阻塞整张表的写入。
> 

### 行级锁 UPDATE

```
-- order_id 是唯一索引 → 行级锁（只锁这一行）
UPDATE user_order
SET order_status = 1
WHERE order_id = 'ORD002' AND order_status = 0;

-- 若 WHERE 没有命中索引 → 表锁（锁住整张表，并发性差）
```

### 表锁触发场景

| **操作** | **说明** |
| --- | --- |
| `ALTER TABLE` | 自动获取表级排它锁，期间阻塞所有读写 |
| `LOCK TABLES` | 手动表锁，影响其他会话 |
| `TRUNCATE TABLE` | 获取排它锁，且无事务、不触发 trigger |