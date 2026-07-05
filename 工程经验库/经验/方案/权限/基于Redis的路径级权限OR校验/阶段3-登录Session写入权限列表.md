---
title: 阶段3-登录Session写入权限列表
level: atomic
parent: 基于Redis的路径级权限OR校验
status: reviewed
tags:
  - permission
  - login
  - session
created_at: 2026-07-05
updated_at: 2026-07-05
confidence: high
related:
  - 阶段2-网关Ant路径OR匹配
source_repo: macrozheng/mall-swarm
source_paths:
  - mall-admin/.../UmsAdminServiceImpl.java
  - mall-gateway/.../StpInterfaceImpl.java
---

# 阶段 3：登录 Session 写入权限列表

## 本项目落地状态

| 维度 | 状态 |
|------|------|
| 代码位置 | `mall-admin/UmsAdminServiceImpl.login` |
| 默认是否启用 | ✅ 仅 admin；portal 不写 permissionList |

## 触发条件

- 管理员 `POST /admin/login` 校验通过（密码 BCrypt、status=1）
- 尚未调用 `StpUtil.login`

## 输入字段

| 字段 | 类型 | 来源 |
|------|------|------|
| adminId | Long | 查库结果 |
| resourceList | List\<UmsResource\> | `getResourceList(adminId)` SQL 联表 |

## 判定规则

```
StpUtil.login(adminId)
permissionList = resourceList.stream()
    .map(item -> item.getId() + ":" + item.getName())
    .toList()
userDto.clientId = "admin-app"
userDto.permissionList = permissionList
StpUtil.getSession().set("adminInfo", userDto)
```

Session 键常量：`STP_ADMIN_INFO = "adminInfo"`。

## 状态读写位置

- 写：Sa-Token Session（底层 Redis，与 gateway 共享）
- 读：网关 `StpInterfaceImpl.getPermissionList` 从 `StpUtil.getSession().get("adminInfo")` 取 `UserDto.permissionList`

## 正常路径

1. 校验用户名密码状态
2. `StpUtil.login(admin.getId())`
3. 查资源列表
4. 组装 UserDto + permissionList
5. 写 Session
6. `insertLoginLog`
7. 返回 `SaTokenInfo`

## 分支路径

- 校验失败 → `Asserts.fail`，不写 Session
- member 登录：写 `memberInfo`，**permissionList 为空/null**，网关 StpInterface 对 member loginType 返回 null

## 失败处理

- 抛 ApiException → 无 token 产生

## 幂等性 / 一致性约束

- 每次登录新建 token（`is-share: false`）时 Session 独立
- permissionList 与 DB 角色资源一致时点快照；角色变更后需重新登录或证据不足：是否支持踢下线刷新

## 代码骨架

```java
StpUtil.login(admin.getId());
UserDto userDto = new UserDto();
userDto.setClientId(AuthConstant.ADMIN_CLIENT_ID);
userDto.setPermissionList(
    resourceList.stream().map(r -> r.getId() + ":" + r.getName()).toList());
StpUtil.getSession().set(AuthConstant.STP_ADMIN_INFO, userDto);
```

```java
// StpInterfaceImpl
public List<String> getPermissionList(Object loginId, String loginType) {
    if (StpUtil.getLoginType().equals(loginType)) {
        UserDto dto = (UserDto) StpUtil.getSession().get(AuthConstant.STP_ADMIN_INFO);
        return dto.getPermissionList();
    }
    return null;
}
```

## 最小验证清单

- 登录后 Session 含 adminInfo.permissionList 条数 = 用户角色资源数
- 每项格式 `数字:字符串` 与 Redis pathResourceMap value 一致
- 网关 checkPermissionOr 使用的列表与登录时相同

## 附录：来源证据

- `UmsAdminServiceImpl.login` L103-113
- `StpInterfaceImpl` L21-30
- `UmsAdminRoleRelationDao.xml` 联表 SQL
