---
title: 基于Redis的路径级权限OR校验
level: pattern
parent:
status: reviewed
tags:
  - permission
  - redis
  - gateway
  - rbac
created_at: 2026-07-05
updated_at: 2026-07-05
confidence: high
related:
  - 网关集中鉴权与双账号体系分离
  - 登录态写入Token Session供网关消费
source_repo: macrozheng/mall-swarm
source_paths:
  - mall-admin/.../UmsResourceServiceImpl.java
  - mall-gateway/.../SaTokenConfig.java
---

# 基于 Redis 的路径级权限 OR 校验

## 本项目落地状态

| 维度 | 状态 |
|------|------|
| 代码位置 | `mall-admin/UmsResourceServiceImpl` + `mall-gateway/SaTokenConfig` |
| 默认是否启用 | ✅ admin 启动预热 Redis；网关每次请求读取 |
| 依赖 | admin 与 gateway 共用 Redis |

详见 Redis 基础设施：[BaseRedisConfig与RedisService封装](../通用/BaseRedisConfig与RedisService封装.md)。

## 1. 问题

管理后台 API 数量多，不宜在网关硬编码每条路径的权限；需动态维护「请求路径 → 所需权限」，且一路径可对应多个权限资源（拥有任一即可访问）。

## 2. 适用约束

- 网关统一鉴权，管理员路径前缀固定
- 权限资源表含 Ant 风格 url 模式（如 `/brand/**`）
- 网关与权限管理服务共享 Redis
- 用户登录时已将 permissionList 写入 Token Session

## 3. 核心思路

权限管理服务将 `{网关可见完整路径 → 权限码}` 写入 Redis Hash；网关每次请求 Ant 匹配 requestPath 收集权限码列表，调用 `checkPermissionOr`；权限码格式与 Session 中列表一致（`id:name`）。

## 4. 通用结构

| 组件 | 职责 |
|------|------|
| PathRulePublisher | 启动/CRUD 时刷新 Redis Hash |
| GatewayPermissionFilter | 读 Hash + Ant 匹配 + OR 校验 |
| LoginSessionWriter | 登录写 permissionList 到 Session |
| StpPermissionProvider | 从 Session 提供权限列表给框架 |

## 5. 处理流程

1. **预热**：服务启动 `@PostConstruct` → 查全部资源 → `key = /{appName}{resource.url}`，`value = {id}:{name}` → `DEL` + `HSETALL`
2. **登录**：用户-角色-资源联表 → `permissionList` → Session
3. **请求**：网关 `checkLogin` → `HGETALL` → Ant 匹配 → `checkPermissionOr`

分阶段原子规格见子目录本方案下阶段 1–3。

## 6. 异常处理

- 未登录：401
- 无任一匹配权限：403
- Redis 不可用：网关鉴权失败（无本地缓存回退）
- 路径无匹配规则：不校验权限（needPermissionList 为空则跳过）

## 7. 具体语言实现（Java + Sa-Token + Redis）

```java
// 发布端（管理服务的 ResourceService）
public Map<String, String> initPathResourceMap() {
    Map<String, String> map = new TreeMap<>();
    for (Resource r : resourceMapper.selectAll()) {
        map.put("/" + applicationName + r.getUrl(), r.getId() + ":" + r.getName());
    }
    redisService.del("auth:pathResourceMap");
    redisService.hSetAll("auth:pathResourceMap", map);
    return map;
}

// 网关消费端（SaReactorFilter setAuth 内）
Map<Object, Object> pathResourceMap = redisTemplate.opsForHash()
    .entries("auth:pathResourceMap");
String requestPath = SaHolder.getRequest().getRequestPath();
List<String> need = new ArrayList<>();
AntPathMatcher matcher = new AntPathMatcher();
for (Map.Entry<Object, Object> e : pathResourceMap.entrySet()) {
    if (matcher.match((String) e.getKey(), requestPath)) {
        need.add((String) e.getValue());
    }
}
if (!need.isEmpty()) {
    StpUtil.checkPermissionOr(need.toArray(new String[0]));
}
```

## 8. 测试点

- 资源 url `/order/**` 匹配 `/app/order/list` 时触发对应权限
- 用户仅有 A 权限、规则要求 A|B 时 OR 通过
- 用户无 A 无 B 时 403
- 资源 CRUD 后刷新 Redis，新路径规则生效

## 9. 适用 / 不适用

**适用**：后台 RBAC + 网关集中鉴权。**不适用**：前台细粒度权限、无共享 Redis 的多区域网关。

## 10. 风险与反模式

- requestPath 是否含服务前缀必须与 Redis key 格式一致，否则永不匹配
- 忘记资源变更后刷新 Redis → 网关规则陈旧
- 将 portal 路径也走权限块但 Session 无 permissionList → 误拒

## 11. 标签

permission, redis, gateway, ant-path, or-check

## 附录：来源证据

- `UmsResourceServiceImpl.initPathResourceMap` L79-87
- `SaTokenConfig` L56-73
- `AuthConstant.PATH_RESOURCE_MAP = "auth:pathResourceMap"`
