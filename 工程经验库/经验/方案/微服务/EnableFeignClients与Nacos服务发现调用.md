---
title: EnableFeignClients与Nacos服务发现调用
level: pattern
parent:
status: reviewed
tags:
  - feign
  - nacos
  - discovery
  - microservice
created_at: 2026-07-05
updated_at: 2026-07-05
confidence: high
related:
  - Feign请求头透传跨服务认证
  - 统一登录聚合服务（clientId 路由到领域服务）
  - Nacos配置缺口与运维注意
source_repo: macrozheng/mall-swarm
source_paths:
  - mall-auth/.../MallAuthApplication.java
  - mall-demo/.../MallDemoApplication.java
  - mall-admin/.../MallAdminApplication.java
  - mall-portal/.../MallPortalApplication.java
  - mall-auth/.../service/UmsAdminService.java
  - mall-auth/.../service/UmsMemberService.java
  - mall-demo/.../service/FeignAdminService.java
---

# EnableFeignClients 与 Nacos 服务发现调用

## 本项目落地状态

| 维度 | 状态 |
|------|------|
| 解析方式 | `@FeignClient("服务名")` → Nacos 按 `spring.application.name` 发现实例 |
| 调用路径 | **服务间直调**，`@FeignClient` **未配置** `url`，不经网关 |
| 主业务 Feign | ✅ `mall-auth` 2 个 Client 聚合登录 |
| 演示 Feign | ✅ `mall-demo` 3 个 Client + 全局 `RequestInterceptor` |
| 空启用 | ⚠️ `mall-admin`/`mall-portal` 有 `@EnableFeignClients` 但**模块内无** `@FeignClient` 接口 |

## 1. 各模块启动类注解（源码对照）

| 模块 | @EnableFeignClients | @EnableDiscoveryClient | pom 含 openfeign |
|------|---------------------|------------------------|------------------|
| mall-auth | ✅ | ✅ | ✅ |
| mall-demo | ✅ | ✅ | ✅ |
| mall-admin | ✅ | ✅ | ✅ |
| mall-portal | ✅ | ✅ | ✅ |
| mall-gateway | ❌ | ✅ | — |
| mall-search | ❌ | ✅ | — |
| mall-monitor | ❌ | ✅ | — |

## 2. 实际 @FeignClient 接口清单（全项目 5 个）

### mall-auth（主业务）

| 接口 | value | 映射 | 调用方 |
|------|-------|------|--------|
| `UmsAdminService` | `mall-admin` | `POST /admin/login` | `AuthController.login`（clientId=admin） |
| `UmsMemberService` | `mall-portal` | `POST /sso/login` | `AuthController.login`（clientId=portal） |

`mall-auth` **无** `RequestInterceptor`，登录接口**不需要**透传 Authorization。

### mall-demo（演示）

| 接口 | value | 方法 |
|------|-------|------|
| `FeignAdminService` | `mall-admin` | `POST /admin/login`、`GET /brand/listAll` |
| `FeignPortalService` | `mall-portal` | `POST /sso/login`、`GET /cart/list` |
| `FeignSearchService` | `mall-search` | `GET /esProduct/search/simple` |

Demo 入口前缀：`/feign/admin`、`/feign/portal`、`/feign/search`（经网关为 `/mall-demo/feign/...`）。

## 3. 与 Nacos 的配合

各 Feign 调用方 `application-dev.yml` 均配置：

```yaml
spring.cloud.nacos.discovery.server-addr: localhost:8848
```

`@FeignClient` 的 `value` 与目标服务 `spring.application.name` 一致：

| value | application.name |
|-------|------------------|
| mall-admin | mall-admin |
| mall-portal | mall-portal |
| mall-search | mall-search |

## 4. 与网关的关系

- 浏览器/App → **网关** → 业务服务：走 `SaTokenConfig` 鉴权
- **Feign 服务间调用**：OpenFeign + Nacos 直连目标实例，**不经过** `mall-gateway`
- 路径示例：网关外部 `POST /mall-admin/admin/login`（StripPrefix=1 后为 admin 内 `/admin/login`）；auth Feign 同样映射 `POST /admin/login` 到 admin 实例——**落点一致，但不经网关**

## 5. RequestInterceptor 范围

| 模块 | Interceptor |
|------|-------------|
| mall-demo | `FeignConfig` 注册 `FeignRequestInterceptor`（全局 Bean，作用于本模块全部 Feign Client） |
| mall-auth | **无** |
| mall-admin / mall-portal | **无**（且无 Feign Client 定义） |

详见 [Feign请求头透传跨服务认证](Feign请求头透传跨服务认证.md)。

## 6. 避免误读

| 说法 | 是否符合代码 |
|------|--------------|
| 「全项目 Feign 都透传 Header」 | ❌ 仅 mall-demo |
| 「admin/portal 有 Feign 远程调用」 | ❌ 有注解开关，**无** Client 接口 |
| 「Feign 经网关转发」 | ❌ 无 `url` 配置，走 Nacos 直调 |
| 「search 模块参与 Feign 调用」 | ⚠️ 仅作为 **被调方**（demo 调它），自身无 `@EnableFeignClients` |

## 7. 最小验证清单

- Nacos 注册列表含 mall-admin、mall-portal、mall-auth
- `POST /mall-auth/auth/login?clientId=admin-app&...` 触发 auth → admin Feign
- demo `GET /mall-demo/feign/admin/getBrandList` 带 Authorization 时下游可收到同名 Header

## 附录：来源证据

- 启动类：`MallAuthApplication`、`MallDemoApplication`、`MallAdminApplication`、`MallPortalApplication`、`MallGatewayApplication`、`MallSearchApplication`
- Feign 接口：`UmsAdminService`、`UmsMemberService`、`FeignAdminService`、`FeignPortalService`、`FeignSearchService`
- `AuthController.java` L40-46
- `pom.xml` openfeign 依赖：admin/auth/portal/demo
- 深读：[T7-mall-demo深读](../../mall-swarm/深读/T7-mall-demo深读.md)
