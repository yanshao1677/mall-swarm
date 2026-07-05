# T1：mall-gateway 模块深读

> **场景**：模板场景二（模块深读）
> **生成时间**：2026-07-05
> **范围**：白名单、SaReactorFilter 鉴权链、Redis 权限 OR 匹配、异常返回

---

## 1. 模块背景

`mall-gateway`（端口 8201）是统一 API 入口，基于 Spring Cloud Gateway + Sa-Token Reactor 过滤器，负责路由转发、CORS、双账号体系登录校验、后台路径权限 OR 校验。

## 2. 核心职责

| 负责 | 不负责 |
|------|--------|
| 路由到 auth/admin/portal/search/demo | 密码校验、token 签发 |
| 白名单放行、OPTIONS 预检 | 权限规则维护（由 mall-admin 写入 Redis） |
| `/mall-portal/**` 会员登录校验 | portal 细粒度权限 |
| `/mall-admin/**` 管理员登录 + 权限 OR 校验 | 业务逻辑 |

## 3. 核心对象

| 对象 | 路径 | 作用 |
|------|------|------|
| `MallGatewayApplication` | `mall-gateway/.../MallGatewayApplication.java` | 启动入口，`@EnableDiscoveryClient` |
| `SaTokenConfig` | `mall-gateway/.../config/SaTokenConfig.java` | 注册 `SaReactorFilter`，编排鉴权链 |
| `IgnoreUrlsConfig` | `mall-gateway/.../config/IgnoreUrlsConfig.java` | 绑定 `secure.ignore.urls` 白名单 |
| `StpMemberUtil` | `mall-gateway/.../util/StpMemberUtil.java` | 前台账号体系，`TYPE="memberLogin"` |
| `StpInterfaceImpl` | `mall-gateway/.../component/StpInterfaceImpl.java` | 为 `StpUtil.checkPermission*` 提供 Session 权限列表 |
| `RedisConfig` | `mall-gateway/.../config/RedisConfig.java` | 继承 `BaseRedisConfig`，提供 `RedisTemplate` |
| `GlobalCorsConfig` | `mall-gateway/.../config/GlobalCorsConfig.java` | `CorsWebFilter` 全局 CORS |

**StpMemberUtil 双体系关键**（`StpMemberUtil.java` 第 44–49 行）：

```java
public static final String TYPE = "memberLogin";
public static StpLogic stpLogic = new StpLogicJwtForSimple(TYPE);
```

## 4. 内部流程（单次请求）

1. **SaReactorFilter 拦截** `/**`，白名单路径由 `IgnoreUrlsConfig.getUrls()` 排除（`SaTokenConfig` 第 43–45 行）
2. **OPTIONS 预检**：`SaRouter.match(SaHttpMethod.OPTIONS).stop()` 直接放行（第 49 行）
3. **Portal 分支**：`/mall-portal/**` → `StpMemberUtil.checkLogin()` → `.stop()`，**不再执行权限块**（第 51 行）
4. **Admin 分支**：`/mall-admin/**` → `StpUtil.checkLogin()`，**不 stop**（第 53 行）
5. **权限块**（对所有未 stop 的请求执行）：
   - `redisTemplate.opsForHash().entries("auth:pathResourceMap")` 取全量规则
   - `AntPathMatcher` 将 `SaHolder.getRequest().getRequestPath()` 与每条 pattern 匹配
   - 收集 `needPermissionList`（值为 `id:name`）
   - 非空则 `StpUtil.checkPermissionOr(...)`（任一权限即可，第 71–72 行）
6. **路由转发**（过滤器之外）：`application.yml` 5 条路由，`lb://` + `StripPrefix=1`

## 5. 对外接口

网关无 REST Controller，对外能力为配置驱动：

| 能力 | 配置位置 |
|------|----------|
| 路由 | `application.yml` 第 19–49 行 |
| 白名单 | `application.yml` 第 56–79 行 → `secure.ignore.urls` |
| Sa-Token | `application.yml` 第 104–126 行：`token-name=Authorization`，`token-prefix=Bearer`，`timeout=2592000` |
| API 文档聚合 | `knife4j.gateway` 服务发现聚合，排除 mall-monitor |

**路由表**

| id | 网关 Path | 下游 |
|----|-----------|------|
| mall-auth | `/mall-auth/**` | `lb://mall-auth` |
| mall-admin | `/mall-admin/**` | `lb://mall-admin` |
| mall-portal | `/mall-portal/**` | `lb://mall-portal` |
| mall-search | `/mall-search/**` | `lb://mall-search` |
| mall-demo | `/mall-demo/**` | `lb://mall-demo` |

## 6. 扩展点

- 新增微服务：在 `application.yml` routes 追加条目 + 白名单/鉴权规则
- 新增白名单路径：修改 `secure.ignore.urls`
- 新增前台公开接口：白名单加 `/mall-portal/xxx/**`

## 7. 错误处理

| 异常 | 响应 | 证据 |
|------|------|------|
| `NotLoginException` | `CommonResult.unauthorized()`，code=401 | `SaTokenConfig` 第 89–90 行 |
| `NotPermissionException` | `CommonResult.forbidden()`，code=403 | 第 91–92 行 |
| 其他 | `CommonResult.failed(e.getMessage())`，code=500 | 第 93–94 行 |

响应强制 JSON + CORS 头（第 84–87 行）。无重试/降级。

## 8. 设计优点（有证据）

- Portal 仅 `checkLogin` + stop，Admin 额外 Redis 路径权限 OR，职责分层清晰
- 鉴权异常统一映射 `CommonResult`，前端可统一处理 401/403

## 9. 设计代价

- 网关 `application.yml` **无** `jwt-secret-key`，admin/portal 均为 `sa-secret-key123`——JWT 密钥一致性待运行时确认
- 权限块对所有未 stop 请求执行，含非 `/mall-admin/**` 路径（若 Redis 有匹配规则）
- `StpInterfaceImpl` 对 member 返回 `null`，与 portal 无网关权限校验一致

## 10. 可提炼候选

| # | 候选 | 层级 |
|---|------|------|
| 1 | 网关集中鉴权 + 双 StpLogic 隔离前后台 token | 架构层 |
| 2 | 白名单外置 YAML + `@ConfigurationProperties` | 方案层 |
| 3 | Redis Hash 存「网关可见路径 → 权限码」，Ant 模式 OR 匹配 | 方案层 |
| 4 | Portal 仅 checkLogin + stop，Admin 再 checkPermissionOr | 方案层 |
| 5 | 鉴权异常统一映射 CommonResult 401/403 | 方案层 |

## 11. 已读证据

- `mall-gateway/src/main/java/com/macro/mall/config/SaTokenConfig.java`
- `mall-gateway/src/main/java/com/macro/mall/config/IgnoreUrlsConfig.java`
- `mall-gateway/src/main/java/com/macro/mall/util/StpMemberUtil.java`
- `mall-gateway/src/main/java/com/macro/mall/component/StpInterfaceImpl.java`
- `mall-gateway/src/main/java/com/macro/mall/config/RedisConfig.java`
- `mall-gateway/src/main/java/com/macro/mall/config/GlobalCorsConfig.java`
- `mall-gateway/src/main/resources/application.yml`
- `mall-admin/src/main/java/com/macro/mall/service/impl/UmsResourceServiceImpl.java`（initPathResourceMap，权限写入来源）

## 12. 待深读问题

1. 网关缺少 `jwt-secret-key` 时 JWT 校验是否仍能通过
2. `requestPath` 是否含 `/mall-admin` 前缀（影响 Ant 匹配）
