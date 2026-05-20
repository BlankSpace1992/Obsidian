---
title: SpringBoot 高并发请求合并：批量查询优化实践
author: Java知音
source: http://mp.weixin.qq.com/s?__biz=MzI4Njc5NjM1NQ==&mid=2247562943&idx=1&sn=2fed536773afb80ecdd1d7f195c471f7
date: 2025-05-14
tags:
  - Spring-Boot
  - Java
  - 高并发
  - 性能优化
  - CompletableFuture
  - 请求合并
---

# SpringBoot 高并发请求合并：批量查询优化实践

> 原文链接：[Java知音](http://mp.weixin.qq.com/s?__biz=MzI4Njc5NjM1NQ==&mid=2247562943&idx=1&sn=2fed536773afb80ecdd1d7f195c471f7)

---

## 前言

假设 3 个用户（id: 1、2、3）都要查询自己的基本信息，传统方式会对数据库发出 3 次请求。

**请求合并**的思路：在服务端把多个请求合并，只发出一条 SQL 查询数据库，返回后根据唯一请求 ID 分组，返回给对应用户。

> 数据库连接资源相当宝贵，RPC 调用也是同理。

---

## 技术手段

- **LinkedBlockingQueue** — 阻塞队列
- **ScheduledThreadPoolExecutor** — 定时任务线程池
- **CompletableFuture** — 异步阻塞机制（Java 8 没有 timeout 机制，后续用队列替代优化）

---

## 代码实现

### 1. 批量查询 Service

```java
@Service
public class UserServiceImpl implements UserService {
    @Resource
    private UsersMapper usersMapper;

    @Override
    public Map<String, Users> queryUserByIdBatch(List<UserWrapBatchService.Request> userReqs) {
        List<Long> userIds = userReqs.stream()
            .map(UserWrapBatchService.Request::getUserId)
            .collect(Collectors.toList());

        QueryWrapper<Users> queryWrapper = new QueryWrapper<>();
        queryWrapper.in("id", userIds);  // 用 IN 语句合并成一条 SQL
        List<Users> users = usersMapper.selectList(queryWrapper);

        // 按 userId 分组
        Map<Long, List<Users>> userGroup = users.stream()
            .collect(Collectors.groupingBy(Users::getId));

        HashMap<String, Users> result = new HashMap<>();
        userReqs.forEach(val -> {
            List<Users> usersList = userGroup.get(val.getUserId());
            if (!CollectionUtils.isEmpty(usersList)) {
                result.put(val.getRequestId(), usersList.get(0));
            } else {
                result.put(val.getRequestId(), null);
            }
        });
        return result;
    }
}
```

### 2. 请求合并核心（CompletableFuture 方案）

```java
@Service
public class UserWrapBatchService {
    @Resource
    private UserService userService;

    public static int MAX_TASK_NUM = 100;  // 单次最大批量数

    // 请求包装类
    public class Request {
        String requestId;                    // 唯一请求 ID
        Long userId;                         // 查询参数
        CompletableFuture<Users> completableFuture;  // 异步结果
    }

    private final Queue<Request> queue = new LinkedBlockingQueue();

    @PostConstruct
    public void init() {
        ScheduledExecutorService executor = Executors.newScheduledThreadPool(1);
        // 初始化 100ms 后执行，每 10ms 周期性执行
        executor.scheduleAtFixedRate(() -> {
            int size = queue.size();
            if (size == 0) return;

            List<Request> list = new ArrayList<>();
            for (int i = 0; i < size && i < MAX_TASK_NUM; i++) {
                list.add(queue.poll());
            }

            // 批量查询（一条 SQL）
            Map<String, Users> response = userService.queryUserByIdBatch(list);

            // 将结果分发给各自的请求
            for (Request request : list) {
                Users result = response.get(request.requestId);
                request.completableFuture.complete(result);
            }
        }, 100, 10, TimeUnit.MILLISECONDS);
    }

    public Users queryUser(Long userId) {
        Request request = new Request();
        request.requestId = UUID.randomUUID().toString().replace("-", "");
        request.userId = userId;
        request.completableFuture = new CompletableFuture<>();
        queue.offer(request);

        try {
            return request.completableFuture.get();  // 阻塞等待结果
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
        return null;
    }
}
```

**流程：**

```
多个请求 → 入队 LinkedBlockingQueue
                ↓
定时任务（每10ms） → 取出队列中所有请求
                ↓
合并为一条 SQL → 批量查询数据库
                ↓
根据 requestId 分发结果 → CompletableFuture.complete()
```

### 3. Controller 调用

```java
@RequestMapping("/merge")
public Callable<Users> merge(Long userId) {
    return () -> userBatchService.queryUser(userId);
}
```

> 使用 `Callable` 实现异步非阻塞。

---

## 优化方案：用队列替代 CompletableFuture 解决超时问题

Java 8 的 `CompletableFuture` 没有 timeout 机制，用 `LinkedBlockingQueue.poll(timeout)` 替代：

```java
public class Request {
    String requestId;
    Long userId;
    LinkedBlockingQueue<Users> usersQueue;  // 替代 CompletableFuture
}

public Users queryUser(Long userId) {
    Request request = new Request();
    request.requestId = UUID.randomUUID().toString().replace("-", "");
    request.userId = userId;
    request.usersQueue = new LinkedBlockingQueue<>();
    queue.offer(request);

    try {
        // 阻塞等待，最多 3 秒超时
        return request.usersQueue.poll(3000, TimeUnit.MILLISECONDS);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return null;
}
```

定时任务中将结果放入队列：

```java
for (Request userReq : userReqs) {
    Users users = response.get(userReq.getRequestId());
    userReq.usersQueue.offer(users);  // 结果入队，阻塞解除
}
```

---

## LinkedBlockingQueue vs ArrayBlockingQueue

| 特性 | LinkedBlockingQueue | ArrayBlockingQueue |
|------|---------------------|--------------------|
| 存储结构 | 链表 | 数组 |
| 默认长度 | Integer.MAX_VALUE（无界） | 需指定 |
| 锁机制 | 读写锁分离（putLock / takeLock） | 读写共用一把锁 |
| 并发性能 | 更高（生产者消费者可并行） | 较低 |
| OOM 风险 | 无界队列需注意 | 固定长度无此问题 |

---

## 注意事项

- ⚠️ SQL IN 语句有长度限制，需通过 `MAX_TASK_NUM` 限制批量大小
- ⚠️ 请求合并增加了等待时间，**不适合低并发场景**
- 适用场景：高并发下的数据库查询、RPC 调用等资源密集型操作

---

## 小结

请求合并、批量的办法能**大幅节省被调用系统的连接资源**。缺点是增加了等待时间，不适合低并发场景。

> 代码地址：[spring-boot-kubernetes v1.0.5](https://gitee.com/apple_1030907690/spring-boot-kubernetes/tree/v1.0.5)
