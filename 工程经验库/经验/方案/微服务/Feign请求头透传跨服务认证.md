---
title: Feign请求头透传跨服务认证
level: pattern
parent:
status: reviewed
tags:
  - feign
  - microservice
  - authentication
  - demo-only
created_at: 2026-07-05
updated_at: 2026-07-05
confidence: medium
related:
  - 统一登录响应token与tokenHead格式
  - EnableFeignClients与Nacos服务发现调用
source_repo: macrozheng/mall-swarm
source_paths:
  - mall-demo/.../FeignRequestInterceptor.java
  - mall-demo/.../config/FeignConfig.java
---

# Feign 请求头透传跨服务认证

## 本项目落地状态

| 维度 | 状态 |
|------|------|
| 代码位置 | **仅 `mall-demo`** 模块（`FeignRequestInterceptor` + `FeignConfig`） |
| 默认是否启用 | ⚠️ **演示模块**；`mall-auth`/`mall-admin`/`mall-portal` **未** 注册此 Interceptor |
| 适用场景 | 学习 Feign 带 token 调下游；**不宜**当作全项目标准 |
| 主业务 Feign | `mall-auth` 调登录接口无需透传 Authorization |

## 1. 问题

BFF 或聚合服务通过 Feign 调下游需登录接口时，必须把用户 `Authorization` 传到下游，否则下游 401。

## 2. 核心思路

实现 `RequestInterceptor`：从 `RequestContextHolder` 取当前 `HttpServletRequest`，遍历 headerNames，**跳过 `content-length`**（防 `Incomplete output stream`），其余 `requestTemplate.header(name, value)`。在 `FeignConfig` 注册该 Interceptor。

## 3. 处理流程

Client → **mall-demo** Controller（带 Authorization）→ Feign apply 透传 → 下游服务 StpUtil 校验通过。

## 4. 原子细节

见子文档「阶段1-跳过content-length头透传」。

## 5. 风险

全量透传可能带入无关或过大 header；生产可改为白名单仅透传 Authorization/Correlation-Id。若要在主业务复用，需在各服务的 FeignConfig **显式注册**，本项目未做。

## 6. 标签

feign, request-interceptor, authorization, demo-only

## 附录：来源证据

- `FeignRequestInterceptor.apply` L17-34（仅 mall-demo）
- `FeignConfig` 注册 Interceptor（仅 mall-demo）
- `mall-auth` Feign 接口无 RequestInterceptor（已 grep 核实）
