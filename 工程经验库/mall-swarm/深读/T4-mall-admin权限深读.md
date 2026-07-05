# T4：mall-admin 权限体系深读

> **场景**：模板场景二（模块深读）
> **生成时间**：2026-07-05
> **范围**：PathResourceRulesHolder、initPathResourceMap、菜单/资源/角色关系、网关 Redis 消费

---

## 1. 模块背景

mall-admin 负责后台 RBAC：用户-角色-资源-菜单。权限规则在启动时写入 Redis `auth:pathResourceMap`，供 mall-gateway 做路径级权限 OR 校验。

## 2. 核心职责

| 负责 | 不负责 |
|------|--------|
| 维护 ums_resource 路径规则 | 网关鉴权执行（gateway 读 Redis） |
| 启动时预热 Redis 路径-权限映射 | portal 会员权限（portal 无细粒度权限） |
| 登录时加载用户 permissionList 到 Session | token 校验（gateway 做） |

## 3. 核心对象

| 对象 | 路径 | 作用 |
|------|------|------|
| `PathResourceRulesHolder` | `mall-admin/.../component/PathResourceRulesHolder.java` | `@PostConstruct` 触发初始化 |
| `UmsResourceServiceImpl` | `mall-admin/.../service/impl/UmsResourceServiceImpl.java` | `initPathResourceMap` 核心逻辑 |
| `UmsAdminServiceImpl` | `mall-admin/.../service/impl/UmsAdminServiceImpl.java` | 登录时 `getResourceList` |
| `UmsResourceController` | `mall-admin/.../controller/UmsResourceController.java` | 手动刷新 API |
| `UmsAdminRoleRelationDao` | `mall-admin/.../dao/UmsAdminRoleRelationDao.xml` | 用户-角色-资源联表 SQL |
| `SaTokenConfig`（gateway） | `mall-gateway/.../config/SaTokenConfig.java` | 消费 Redis 规则 |

## 4. 内部流程

### 4.1 启动加载路径-资源映射

```
mall-admin 启动
  → PathResourceRulesHolder.@PostConstruct initPathResourceMap
  → UmsResourceServiceImpl.initPathResourceMap
```

**initPathResourceMap 逻辑**（第 79–87 行）：

1. 查全部 `ums_resource`
2. 构建 Map：`"/"+applicationName+resource.getUrl"` → `"id:name"`
   - 例：`/mall-admin/brand/**` → `1:商品品牌管理`
3. `redisService.del("auth:pathResourceMap")`
4. `redisService.hSetAll(PATH_RESOURCE_MAP, map)`

常量：`AuthConstant.PATH_RESOURCE_MAP = "auth:pathResourceMap"`。

### 4.2 增量刷新触发点

| 触发 | 方法 | 行号 |
|------|------|------|
| 资源 create/update/delete | `UmsResourceServiceImpl` | L32/L40/L52 |
| 角色分配资源 | `UmsRoleServiceImpl.allocResource` | L116 |
| 角色删除 | `UmsRoleServiceImpl.delete` | L53 |
| 手动 API | `GET /resource/initPathResourceMap` | `UmsResourceController` L95–99 |

**未确认**：`UmsRoleServiceImpl.update` 改角色名不会刷新 pathResourceMap。

### 4.3 登录时权限加载

```
POST /admin/login
  → UmsAdminServiceImpl.login
  → getResourceList(adminId)  // SQL 联表
  → permissionList = resource.id + ":" + resource.name
  → StpUtil.getSession().set("adminInfo", UserDto{permissionList})
```

**SQL 链路**（`UmsAdminRoleRelationDao.xml`）：

```
ums_admin_role_relation → ums_role → ums_role_resource_relation → ums_resource
WHERE admin_id = #{adminId} GROUP BY ur.id
```

权限码格式 `"资源ID:资源名称"` 与 Redis pathResourceMap 的 value **一致**。

### 4.4 网关消费（与 T1 衔接）

```
请求 GET /mall-admin/order/list
  → StpUtil.checkLogin()
  → Redis HGETALL auth:pathResourceMap
  → AntPathMatcher 匹配 requestPath
  → needPermissionList 非空 → StpUtil.checkPermissionOr
  → StpInterfaceImpl 从 Session adminInfo.permissionList 取权限
```

`StpInterfaceImpl` 第 21–30 行：从 Session 读 `adminInfo.permissionList`。

## 5. 对外接口

| 接口 | 作用 |
|------|------|
| `GET /resource/initPathResourceMap` | 手动刷新 Redis 路径规则 |
| 资源 CRUD | 自动触发 initPathResourceMap |
| `POST /admin/login` | 登录并加载 permissionList 到 Session |

## 6. 扩展点

- 新增后台 API：在 `ums_resource` 表加 url 规则（Ant 模式），刷新映射
- 新增角色权限：通过 `allocResource` 分配资源，用户登录时自动加载

## 7. 错误处理

- 登录失败：`Asserts.fail` → `ApiException` → `CommonResult.failed`
- 网关权限不足：`NotPermissionException` → 403

## 8. 设计优点

- 路径规则与 Session 权限码格式统一（`id:name`），网关 OR 匹配简单
- 资源变更自动刷新 Redis，无需重启 gateway

## 9. 设计代价

- 权限规则存 Redis，gateway 与 admin 必须共享同一 Redis
- 路径未配置资源规则时，gateway 不校验权限（needPermissionList 为空）
- 角色仅改名不触发 pathResourceMap 刷新

## 10. 可提炼候选

| # | 候选 | 层级 |
|---|------|------|
| 1 | 启动 @PostConstruct 预热 Redis 路径-权限 Hash | 方案层 |
| 2 | 资源 CRUD 后同步刷新 Redis，保持网关规则一致 | 方案层 |
| 3 | 登录时 permissionList 写入 Session，网关 StpInterface 读取 | 方案层 |
| 4 | Ant 路径模式 + OR 权限（一路径多资源任一即可） | 方案层 |

## 11. 已读证据

- `mall-admin/src/main/java/com/macro/mall/component/PathResourceRulesHolder.java`
- `mall-admin/src/main/java/com/macro/mall/service/impl/UmsResourceServiceImpl.java`
- `mall-admin/src/main/java/com/macro/mall/service/impl/UmsAdminServiceImpl.java`
- `mall-admin/src/main/java/com/macro/mall/controller/UmsResourceController.java`
- `mall-gateway/src/main/java/com/macro/mall/config/SaTokenConfig.java`
- `mall-gateway/src/main/java/com/macro/mall/component/StpInterfaceImpl.java`
- `mall-common/src/main/java/com/macro/mall/common/constant/AuthConstant.java`

## 12. 待深读问题

1. `requestPath` 是否含 `/mall-admin` 前缀，与 Redis key 格式是否一致
2. 角色仅 update 名称时是否需要手动刷新 pathResourceMap
