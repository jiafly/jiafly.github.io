# 安全工程

> 不是"OWASP Top 10 背一遍"。这一节讲：**Spring Security 怎么不留坑、JWT 真正的风险、密码学常见误用**。

---

## 已完成的深文

- ✅ **[Web 安全实战 —— OAuth2 / JWT / HTTPS / OWASP 全防御](/article/security/security-deep)** · 1863 行
- ✅ **[Spring Security 6 深入 —— 过滤器链、OAuth2 RS、方法级安全](/article/security/spring-security)** · 1784 行

---

## 规划中的深文

### 2. Spring Security 6 深入 ✅ 已完成 → [阅读深文](/article/security/spring-security)
**核心问题**：Spring Security 6 的过滤器链是怎么编排的？`SecurityFilterChain` 与传统配置的区别？方法级安全（@PreAuthorize）的代价？

### 3. WebAuthn / Passkey 实战
**核心问题**：FIDO2 凭什么取代密码？WebAuthn 的工程化集成？Passkey 跨设备同步？

### 4. mTLS 与零信任架构
**核心问题**：服务间 mTLS 怎么不漏？SPIFFE / SPIRE 自动证书？BeyondCorp 模型？

### 5. SBOM 与供应链安全
**核心问题**：CycloneDX vs SPDX？依赖图扫描工具（Snyk / Dependabot / Trivy）选型？签名与制品溯源（SLSA / Sigstore）？

### 6. 密码学库选型与 KMS
**核心问题**：Bouncy Castle vs JDK 内置？AWS KMS / Vault / 阿里 KMS 选型？字段级加密 vs 全表加密？

### 7. DAST / SAST / IAST 工具链
**核心问题**：三种安全测试的本质差别？SonarQube + 安全规则？Burp Suite / OWASP ZAP 在 CI 中的集成？

---

## 关联现有文章
- [API 设计](/article/architecture/api-design)
- [HTTP/1→2→3 演进](/article/network/http-evolution)
- [DevOps CI/CD](/article/devops/cicd)

[告诉我](/article/about) 你最想先看哪个主题。
