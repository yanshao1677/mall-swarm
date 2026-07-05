---
title: function_score多字段加权商品搜索
level: pattern
parent:
status: reviewed
tags:
  - search
  - elasticsearch
  - function-score
created_at: 2026-07-05
updated_at: 2026-07-05
confidence: high
related:
  - 手动API触发关系库到搜索引擎同步
  - Nested聚合筛选商品属性
source_repo: macrozheng/mall-swarm
source_paths:
  - mall-search/.../EsProductServiceImpl.java
---

# function_score 多字段加权商品搜索

## 本项目落地状态

| 维度 | 状态 |
|------|------|
| 代码位置 | `mall-search/EsProductServiceImpl.search`（综合搜索）、`recommend`（商品推荐） |
| 默认是否启用 | ✅ `GET /esProduct/search`；✅ `GET /esProduct/recommend/{id}` |
| 前置 | 索引需先手动 import/create |

## 1. 问题

关键词搜索需让标题匹配优先于副标题、关键字，并支持品牌/分类筛选与销量价格排序。

## 2. 核心思路

无 keyword 时 `match_all`；有 keyword 时 `function_score` 三 filter 加权：name×10.0、subTitle×5.0、keywords×2.0，`scoreMode=Sum`，`minScore=2.0`。Filter 层 term 过滤 brandId/productCategoryId。sort 参数：1=id desc，2=sale desc，3=price asc，4=price desc，最后追加 `_score` desc。

## 3. 具体 DSL 构建（Java API）

```java
functionScoreList.add(match("name", keyword).weight(10.0));
functionScoreList.add(match("subTitle", keyword).weight(5.0));
functionScoreList.add(match("keywords", keyword).weight(2.0));
QueryBuilders.functionScore()
    .functions(functionScoreList)
    .scoreMode(FunctionScoreMode.Sum)
    .minScore(2.0);
```

## 4. 商品推荐场景（`recommend` 方法，源码 L161-211）

**API**：`GET /esProduct/recommend/{id}`

与综合搜索**共用** `function_score` + `scoreMode=Sum` + `minScore=2.0`，但权重与条件不同：

| filter 字段 | weight |
|-------------|--------|
| name（取自当前商品 name） | 8.0 |
| subTitle | 2.0 |
| keywords | 2.0 |
| brandId（当前商品品牌） | 5.0 |
| productCategoryId（当前商品分类） | 3.0 |

额外 filter：`mustNot term id = {当前商品id}`（排除自身）。

无 sort 参数；**不使用** Nested 聚合（见 [Nested聚合筛选商品属性](Nested聚合筛选商品属性.md)）。

## 5. 测试点

- 标题含关键词排在仅关键字含的前面
- minScore=2.0 过滤弱相关
- sort=3 价格升序生效
- recommend 结果不含当前 id 商品

## 6. 标签

search, elasticsearch, function-score, ranking

## 附录：来源证据

- `EsProductServiceImpl.search` L99-149
- `EsProductServiceImpl.recommend` L161-211
- 综合搜索权重 10.0 / 5.0 / 2.0；推荐权重 8.0 / 2.0 / 2.0 / 5.0 / 3.0；minScore 均为 2.0
