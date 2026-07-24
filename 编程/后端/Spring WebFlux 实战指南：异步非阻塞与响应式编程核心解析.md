---
title: Spring WebFlux 实战指南：异步非阻塞与响应式编程核心解析
author: 苏三
source: https://mp.weixin.qq.com/s/8yB-XLQ1rQ4hZ3RRm3dbrg
date: 2025-07-24
tags:
  - Spring-WebFlux
  - 响应式编程
  - Reactor
  - Netty
  - R2DBC
  - 高并发
  - 后端
---

# Spring WebFlux 实战指南：异步非阻塞与响应式编程核心解析

> 原文链接：[苏三](https://mp.weixin.qq.com/s/8yB-XLQ1rQ4hZ3RRm3dbrg)

---

## 一、WebFlux 要解决什么问题？

### 传统 Spring MVC 的瓶颈

```java
@GetMapping("/order/{id}")
public Order getOrder(@PathVariable Long id) {
    User user = userService.getUser(order.getUserId());  // 线程阻塞 2 秒！
    return order;
}
```

**"一个请求，一个线程"模型**下，线程在等待 I/O 时完全空闲，但资源却被占着不放。1000 个并发就需要 1000 个线程，每线程约 1MB 栈内存，大量时间浪费在线程上下文切换上。

> 核心问题：投入大量线程资源，仅仅是为了"等待"，而不是"计算"。

---

## 二、异步非阻塞 + 响应式流

### 线程模型对比

| 维度 | Spring MVC | Spring WebFlux |
|------|-----------|---------------|
| 编程模型 | 命令式 / 同步阻塞 | 声明式 / 异步非阻塞 |
| 线程模型 | Thread-per-Request | Event-Loop + 少量线程 |
| I/O 模型 | 阻塞式 | 非阻塞式 |
| 并发能力 | 线程池大小决定（200-500） | 数万～数十万连接 |
| 背压支持 | 无 | 原生支持 |
| 默认服务器 | Tomcat / Jetty | Netty |

> 万级并发场景下，WebFlux 相比传统 Servlet 容器可减少 **30%-50%** 内存消耗，CPU 利用率低 **40%**，P99 延迟更稳定。

### Mono 与 Flux

```java
// 0 或 1 个结果 → Mono
@GetMapping("/user/{id}")
public Mono<User> getUser(@PathVariable Long id) {
    return userRepository.findById(id);   // 立即返回，不等待
}

// 0 到 N 个结果 → Flux
@GetMapping("/users")
public Flux<User> getUsers() {
    return userRepository.findAll();
}
```

---

## 三、从零搭建 WebFlux 应用

### 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
<!-- 注意：不需要 spring-boot-starter-web，两者互斥 -->
```

### 响应式 Controller

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    @GetMapping("/{id}")
    public Mono<User> getUser(@PathVariable Long id) {
        return userService.findById(id)
            .switchIfEmpty(Mono.error(new UserNotFoundException(id)));
    }

    @GetMapping
    public Flux<User> getUsers(@RequestParam(defaultValue = "0") int page,
                                @RequestParam(defaultValue = "20") int size) {
        return userService.findAll(page, size);
    }

    @PostMapping
    public Mono<User> createUser(@RequestBody Mono<User> userMono) {
        return userMono.flatMap(userService::save);
    }
}
```

### 响应式 Service

```java
@Service
public class UserService {
    private final ReactiveUserRepository userRepository;

    public Mono<User> findById(Long id) {
        return userRepository.findById(id);
    }

    public Flux<User> findAll(int page, int size) {
        return userRepository.findAll().skip((long) page * size).take(size);
    }
}
```

### R2DBC：端到端非阻塞的关键

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-r2dbc</artifactId>
</dependency>
<dependency>
    <groupId>io.r2dbc</groupId>
    <artifactId>r2dbc-postgresql</artifactId>
</dependency>
```

```java
@Repository
public interface ReactiveUserRepository 
        extends ReactiveCrudRepository<User, Long> {
    Mono<User> findByEmail(String email);
    Flux<User> findByAgeGreaterThan(int age);
}
```

> ⚠️ 没有 R2DBC 而只用 WebFlux + 传统 JDBC，数据库访问仍是阻塞的，端到端非阻塞就断在了数据库这一层。

---

## 四、WebClient：响应式 HTTP 客户端

替代 RestTemplate，支持并行调用：

```java
@Service
public class OrderService {
    private final WebClient webClient;

    public Mono<User> getUserWithOrders(Long userId) {
        // 两个调用并行执行
        Mono<User> userMono = webClient.get()
            .uri("/api/users/{id}", userId).retrieve().bodyToMono(User.class);

        Flux<Order> ordersFlux = webClient.get()
            .uri("/api/orders?userId={id}", userId).retrieve().bodyToFlux(Order.class);

        return userMono.zipWith(ordersFlux.collectList())
            .map(tuple -> { User u = tuple.getT1(); u.setOrders(tuple.getT2()); return u; });
    }
}
```

> RestTemplate 串行耗时 = 两者之和；WebClient 并行耗时 = 两者最大值。

---

## 五、为什么越来越多的人使用？

1. **资源效率革命**：同等硬件吞吐量提升 3-5 倍，延迟降低 60%+。Netty Event Loop 线程数 = CPU 核心数，而 Tomcat 线程池可达数百上千
2. **流式数据原生支持**：SSE、WebSocket 无需额外处理
   ```java
   @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
   public Flux<StockPrice> streamStockPrices() {
       return stockService.priceStream().delayElements(Duration.ofSeconds(1));
   }
   ```
3. **背压机制**：消费者可主动告诉生产者"慢一点"，防止系统过载
4. **GraalVM 原生镜像**：启动时间优化至 100ms 以内，适配 Serverless 弹性伸缩
5. **与 Spring 生态深度融合**：无需引入额外框架

---

## 六、缺点

| 缺点 | 说明 |
|------|------|
| 学习曲线陡峭 | 需理解响应式思维、Mono/Flux 操作符、背压等新概念 |
| 调试难度高 | 堆栈跟踪不直观，flatMap 链中异常可能跨多个操作符 |
| 生态适配待完善 | 部分传统 JDBC/ORM 库无响应式版本，需 R2DBC 替代 |
| 简单 CRUD 杀鸡牛刀 | 性能提升可能不足以抵消复杂性 |
| 配置调优要求高 | 高并发场景需调 Netty 参数 |

---

## 七、适用场景

| 场景 | 推荐 | 理由 |
|------|------|------|
| API 网关 | ⭐强烈推荐 | 高并发、I/O 密集、服务聚合 |
| 实时数据推送 | ⭐强烈推荐 | SSE/WebSocket 原生支持 |
| 微服务间调用 | ⭐强烈推荐 | WebClient 并行调用，延迟明显降低 |
| 流式数据处理 | ⭐强烈推荐 | 背压机制天然适配 |
| Serverless/FaaS | ⭐推荐 | GraalVM 原生镜像启动快 |
| 传统 CRUD | ⚠️ 需评估 | 收益不明显 |
| CPU 密集型 | ❌ 不推荐 | 增加了调度开销 |

> 判断法则：**I/O 密集型用 WebFlux，CPU 密集型用 MVC + 虚拟线程。**

---

## 八、虚拟线程 vs WebFlux

2026 年 Java 虚拟线程（Project Loom）让同步代码也能低成本支撑高并发。

| | 虚拟线程 | WebFlux |
|---|---------|---------|
| 编程模型 | 同步（易） | 响应式（陡） |
| 背压 | 不支持 | 原生支持 |
| 流式处理 | 别扭 | 天然适配 |
| 并发能力 | 接近 WebFlux | 极致 |

> 二者互补：虚拟线程覆盖大部分传统业务，WebFlux 在流式处理、背压、细粒度调度场景仍有不可替代的优势。

---

> WebFlux 不是银弹。I/O 密集型需要支撑高并发时，WebFlux 值得深入。简单 CRUD 用 MVC + 虚拟线程更务实。技术选型没有标准答案——理解自己的业务场景，选择最匹配的方案。
