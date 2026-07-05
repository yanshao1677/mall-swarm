---
title: 阶段1-跳过content-length头透传
level: atomic
parent: Feign请求头透传跨服务认证
status: reviewed
tags:
  - feign
  - header
created_at: 2026-07-05
updated_at: 2026-07-05
confidence: high
related: []
source_repo: macrozheng/mall-swarm
source_paths:
  - mall-demo/.../FeignRequestInterceptor.java
---

# 阶段 1：跳过 content-length 头透传

## 本项目落地状态

| 维度 | 状态 |
|------|------|
| 代码位置 | **仅** `mall-demo/FeignRequestInterceptor` |
| 主业务 | mall-auth/admin/portal **未使用** |

## 触发条件

- 当前线程存在 `ServletRequestAttributes`（Feign 在 Web 请求线程内调用）
- `RequestInterceptor.apply(RequestTemplate)` 被 Feign 执行

## 输入字段

| 字段 | 类型 | 来源 |
|------|------|------|
| headerNames | Enumeration\<String\> | `HttpServletRequest.getHeaderNames()` |
| name | String | 每个 header 名 |
| values | String | `request.getHeader(name)` |

## 判定规则

```
IF attributes == null:
    不添加任何头（非 Web 上下文）
ELSE:
    WHILE headerNames.hasMoreElements():
        name = next
        IF name.equalsIgnoreCase("content-length"):
            CONTINUE   // 跳过
        requestTemplate.header(name, values)
```

跳过原因（代码注释）：传递 content-length 会导致 `java.io.IOException: Incomplete output stream`。

## 状态读写位置

- 读：当前 HTTP 请求头
- 写：Feign `RequestTemplate` headers

## 正常路径

1. 取 RequestContextHolder attributes
2. 遍历所有 header
3. 非 content-length → 复制到 template
4. Feign 发起下游请求（含 Authorization 等）

## 分支路径

- 无 Servlet 上下文 → apply 空操作
- content-length → 跳过
- 其他头（含 Authorization、User-Agent 等）→ 原样透传

## 失败处理

- 无 try-catch；下游 401 由业务 CommonResult 返回

## 幂等性 / 一致性约束

- 只读当前请求头，无副作用
- 多次 Feign 调用各透传当时请求头

## 代码骨架

```java
public void apply(RequestTemplate template) {
    ServletRequestAttributes attrs = (ServletRequestAttributes)
        RequestContextHolder.getRequestAttributes();
    if (attrs == null) return;
    HttpServletRequest request = attrs.getRequest();
    Enumeration<String> names = request.getHeaderNames();
    if (names == null) return;
    while (names.hasMoreElements()) {
        String name = names.nextElement();
        if ("content-length".equalsIgnoreCase(name)) continue;
        template.header(name, request.getHeader(name));
    }
}
```

## 最小验证清单

- 带 Authorization 调 Feign 下游 → 下游 checkLogin 通过
- 透传不含 content-length 头
- 无 Web 上下文调 Feign → 不 NPE，无头透传

## 附录：来源证据

- `FeignRequestInterceptor.java` L17-34
