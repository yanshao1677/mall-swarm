---
title: Nested聚合筛选商品属性
level: pattern
parent:
status: reviewed
tags:
  - search
  - elasticsearch
  - nested
  - aggregation
created_at: 2026-07-05
updated_at: 2026-07-05
confidence: high
related:
  - function_score多字段加权商品搜索
  - 手动API触发关系库到搜索引擎同步
source_repo: macrozheng/mall-swarm
source_paths:
  - mall-search/.../EsProductServiceImpl.java
  - mall-search/.../EsProduct.java
  - mall-search/.../EsProductController.java
---

# Nested 聚合筛选商品属性

## 本项目落地状态

| 维度 | 状态 |
|------|------|
| 代码位置 | `EsProductServiceImpl.searchRelatedInfo` |
| API | ✅ `GET /esProduct/search/relate`（网关白名单 `/mall-search/**`） |
| 索引字段 | `EsProduct.attrValueList`，`@Field(type = Nested)` |
| 未使用 Nested 的接口 | `search`、`recommend` 仅用 function_score，**无** Nested 聚合 |

## 1. 问题

关键词搜索后，前端需要展示可筛选的**品牌、分类、商品属性**列表；属性在文档中是嵌套数组 `attrValueList`，需 Nested 聚合避免跨商品属性错配。

## 2. 查询结构（源码 L215-239）

**搜索条件**：

- keyword 为空 → `match_all`
- keyword 非空 → `multi_match` 字段 `name, subTitle, keywords`

**三类聚合**：

| 聚合名 | 类型 | 字段 |
|--------|------|------|
| brandNames | terms | `brandName`，size=10 |
| productCategoryNames | terms | `productCategoryName`，size=10 |
| allAttrValues | **nested** path=`attrValueList` | 见下 |

**Nested 子聚合链**（源码 L228-234）：

```
nested(path: "attrValueList")
  └─ filter: attrValueList.type = "1"     // 仅销售属性，排除 type=0
       └─ terms: attrValueList.productAttributeId (size=10)
            ├─ terms: attrValueList.value (size=10)
            └─ terms: attrValueList.name (size=10)
```

## 3. 结果解析（源码 L245-285）

`convertProductRelatedInfo` 从 `SearchHits` 聚合结果组装 `EsProductRelatedInfo`：

- `brandNames` / `productCategoryNames` → `List<String>`
- `allAttrValues` → 遍历 `attrIds` buckets，取 `attrValues`、`attrNames` → `List<ProductAttr>`

## 4. 与 recommend 的边界

| 方法 | API | 是否用 Nested |
|------|-----|---------------|
| `searchRelatedInfo` | `/search/relate` | ✅ |
| `recommend` | `/recommend/{id}` | ❌（function_score，见 function_score 方案卡） |

## 5. 前置条件

- 商品文档已 import/create 进 ES，且 `attrValueList` 有数据
- 索引 mapping 中 `attrValueList` 为 nested 类型（`EsProduct.java` L45-46）

## 6. 最小验证清单

- `GET /esProduct/search/relate?keyword=手机` 返回 brandNames、productCategoryNames、productAttrs
- productAttrs 中 attrId 对应多个 attrValues
- type=0 的属性不出现在聚合结果（filter type=1）

## 附录：来源证据

- `EsProductServiceImpl.searchRelatedInfo` L215-239
- `EsProductServiceImpl.convertProductRelatedInfo` L245-285
- `EsProduct.java` L45-46
- `EsProductController` L101-106
