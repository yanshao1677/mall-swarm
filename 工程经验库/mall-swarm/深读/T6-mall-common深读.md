# T6：mall-common 通用层深读

> **场景**：模板场景二（模块深读）
> **生成时间**：2026-07-05
> **范围**：CommonResult、GlobalExceptionHandler、RedisService、AuthConstant

---

## 1. 模块背景

`mall-common` 是 jar 库模块（无 Application），提供各微服务共享的 API 封装、Redis 工具、认证常量、全局异常处理、Web 日志切面。

## 2. 核心职责

| 负责 | 不负责 |
|------|--------|
| 统一 HTTP 响应格式 `CommonResult` | 业务逻辑 |
| 全局异常转 CommonResult | 鉴权执行 |
| Redis 基础封装 | 业务级缓存策略 |
| 认证相关常量定义 | token 签发 |

## 3. 核心对象（15 个 Java 文件）

| 对象 | 路径 | 作用 |
|------|------|------|
| `CommonResult<T>` | `common/api/CommonResult.java` | 统一响应：code/message/data |
| `ResultCode` | `common/api/ResultCode.java` | 错误码枚举 |
| `CommonPage<T>` | `common/api/CommonPage.java` | 分页封装 |
| `IErrorCode` | `common/api/IErrorCode.java` | 错误码接口 |
| `GlobalExceptionHandler` | `common/exception/GlobalExceptionHandler.java` | 全局异常处理 |
| `ApiException` / `Asserts` | `common/exception/` | 业务异常与断言 |
| `RedisService` / `RedisServiceImpl` | `common/service/` | Redis 操作封装 |
| `BaseRedisConfig` | `common/config/BaseRedisConfig.java` | RedisTemplate + CacheManager |
| `AuthConstant` | `common/constant/AuthConstant.java` | 认证常量 |
| `WebLogAspect` | `common/log/WebLogAspect.java` | Controller 请求日志 |
| `UserDto` | `common/dto/UserDto.java` | Session 用户信息 |
| `WebLog` | `common/domain/WebLog.java` | 日志实体 |
| `CacheException` | `common/annotation/CacheException.java` | 缓存异常注解 |

## 4. 内部流程

### 4.1 CommonResult 工厂方法

| 方法 | code | 证据 |
|------|------|------|
| `success(data)` | 200 | `CommonResult.java` L28–29 |
| `failed()` / `failed(message)` | 500 | L63–71 |
| `validateFailed()` | 404 | L77–86 |
| `unauthorized(data)` | 401 | L92–93 |
| `forbidden(data)` | 403 | L99–100 |

**ResultCode 枚举**：SUCCESS(200)、FAILED(500)、VALIDATE_FAILED(404)、UNAUTHORIZED(401)、FORBIDDEN(403)。

### 4.2 GlobalExceptionHandler

| 异常 | 处理 |
|------|------|
| `ApiException` | `CommonResult.failed(e.getMessage())` |
| `MethodArgumentNotValidException` | 取首个 FieldError → `validateFailed` |
| `BindException` | 同上 |

### 4.3 Redis 封装

**BaseRedisConfig**：

- `RedisTemplate<String,Object>`：String key + Jackson JSON value（含 default typing）
- `RedisCacheManager`：TTL 1 天
- 手动 `@Bean redisService()` → `RedisServiceImpl`（Impl 无 `@Service`）

**RedisService**：封装 String/Hash/Set/List 操作，过期单位均为**秒**。

### 4.4 WebLogAspect

- 切点：`com.macro.mall.controller.*` 或 `com.macro.mall.*.controller.*`
- `@Around`：记录 url/method/parameter/spendTime，Logstash Marker 输出 JSON
- 注意：`webLog.setIp(request.getRemoteUser())` 实际取 RemoteUser 而非 IP

### 4.5 AuthConstant 关键常量

| 常量 | 值 |
|------|-----|
| `JWT_TOKEN_HEADER` | `Authorization` |
| `JWT_TOKEN_PREFIX` | `Bearer ` |
| `ADMIN_CLIENT_ID` / `PORTAL_CLIENT_ID` | `admin-app` / `portal-app` |
| `PATH_RESOURCE_MAP` | `auth:pathResourceMap` |
| `STP_ADMIN_INFO` / `STP_MEMBER_INFO` | Session 键 |

## 5. 对外接口

mall-common 不直接对外暴露 HTTP，而是以依赖形式被各服务引用：

- 所有 Controller 返回 `CommonResult`
- gateway 鉴权异常返回 `CommonResult.unauthorized/forbidden`
- admin 权限初始化用 `RedisService.hSetAll(PATH_RESOURCE_MAP, ...)`

## 6. 扩展点

- 新增错误码：扩展 `ResultCode` 枚举
- 新增 Redis 操作：在 `RedisService` 接口加方法

## 7. 错误处理

- 业务层 `Asserts.fail(msg)` → `ApiException` → `GlobalExceptionHandler` → `CommonResult.failed`
- 参数校验失败 → `validateFailed` code=404

## 8. 设计优点

- 全项目统一响应格式，gateway 与各服务异常语义一致
- BaseRedisConfig 可被各服务继承复用

## 9. 设计代价

- `ApiException` 统一映射 code=500，业务错误与系统错误未细分
- WebLogAspect IP 字段取值有误（RemoteUser）

## 10. 可提炼候选

| # | 候选 | 层级 |
|---|------|------|
| 1 | CommonResult 三字段统一响应 + ResultCode 枚举 | 方案层 |
| 2 | Asserts.fail → ApiException → GlobalExceptionHandler 链 | 方案层 |
| 3 | BaseRedisConfig 抽象 + RedisService 业务封装 | 方案层 |
| 4 | AuthConstant 集中定义跨服务认证契约 | 方案层 |

## 11. 已读证据

- `mall-common/src/main/java/com/macro/mall/common/api/CommonResult.java`
- `mall-common/src/main/java/com/macro/mall/common/api/ResultCode.java`
- `mall-common/src/main/java/com/macro/mall/common/api/CommonPage.java`
- `mall-common/src/main/java/com/macro/mall/common/exception/GlobalExceptionHandler.java`
- `mall-common/src/main/java/com/macro/mall/common/exception/Asserts.java`
- `mall-common/src/main/java/com/macro/mall/common/service/RedisService.java`
- `mall-common/src/main/java/com/macro/mall/common/service/impl/RedisServiceImpl.java`
- `mall-common/src/main/java/com/macro/mall/common/config/BaseRedisConfig.java`
- `mall-common/src/main/java/com/macro/mall/common/constant/AuthConstant.java`
- `mall-common/src/main/java/com/macro/mall/common/log/WebLogAspect.java`

## 12. 待深读问题

1. 各服务是否都注册了 `GlobalExceptionHandler`（需确认 `@RestControllerAdvice` 扫描范围）
