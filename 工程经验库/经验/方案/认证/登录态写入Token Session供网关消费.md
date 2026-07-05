---
title: 登录态写入Token Session供网关消费
level: pattern
parent:
status: reviewed
tags:
  - authentication
  - session
created_at: 2026-07-05
updated_at: 2026-07-05
confidence: high
related:
  - 基于Redis的路径级权限OR校验
  - 统一登录聚合服务（clientId 路由到领域服务）
source_repo: macrozheng/mall-swarm
source_paths:
  - mall-admin/.../UmsAdminServiceImpl.java
  - mall-gateway/.../StpInterfaceImpl.java
---

# 登录态写入 Token Session 供网关消费

## 本项目落地状态

| 维度 | 状态 |
|------|------|
| 代码位置 | admin `UmsAdminServiceImpl.login`；portal 写 memberInfo **无** permissionList |
| 默认是否启用 | ✅ |

## 1. 问题

网关校验权限时需知用户拥有哪些权限码，但不希望每次请求查库；需在登录时一次性加载并随 token 存共享 Session。

## 2. 核心思路

登录成功后 `StpUtil.login(userId)`，组装 `UserDto`（含 `clientId`、`permissionList`），`StpUtil.getSession().set("adminInfo", userDto)`。网关 `StpInterface.getPermissionList` 从 Session 读取 `permissionList` 供 `checkPermissionOr` 使用。

## 3. 权限码格式

`permissionList` 每项 = `{resourceId}:{resourceName}`，与 Redis pathResourceMap 的 value 格式一致。

## 4. 前台差异

会员登录写 `memberInfo` 到 Session，**不写 permissionList**；网关对 portal 路径仅 checkLogin。

## 5. 测试点

- 登录后 token 在网关可 checkLogin
- 管理员 permissionList 与 DB 角色资源一致
- 网关 StpInterface 返回的列表与登录时一致

## 6. 标签

authentication, session, permission-list

## 附录：来源证据

- `UmsAdminServiceImpl.login` L103-113
- `UmsMemberServiceImpl.login` L162-168（无 permissionList）
- `StpInterfaceImpl` L21-30
- `AuthConstant.STP_ADMIN_INFO` / `STP_MEMBER_INFO`
