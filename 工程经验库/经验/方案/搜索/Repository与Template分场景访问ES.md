---
title: Repository与Template分场景访问ES
level: pattern
parent:
status: reviewed
tags:
  - search
  - elasticsearch
  - repository
  - template
created_at: 2026-07-05
updated_at: 2026-07-05
confidence: high
related:
  - 手动API触发关系库到搜索引擎同步
  - function_score多字段加权商品搜索
  - Nested聚合筛选商品属性
source_repo: macrozheng/mall-swarm
source_paths:
  - mall-search/.../EsProductServiceImpl.java
  - mall-search/.../EsProductRepository.java
---

# Repository 与 Template 分场景访问 ES

## 本项目落地状态

| 维度 | 状态 |
|------|------|
| 代码位置 | `EsProductServiceImpl` 内同时使用两种客户端 |
| 索引 | `pms`（`@Document(indexName = "pms")`） |
| **非双写** | 同一文档**不会**同时经两条路径写入；按方法分工 |

## 1. 分工对照表（`EsProductServiceImpl` 逐方法）

| 方法 | 客户端 | 操作类型 | API 入口 |
|------|--------|----------|----------|
| `importAll` | `EsProductRepository.saveAll` | 写 | `POST /importAll` |
| `create` | `EsProductRepository.save` | 写 | `POST /create/{id}` |
| `delete` / `delete(List)` | `Repository.deleteById` / `deleteAll` | 写 | `GET /delete/{id}`、`POST /delete/batch` |
| `search(keyword, page…)` 单参数 | `Repository.findByNameOrSubTitleOrKeywords` | 读 | `GET /search/simple` |
| `search(…, brandId, categoryId, sort)` | `ElasticsearchTemplate.search(NativeQuery)` | 读 | `GET /search` |
| `recommend` | `ElasticsearchTemplate.search(NativeQuery)` | 读 | `GET /recommend/{id}` |
| `searchRelatedInfo` | `ElasticsearchTemplate.search(NativeQuery)` | 读 | `GET /search/relate` |

## 2. 写路径数据流（Repository）

```
MySQL（EsProductDao.getAllEsProductList）
    → List<EsProduct>
    → productRepository.save / saveAll / delete*
```

- 写前数据来自 **MyBatis 联表 SQL**，不是 Template 查询结果回写
- `EsProductRepository` 仅声明一个派生查询方法（L21），**无**自定义写逻辑

## 3. 读路径为何用 Template

带 `function_score`、bool filter、排序、Nested 聚合的查询，在 `EsProductRepository` 中**无**对应方法，由 `NativeQueryBuilder` + `elasticsearchTemplate.search` 执行（L95-157、L170-209、L216-238）。

简单关键词搜索（`/search/simple`）用 Repository 派生方法即可（L87-89）。

## 4. 边界说明（避免误读）

| 说法 | 是否符合代码 |
|------|--------------|
| 「Repository 和 Template 双写同一索引」 | ❌ 写操作**仅** Repository |
| 「复杂搜索也走 Repository」 | ❌ 综合搜索/推荐/关联信息走 Template |
| 「两条读路径会写 ES」 | ❌ Template 方法均为读 |

## 5. 最小验证清单

- `POST /importAll` 后 ES `pms` 文档数增加（Repository 写）
- `GET /search/simple?keyword=x` 有结果（Repository 读）
- `GET /search?keyword=x&sort=2` 按销量排序（Template 读，含 function_score）

## 附录：来源证据

- `EsProductServiceImpl.java` L41-43 双注入；L45-89 Repository；L93-157、L161-211、L215-239 Template
- `EsProductRepository.java` L12-21（仅 `findByNameOrSubTitleOrKeywords`）
