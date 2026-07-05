---
title: 手动API触发关系库到搜索引擎同步
level: pattern
parent:
status: reviewed
tags:
  - search
  - elasticsearch
  - sync
created_at: 2026-07-05
updated_at: 2026-07-05
confidence: high
related:
  - 逻辑微服务与共享单库数据访问
  - Repository与Template分场景访问ES
source_repo: macrozheng/mall-swarm
source_paths:
  - mall-search/.../EsProductServiceImpl.java
  - mall-search/.../dao/EsProductDao.xml
---

# 手动 API 触发关系库到搜索引擎同步

## 本项目落地状态

| 维度 | 状态 |
|------|------|
| 代码位置 | `mall-search/EsProductController`：`importAll`、`create/{id}` |
| 默认是否启用 | ✅ 搜索 API 可用 |
| **未接线** | `mall-admin/PmsProductController` **无** 调用 search 同步；上架/改商品 **不会** 自动进 ES |
| 运维动作 | 需人工或脚本调 `POST /esProduct/create/{id}` 或 `importAll` |
| 网关 | `/mall-search/**` 整模块白名单，导入接口无鉴权 |

## 1. 问题

商品主数据在 MySQL，搜索在 ES；无 CDC 时需在商品变更后把可读副本写入索引。

## 2. 核心思路

搜索服务通过 MyBatis 联表查询组装 `EsProduct` 文档，`EsProductRepository.save/saveAll` 写入索引 `pms`。提供 `POST /importAll` 全量与 `POST /create/{id}` 单条 API，由运营或后台在商品上架后手动触发。

## 3. SQL 过滤条件

`delete_status=0 AND publish_status=1`（仅上架未删商品进索引）。

## 4. 处理流程

importAll：查全量 → saveAll → 返回计数。create(id)：按 id 查 → 有则 save 单条 → 无则 failed。

## 5. 风险

商品更新后未调 create → ES 陈旧；删除商品需调 delete API。

## 6. 标签

search, elasticsearch, manual-sync

## 附录：来源证据

- `EsProductServiceImpl.importAll` L45-54
- `EsProductServiceImpl.create` L63-70
- `EsProductDao.xml` 条件 delete_status/publish_status
- `@Document(indexName="pms")`
