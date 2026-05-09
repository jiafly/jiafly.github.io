# 架构设计

> 不是"四层 vs 三层"的 PPT 架构。这一节讲的是**怎么在团队里真正落地 DDD、API 怎么设计才不留坑**。

---

## 已完成的深文

- ✅ **[DDD 战术与战略设计 —— Aggregate 到 Bounded Context](/article/architecture/ddd)** · 1974 行
- ✅ **[API 设计与 OpenAPI](/article/architecture/api-design)** · 2024 行
- ✅ **[Hexagonal/Clean/Onion 架构对比](/article/architecture/hexagonal)** · 1917 行
- ✅ **[事件驱动架构（EDA）与 Event Sourcing](/article/architecture/eda)** · 1915 行
- ✅ **[微服务拆分演进 —— 单体到 Strangler、Cell-based](/article/architecture/microservices)** · 1951 行

---

## 规划中的深文

### 3. Hexagonal / Clean / Onion 架构对比 ✅ 已完成 → [阅读深文](/article/architecture/hexagonal)
**核心问题**：六边形架构和 DDD 是什么关系？Clean Architecture 真的能落地吗？端口与适配器的工程化？

### 4. EDA 与 Event Sourcing ✅ 已完成 → [阅读深文](/article/architecture/eda)
**核心问题**：事件驱动 vs 请求响应的本质边界？事件溯源（Event Sourcing）的工程化代价？事件版本管理？

### 5. CQRS 实战
**核心问题**：CQRS 什么时候真的需要？读写分离的存储选型？最终一致 vs 强一致？

### 6. 微服务边界与拆分 ✅ 已完成 → [阅读深文](/article/architecture/microservices)
**核心问题**：单体 → 微服务 → 服务网格的演进路径？什么时候拆、什么时候合？Strangler Pattern？

### 7. BFF 与 GraphQL Federation
**核心问题**：每个前端配一个 BFF 的成本？GraphQL Federation 与 BFF 的边界？

---

## 关联现有文章
- [分布式事务](/article/distributed/dist-tx) — Saga 与 DDD 聚合的关系
- [系统设计主题](/article/system-design/)
- [HTTP/1→2→3 演进](/article/network/http-evolution)

[告诉我](/article/about) 你最想先看哪个主题。
