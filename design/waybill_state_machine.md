# 运单状态机高效可靠设计方案

## 1. 设计目标
- **高效**：状态流转简单明了，易于扩展，性能优良。
- **可靠**：状态变更具备幂等性，防止重复变更，支持异常恢复。
- **可追溯**：每一次状态变更均有日志记录，便于审计和问题追踪。
- **合规**：遵循阿里巴巴Java开发手册，代码规范，易于维护。

## 2. 状态机核心原则
- 状态枚举唯一且不可变。
- 状态流转路径清晰，非法流转需拦截。
- 状态变更需持久化，支持分布式场景下的一致性。
- 状态变更操作具备幂等性。
- 所有状态变更需记录操作日志。

## 3. 运单状态流转图
```mermaid
graph TD
  "CREATED"("已创建") --> "PAID"("已支付")
  "PAID" --> "ASSIGNED"("已分配")
  "ASSIGNED" --> "PICKED_UP"("已揽件")
  "PICKED_UP" --> "IN_TRANSIT"("运输中")
  "IN_TRANSIT" --> "DELIVERED"("已送达")
  "DELIVERED" --> "SIGNED"("已签收")
  "SIGNED" --> "COMPLETED"("已完成")
  "*" --> "CANCELLED"("已取消")
```

## 4. 核心类结构设计
```java
/**
 * 运单状态枚举
 * 定义了运单从创建到完成的所有状态节点
 * 状态流转需遵循预设规则，避免非法流转
 */
public enum WaybillStatus {
    /**
     * 运单已创建
     * - 初始状态，运单信息已录入系统
     * - 此时运单已分配运单号，但未支付
     * - 可流转至：PAID, CANCELLED
     */
    CREATED,

    /**
     * 运单已支付
     * - 客户完成支付，等待分配配送人员
     * - 支付完成后进入实际配送流程
     * - 可流转至：ASSIGNED, CANCELLED
     */
    PAID,

    /**
     * 运单已分配
     * - 系统已将运单分配给具体配送员
     * - 配送员已接单但未开始取件
     * - 可流转至：PICKED_UP, CANCELLED
     */
    ASSIGNED,

    /**
     * 运单已取件
     * - 配送员已从发件人处取得包裹
     * - 标志着包裹正式进入物流环节
     * - 可流转至：IN_TRANSIT, CANCELLED
     */
    PICKED_UP,

    /**
     * 运单运输中
     * - 包裹在物流网络中转运
     * - 可能经过多个物流节点
     * - 此状态期间会有多个物流轨迹更新
     * - 可流转至：DELIVERED, CANCELLED
     */
    IN_TRANSIT,

    /**
     * 运单已送达
     * - 包裹已到达收件人地址
     * - 等待收件人签收
     * - 可流转至：SIGNED, CANCELLED
     */
    DELIVERED,

    /**
     * 运单已签收
     * - 收件人已确认签收包裹
     * - 标志着配送服务完成
     * - 可流转至：COMPLETED
     */
    SIGNED,

    /**
     * 运单已完成
     * - 运单生命周期的最终状态
     * - 所有配送流程和支付结算均已完成
     * - 终态，不可再变更
     */
    COMPLETED,

    /**
     * 运单已取消
     * - 因各种原因导致运单终止
     * - 取消原因需在日志中详细记录
     * - 可能发生退款等后续处理
     * - 终态，不可再变更
     */
    CANCELLED;
}

// 状态流转定义
public enum WaybillStatusTransition {
    CREATE_TO_PAID(CREATED, PAID),
    PAID_TO_ASSIGNED(PAID, ASSIGNED),
    ASSIGNED_TO_PICKED_UP(ASSIGNED, PICKED_UP),
    PICKED_UP_TO_IN_TRANSIT(PICKED_UP, IN_TRANSIT),
    IN_TRANSIT_TO_DELIVERED(IN_TRANSIT, DELIVERED),
    DELIVERED_TO_SIGNED(DELIVERED, SIGNED),
    SIGNED_TO_COMPLETED(SIGNED, COMPLETED),
    ANY_TO_CANCELLED(null, CANCELLED);
    // ... 构造方法与校验逻辑
}

// 运单实体
public class Waybill {
    private Long id;
    private WaybillStatus status;
    // ... 其他属性
}

// 状态机服务接口
public interface WaybillStateMachineService {
    boolean canTransfer(WaybillStatus from, WaybillStatus to);
    void transferStatus(Long waybillId, WaybillStatus to, String operator);
}
```

## 5. 状态变更流程
1. 校验当前状态与目标状态是否合法流转。
2. 幂等校验（如状态已是目标状态则直接返回成功）。
3. 持久化状态变更（建议使用数据库乐观锁或分布式锁）。
4. 记录状态变更日志（含操作人、时间、前后状态、备注等）。
5. 返回变更结果。

## 6. 幂等性与可追溯性实现
- 幂等性：每次变更前先查询当前状态，若已是目标状态则不重复操作。
- 可追溯性：设计`waybill_status_log`表，记录每一次状态变更。

```sql
CREATE TABLE waybill_status_log (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  waybill_id BIGINT NOT NULL,
  from_status VARCHAR(32),
  to_status VARCHAR(32),
  operator VARCHAR(64),
  operate_time DATETIME DEFAULT CURRENT_TIMESTAMP,
  remark VARCHAR(255)
);
```

## 7. 伪代码示例
```java
@Transactional
public void transferStatus(Long waybillId, WaybillStatus to, String operator) {
    Waybill waybill = waybillRepository.findById(waybillId);
    WaybillStatus from = waybill.getStatus();
    if (from == to) return; // 幂等
    if (!canTransfer(from, to)) throw new BizException("非法状态流转");
    // 乐观锁更新
    int updated = waybillRepository.updateStatus(waybillId, from, to);
    if (updated == 0) throw new BizException("状态已变更，请重试");
    // 记录日志
    waybillStatusLogRepository.insert(new WaybillStatusLog(waybillId, from, to, operator, ...));
}
```

## 8. 设计要点总结
- 状态机设计需前置规划所有状态与流转路径，避免后期频繁变更。
- 状态变更需保证幂等，防止并发下重复操作。
- 日志表设计需包含所有关键信息，便于追溯。
- 代码实现需遵循阿里巴巴Java开发手册，命名规范、异常处理、事务控制等均需合规。 