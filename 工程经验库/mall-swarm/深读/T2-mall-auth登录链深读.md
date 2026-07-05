# T2：mall-auth 统一认证 + 登录链深读

> **场景**：模板场景二（模块深读）
> **生成时间**：2026-07-05
> **范围**：Feign 调用链、token 签发、双 clientId 分支

---

## 1. 模块背景

`mall-auth`（端口 8401）提供统一登录入口，按 `clientId` 分流到 mall-admin / mall-portal 完成密码校验与 Sa-Token 签发。认证中心本身无数据库。

## 2. 核心职责

| mall-auth | mall-admin / mall-portal |
|-----------|--------------------------|
| 接收 `clientId+username+password` | 查库、BCrypt 校验、状态检查 |
| Feign 转发 | `StpUtil` / `StpMemberUtil.login` 签发 token |
| clientId 路由 | Session 写入 `UserDto`（含权限/clientId） |

## 3. 核心对象

| 对象 | 路径 | 作用 |
|------|------|------|
| `AuthController` | `mall-auth/.../controller/AuthController.java` | `POST /auth/login`，clientId 分支 |
| `UmsAdminService`（Feign） | `mall-auth/.../service/UmsAdminService.java` | `@FeignClient("mall-admin")` → `POST /admin/login` |
| `UmsMemberService`（Feign） | `mall-auth/.../service/UmsMemberService.java` | `@FeignClient("mall-portal")` → `POST /sso/login` |
| `UmsAdminController` | `mall-admin/.../controller/UmsAdminController.java` | 后台 REST 登录 |
| `UmsAdminServiceImpl` | `mall-admin/.../service/impl/UmsAdminServiceImpl.java` | 后台登录核心 |
| `UmsMemberController` | `mall-portal/.../controller/UmsMemberController.java` | 前台 REST 登录 |
| `UmsMemberServiceImpl` | `mall-portal/.../service/impl/UmsMemberServiceImpl.java` | 前台登录核心 |
| `SaTokenConfigure` | `mall-admin/.../config/SaTokenConfigure.java` | 注册默认 `StpLogicJwtForSimple()` |
| `AuthConstant` | `mall-common/.../constant/AuthConstant.java` | clientId、Session key 常量 |

**clientId 常量**（`AuthConstant.java` 第 22–27 行）：

- `ADMIN_CLIENT_ID = "admin-app"`
- `PORTAL_CLIENT_ID = "portal-app"`

## 4. 内部流程

### 路径 A：经 mall-auth 统一登录

```
POST /mall-auth/auth/login?clientId&username&password
  → 网关白名单 /mall-auth/** 跳过鉴权
  → StripPrefix=1 → mall-auth POST /auth/login
  → AuthController.login 分支 clientId
```

`AuthController` 第 37–49 行：

- `admin-app` → Feign `mall-admin POST /admin/login`（JSON body）
- `portal-app` → Feign `mall-portal POST /sso/login`（query 参数）
- 其他 → `CommonResult.failed("clientId不正确")`

### 路径 B：Admin 直连（白名单 `/mall-admin/admin/login`）

`UmsAdminServiceImpl.login` 8 步（第 89–118 行）：

1. 非空校验 → `Asserts.fail`
2. `getAdminByUsername`
3. `BCrypt.checkpw`
4. `status==1`
5. `StpUtil.login(admin.getId())`
6. 组装 `UserDto`：`clientId=admin-app`，`permissionList` = 资源 `id:name` 列表
7. `StpUtil.getSession().set(STP_ADMIN_INFO, userDto)`
8. `insertLoginLog` + 返回 `SaTokenInfo`

Controller 包装（`UmsAdminController` 第 57–65 行）：`{token, tokenHead}`，`tokenHead` = `Bearer `。

### 路径 C：Portal 直连 / Feign

`UmsMemberServiceImpl.login` 7 步（第 148–171 行）：

