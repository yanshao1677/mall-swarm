# T5：mall-search MySQL→ES 深读

> **场景**：模板场景二（模块深读）
> **生成时间**：2026-07-05
> **范围**：EsProduct 导入/创建、字段映射、搜索 DSL

---

## 1. 模块背景

`mall-search`（端口 8081）负责 Elasticsearch 商品搜索，从共享 MySQL（mall-mbg）读取商品数据，写入 ES 索引 `pms`，提供简单搜索与综合 function_score 搜索。

## 2. 核心职责

| 负责 | 不负责 |
|------|--------|
| MySQL → ES 数据同步（手动触发） | 商品 CRUD（在 mall-admin） |
| ES 简单/综合/推荐搜索 | 自动 CDC 同步 |
| 搜索聚合（品牌/分类/属性） | 订单/用户业务 |

## 3. 核心对象

| 对象 | 路径 | 作用 |
|------|------|------|
| `EsProductController` | `mall-search/.../controller/EsProductController.java` | HTTP 入口 |
| `EsProductServiceImpl` | `mall-search/.../service/impl/EsProductServiceImpl.java` | 业务编排 |
| `EsProductDao` + XML | `mall-search/.../dao/` | MySQL 组装 EsProduct |
| `EsProductRepository` | `mall-search/.../repository/EsProductRepository.java` | Spring Data ES CRUD |
| `EsProduct` | `mall-search/.../domain/EsProduct.java` | ES 文档模型 |
| `ElasticsearchTemplate` | Service 注入 | 复杂 DSL 查询 |

## 4. 内部流程

### 4.1 MySQL → ES 同步

```
POST /esProduct/importAll
  → productDao.getAllEsProductList(null)
  → productRepository.saveAll()
  → 返回导入条数

POST /esProduct/create/{id}
  → getAllEsProductList(id)
  → 有数据则 save 单条
```

**SQL 条件**（`EsProductDao.xml`）：`delete_status=0 AND publish_status=1`。

证据：`EsProductServiceImpl.java` 第 45–54、63–70 行。

### 4.2 删除

- 单删：`deleteById(id)`
- 批删：构造仅含 id 的列表 → `deleteAll`

### 4.3 简单搜索

```
GET /esProduct/search/simple?keyword=&pageNum=&pageSize=
  → findByNameOrSubTitleOrKeywords(keyword, keyword, keyword, pageable)
```

证据：`EsProductRepository.java` 第 21 行。

### 4.4 综合搜索（NativeQuery + function_score）

```
GET /esProduct/search?keyword=&brandId=&productCategoryId=&sort=
```

流程：

1. **Filter**：`brandId` / `productCategoryId` term 过滤
2. **Query**：无 keyword → `match_all`；有 keyword → `function_score`（name×10, subTitle×5, keywords×2, minScore=2.0）
3. **Sort**：0=相关度；1=新品(id desc)；2=销量；3=价格 asc；4=价格 desc；最后追加 `_score` desc
4. `elasticsearchTemplate.search()`

证据：`EsProductServiceImpl.java` 第 99–157 行。

### 4.5 商品推荐

```
GET /esProduct/recommend/{id}
  → 从 MySQL 取当前商品
  → function_score 用 name/brandId/productCategoryId，mustNot 排除自身 id
```

### 4.6 搜索关联聚合

```
GET /esProduct/search/relate
  → 聚合 brandNames、productCategoryNames、allAttrValues（nested）
```

## 5. 关键字段（EsProduct）

| 字段 | ES 映射 | 用途 |
|------|---------|------|
| `id` | `@Id` | 主键 |
| `name/subTitle/keywords` | Text + ik_max_word | 全文检索、function_score |
| `brandId/productCategoryId` | 默认 | 筛选、推荐权重 |
| `brandName/productCategoryName` | Keyword | 聚合 |
| `price/sale` | 默认 | 排序 |
| `attrValueList` | Nested | 属性筛选聚合 |
| 索引 | `@Document(indexName="pms")`, shards=1, replicas=0 | |

嵌套属性 `EsProductAttributeValue`：`type` 0=规格, 1=参数。

## 6. 对外接口

| 接口 | 方法 | 说明 |
|------|------|------|
| `/esProduct/importAll` | POST | 全量导入 |
| `/esProduct/create/{id}` | POST | 单条创建 |
| `/esProduct/delete/{id}` | GET | 单删 |
| `/esProduct/search/simple` | GET | 简单搜索 |
| `/esProduct/search` | GET | 综合搜索 |
| `/esProduct/recommend/{id}` | GET | 推荐 |
| `/esProduct/search/relate` | GET | 聚合 |

网关白名单：`/mall-search/**` 整体免登录。

## 7. 错误处理

- 创建失败（MySQL 无数据）：`CommonResult.failed()`
- ES 连接：本地 `spring.elasticsearch.uris: localhost:9200`；Nacos `mall-search-dev.yaml` 可外部化

## 8. 设计优点

- MyBatis 联表 + collection 映射嵌套属性，一次 SQL 组装完整 ES 文档
- 简单搜索用 Repository 派生，复杂搜索用 ElasticsearchTemplate，分层清晰

## 9. 设计代价

- **无自动同步**：商品变更后需手动调 `/esProduct/create/{id}` 或 `importAll`
- 网关整模块白名单，搜索/导入接口均无鉴权

## 10. 可提炼候选

| # | 候选 | 层级 |
|---|------|------|
| 1 | 手动 API 触发 MySQL→ES 同步（非 CDC） | 方案层 |
| 2 | function_score 多字段加权搜索（name×10 等） | 方案层 |
| 3 | Nested 属性聚合筛选 | 方案层 |
| 4 | 双写路径：Repository 简单 CRUD + Template 复杂 DSL | 方案层 |

## 11. 已读证据

- `mall-search/src/main/java/com/macro/mall/search/controller/EsProductController.java`
- `mall-search/src/main/java/com/macro/mall/search/service/impl/EsProductServiceImpl.java`
- `mall-search/src/main/java/com/macro/mall/search/dao/EsProductDao.java`
- `mall-search/src/main/resources/dao/EsProductDao.xml`
- `mall-search/src/main/java/com/macro/mall/search/repository/EsProductRepository.java`
- `mall-search/src/main/java/com/macro/mall/search/domain/EsProduct.java`
- `config/search/mall-search-dev.yaml`

## 12. 待深读问题

1. mall-admin 商品保存后是否调用 search 同步（本模块未见自动触发）
2. IK 分词器是否在 ES 索引中预配置
