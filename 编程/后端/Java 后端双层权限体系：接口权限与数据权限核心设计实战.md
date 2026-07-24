---
title: Java 后端双层权限体系：接口权限与数据权限核心设计实战
author: 程序员进阶笔记
source: https://mp.weixin.qq.com/s/pWCZJIYFvnW1y-w-w1sazA
date: 2025-07-01
tags:
  - Spring-Security
  - MyBatis-Plus
  - 权限设计
  - 数据权限
  - RBAC
  - 后端
---

# Java 后端双层权限体系：接口权限与数据权限核心设计实战

> 原文链接：[程序员进阶笔记](https://mp.weixin.qq.com/s/pWCZJIYFvnW1y-w-w1sazA)

---

## 一、两层权限分层体系

| 维度 | 功能权限（接口权限） | 数据权限 |
|------|-------------------|---------|
| 管控对象 | 接口、按钮、菜单 | 数据库行数据 |
| 解决问题 | "能否访问功能" | "能访问哪些数据" |
| 校验时机 | 进入业务逻辑之前 | SQL 执行查询之前 |
| 实现技术 | SpringSecurity `@PreAuthorize` | MyBatis-Plus 拦截器动态改写 SQL |
| 安全意义 | 拦截非法入口 | 隔离同接口下不同用户数据 |

**依赖关系**：功能权限是第一道防线（无权限直接 403），拥有功能权限才会执行 SQL，再由数据权限过滤行数据。二者缺一不可。

---

## 二、RBAC 权限模型

### 链路

```
用户 → 角色 → 权限
```

用户不直接绑定权限，通过角色做中转，方便批量分配和维护。

### 5 种数据范围

| 级别 | 数据范围 | 编码 |
|------|---------|------|
| 1 | 全部数据（管理员最高权限） | 最小 |
| 2 | 本部门及所有下级部门 | ↓ |
| 3 | 仅直属本部门 | ↓ |
| 4 | 仅本人创建数据 | ↓ |
| 5 | 自定义指定部门 | 最大 |

### 多角色合并规则

- **功能权限**：取并集，任意角色拥有即可访问
- **数据权限**：取编码最小值，编码越小权限越高，权限向上兼容

---

## 三、完整请求执行全链路

```
前端请求携带 Token
    ↓
TokenAuthFilter 拦截 → 解析 Token → 封装 LoginUser → 存入 ThreadLocal
    ↓
SpringSecurity @PreAuthorize 校验功能权限 → 无权限返回 403
    ↓
Controller → Service 执行业务（不处理权限逻辑）
    ↓
Mapper 查询时触发 DataScopeInnerInterceptor → 自动拼接过滤条件改写 SQL
    ↓
数据库返回过滤后数据
    ↓
请求结束 → finally 强制 clear() ThreadLocal（避免线程池权限串号）
```

---

## 四、核心组件

### 1. TokenAuthFilter

```java
@Component
public class TokenAuthFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req,
            HttpServletResponse resp, FilterChain chain) {
        try {
            String token = req.getHeader("Authorization");
            LoginUser user = tokenService.parseToken(token);
            SecurityContextHolder.set(user);
            chain.doFilter(req, resp);
        } finally {
            SecurityContextHolder.clear();
        }
    }
}
```

### 2. SecurityContextHolder

```java
public class SecurityContextHolder {
    private static final ThreadLocal<LoginUser> THREAD_LOCAL = new ThreadLocal<>();

    public static void set(LoginUser user) { THREAD_LOCAL.set(user); }
    public static LoginUser get() { return THREAD_LOCAL.get(); }
    public static void clear() { THREAD_LOCAL.remove(); }
}
```

### 3. SpringSecurity 方法鉴权

```java
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class SecurityConfig {}
```

### 4. DataScopeInnerInterceptor

MyBatis-Plus 自定义 SQL 拦截器，仅处理查询语句，自动拼接数据过滤条件，同步处理分页 count。

### 5. @IgnoreDataScope

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface IgnoreDataScope {}

// 导出、报表接口跳过数据过滤，查询全量数据
@GetMapping("/export")
@PreAuthorize("hasPerm('biz:order:export')")
@IgnoreDataScope
public void exportExcel() {}
```

---

## 五、SpringSecurity 原生表达式全集

### 身份校验

| 表达式 | 说明 |
|--------|------|
| `isAuthenticated()` | 登录成功后为 true，所有登录用户可访问 |
| `isAnonymous()` | 仅未登录游客可访问 |
| `isFullyAuthenticated()` | 仅手动账号密码登录通过，记住密码自动登录不通过（高危敏感操作） |
| `rememberMe()` | 通过记住密码 Cookie 自动登录时返回 true |

### 角色校验（自动携带 ROLE_ 前缀）

```java
@PreAuthorize("hasRole('admin')")
@PreAuthorize("hasAnyRole('admin','manager')")
```

### 登录主体取值

```java
@PreAuthorize("authentication.principal.userId == 10001")
```

### 逻辑运算符

```java
// 管理员或拥有导出权限
@PreAuthorize("hasRole('admin') || hasPerm('biz:order:export')")
```

> `hasPerm`、`hasAnyPerm` 不属于 SpringSecurity 原生表达式，为项目自定义扩展实现。

---

## 六、实战示例

### 功能权限完整实战

```java
@RestController
@RequestMapping("/biz/order")
public class OrderController {

    // 任意登录用户可访问
    @GetMapping("/all")
    @PreAuthorize("isAuthenticated()")
    public List<Order> allOrder() { return orderService.list(); }

    // 必须具备订单列表权限
    @GetMapping("/list")
    @PreAuthorize("hasPerm('biz:order:list')")
    public Page<OrderVO> list(OrderQuery query) { return orderService.pageList(query); }

    // 仅 admin 角色可删除
    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('admin')")
    public void remove(@PathVariable Long id) { orderService.removeById(id); }

    // 同时拥有列表、编辑权限
    @PutMapping("/batch")
    @PreAuthorize("hasPerm('biz:order:list') && hasPerm('biz:order:edit')")
    public void batchEdit(@RequestBody List<Order> orders) { orderService.batchUpdate(orders); }

    // 管理员或拥有导出权限，跳过数据权限
    @GetMapping("/export")
    @PreAuthorize("hasRole('admin') || hasPerm('biz:order:export')")
    @IgnoreDataScope
    public void export(OrderQuery query) { orderService.exportExcel(query); }
}
```

### 自定义 hasPerm 底层实现

```java
@Component
public class CustomMethodExpressionHandler
        extends DefaultMethodSecurityExpressionHandler {
    @Override
    protected MethodSecurityExpressionOperations createSecurityExpressionRoot(
            Authentication authentication, MethodInvocation invocation) {
        CustomExpressionRoot root = new CustomExpressionRoot(authentication);
        root.setPermissionEvaluator(getPermissionEvaluator());
        return root;
    }
}

public class CustomExpressionRoot extends MethodSecurityExpressionRoot {
    public boolean hasPerm(String perm) {
        LoginUser loginUser = SecurityContextHolder.get();
        return loginUser != null && loginUser.getPermSet().contains(perm);
    }

    public boolean hasAnyPerm(String... perms) {
        LoginUser loginUser = SecurityContextHolder.get();
        if (loginUser == null) return false;
        for (String p : perms) {
            if (loginUser.getPermSet().contains(p)) return true;
        }
        return false;
    }
}
```

### 登录预加载 scopeUserIds

```java
@Service
public class LoginServiceImpl implements LoginService {
    @Override
    public String login(String username, String password) {
        SysUser user = userMapper.selectByUsername(username);
        List<SysRole> roleList = roleMapper.selectRoleByUserId(user.getId());

        // 合并功能权限（取并集）
        Set<String> permSet = new HashSet<>();
        for (SysRole role : roleList) {
            permSet.addAll(permMapper.selectPermByRoleId(role.getId()));
        }

        // 合并数据范围（取最小值，编码越小权限越高）
        Integer maxDataScope = roleList.stream()
                .map(SysRole::getDataScope)
                .min(Integer::compareTo).get();

        // 查询可见用户 ID 集合
        Set<Long> scopeUserIds = switch (DataScopeEnum.values()[maxDataScope]) {
            case ALL -> Collections.emptySet();
            case SELF -> Set.of(user.getId());
            case DEPT_SELF -> userMapper.selectUserIdByDeptId(user.getDeptId());
            case DEPT_AND_CHILD -> {
                Set<Long> childDept = deptMapper.selectAllChildDeptId(user.getDeptId());
                yield userMapper.selectUserByDeptIds(childDept);
            }
            case CUSTOM -> {
                Set<Long> customDept = roleMapper.selectCustomDeptByRoleList(roleList);
                yield userMapper.selectUserByDeptIds(customDept);
            }
        };

        // 封装上下文
        LoginUser loginUser = new LoginUser();
        loginUser.setUserId(user.getId());
        loginUser.setDeptId(user.getDeptId());
        loginUser.setPermSet(permSet);
        loginUser.setDataScope(maxDataScope);
        loginUser.setScopeUserIds(scopeUserIds);
        SecurityContextHolder.set(loginUser);

        return tokenService.createToken(loginUser);
    }
}
```

### SQL 自动改写效果

```sql
-- 原始 SQL
SELECT id, order_no, amount FROM biz_order WHERE amount > 100

-- 普通销售（仅本人）
AND create_user_id IN (10001)

-- 部门主管（本部门及下级）
AND create_user_id IN (10001, 10002, 10003)

-- 管理员（全部数据）
-- 无额外过滤条件

-- @IgnoreDataScope 导出接口
-- 不拼接过滤，查询全表
```

### 分页兼容性

**注册顺序决定正确性**：分页插件必须优先注册，数据权限拦截器后置。这样 count SQL 和列表 SQL 都会被拦截器同步改写，分页总数与列表数据完全对应。

### 异步任务权限兼容

```java
@Service
public class OrderAsyncService {
    @Async
    public void asyncExport(LoginUser loginUser, OrderQuery query) {
        try {
            SecurityContextHolder.set(loginUser);  // 子线程手动 set
            List<Order> list = orderMapper.selectList(query);
            // 导出逻辑...
        } finally {
            SecurityContextHolder.clear();         // 用完清空
        }
    }
}

// 调用方
@GetMapping("/async/export")
@PreAuthorize("hasPerm('biz:order:export')")
public Result<Void> asyncExport(OrderQuery query) {
    LoginUser loginUser = SecurityContextHolder.get();
    orderAsyncService.asyncExport(loginUser, query);
    return Result.success();
}
```

### 存量无 user_id 业务表兼容

```sql
CREATE TABLE biz_data_share (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    biz_type VARCHAR(32) NOT NULL COMMENT '业务类型',
    biz_id BIGINT NOT NULL COMMENT '单据主键',
    visible_user_id BIGINT COMMENT '可查看用户',
    visible_dept_id BIGINT COMMENT '可查看部门',
    UNIQUE KEY uk_biz_type_id (biz_type, biz_id, visible_user_id)
);
```

拦截器识别旧表自动 JOIN 中间表过滤，无需修改原有业务表任何字段。

---

## 七、数据权限两种业务模型

### 创建人归属模型（推荐）

业务表仅需 `create_user_id`，部门关系动态查询。**员工调岗不会修改历史单据归属，满足审计要求。**

> 规范：项目初始化统一为所有业务表增设 `create_user_id`、`create_time`、`update_user_id`、`update_time` 四个基础字段。

### 单据独立归属模型

业务表同时存在 `create_user_id` 和 `belong_dept_id`，单据归属部门和创建人无关，适用于财务数据、公共跨部门项目。

---

## 八、生产避坑清单

### 数据权限相关（8 坑）

| 坑 | 现象 | 根因 | 解决 |
|----|------|------|------|
| ThreadLocal 不清除 | 不同用户数据串号 | 线程池复用，上一请求残留 | finally 执行 clear() |
| 分页插件注册顺序颠倒 | 分页 total 是全表数据 | count SQL 没被拦截器改写 | 分页插件优先注册 |
| 依赖前端传 userId 过滤 | 修改参数可查全平台数据 | 前端参数可篡改 | 全部读取 SecurityContextHolder |
| 员工调岗同步更新单据部门 | 历史单据归属变更，统计失真 | dept_id 静态绑定 | 仅保留 create_user_id |
| 大部门 IN 超长 SQL | 接口 500 | 超出 max_allowed_packet | 超阈值部门分配 ALL 权限 |
| 历史数据 create_user_id 为 null | 旧单据不可见 | NULL 无法匹配任何用户 ID | 回填或赋值 0 |
| UNION/CTE 复杂 SQL 解析失败 | 报表接口 500 | JSqlParser 兼容性有限 | 加 @IgnoreDataScope |
| 联查无别名导致字段歧义 | 联查提示字段模糊 | 无别名拼接引发冲突 | 规范添加表别名 |

### 功能权限相关（3 坑）

| 坑 | 解决 |
|----|------|
| 未开启 `@EnableGlobalMethodSecurity` → 注解全部失效 | SecurityConfig 添加 `prePostEnabled = true` |
| `@Async` 丢失登录上下文 → 导出返回全量数据 | 手动传入 LoginUser，用完 clear |
| 登录/验证码接口未放行 → 无法登录 | HttpSecurity 放行 `/login`、`/captcha` |

### 架构层面（3 坑）

| 坑 | 解决 |
|----|------|
| 多角色 dataScope 取 max 而非 min → 管理员权限降级 | 统一使用 `stream.min()` |
| 业务手写过滤 + 全局拦截器两套并行 | 禁用业务手写过滤，统一拦截器 |
| Token 刷新未重算 scopeUserIds → 权限变更滞后 | 刷新 Token 时重新查询角色 |

---

## 九、核心设计原则总结

1. **双层防护**：功能权限管控接口访问入口，数据权限隔离行级业务数据，缺一不可
2. **RBAC 模型**：用户 → 角色 → 权限，多角色合并遵循权限最大化
3. **建表规范**：统一预留创建人字段，低成本实现数据权限和审计
4. **无侵入过滤**：MyBatis-Plus 拦截器 + 登录预加载 scopeUserIds，业务零侵入
5. **SpringSecurity 注解**：原生身份/角色表达式 + 自定义 hasPerm 精准控制
6. **ThreadLocal 生命周期**：请求结束必须清理，防止线程池复用串权

> 相关笔记：[[SpringBoot 权限校验：从 RBAC 到 ABAC 基于 JCasbin]]