1–4. 同 admin（非空、查用户、BCrypt、status）
5. `StpMemberUtil.login(member.getId())`
6. `UserDto`：`clientId=portal-app`，**无 permissionList**
7. Session `STP_MEMBER_INFO` + 返回 `StpMemberUtil.getTokenInfo()`

### Sa-Token 双体系

| 体系 | 实现 | 网关校验 |
|------|------|----------|
| Admin | `SaTokenConfigure` → `StpLogicJwtForSimple()` | `StpUtil.checkLogin` |
| Portal | `StpMemberUtil`，`TYPE=memberLogin` | `StpMemberUtil.checkLogin` |

## 5. 对外接口

| 入口 | 方法 | 参数 | 返回 |
|------|------|------|------|
| 统一登录 | `POST /mall-auth/auth/login` | `clientId`, `username`, `password`（query） | 下游 `CommonResult` 透传 |
| Admin 登录 | `POST /mall-admin/admin/login` | JSON `UmsAdminLoginParam` | `{token, tokenHead}` |
| Portal 登录 | `POST /mall-portal/sso/login` | `username`, `password`（query） | `{token, tokenHead}` |

## 6. 扩展点

- 新增客户端类型：在 `AuthConstant` 加 clientId，`AuthController` 加分支，新建 Feign 接口
- 登录后权限：admin 在 `getResourceList(adminId)` 加载，portal 无权限列表

## 7. 错误处理

| 层级 | 机制 | 典型结果 |
|------|------|----------|
| Service 校验 | `Asserts.fail(msg)` → `ApiException` | `GlobalExceptionHandler` → `CommonResult.failed(msg)` code=500 |
| 非法 clientId | `AuthController` | `failed("clientId不正确")` |
| 网关后续 | token 无效 | 401；权限不足 403 |

Service 层失败均抛异常，非返回 null。Feign 无自定义 Fallback。

## 8. 设计优点

- clientId 单点路由，Feign 解耦认证中心与用户服务
- 权限在 admin 登录时一次性加载进 Session，网关 O(1) 读取
- 双 StpLogic 隔离前后台 token

## 9. 设计代价

- 三条登录路径并存（auth 聚合 + admin/portal 直连），需保持白名单同步
- `UmsAdminController.login` 中 `saTokenInfo==null` 分支与 Service 全抛异常可能冗余

## 10. 可提炼候选

| # | 候选 | 层级 |
|---|------|------|
| 1 | 认证中心 + clientId 路由，Feign 调用领域用户服务 | 架构层 |
| 2 | 同一 Sa-Token 框架下双 loginType 隔离前后台 JWT | 架构层 |
| 3 | 登录后将 UserDto 写入 Token Session，供网关/StpInterface 消费 | 方案层 |
| 4 | 权限码格式 resourceId:resourceName 与 Redis 路径规则同源 | 方案层 |
| 5 | 统一返回 {token, tokenHead} 便于前端封装 Authorization 头 | 方案层 |

## 11. 已读证据

- `mall-auth/src/main/java/com/macro/mall/auth/controller/AuthController.java`
- `mall-auth/src/main/java/com/macro/mall/auth/service/UmsAdminService.java`
- `mall-auth/src/main/java/com/macro/mall/auth/service/UmsMemberService.java`
- `mall-admin/src/main/java/com/macro/mall/controller/UmsAdminController.java`
- `mall-admin/src/main/java/com/macro/mall/service/impl/UmsAdminServiceImpl.java`
- `mall-portal/src/main/java/com/macro/mall/portal/controller/UmsMemberController.java`
- `mall-portal/src/main/java/com/macro/mall/portal/service/impl/UmsMemberServiceImpl.java`
- `mall-admin/src/main/java/com/macro/mall/config/SaTokenConfigure.java`
- `mall-common/src/main/java/com/macro/mall/common/constant/AuthConstant.java`

## 12. 待深读问题

1. Feign 调用链上 `ApiException` 如何跨服务传播为 `CommonResult`
2. 网关 JWT 密钥与 admin/portal 对齐机制
