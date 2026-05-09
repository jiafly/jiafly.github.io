# DevOps 与工程实践

> 不是"DevOps 文化"鸡汤。这一节讲：**Git 怎么不出事、CI/CD 怎么不慢、镜像怎么不膨胀、灰度怎么真的灰**。

---

## 已完成的深文

- ✅ **[Git 工作流深入 —— 对象模型到 Trunk-based 实战](/article/devops/git-workflow)** · 1724 行 · 4 种 Git 对象 + 工作流对比 + 9 个事故复盘
- ✅ **[DevOps 与 CI/CD 流水线 —— Jenkins 到 GitOps 实战](/article/devops/cicd)** · 1828 行 · GitHub Actions / ArgoCD / 9 个事故 + Helm vs Kustomize

---

## 规划中的深文

### 3. Docker 镜像优化与多阶段构建
**核心问题**：BuildKit cache mount 的真实效果？distroless / chainguard 镜像选型？层级排序的玄学？

### 4. Helm vs Kustomize 实战对比
**核心问题**：Helm chart 写得好和写得烂的差别？Kustomize overlay 的复杂度边界？两者怎么混用？

### 5. Feature Flag 系统设计
**核心问题**：自研 vs LaunchDarkly / Unleash？灰度配置爆炸？长期 Flag 的清理？

### 6. SLSA 供应链安全
**核心问题**：SBOM 怎么生成？依赖漏洞自动扫描？签名与制品溯源？

### 7. DORA 指标与 DevOps 度量
**核心问题**：4 个 DORA 指标怎么真的采集？低 / 中 / 高 / 精英团队的差距？

---

## 关联现有文章
- [Kubernetes 实战](/article/k8s/k8s-essentials)
- [单元测试与 Mock](/article/testing/unit-testing)
- [安全实战](/article/security/security-deep) — 流水线里的安全

[告诉我](/article/about) 你最想先看哪个主题。
