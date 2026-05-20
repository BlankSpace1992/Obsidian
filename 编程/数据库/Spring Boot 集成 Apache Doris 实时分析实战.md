---
title: Spring Boot 集成 Apache Doris 实时分析实战
author: Java知音
source: https://mp.weixin.qq.com/s/NKl489tCmlxxHEUWIn5SpQ
date: 2025-05-14
tags:
  - Spring-Boot
  - Apache-Doris
  - 数据库
  - OLAP
  - 实时分析
  - Java
---

# Spring Boot 集成 Apache Doris 实时分析实战

> 原文链接：[Java知音](https://mp.weixin.qq.com/s/NKl489tCmlxxHEUWIn5SpQ)

---

## 前言

在大数据爆发的今天，实时数据分析已成为企业决策的核心支撑——电商需要实时监控爆款库存，运维需要秒级定位系统异常，营销需要实时调整投放策略。

MySQL 在中小规模数据和简单查询中表现稳定，但面对**千万级以上数据量的复杂聚合分析**时（如多维度统计、漏斗转化计算），往往查询超时、性能暴跌。

**Apache Doris** 专为实时数据分析场景设计，完美弥补了 MySQL 的短板。

---

## Apache Doris 简介

Apache Doris 是基于 **MPP（大规模并行处理）** 架构的开源分析型数据库，起源于百度，2018 年开源并捐赠给 Apache 基金会。

专为 **OLAP** 场景设计，支持 PB 级数据低延迟查询，无需复杂的分库分表或数据预处理。

### 核心特性

1. **极致查询性能** — MPP 架构多节点并行计算，响应低至毫秒级，比 MySQL 快 10-100 倍
2. **实时数据接入** — 支持 Kafka、Flink 实时导入，也可同步 MySQL、Hive，延迟低至秒级
3. **兼容 MySQL 协议** — 无需修改 SQL 语法，支持 JDBC/ODBC，集成成本极低
4. **高并发支持** — 轻松支撑每秒数千次查询，适合数据看板、自助分析平台
5. **极简运维** — 支持单节点到数百节点横向扩展，自动分片、负载均衡

### Doris vs MySQL 适用场景

| 维度 | MySQL | Apache Doris |
|------|-------|-------------|
| 定位 | 事务型（OLTP） | 分析型（OLAP） |
| 数据规模 | 千万级以下 | PB 级 |
| 查询延迟 | 秒级~分钟级 | 毫秒级~秒级 |
| 并发能力 | 中等 | 高 |
| 最佳场景 | 写多查少（订单、注册） | 读多写少（统计、报表） |

> 一句话：MySQL 适合"写多查少"的业务交易，Doris 适合"读多写少"的实时分析。

---

## Spring Boot 集成实战

### 环境要求

- JDK 1.8+
- Spring Boot 2.3+
- Maven 3.6+
- Apache Doris 已部署（推荐 Docker 快速部署）

### 第一步：创建项目

通过 Spring Initializr 创建，选择：
- Spring Boot 版本：2.7.x
- 依赖：Spring Web、Spring Data JPA、MySQL Driver

```
src/
├── main/
│   ├── java/com/example/dorisdemo/
│   │   ├── controller/     // 接口层
│   │   ├── entity/         // 实体类
│   │   ├── repository/     // 数据访问层
│   │   ├── service/        // 业务层
│   │   └── DorisDemoApplication.java
│   └── resources/
│       └── application.yml
└── pom.xml
```

### 第二步：核心配置

#### pom.xml

无需额外引入 Doris 专属依赖，直接使用 MySQL JDBC 驱动：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<!-- MySQL JDBC驱动（兼容Doris） -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>
```

#### application.yml

```yaml
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:9030/demo_db?useSSL=false&serverTimezone=Asia/Shanghai
    username: root
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: none          # Doris 表手动创建
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQL8Dialect
        format_sql: true
    show-sql: true
```

> 💡 Doris 的 JDBC 端口默认 9030（FE 查询端口）

### 第三步：创建 Doris 数据表

```sql
CREATE DATABASE IF NOT EXISTS demo_db;
USE demo_db;

CREATE TABLE IF NOT EXISTS user_behavior (
    user_id       BIGINT      COMMENT '用户ID',
    product_id    BIGINT      COMMENT '商品ID',
    category_id   INT         COMMENT '商品分类ID',
    behavior_type VARCHAR(20) COMMENT '行为类型（click/purchase/collect）',
    create_time   DATETIME    COMMENT '行为时间'
) ENGINE=OLAP
DUPLICATE KEY(user_id, product_id)
PARTITION BY RANGE(create_time) (
    PARTITION p202401 VALUES LESS THAN ('2024-02-01'),
    PARTITION p202402 VALUES LESS THAN ('2024-03-01')
)
DISTRIBUTED BY HASH(user_id) BUCKETS 10
PROPERTIES (
    "storage_medium" = "HDD",
    "storage_ttl" = "30 DAY"
);
```

### 第四步：编写代码

#### 实体类

```java
@Data
@Entity
@Table(name = "user_behavior")
@DynamicInsert
@DynamicUpdate
public class UserBehavior {
    @Id
    @Column(name = "user_id")
    private Long userId;
    @Column(name = "product_id")
    private Long productId;
    @Column(name = "category_id")
    private Integer categoryId;
    @Column(name = "behavior_type")
    private String behaviorType;
    @Column(name = "create_time")
    private LocalDateTime createTime;
}
```

#### Repository

```java
@Repository
public interface UserBehaviorRepository extends JpaRepository<UserBehavior, Long> {
    List<UserBehavior> findByCreateTimeBetween(LocalDateTime start, LocalDateTime end);

    @Query(value = "SELECT COUNT(*) FROM user_behavior WHERE category_id = :categoryId AND behavior_type = 'click' AND create_time BETWEEN :start AND :end", nativeQuery = true)
    Long countClickByCategoryId(@Param("categoryId") Integer categoryId,
                                @Param("start") LocalDateTime start,
                                @Param("end") LocalDateTime end);
}
```

#### Service

```java
@Service
@RequiredArgsConstructor
public class UserBehaviorService {
    private final UserBehaviorRepository behaviorRepository;

    public UserBehavior save(UserBehavior behavior) {
        return behaviorRepository.save(behavior);
    }

    public List<UserBehavior> batchSave(List<UserBehavior> behaviors) {
        return behaviorRepository.saveAll(behaviors);
    }

    public List<UserBehavior> getBehaviorByDateRange(LocalDateTime start, LocalDateTime end) {
        return behaviorRepository.findByCreateTimeBetween(start, end);
    }

    public Long getCategoryClickCount(Integer categoryId, LocalDateTime start, LocalDateTime end) {
        return behaviorRepository.countClickByCategoryId(categoryId, start, end);
    }
}
```

#### Controller

```java
@RestController
@RequestMapping("/user-behavior")
@RequiredArgsConstructor
public class UserBehaviorController {
    private final UserBehaviorService behaviorService;

    @PostMapping
    public UserBehavior save(@RequestBody UserBehavior behavior) {
        return behaviorService.save(behavior);
    }

    @PostMapping("/batch")
    public List<UserBehavior> batchSave(@RequestBody List<UserBehavior> behaviors) {
        return behaviorService.batchSave(behaviors);
    }

    @GetMapping("/range")
    public List<UserBehavior> getByRange(
            @RequestParam @DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss") LocalDateTime start,
            @RequestParam @DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss") LocalDateTime end) {
        return behaviorService.getBehaviorByDateRange(start, end);
    }

    @GetMapping("/click-count")
    public Long getClickCount(
            @RequestParam Integer categoryId,
            @RequestParam @DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss") LocalDateTime start,
            @RequestParam @DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss") LocalDateTime end) {
        return behaviorService.getCategoryClickCount(categoryId, start, end);
    }
}
```

### 第五步：测试验证

**批量插入测试数据：**

```bash
POST /user-behavior/batch
Content-Type: application/json

[
    {"userId":1001,"productId":2001,"categoryId":301,"behaviorType":"click","createTime":"2024-01-15 10:30:00"},
    {"userId":1002,"productId":2002,"categoryId":301,"behaviorType":"click","createTime":"2024-01-15 11:20:00"},
    {"userId":1003,"productId":2003,"categoryId":302,"behaviorType":"purchase","createTime":"2024-01-15 14:10:00"}
]
```

**统计分类点击量：**

```bash
GET /user-behavior/click-count?categoryId=301&start=2024-01-01 00:00:00&end=2024-01-31 23:59:59
```

---

## 性能对比

100 万条用户行为数据测试，Doris 在分析场景下的性能远超 MySQL，数据量越大优势越明显。

---

## 小结

Spring Boot 与 Apache Doris 集成极其简单（兼容 MySQL 协议），但带来的性能提升是革命性的——无需关注复杂分布式架构，就能快速实现 PB 级数据的实时分析能力。

> 只要涉及大数据量的实时统计、报表生成、多维度分析，"Spring Boot + Apache Doris" 都是值得优先选择的方案。
