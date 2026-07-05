# T8：Nacos 配置缺口核实

> **场景**：配置核查（总纲 T8）
> **生成时间**：2026-07-05
> **范围**：config/ 目录与各服务 application-dev/prod.yml import 对照

---

## 1. 核查方法

对比 `config/` 目录现有 YAML 与各服务 `application-dev.yml` / `application-prod.yml` 中 `spring.config.import` 的 nacos DataId 引用。

## 2. config/ 目录现有文件（10 个）

| 子目录 | 文件 |
|--------|------|
| `config/admin/` | `mall-admin-dev.yaml`, `mall-admin-prod.yaml` |
| `config/demo/` | `mall-demo-dev.yaml`, `mall-demo-prod.yaml` |
| `config/gateway/` | `mall-gateway-dev.yaml`, `mall-gateway-prod.yaml` |
| `config/portal/` | `mall-portal-dev.yaml`, `mall-portal-prod.yaml` |
| `config/search/` | `mall-search-dev.yaml`, `mall-search-prod.yaml` |

**不存在**：`config/auth/`、`config/monitor/`。

## 3. 各服务 import 对照表

| 服务 | dev import DataId | config/ 模板 | prod import | 状态 |
|------|-------------------|--------------|-------------|------|
| mall-admin | `mall-admin-dev.yaml` | ✅ | `mall-admin-prod.yaml` | ✅ 完整 |
| mall-gateway | `mall-gateway-dev.yaml` | ✅ | `mall-gateway-prod.yaml` | ✅ 完整 |
| mall-portal | `mall-portal-dev.yaml` | ✅ | `mall-portal-prod.yaml` | ✅ 完整 |
| mall-search | `mall-search-dev.yaml` | ✅ | `mall-search-prod.yaml` | ✅ 完整 |
| mall-demo | `mall-demo-dev.yaml` | ✅ | `mall-demo-prod.yaml` | ✅ 完整 |
| **mall-auth** | **`mall-auth-dev.yaml`** | **❌ 缺失** | **`mall-auth-prod.yaml`** | **❌ 缺失** |
| mall-monitor | （无 import） | ❌ 无 | （无 import） | 未接入 Nacos Config |

## 4. 缺失 / 不一致详情

### 4.1 缺失的配置文件（被 import 引用但 config/ 无模板）

| DataId | 引用位置 |
|--------|----------|
| `mall-auth-dev.yaml` | `mall-auth/src/main/resources/application-dev.yml` 第 11 行 |
| `mall-auth-prod.yaml` | `mall-auth/src/main/resources/application-prod.yml` 第 11 行 |

**影响**：mall-auth 启动 dev/prod profile 时会尝试从 Nacos 拉取不存在的 DataId。行为取决于 Nacos 配置（可能告警继续，或启动失败）——**未运行时确认**。

mall-auth 本地 `application.yml` 仅有端口 8401、springdoc 等基础配置，无数据源。

### 4.2 未接入 Nacos Config 的服务

**mall-monitor**：

- `application-dev.yml` 仅有 `nacos.discovery`，无 `config.import`
- 配置全在本地 `application.yml`（端口 8101、security 账号 macro:123456）
- `config/` 无 monitor 模板

### 4.3 Nacos 地址格式不一致

| 服务 | dev server-addr |
|------|-----------------|
| admin/demo/search/portal/gateway/auth | `localhost:8848` |
| **mall-monitor** | `http://localhost:8848`（带 http:// 前缀） |

证据：`mall-monitor/application-dev.yml` 第 5 行 vs 其他服务。

### 4.4 配置内容差异（非错误）

- `config/search/mall-search-dev.yaml` 数据源 URL 无 `useSSL=false`
- admin dev 配置含 Redis/MinIO/OSS；search dev 仅 datasource + elasticsearch + logstash
- 属业务差异，非缺口

## 5. 建议处理（仅记录，本次不修改代码）

| 缺口 | 建议 |
|------|------|
| mall-auth-dev/prod.yaml | 在 `config/auth/` 新增空模板或最小配置（logging 等），或从 application-dev.yml 移除 import |
| mall-monitor | 可选：新增 `config/monitor/` 并将 security 等外部化；或保持本地配置并统一 server-addr 格式 |
| monitor server-addr | 去掉 `http://` 前缀与其他服务一致 |

## 6. 已确认事实

1. 5 个服务（admin/gateway/portal/search/demo）的 dev+prod 配置在 `config/` 均有对应模板
2. mall-auth 引用 2 个不存在的 DataId
3. mall-monitor 仅用 Nacos Discovery，不用 Nacos Config
4. mall-auth 无数据库依赖，缺失的远程配置可能影响较小（仅 logging 等）

## 7. 证据列表

- `config/` 目录 glob（10 个 yaml）
- `mall-auth/src/main/resources/application-dev.yml`
- `mall-auth/src/main/resources/application-prod.yml`
- `mall-monitor/src/main/resources/application-dev.yml`
- `mall-monitor/src/main/resources/application-prod.yml`
- 各服务 `application-dev.yml` 的 `spring.config.import` 行
