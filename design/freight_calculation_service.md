# 多维度运费计算服务设计方案

## 1. 系统概述

设计一个高性能、高可用的运费计算服务，支持多维度计费规则，包括重量、体积、区域、时效等因素。

## 2. 核心功能需求

- 支持多维度计费规则配置和管理
- 高性能运费计算
- 规则版本管理
- 计费历史记录和追溯
- 支持规则热更新
- 提供实时计算和批量计算能力

## 3. 技术架构设计

### 3.1 整体架构

采用微服务架构，主要包含以下服务：

1. **计费规则管理服务**
   - 负责规则的CRUD操作
   - 规则版本控制
   - 规则配置管理

2. **运费计算服务**
   - 核心计算引擎
   - 规则解析和执行
   - 计算结果缓存

3. **数据存储服务**
   - 规则持久化
   - 计算记录存储
   - 历史数据管理

### 3.2 技术选型

- **开发语言**: Java 17
- **微服务框架**: Spring Cloud Alibaba
- **服务注册与发现**: Nacos
- **配置中心**: Nacos Config
- **缓存**: Redis Cluster
- **消息队列**: RocketMQ
- **数据库**: 
  - MySQL (规则存储)
  - MongoDB (计算记录)
- **规则引擎**: Drools
- **负载均衡**: Nginx + Spring Cloud LoadBalancer
- **服务熔断与限流**: Sentinel
- **监控**: Prometheus + Grafana
- **链路追踪**: SkyWalking

## 4. 高并发设计

### 4.1 缓存策略

1. **多级缓存**
   - L1: 本地缓存 (Caffeine)
   - L2: 分布式缓存 (Redis Cluster)
   - 缓存预热机制
   - 缓存更新策略 (采用双删策略)

2. **计算结果缓存**
   - 相同参数的计算结果缓存
   - LRU淘汰策略
   - 缓存击穿保护 (Bloom Filter)

### 4.2 性能优化

1. **规则预编译**
   - 规则变更时预编译
   - 编译结果缓存
   - 热点规则优先级提升

2. **异步处理**
   - 批量计算异步化
   - 计算记录异步写入
   - 规则更新异步通知

3. **分库分表**
   - 计算记录按时间和业务维度分表
   - 读写分离
   - 分片策略优化

### 4.3 并发控制

1. **限流策略**
   - 接口级限流
   - 用户级限流
   - 服务级限流
   - 动态调整限流阈值

2. **线程池管理**
   - 核心线程池隔离
   - 动态线程池
   - 线程池监控

## 5. 高可用设计

### 5.1 服务高可用

1. **集群部署**
   - 服务多节点部署
   - 跨机房部署
   - 自动扩缩容

2. **故障转移**
   - 服务熔断
   - 服务降级
   - 快速失败策略

### 5.2 数据高可用

1. **数据存储**
   - MySQL主从复制
   - Redis Cluster
   - 数据备份策略

2. **数据一致性**
   - 最终一致性
   - 补偿机制
   - 版本控制

## 6. 核心类设计

```java
// 运费计算请求
public class FreightCalculationRequest {
    private String orderId;
    private Double weight;
    private Double volume;
    private String fromAddress;
    private String toAddress;
    private String serviceLevel;
    private Map<String, Object> extraParams;
}

// 运费计算结果
public class FreightCalculationResult {
    private String orderId;
    private BigDecimal totalAmount;
    private List<FreightDetail> details;
    private String ruleVersion;
    private Long calculateTime;
}

// 计费规则
public class FreightRule {
    private String ruleId;
    private String ruleName;
    private String ruleType;
    private String ruleContent;
    private String version;
    private Boolean isActive;
    private Date effectiveTime;
    private Date expiryTime;
}

// 计费引擎接口
public interface FreightCalculationEngine {
    FreightCalculationResult calculate(FreightCalculationRequest request);
    void refreshRules();
    void preCompileRules();
}
```

## 7. 接口设计

### 7.1 REST API

```
POST /api/v1/freight/calculate
GET /api/v1/freight/rules
POST /api/v1/freight/rules
PUT /api/v1/freight/rules/{ruleId}
DELETE /api/v1/freight/rules/{ruleId}
POST /api/v1/freight/batch-calculate
```

### 7.2 内部接口

```java
// 规则管理接口
public interface RuleManager {
    void addRule(FreightRule rule);
    void updateRule(FreightRule rule);
    void deleteRule(String ruleId);
    List<FreightRule> getActiveRules();
}

// 缓存管理接口
public interface CacheManager {
    void put(String key, Object value);
    Optional<Object> get(String key);
    void invalidate(String key);
    void preHeat(List<String> keys);
}
```

## 8. 监控告警

1. **关键指标**
   - 计算QPS
   - 计算耗时分布
   - 缓存命中率
   - 错误率
   - 规则更新频率

2. **告警策略**
   - 服务可用性告警
   - 性能阈值告警
   - 错误率告警
   - 容量告警

## 9. 部署架构

1. **环境划分**
   - 开发环境
   - 测试环境
   - 预发环境
   - 生产环境

2. **容器化部署**
   - 使用Kubernetes编排
   - 服务网格化
   - 自动化部署流程

## 10. 扩展性考虑

1. **规则扩展**
   - 支持插件式规则扩展
   - 规则模板机制
   - 规则组合能力

2. **业务扩展**
   - 支持新的计费维度
   - 支持新的业务场景
   - 支持新的计算模式

## 11. 安全性设计

1. **接口安全**
   - 接口鉴权
   - 参数加密
   - 防重放攻击

2. **数据安全**
   - 敏感数据加密
   - 操作审计
   - 访问控制 