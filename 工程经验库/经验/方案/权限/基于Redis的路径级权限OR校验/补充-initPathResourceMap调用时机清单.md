---
title: 补充-initPathResourceMap调用时机清单
level: atomic
parent: 基于Redis的路径级权限OR校验
status: reviewed
tags:
  - permission
  - redis
  - refresh
created_at: 2026-07-05
updated_at: 2026-07-05
confidence: high
related:
  - 阶段1-Redis路径规则预热
  - 阶段3-登录Session写入权限列表
source_repo: macrozheng/mall-swarm
source_paths:
  - mall-admin/.../PathResourceRulesHolder.java
  - mall-admin/.../UmsResourceServiceImpl.java
  - mall-admin/.../UmsRoleServiceImpl.java
  - mall-admin/.../UmsResourceController.java
---

# 补充：`initPathResourceMap` 调用时机清单

## 本项目落地状态

| 维度 | 状态 |
|------|------|
| 代码位置 | `UmsResourceServiceImpl.initPathResourceMap` |
| 方法行为 | 仅读 `ums_resource` 全表 → 写 Redis `auth:pathResourceMap` |
| 默认是否启用 | ✅ admin 启动与各调用点均会执行 |

## 方法实际做什么（避免误解）

```
initPathResourceMap():
    SELECT * FROM ums_resource
    FOR EACH resource:
        key = "/" + spring.application.name + resource.url   // 如 /mall-admin/brand/**
        value = resource.id + ":" + resource.name
    Redis DEL auth:pathResourceMap
    Redis HSETALL auth:pathResourceMap
```

**不读取**：`ums_role`、`ums_role_resource_relation`、用户 Session。  
**因此**：角色分配资源（`allocResource`）**不会改变** Redis 中 path→resource 映射内容（除非并发发生了资源 CRUD）。

角色权限变更生效路径是 [阶段3-登录Session写入权限列表](阶段3-登录Session写入权限列表.md)（用户需重新登录后 Session 中的 `permissionList` 才更新）。

## 调用时机对照表（源码逐行）

| # | 触发场景 | 调用位置 | 是否改变 path map 内容 |
|---|----------|----------|------------------------|
| 1 | admin 启动 | `PathResourceRulesHolder.@PostConstruct` L20-21 | ✅ 首次写入 |
| 2 | 资源新增 | `UmsResourceServiceImpl.create` L32 | ✅ url 新增 |
| 3 | 资源修改 | `UmsResourceServiceImpl.update` L40 | ✅ url/name 可能变 |
| 4 | 资源删除 | `UmsResourceServiceImpl.delete` L52 | ✅ 条目减少 |
| 5 | 手动 API | `GET /resource/initPathResourceMap`（网关路径 `/mall-admin/resource/initPathResourceMap`）L97-98 | ✅ 全量重载 |
| 6 | 角色分配资源 | `UmsRoleServiceImpl.allocResource` L116 | ❌ 代码调用了，但数据源 `ums_resource` 未变 |
| 7 | 角色批量删除 | `UmsRoleServiceImpl.delete` L53 | ❌ 同上 |

## 未调用 initPathResourceMap 的操作（源码确认）

| 操作 | 位置 | 说明 |
|------|------|------|
| 角色新增 | `UmsRoleServiceImpl.create` L35-39 | 无调用 |
| 角色修改 | `UmsRoleServiceImpl.update` L43-45 | 无调用 |
| 菜单分配 | `UmsRoleServiceImpl.allocMenu` L88-100 | 无调用 |

## 网关侧影响

- path map **有变更**（#1–5）→ [阶段2-网关Ant路径OR匹配](阶段2-网关Ant路径OR匹配.md) 匹配的 pattern 集合变化
- path map **无实质变更**（#6–7）→ Redis 内容重写但 key/value 与刷新前相同
- 角色权限变更（#6）→ 已登录用户的 `permissionList` **不会**自动更新，需重新登录

## 最小验证清单

- 新增 `ums_resource` 后 Redis 出现新 key `/mall-admin{url}`
- `allocResource` 后 Redis `HGETALL` 条数与内容不变（无并发资源 CRUD 时）
- `GET /mall-admin/resource/initPathResourceMap` 返回当前 map

## 附录：来源证据

- `PathResourceRulesHolder.java` L19-21
- `UmsResourceServiceImpl.java` L29-52、L79-87
- `UmsRoleServiceImpl.java` L49-54、L104-117
- `UmsResourceController.java` L95-99
