---
title: CommonResult统一API响应封装
level: pattern
parent:
status: reviewed
tags:
  - api
  - common
created_at: 2026-07-05
updated_at: 2026-07-05
confidence: high
related:
  - 业务断言失败经全局异常转响应
source_repo: macrozheng/mall-swarm
source_paths:
  - mall-common/.../CommonResult.java
  - mall-common/.../ResultCode.java
---

# CommonResult 统一 API 响应封装

## 本项目落地状态

| 维度 | 状态 |
|------|------|
| 代码位置 | `mall-common/CommonResult` + `ResultCode` |
| 默认是否启用 | ✅ 各服务 Controller 与 gateway 异常处理均使用 |

## 1. 问题

多微服务若响应格式不一，网关聚合与前端统一处理困难。

## 2. 核心思路

所有 API 返回 `CommonResult<T>`：`code`(long)、`message`(String)、`data`(T)。`ResultCode` 枚举：SUCCESS=200、FAILED=500、VALIDATE_FAILED=404、UNAUTHORIZED=401、FORBIDDEN=403。工厂方法 `success/failed/validateFailed/unauthorized/forbidden`。

## 3. 分页

`CommonPage<T>` 适配 PageHelper List 与 Spring Data `Page`，字段 pageNum/pageSize/totalPage/total/list。

## 4. 测试点

- 成功 code=200 且 data 非空
- 校验失败 code=404
- 网关鉴权失败 code=401/403 与业务一致

## 5. 标签

api, common-result, envelope

## 附录：来源证据

- `CommonResult.java` 工厂方法 L28-100
- `ResultCode.java` L7-12
