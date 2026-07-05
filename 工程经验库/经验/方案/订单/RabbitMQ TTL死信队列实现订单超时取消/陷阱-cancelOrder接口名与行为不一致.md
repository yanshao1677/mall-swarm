---
title: 陷阱-cancelOrder接口名与行为不一致
level: atomic
parent: RabbitMQ TTL死信队列实现订单超时取消
status: reviewed
tags:
  - order
  - api
  - naming
  - trap
created_at: 2026-07-05
updated_at: 2026-07-05
confidence: high
related:
  - 阶段1-延迟消息发送
  - 阶段2-幂等取消执行
source_repo: macrozheng/mall-swarm
source_paths:
  - mall-portal/.../controller/OmsPortalOrderController.java
  - mall-portal/.../service/impl/OmsPortalOrderServiceImpl.java
---

# 陷阱：`cancelOrder` 接口名与行为不一致

## 本项目落地状态

| 维度 | 状态 |
|------|------|
| 代码位置 | `OmsPortalOrderController` + `OmsPortalOrderServiceImpl` |
| 默认是否启用 | ✅ 两个 POST 接口均对外暴露 |
| 易踩坑点 | ⚠️ 路径名 `cancelOrder` **不会**立即关单，仅重发延迟 MQ |

## 触发条件

- 调用方需要「取消待付款订单」或「补救超时未取消的订单」
- 误按 URL 字面语义选择接口

## 接口对照表

| HTTP 路径 | Swagger 摘要 | 实际调用 | 行为 |
|-----------|--------------|----------|------|
| `POST /order/cancelOrder` | 取消单个超时订单 | `sendDelayMessageCancelOrder(orderId)` | **重发** TTL 延迟消息，到期后由 MQ 消费执行关单 |
| `POST /order/cancelUserOrder` | 用户取消订单 | `cancelOrder(orderId)` | **立即**执行关单（status=0→4，释放库存/券/积分） |
| `POST /order/cancelTimeOutOrder` | 取消超时订单 | `cancelTimeOutOrder()` | 批量扫表取消（定时任务同款逻辑，API 触发） |

## 判定规则

```
IF 调用 POST /order/cancelOrder:
    → 仅向 TTL 队列再发一条 orderId 消息
    → 不查 status、不改库、不释放库存

IF 调用 POST /order/cancelUserOrder:
    → 执行 cancelOrder 服务方法
    → 仅 status=0 且 deleteStatus=0 时生效（幂等）

IF 需要「立刻关单」:
    → 必须用 cancelUserOrder（或等 MQ 消费，或 cancelTimeOutOrder 批量）
```

## 状态读写位置

| 接口 | 读 | 写 |
|------|----|----|
| cancelOrder（Controller） | `oms_order_setting`（读超时分钟） | RabbitMQ TTL 队列 |
| cancelUserOrder | `oms_order` status=0 | `oms_order` status=4 + 库存/券/积分回滚 |

## 正常路径 vs 误用路径

**期望：用户点「取消订单」立即关单**

1. ✅ `POST /order/cancelUserOrder?orderId={id}`
2. ❌ `POST /order/cancelOrder?orderId={id}` → 订单仍为待付款，仅多一条延迟消息

**期望：补救某单未进 MQ 或消息丢失**

1. ✅ `POST /order/cancelOrder?orderId={id}` 重发延迟消息
2. 或 ✅ `POST /order/cancelUserOrder` 若需立即关单

## 分支路径

- `cancelOrder`（Controller）多次调用 → 多条 TTL 消息；阶段 2 幂等消化，已支付单不受影响
- `cancelUserOrder` 对 status≠0 → `cancelOrder` 服务方法查无 status=0 记录，**静默 return**

## 失败处理

- 误用 `cancelOrder` 期望即时取消 → 前端仍显示待付款，**无错误响应**（`CommonResult.success`）
- MQ 不可用时调用 Controller `cancelOrder` → 发送异常，订单仍待付款

## 幂等性 / 一致性约束

- 立即取消：`cancelOrder` 服务方法以 status=0 为闸门，重复调用安全
- 延迟重发：与 [阶段1-延迟消息发送](阶段1-延迟消息发送.md) 相同，多次发送由 [阶段2-幂等取消执行](阶段2-幂等取消执行.md) 消化

## 命名改进建议（仅记录，未改代码）

| 现状 | 更清晰命名 |
|------|------------|
| `POST /order/cancelOrder` | `resendCancelDelayMessage` / `retryOrderTimeoutCancel` |
| `POST /order/cancelUserOrder` | 保持或改为 `cancelOrderImmediately` |

## 代码骨架

```java
// Controller — 注意方法名与路径的误导性
@RequestMapping(value = "/cancelOrder", method = RequestMethod.POST)
public CommonResult cancelOrder(Long orderId) {
    portalOrderService.sendDelayMessageCancelOrder(orderId);  // 非 cancelOrder 服务
    return CommonResult.success(null);
}

@RequestMapping(value = "/cancelUserOrder", method = RequestMethod.POST)
public CommonResult cancelUserOrder(Long orderId) {
    portalOrderService.cancelOrder(orderId);  // 真·关单
    return CommonResult.success(null);
}
```

## 最小验证清单

- `POST /order/cancelOrder` 后：`oms_order.status` 仍为 0，MQ 出现新 TTL 消息
- `POST /order/cancelUserOrder` 后：status=0 订单变为 4，`pms_sku_stock.lock_stock` 减少
- 对已支付单（status=1）调 `cancelUserOrder` → 无状态变更
- 前端/文档勿将 `cancelOrder` 标为「用户取消」

## 附录：来源证据

- `OmsPortalOrderController` L64-70（cancelOrder）、L92-97（cancelUserOrder）
- `OmsPortalOrderServiceImpl.sendDelayMessageCancelOrder` L328-333
- `OmsPortalOrderServiceImpl.cancelOrder` L297-324
- 深读：[T3-订单流程深读](../../../mall-swarm/深读/T3-订单流程深读.md) § 接口表
