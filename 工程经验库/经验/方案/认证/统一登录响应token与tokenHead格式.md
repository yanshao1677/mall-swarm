---
title: 统一登录响应token与tokenHead格式
level: pattern
parent:
status: reviewed
tags:
  - authentication
  - api-contract
created_at: 2026-07-05
updated_at: 2026-07-05
confidence: high
related:
  - Feign请求头透传跨服务认证
source_repo: macrozheng/mall-swarm
source_paths:
  - mall-admin/.../UmsAdminController.java
  - mall-portal/.../UmsMemberController.java
---

# 统一登录响应 token 与 tokenHead 格式

## 本项目落地状态

| 维度 | 状态 |
|------|------|
| 代码位置 | `UmsAdminController` / `UmsMemberController` login 返回 |
| 默认是否启用 | ✅ |

## 1. 问题

多端客户端需统一拼装 `Authorization` 头，避免有的服务只返回裸 token、前缀不一致。

## 2. 核心思路

登录成功 `CommonResult.success(Map)`，Map 固定两键：
- `token`：`SaTokenInfo.getTokenValue()`
- `tokenHead`：配置项 `sa-token.token-prefix` + 空格（如 `Bearer `）

客户端请求头：`Authorization: tokenHead + token`。

## 3. 配置

`sa-token.token-prefix: Bearer`，`is-read-header: true`，`is-read-cookie: false`。

## 4. 测试点

- 登录响应含 token 与 tokenHead
- 后续请求 `Authorization: Bearer {token}` 通过网关鉴权

## 5. 标签

authentication, api-contract, bearer-token

## 附录：来源证据

- `UmsAdminController.login` L57-65
- `UmsMemberController.login` L50-57
- `application.yml` sa-token 块 token-prefix/is-read-header
