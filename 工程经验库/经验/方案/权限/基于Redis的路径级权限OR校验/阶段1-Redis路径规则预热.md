---
title: 阶段1-Redis路径规则预热
level: atomic
parent: 基于Redis的路径级权限OR校验
status: reviewed
tags:
  - permission
  - redis
  - startup
created_at: 2026-07-05
updated_at: 2026-07-05
confidence: high
related:
  - 阶段2-网关Ant路径OR匹配
  - 补充-initPathResourceMap调用时机清单
source_repo: macrozheng/mall-swarm
source_paths:
  - mall-admin/.../PathResourceRulesHolder.java
  - mall-admin/.../UmsResourceServiceImpl.java
---

# 阶段 1：Redis 路径规则预热

## 本项目落地状态

| 维度 | 状态 |
|------|------|
| 代码位置 | `mall-admin/PathResourceRulesHolder` + `UmsResourceServiceImpl` |
| 默认是否启用 | ✅ admin 启动时 `@PostConstruct`；资源 CRUD 后也会刷新 |

## 触发条件

- 权限管理服务（管理后台）Spring 容器启动完成
- `PathResourceRulesHolder` Bean 已注入 `UmsResourceService`

## 输入字段

| 字段 | 类型 | 来源 |
|------|------|------|
| applicationName | String | `spring.application.name`（如 `mall-admin`） |
| resourceList | List\<Resource\> | `ums_resource` 全表 |
| resource.url | String | 资源 Ant 路径（如 `/brand/**`） |
| resource.id | Long | 资源 ID |
| resource.name | String | 资源名称 |

## 判定规则

```
pathKey   = "/" + applicationName + resource.url
pathValue = resource.id + ":" + resource.name
```

示例：`/mall-admin/brand/**` → `1:商品品牌管理`

Redis：
- key（Hash 名）= `auth:pathResourceMap`（常量 PATH_RESOURCE_MAP）
- 操作顺序：**先 DEL 再 HSETALL**（全量替换，非增量）

## 状态读写位置

- 写：Redis Hash `auth:pathResourceMap`
- 读：本阶段无读

## 正常路径

1. `@PostConstruct initPathResourceMap()` 调用 Service
2. `selectByExample` 查全部资源
3. 循环构建 TreeMap
4. `redisService.del(PATH_RESOURCE_MAP)`
5. `redisService.hSetAll(PATH_RESOURCE_MAP, pathResourceMap)`
6. 返回 map（可供管理 API 展示）

## 分支路径

- resourceList 为空 → 仍 DEL + HSETALL 空 map（网关则无任何路径需权限）
- 资源 CRUD 后 → 同逻辑再次调用 initPathResourceMap（非仅启动一次）

## 失败处理

- Redis 连接失败 → 启动异常中断（无本地降级）
- 证据不足：单条 resource.url 为空时的 key 形态需实现时确认

## 幂等性 / 一致性约束

- 全量替换：并发两次刷新后者覆盖前者，最终一致
- 网关依赖此 Hash；刷新窗口内可能短暂空 map（DEL 与 HSETALL 之间）

## 代码骨架

```java
@Component
public class PathResourceRulesHolder {
    @Autowired private UmsResourceService resourceService;
    @PostConstruct
    public void initPathResourceMap() {
        resourceService.initPathResourceMap();
    }
}

public Map<String, String> initPathResourceMap() {
    Map<String, String> map = new TreeMap<>();
    for (UmsResource r : resourceMapper.selectByExample(new UmsResourceExample())) {
        map.put("/" + applicationName + r.getUrl(), r.getId() + ":" + r.getName());
    }
    redisService.del(AuthConstant.PATH_RESOURCE_MAP);
    redisService.hSetAll(AuthConstant.PATH_RESOURCE_MAP, map);
    return map;
}
```

## 最小验证清单

- 启动后 Redis `HGETALL auth:pathResourceMap` 条数 = ums_resource 表行数
- key 含 `/mall-admin` 前缀 + 资源 url
- 资源新增后调 init，新 key 出现

## 附录：来源证据

- `PathResourceRulesHolder.java` L19-21
- `UmsResourceServiceImpl.initPathResourceMap` L79-87
