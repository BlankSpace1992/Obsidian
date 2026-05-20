---
title: SpringBoot 接口防抖：防重复提交实现方案
author: java1234
source: http://mp.weixin.qq.com/s?__biz=MzIxNTAwNjA4OQ==&mid=2247566783&idx=1&sn=974f61d4f4f59161cf03c7e88135027a
date: 2025-05-14
tags:
  - Spring-Boot
  - Java
  - 接口防抖
  - Redis
  - Redisson
  - 幂等性
---

# SpringBoot 接口防抖：防重复提交实现方案

> 原文链接：[java1234](http://mp.weixin.qq.com/s?__biz=MzIxNTAwNjA4OQ==&mid=2247566783&idx=1&sn=974f61d4f4f59161cf03c7e88135027a)

---

## 前言

所谓**防抖**，一是防用户手抖，二是防网络抖动。

在 Web 系统中，如果不加控制，容易因用户误操作或网络延迟导致同一请求被发送多次，生成重复数据。

- **前端**：按钮 loading 状态，阻止多次点击
- **后端**：网络波动造成的请求重发，仅靠前端不够，后端也需防抖

理想的防抖机制应具备：
- 逻辑正确，不误判
- 响应迅速，不能太慢
- 易于集成，逻辑与业务解耦
- 良好的用户反馈（如提示"您点击的太快了"）

---

## 哪些接口需要防抖？

- **用户输入类**：搜索框输入、表单输入等频繁触发的接口
- **按钮点击类**：提交表单、保存设置等
- **滚动加载类**：下拉刷新、上拉加载更多

---

## 如何确定请求是重复的？

1. **时间间隔**：大于一定时间间隔的一定不是重复提交
2. **参数比对**：选择标识性强的参数（不一定要全部参数）
3. **请求地址**：可选，进一步减少误判

---

## 分布式部署方案

### 方案一：共享缓存（Redis）

利用 Redis 的 `SET IF ABSENT` 特性，请求时写入一个带过期时间的 key，存在则视为重复。

### 方案二：分布式锁

利用 Redisson 等分布式锁组件，请求时尝试抢锁，抢不到则视为重复。

---

## 具体实现

### 基础接口

```java
@PostMapping("/add")
@RequiresPermissions(value = "add")
@Log(methodDesc = "添加用户")
public ResponseEntity<String> add(@RequestBody AddReq addReq) {
    return userService.add(addReq);
}
```

```java
@Data
public class AddReq {
    private String userName;
    private String userPhone;
    private List<Long> roleIdList;
}
```

### 1. 定义 @RequestLock 注解

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RequestLock {
    String prefix() default "";        // redis 锁前缀
    long expire() default 5;           // 过期时间
    TimeUnit timeUnit() default TimeUnit.SECONDS;  // 时间单位
    String delimiter() default "&";    // key 分隔符
}
```

### 2. 定义 @RequestKeyParam 注解

标记哪些参数/字段参与唯一 key 的生成：

```java
@Target({ElementType.METHOD, ElementType.PARAMETER, ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
public @interface RequestKeyParam {
}
```

### 3. 唯一 key 生成器

```java
public class RequestKeyGenerator {
    public static String getLockKey(ProceedingJoinPoint joinPoint) {
        MethodSignature methodSignature = (MethodSignature) joinPoint.getSignature();
        Method method = methodSignature.getMethod();
        RequestLock requestLock = method.getAnnotation(RequestLock.class);
        final Object[] args = joinPoint.getArgs();
        final Parameter[] parameters = method.getParameters();
        StringBuilder sb = new StringBuilder();

        // 优先检查方法参数上的 @RequestKeyParam
        for (int i = 0; i < parameters.length; i++) {
            final RequestKeyParam keyParam = parameters[i].getAnnotation(RequestKeyParam.class);
            if (keyParam == null) continue;
            sb.append(requestLock.delimiter()).append(args[i]);
        }

        // 若方法参数上没有注解，则检查对象属性上的注解
        if (StringUtils.isEmpty(sb.toString())) {
            final Annotation[][] parameterAnnotations = method.getParameterAnnotations();
            for (int i = 0; i < parameterAnnotations.length; i++) {
                final Object object = args[i];
                final Field[] fields = object.getClass().getDeclaredFields();
                for (Field field : fields) {
                    final RequestKeyParam annotation = field.getAnnotation(RequestKeyParam.class);
                    if (annotation == null) continue;
                    field.setAccessible(true);
                    sb.append(requestLock.delimiter())
                      .append(ReflectionUtils.getField(field, object));
                }
            }
        }
        return requestLock.prefix() + sb;
    }
}
```

### 4A. Redis 缓存方式实现

```java
@Aspect
@Configuration
@Order(2)
public class RedisRequestLockAspect {

    private final StringRedisTemplate stringRedisTemplate;

    @Autowired
    public RedisRequestLockAspect(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }

    @Around("execution(public * * (..)) && @annotation(com.summo.demo.config.requestlock.RequestLock)")
    public Object interceptor(ProceedingJoinPoint joinPoint) {
        MethodSignature methodSignature = (MethodSignature) joinPoint.getSignature();
        Method method = methodSignature.getMethod();
        RequestLock requestLock = method.getAnnotation(RequestLock.class);

        if (StringUtils.isEmpty(requestLock.prefix())) {
            throw new BizException(ResponseCodeEnum.BIZ_CHECK_FAIL, "重复提交前缀不能为空");
        }

        final String lockKey = RequestKeyGenerator.getLockKey(joinPoint);

        // SET IF ABSENT：键不存在则设置，已存在则不设置
        final Boolean success = stringRedisTemplate.execute(
            (RedisCallback<Boolean>) connection -> connection.set(
                lockKey.getBytes(), new byte[0],
                Expiration.from(requestLock.expire(), requestLock.timeUnit()),
                RedisStringCommands.SetOption.SET_IF_ABSENT));

        if (!success) {
            throw new BizException(ResponseCodeEnum.BIZ_CHECK_FAIL, "您的操作太快了,请稍后重试");
        }
        try {
            return joinPoint.proceed();
        } catch (Throwable throwable) {
            throw new BizException(ResponseCodeEnum.BIZ_CHECK_FAIL, "系统异常");
        }
    }
}
```

### 4B. Redisson 分布式锁方式实现

引入依赖：

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.10.6</version>
</dependency>
```

```java
@Aspect
@Configuration
@Order(2)
public class RedissonRequestLockAspect {

    private RedissonClient redissonClient;

    @Autowired
    public RedissonRequestLockAspect(RedissonClient redissonClient) {
        this.redissonClient = redissonClient;
    }

    @Around("execution(public * * (..)) && @annotation(com.summo.demo.config.requestlock.RequestLock)")
    public Object interceptor(ProceedingJoinPoint joinPoint) {
        MethodSignature methodSignature = (MethodSignature) joinPoint.getSignature();
        Method method = methodSignature.getMethod();
        RequestLock requestLock = method.getAnnotation(RequestLock.class);

        if (StringUtils.isEmpty(requestLock.prefix())) {
            throw new BizException(ResponseCodeEnum.BIZ_CHECK_FAIL, "重复提交前缀不能为空");
        }

        final String lockKey = RequestKeyGenerator.getLockKey(joinPoint);
        RLock lock = redissonClient.getLock(lockKey);
        boolean isLocked = false;

        try {
            isLocked = lock.tryLock();
            if (!isLocked) {
                throw new BizException(ResponseCodeEnum.BIZ_CHECK_FAIL, "您的操作太快了,请稍后重试");
            }
            lock.lock(requestLock.expire(), requestLock.timeUnit());
            try {
                return joinPoint.proceed();
            } catch (Throwable throwable) {
                throw new BizException(ResponseCodeEnum.BIZ_CHECK_FAIL, "系统异常");
            }
        } catch (Exception e) {
            throw new BizException(ResponseCodeEnum.BIZ_CHECK_FAIL, "您的操作太快了,请稍后重试");
        } finally {
            if (isLocked && lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

### 5. 使用方式

```java
@RequestLock(prefix = "user:add", expire = 5)
@PostMapping("/add")
public ResponseEntity<String> add(@RequestBody @RequestKeyParam AddReq addReq) {
    return userService.add(addReq);
}
```

---

## 测试结果

- 第一次提交：✅ 添加用户成功
- 短时间重复提交：❌ "您的操作太快了,请稍后重试"
- 过几秒后再次提交：✅ 添加用户成功

---

## 注意事项

> 仅靠防抖不能完全保证接口幂等性，还需配合：
> - 业务代码判断
> - 数据库表 UK 索引
> - 唯一 key 中加入用户 ID、IP 等信息以减少误判
