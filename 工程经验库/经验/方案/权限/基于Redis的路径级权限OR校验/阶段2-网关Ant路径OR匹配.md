---
title: 阶段2-网关Ant路径OR匹配
level: atomic
parent: 基于Redis的路径级权限OR校验
status: reviewed
tags:
  - permission
  - gateway
  - ant-path
created_at: 2026-07-05
updated_at: 2026-07-05
confidence: high
related:
  - 阶段1-Redis路径规则预热
  - 阶段3-登录Session写入权限列表
source_repo: macrozheng/mall-swarm
source_paths:
  - mall-gateway/.../SaTokenConfig.java
---

# 阶段 2：网关 Ant 路径 OR 匹配

## 本项目落地状态

| 维度 | 状态 |
|------|------|
| 代码位置 | `mall-gateway/SaTokenConfig` L56-73 |
| 默认是否启用 | ✅ |
| 待确认 | `requestPath` 与 Redis key 前缀是否一致需实测 |

## 触发条件

- 请求未命中白名单（`setExcludeList`）
- 非 OPTIONS 预检
- `/mall-portal/**` 已在上一规则 `checkLogin` 后 **stop**（本阶段对 portal 不执行）
- `/mall-admin/**` 已通过 `StpUtil.checkLogin()`

## 输入字段

| 字段 | 类型 | 来源 |
|------|------|------|
| requestPath | String | `SaHolder.getRequest().getRequestPath()` |
| pathResourceMap | Map | Redis `HGETALL auth:pathResourceMap` |
| pattern | String | map 每条 key |
| permissionCode | String | map 每条 value（`id:name`） |

## 判定规则

```
needPermissionList = []
FOR each (pattern, permissionCode) IN pathResourceMap:
    IF AntPathMatcher.match(pattern, requestPath):
        needPermissionList.add(permissionCode)

IF needPermissionList 非空:
    StpUtil.checkPermissionOr(needPermissionList 转 String[])
ELSE:
    跳过权限校验（该路径未配置资源）
```

**OR 语义**：注释明确「一个路径对应多个资源时，拥有任意一个资源都可以访问」。

Matcher 实现：`org.springframework.util.AntPathMatcher`。

## 状态读写位置

- 读：Redis Hash `auth:pathResourceMap`（每次请求全量 entries）
- 读：Token Session（经 StpInterface 取 permissionList，阶段 3）

## 正常路径

1. `redisTemplate.opsForHash().entries(PATH_RESOURCE_MAP)`
2. 取 requestPath
3. 双重循环匹配收集 needPermissionList
4. 非空 → `SaRouter.match(requestPath, r -> StpUtil.checkPermissionOr(...))`

## 分支路径

- needPermissionList 为空 → 不调用 checkPermissionOr，请求继续路由
- checkPermissionOr 失败 → `NotPermissionException` → 403
- 非 NotLogin/NotPermission 异常 → `CommonResult.failed(message)`

## 失败处理

- Redis 不可用 → 过滤器抛错，请求失败
- 证据不足：requestPath 是否含 `/mall-admin` 前缀需与阶段 1 key 格式对齐实测

## 幂等性 / 一致性约束

- 只读 Redis；规则陈旧直到管理端 refresh
- 无本地缓存，每次请求打 Redis

## 代码骨架

```java
Map<Object, Object> pathResourceMap = redisTemplate.opsForHash()
    .entries(AuthConstant.PATH_RESOURCE_MAP);
String requestPath = SaHolder.getRequest().getRequestPath();
AntPathMatcher pathMatcher = new AntPathMatcher();
List<String> need = new ArrayList<>();
for (Map.Entry<Object, Object> e : pathResourceMap.entrySet()) {
    if (pathMatcher.match((String) e.getKey(), requestPath)) {
        need.add((String) e.getValue());
    }
}
if (CollUtil.isNotEmpty(need)) {
    StpUtil.checkPermissionOr(Convert.toStrArray(need));
}
```

## 最小验证清单

- 规则 `/mall-admin/order/**` + 请求 `/mall-admin/order/list` → 触发校验
- 用户 permissionList 含规则中任一 value → 200
- 用户 permissionList 与规则无交集 → 403
- 无匹配规则的路径 → 仅 checkLogin 即通过

## 附录：来源证据

- `SaTokenConfig.java` L56-73
