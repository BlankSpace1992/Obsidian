---
title: SpringBoot 权限校验：从 RBAC 到 ABAC 基于 JCasbin
author: 风象南
source: https://mp.weixin.qq.com/s?__biz=MzU3NTgwOTE4NQ==&mid=2247484851&idx=1&sn=309991bacb1bf3b22669dca9d48fa270
date: 2025-05-14
tags:
  - Spring-Boot
  - 权限
  - RBAC
  - ABAC
  - JCasbin
  - Java
  - 安全
---

# SpringBoot 权限校验：从 RBAC 到 ABAC 基于 JCasbin

> 原文链接：[风象南](https://mp.weixin.qq.com/s?__biz=MzU3NTgwOTE4NQ==&mid=2247484851&idx=1&sn=309991bacb1bf3b22669dca9d48fa270)

---

## 一、前言：当权限判断写满业务代码

几乎所有企业系统都逃不过"权限"这道关。从"谁能看"、"谁能改"到"谁能审批"，权限逻辑贯穿业务方方面面。

起初使用最常见的 **RBAC（基于角色的访问控制）**：

```java
if (user.hasRole("admin")) {
    documentService.update(doc);
}
```

逻辑简单、上手快，但随着业务复杂度上升，RBAC 很快失控。比如：

- "文档的作者可以编辑自己的文档"
- "同部门的经理也可以编辑该文档"
- "外部合作方仅能查看共享文档"
- "项目归档后，所有人都只读"

这些场景无法用"角色"简单定义，权限判断开始蔓延在业务代码各处：

```java
if (user.getId().equals(doc.getOwnerId()) 
    || (user.getDept().equals(doc.getDept()) && user.isManager())) {
    documentService.update(doc);
} else {
    throw new AccessDeniedException("无权限");
}
```

权限逻辑与业务逻辑纠缠不清，可维护性、可测试性、可演化性统统崩盘。

---

## 二、RBAC 的天花板

RBAC 的问题在于：**它过于静态**。"角色"可以描述一类人，但描述不了上下文。

例如：研发经理能编辑本部门文档，但不能编辑市场部的。在 RBAC 下只能不断创建新角色，最终角色爆炸。

现实世界的权限往往与**属性**有关：
- 用户的部门
- 资源的拥有者
- 操作发生的时间/状态

这些动态因素是 RBAC 无法覆盖的，于是需要更灵活的模型——**ABAC**。

---

## 三、ABAC：基于属性的访问控制

**ABAC（Attribute-Based Access Control）** 核心理念：

> 授权决策 = 函数（主体属性、资源属性、操作属性、环境属性）

| 概念 | 含义 | 示例 |
|------|------|------|
| Subject（主体） | 谁在访问 | 用户A，部门=研发部 |
| Object（资源） | 访问什么 | 文档1，ownerId=A，部门=研发部 |
| Action（操作） | 做什么 | edit / read / delete |
| Policy（策略） | 允许条件 | user.dept == doc.dept && act == "edit" |

> 一句话：ABAC 不关心用户是谁，而关心"用户和资源具有什么属性"。

---

## 四、引入 JCasbin

[JCasbin](https://github.com/casbin/jcasbin) 是一个优秀的 Java 权限引擎，支持 RBAC、ABAC 等多种模型。

核心价值：**把授权逻辑从代码中抽离，让代码只负责执行业务。**

通过定义：
- **模型文件（model）**：规则框架
- **策略文件（policy）**：具体规则

由 Casbin 引擎执行判断。

---

## 五、核心实现

### 模型文件 model.conf

```
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub_rule, obj_rule, act

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = eval(p.sub_rule) && eval(p.obj_rule) && r.act == p.act
```

### 策略文件 policy.csv

```
p, r.sub.dept == r.obj.dept, true, edit
p, r.sub.id == r.obj.ownerId, true, edit
p, true, true, read
```

解释：
- 同部门可编辑
- 作者可编辑
- 所有人可阅读

### 代码调用

```java
Enforcer enforcer = new Enforcer("model.conf", "policy.csv");

User user = new User("u1", "研发部");
Document doc = new Document("d1", "研发部", "u1");

boolean canEdit = enforcer.enforce(user, doc, "edit");
// 输出：是否有编辑权限：true
```

无需任何 if-else，逻辑全在外部配置中定义。

---

## 六、Spring Boot 无感校验（注解 + AOP）

### 定义注解

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface CheckPermission {
    String action();
}
```

### 编写切面

```java
@Aspect
@Component
public class PermissionAspect {
    @Autowired
    private Enforcer enforcer;

    @Before("@annotation(checkPermission)")
    public void checkAuth(JoinPoint jp, CheckPermission checkPermission) {
        Object user = getCurrentUser();
        Object resource = getRequestResource(jp);
        String action = checkPermission.action();

        if (!enforcer.enforce(user, resource, action)) {
            throw new AccessDeniedException("无权限执行操作：" + action);
        }
    }
}
```

### 业务代码使用

```java
@CheckPermission(action = "edit")
@PostMapping("/doc/edit")
public void editDoc(@RequestBody Document doc) {
    documentService.update(doc);
}
```

> ✅ 授权逻辑彻底从业务中解耦，权限统一由 Casbin 引擎处理。

---

## 七、策略动态化与分布式支持

生产环境中权限策略存储在数据库中：

```java
JDBCAdapter adapter = new JDBCAdapter(dataSource);
Enforcer enforcer = new Enforcer("model.conf", adapter);
```

支持特性：
- **MySQL / PostgreSQL** 持久化
- **Redis Watcher** 实现多节点策略热更新
- **SyncedEnforcer** 支持高并发一致性

> 修改权限规则无需重新部署代码，即改即生效。

---

## 八、总结

| 优势 | 描述 |
|------|------|
| 逻辑解耦 | 授权逻辑完全从业务代码中剥离 |
| 灵活配置 | 权限规则动态可改、可热更新 |
| 可扩展 | 可根据属性定义复杂条件 |
| 统一决策 | 所有权限判断走同一引擎 |
| 可测试 | 策略可单测，无需跑整套业务流程 |

> 真正的架构能力，不是多写逻辑，而是让逻辑有边界。

**示例代码：** [springboot-permission](https://github.com/yuboon/java-examples/tree/master/springboot-permission)
