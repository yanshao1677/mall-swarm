---
title: BaseRedisConfig与RedisService封装
level: pattern
parent:
status: reviewed
tags:
  - redis
  - common
  - cache
created_at: 2026-07-05
updated_at: 2026-07-05
confidence: high
related:
  - 基于Redis的路径级权限OR校验
source_repo: macrozheng/mall-swarm
source_paths:
  - mall-common/.../BaseRedisConfig.java
  - mall-common/.../RedisService.java
  - mall-common/.../RedisServiceImpl.java
  - mall-admin/.../RedisConfig.java
  - mall-portal/.../RedisConfig.java
  - mall-gateway/.../RedisConfig.java
---

# BaseRedisConfig 与 RedisService 封装

## 本项目落地状态

| 维度 | 状态 |
|------|------|
| 代码位置 | `mall-common/BaseRedisConfig` + `RedisService`/`RedisServiceImpl` |
| 接入模块 | ✅ **仅** mall-admin、mall-portal、mall-gateway（各 `RedisConfig extends BaseRedisConfig`） |
| 未接入 | mall-search、mall-auth、mall-demo、mall-monitor **无** `RedisConfig` |
| 网关用法 | ⚠️ 网关注入 `RedisTemplate` 直读 Hash，**不用** `RedisService` |

## 1. BaseRedisConfig 提供的 Bean（源码 L28-64）

| Bean | 作用 |
|------|------|
| `RedisTemplate<String, Object>` | key/hashKey 用 `StringRedisSerializer`；value/hashValue 用 Jackson JSON |
| `RedisSerializer<Object>` | `Jackson2JsonRedisSerializer` + `ObjectMapper.activateDefaultTyping(NON_FINAL)` |
| `RedisCacheManager` | 默认缓存 TTL **1 天** |
| `RedisService` | `return new RedisServiceImpl()`（**非** `@Service` 组件扫描） |

各模块 `RedisConfig` 为空继承，仅 `@Configuration` 激活上述 Bean。portal 额外有 `@EnableCaching`。

## 2. RedisService 能力范围

对 `RedisTemplate` 的薄封装，覆盖：String（set/get/del/incr）、Hash（hGet/hSetAll/hDel…）、Set、List。

业务侧通过 `@Autowired RedisService` 调用，不直接操作 Template（**网关除外**）。

## 3. 本项目实际调用点（glob 核实）

| 模块 | 类 | 用途 | 典型 key 形态 |
|------|-----|------|---------------|
| mall-admin | `UmsResourceServiceImpl` | `del` + `hSetAll` 写 `auth:pathResourceMap` | 常量 `AuthConstant.PATH_RESOURCE_MAP` |
| mall-admin | `UmsAdminCacheServiceImpl` | admin 对象缓存 | `{redis.database}:{redis.key.admin}:{adminId}` |
| mall-portal | `UmsMemberCacheServiceImpl` | 会员缓存、短信验证码 | `...:ums:member:{id}`、`...:ums:authCode:{phone}` |
| mall-portal | `OmsPortalOrderServiceImpl` | 订单号自增 | `...:oms:orderId:{yyyyMMdd}` + `incr` |

`redis.database`、`redis.key.*`、`redis.expire.*` 定义在 **各服务本地** `application.yml`（非 Nacos config 模板中的 `spring.data.redis` 连接项）。

示例（`mall-admin/application.yml` L53-58）：

```yaml
redis:
  database: mall-swarm
  key:
    admin: 'ums:admin'
  expire:
    common: 86400
```

## 4. 网关侧差异

`SaTokenConfig` 注入 `RedisTemplate<String, Object>`，调用 `opsForHash().entries(AuthConstant.PATH_RESOURCE_MAP)` 读权限规则——与 admin 写入的 Hash **共用同一 Redis 实例与序列化配置**。

## 5. 最小验证清单

- admin 启动后 `HGETALL auth:pathResourceMap` 有数据
- 登录后 Redis 出现 `mall-swarm:ums:admin:{id}` 类 key
- 下单后 `INCR mall-swarm:oms:orderId:{日期}` 递增

## 附录：来源证据

- `BaseRedisConfig.java` L28-64
- `RedisServiceImpl.java`（全文件委托 `redisTemplate`）
- `RedisConfig.java`：admin/portal/gateway 三处
- `SaTokenConfig.java` L34、L56
- `UmsResourceServiceImpl` L85-86；`UmsAdminCacheServiceImpl` L27-41
- `UmsMemberCacheServiceImpl` L31-60；`OmsPortalOrderServiceImpl` L439-449
