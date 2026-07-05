---
title: RabbitMQ TTL死信队列实现订单超时取消
level: pattern
parent:
status: reviewed
tags:
  - order
  - rabbitmq
  - ttl
  - dead-letter
created_at: 2026-07-05
updated_at: 2026-07-05
confidence: high
related:
  - 下单事务内锁库存与支付后扣减
source_repo: macrozheng/mall-swarm
source_paths:
  - mall-portal/.../config/RabbitMqConfig.java
  - mall-portal/.../component/CancelOrderSender.java
  - mall-portal/.../component/CancelOrderReceiver.java
  - mall-portal/.../component/OrderTimeOutCancelTask.java
---

# RabbitMQ TTL 死信队列实现订单超时取消

## 本项目落地状态

| 维度 | 状态 |
|------|------|
| 主路径（默认启用） | ✅ 下单后 `sendDelayMessageCancelOrder` → TTL 队列 → `cancelOrder` |
| 备用路径 | ⚠️ `OrderTimeOutCancelTask` 定时扫表代码存在，但 **`//@Component` 未启用** |
| 手动补救 | `POST /order/cancelTimeOutOrder` API 可触发批量取消 |
| API 陷阱 | ⚠️ `POST /order/cancelOrder` 仅重发 MQ；立即关单用 `cancelUserOrder` → 见陷阱原子卡 |
| 依赖 | RabbitMQ + `oms_order_setting.normal_order_overtime`（种子数据 120 分钟） |

## 1. 问题

待付款订单超时需自动关闭、释放锁定库存、恢复优惠券与积分；需单笔订单精确延迟。**本项目默认靠 MQ，不靠定时任务。**

## 2. 适用约束

- 已有 RabbitMQ
- 超时时间配置在 DB（如 `normal_order_overtime` 分钟）
- 下单与发消息在同一业务流程（下单成功后发送）
- 取消逻辑需幂等（支付后 MQ 迟到不误关）

## 3. 核心思路

下单成功后向 **TTL 队列**发送消息，设置 per-message `expiration` 毫秒；TTL 队列配置死信交换机转发到 **实际消费队列**；消费者执行幂等 `cancelOrder`（仅 status=0 生效）。

## 4. 通用结构

| 对象 | 名称示例 |
|------|----------|
| TTL Exchange | `order.direct.ttl` |
| TTL Queue | `order.cancel.ttl`（x-dead-letter-exchange/key 指向实际交换机） |
| Work Exchange | `order.direct` |
| Work Queue | `order.cancel` |
| Sender | 设置 message.expiration |
| Receiver | `@RabbitListener` + 幂等取消 |

## 5. 处理流程

1. 下单事务提交后 `sendDelayMessageCancelOrder(orderId)`
2. 读 `normalOrderOvertime`（分钟）→ `delayMs = minute * 60 * 1000`
3. `convertAndSend(ttlExchange, ttlRouteKey, orderId, setExpiration(delayMs))`
4. TTL 到期 → 死信 → work 队列
5. Receiver → `cancelOrder(orderId)`：仅 status=0 且 deleteStatus=0 时改 status=4 并回滚库存/券/积分

原子阶段见：阶段1-延迟消息发送、阶段2-幂等取消执行。

## 6. 异常处理

- MQ 发送失败：订单已创建但无延迟取消，需定时扫表补救（本项目有 `cancelTimeOutOrder` 但默认定时未启用）
- 重复消费：`cancelOrder` 查不到 status=0 则静默 return
- 已支付：status=1，取消逻辑跳过

## 7. 具体语言实现（Spring AMQP）

```java
// TTL 队列声明
@Bean
public Queue orderTtlQueue() {
    return QueueBuilder.durable("order.cancel.ttl")
        .withArgument("x-dead-letter-exchange", "order.direct")
        .withArgument("x-dead-letter-routing-key", "order.cancel")
        .build();
}

// 发送
public void sendMessage(Long orderId, long delayTimes) {
    amqpTemplate.convertAndSend("order.direct.ttl", "order.cancel.ttl", orderId,
        message -> {
            message.getMessageProperties().setExpiration(String.valueOf(delayTimes));
            return message;
        });
}

// 消费
@RabbitListener(queues = "order.cancel")
public void handle(Long orderId) {
    portalOrderService.cancelOrder(orderId);
}
```

## 8. 测试点

- 下单后 delay 内未支付 → 订单 status=4，lock_stock 释放
- 支付后 MQ 到达 → 订单保持 status=1
- 同一 orderId 重复投递 → 第二次静默无变更

## 9. 适用 / 不适用

**适用**：超时分钟数可配置、单笔精确延迟。**不适用**：无 MQ、要求秒级大量扫表即可满足（可用纯定时任务）。

## 10. 风险与反模式

- 仅依赖 MQ 无扫表兜底 → 发送失败订单永不关闭
- `cancelOrder` 不幂等 → 重复消费误操作已支付单
- per-message TTL 与 queue TTL 混用需确认 broker 行为

## 11. 标签

order, rabbitmq, ttl, dead-letter, timeout-cancel

## 附录：来源证据

- `QueueEnum`：exchange `mall.order.direct.ttl` / `mall.order.direct`，queue `mall.order.cancel.ttl` / `mall.order.cancel`
- `RabbitMqConfig.java` L48-54 死信参数
- `sendDelayMessageCancelOrder`：`normalOrderOvertime * 60 * 1000`
- SQL seed：`normal_order_overtime=120` 分钟
