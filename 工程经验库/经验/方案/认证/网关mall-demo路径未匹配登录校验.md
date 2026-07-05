---
title: 网关mall-demo路径未匹配登录校验
level: atomic
parent: 网关集中鉴权与双账号体系分离
status: reviewed
tags:
  - gateway
  - authentication
  - mall-demo
  - gap
created_at: 2026-07-05
updated_at: 2026-07-05
confidence: high
related:
  - 网关集中鉴权与双账号体系分离
  - 网关路径白名单外置配置
source_repo: macrozheng/mall-swarm
source_paths:
  - mall-gateway/.../SaTokenConfig.java
  - mall-gateway/.../application.yml
---

# 网关：`/mall-demo/**` 未匹配登录校验

## 本项目落地状态

| 维度 | 状态 |
|------|------|
| 代码位置 | `mall-gateway/SaTokenConfig` + `application.yml` 路由与白名单 |
| 路由 | ✅ `Path=/mall-demo/**` → `lb://mall-demo`（L44-49） |
| 白名单 | ❌ `secure.ignore.urls` **不含** `/mall-demo/**` |
| 登录校验 | ⚠️ 过滤器 **未** 对 `/mall-demo/**` 调用 `checkLogin` |
| mall-demo 服务内 | ❌ 无 Sa-Token / `checkLogin` 代码（glob 无匹配） |

## 网关过滤器实际执行顺序（源码 L47-73）

对**非白名单**请求进入 `setAuth` 后：

```
1. OPTIONS → stop（放行）
2. match /mall-portal/** → StpMemberUtil.checkLogin() → stop
3. match /mall-admin/**  → StpUtil.checkLogin()        // 无 stop，继续往下
4. 读 Redis auth:pathResourceMap，Ant 匹配 requestPath
5. IF needPermissionList 非空 → StpUtil.checkPermissionOr(...)
```

**`/mall-demo/**` 请求**：

- 步骤 2、3 均不匹配 → **无 checkLogin**
- 步骤 4：Redis key 形如 `/mall-admin/...`（见 `UmsResourceServiceImpl` L83），**通常不匹配** `/mall-demo/...`
- 步骤 5：`needPermissionList` 为空 → **无 checkPermissionOr**
- 结果：**网关层不校验登录、不校验权限**，直接路由到 mall-demo

## 与白名单路径的区别

| 路径类型 | 是否进入 setAuth | 网关登录校验 |
|----------|------------------|--------------|
| 白名单（如 `/mall-search/**`） | 否（excludeList） | 跳过过滤器 |
| `/mall-portal/**`（非白名单子路径） | 是 | `StpMemberUtil.checkLogin` |
| `/mall-admin/**`（非白名单子路径） | 是 | `StpUtil.checkLogin` + 权限 OR |
| **`/mall-demo/**`** | **是** | **无 checkLogin** |

## 代码事实（不作安全评级）

- 这是 `SaTokenConfig` 现有分支结构的**直接结果**，不是配置遗漏白名单
- mall-demo 服务自身也未集成鉴权
- 若需鉴权，代码中**尚无**对应实现；需改网关 `SaRouter` 或 mall-demo 服务（**超出本文档范围，仓库未做**）

## 最小验证清单

- 不带 Token 请求 `/mall-demo/...`（非网关白名单的其他路径）→ 网关不返回 401（由 demo 业务决定响应）
- 对比 `/mall-admin/...` 无 Token → 401
- Redis `HGETALL auth:pathResourceMap` 无 `/mall-demo/` 前缀 key

## 附录：来源证据

- `SaTokenConfig.java` L45-73
- `application.yml` L44-49（mall-demo 路由）、L56-79（白名单，无 mall-demo）
- `mall-demo/**/*.java` 无 Sa-Token 引用
