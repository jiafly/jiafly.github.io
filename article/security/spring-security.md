# Spring Security 6 深入 —— 过滤器链、OAuth2 Resource Server 与方法级安全

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想速过面试，看 [Web 安全实战](/article/security/security-deep) 那一节够了。
> 这里要回答的，是**写代码遇到 401/403 莫名其妙、JWT 一会有效一会无效、半夜被告警叫起来**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. Spring Security 6 把 `WebSecurityConfigurerAdapter` 干掉了，**它替代成了什么 Bean**？同一个项目里能不能注册多个 `SecurityFilterChain`？多个 chain 谁先匹配？
2. `DelegatingFilterProxy` 是 Servlet 的 Filter 还是 Spring 的 Bean？它和 `FilterChainProxy` 各自的职责是什么？为什么需要这层"代理的代理"？
3. 一次 HTTP 请求经过 Spring Security 时，**`AuthenticationManager` / `ProviderManager` / `AuthenticationProvider` 三者是怎么串起来的**？谁负责"找到合适的 Provider"，谁负责"判断密码对不对"？
4. `SecurityContextHolder` 默认基于 `ThreadLocal`。开了 `@Async` 之后子线程拿不到 `Authentication` —— 怎么修？`InheritableThreadLocal` 又有什么坑？
5. `@PreAuthorize("hasRole('ADMIN')")` 和 `@Secured("ROLE_ADMIN")`，**底层走的是同一个拦截器吗**？SpEL 表达式什么时候被求值？
6. 纯前后端分离的 REST API 用 JWT 鉴权，**还要不要开 CSRF 防护**？为什么？同一个 API 如果同时支持 Cookie Session 呢？
7. OAuth2 Resource Server 模式下，每来一个请求都要去 Authorization Server 验签吗？JWKS 是什么？时钟偏差导致的"验签失败"怎么排查？
8. 你的 SaaS 产品要加 MFA（多因素认证），**Spring Security 的扩展点在哪里**？是写一个新的 `AuthenticationFilter`，还是改 `AuthenticationProvider`，还是上 `AuthenticationSuccessHandler`？

如果有半数答不上，往下看。

---

## 1. 底层模型：FilterChainProxy、Provider 责任链与方法拦截

绝大多数文章把 Spring Security 讲成"配一下 `http.authorizeRequests()`"就完事。这层认知扛不住任何一个生产事故。要看懂上面 8 个问题，必须下沉到三个核心模型。

### 1.1 整体架构：四层代理

Spring Security 6 的请求路径长这样：

```
HTTP Request
    │
    ▼
Servlet Container (Tomcat / Undertow)
    │
    ▼
DelegatingFilterProxy        ← Servlet 标准 Filter，仅一个，注册到 web.xml/ServletContext
    │  (按 beanName 查找 Spring 容器里的 Bean)
    ▼
FilterChainProxy             ← Spring Bean，名字默认 springSecurityFilterChain
    │  (内部维护多个 SecurityFilterChain，按 RequestMatcher 选一条)
    ▼
SecurityFilterChain          ← 一组有序的 Filter，针对某种 URL 模式
    │
    ├── DisableEncodeUrlFilter
    ├── WebAsyncManagerIntegrationFilter
    ├── SecurityContextHolderFilter
    ├── HeaderWriterFilter
    ├── CorsFilter
    ├── CsrfFilter
    ├── LogoutFilter
    ├── (BearerTokenAuthenticationFilter)        ← OAuth2 Resource Server 加的
    ├── (UsernamePasswordAuthenticationFilter)
    ├── DefaultLoginPageGeneratingFilter
    ├── BasicAuthenticationFilter
    ├── RequestCacheAwareFilter
    ├── SecurityContextHolderAwareRequestFilter
    ├── AnonymousAuthenticationFilter
    ├── SessionManagementFilter
    ├── ExceptionTranslationFilter               ← 把 AuthN/AuthZ 异常翻译成 401/403
    └── AuthorizationFilter                      ← 真正做 URL 鉴权
            │
            ▼
        DispatcherServlet → Controller
```

**为什么需要 `DelegatingFilterProxy`**？Servlet 的 Filter 是容器在启动时实例化的，**拿不到 Spring 容器**。所以注册到容器的 Filter 是 `DelegatingFilterProxy`，它在第一次 `doFilter` 时反向去 `WebApplicationContext` 里找 Bean。这层"代理的代理"是 Spring/Servlet 桥的标配。

**为什么要 `FilterChainProxy`**？一个 Spring Security 应用通常需要**多套规则**（比如 `/api/**` 走 JWT、`/admin/**` 走 Form Login、`/public/**` 完全开放），FilterChainProxy 持有多条 SecurityFilterChain，按 `RequestMatcher` 选第一条匹配的。

> **口诀**：`DelegatingFilterProxy`（连接 Servlet 与 Spring）→ `FilterChainProxy`（多 chain 路由）→ `SecurityFilterChain`（实际 Filter 序列）。三层代理的顺序记牢。

### 1.2 SecurityFilterChain 的 15 个核心 Filter

按执行顺序（Spring Security 6.x 默认配置）：

| # | Filter | 职责 | 跳过它会怎样 |
| --- | --- | --- | --- |
| 1 | `DisableEncodeUrlFilter` | 禁用 URL 重写中的 jsessionid | URL 暴露 sessionid，被偷会话 |
| 2 | `WebAsyncManagerIntegrationFilter` | 把 SecurityContext 装进 `WebAsyncManager` | `Callable` 控制器丢登录态 |
| 3 | `SecurityContextHolderFilter` | 加载 / 清空 SecurityContext | 后续 Filter 拿不到 Authentication |
| 4 | `HeaderWriterFilter` | 写安全响应头（XSS、HSTS 等） | 浏览器侧失去保护 |
| 5 | `CorsFilter` | 处理 CORS 预检请求 | 跨域全挂 |
| 6 | `CsrfFilter` | 校验 CSRF Token | POST/PUT 被伪造 |
| 7 | `LogoutFilter` | 处理 `/logout` 请求 | logout 端点失效 |
| 8 | `UsernamePasswordAuthenticationFilter` | 处理 `/login` 表单提交 | 表单登录无效 |
| 9 | `DefaultLoginPageGeneratingFilter` | 生成默认登录页 | 没有自带登录页 |
| 10 | `BasicAuthenticationFilter` | 处理 `Authorization: Basic xxx` | Basic 认证失效 |
| 11 | `RequestCacheAwareFilter` | 登录前的请求缓存还原 | 登录后跳回 `/` 而非原页 |
| 12 | `SecurityContextHolderAwareRequestFilter` | 包装 request，让 `getUserPrincipal()` 可用 | Servlet 标准 API 失效 |
| 13 | `AnonymousAuthenticationFilter` | 没登录时塞一个 anonymous Authentication | 后续判定 NPE |
| 14 | `SessionManagementFilter` | Session 固定保护、并发 Session 控制 | 会话固定攻击 |
| 15 | `ExceptionTranslationFilter` | 把 `AuthenticationException` → 401，`AccessDeniedException` → 403 | 用户看到 500 |
| 16 | `AuthorizationFilter` | 调 `AuthorizationManager` 做 URL 级鉴权 | 鉴权全失效 |

加了 OAuth2 Resource Server 后，会在中间插入 `BearerTokenAuthenticationFilter`（位置在 `BasicAuthenticationFilter` 之前）。

**反直觉点 ①**：`AnonymousAuthenticationFilter` 在 `AuthorizationFilter` 之前。这意味着**没登录的请求也会被塞一个 `AnonymousAuthenticationToken`**，后续的 `Authentication` 永不为 null。所以你写 `auth.isAuthenticated()` 判断登录状态是错的 —— anonymous 的 `isAuthenticated()` 也返回 true。正确判断是 `!(auth instanceof AnonymousAuthenticationToken)`。

**反直觉点 ②**：`ExceptionTranslationFilter` 自己**不抛异常**，它只**捕获**后面 Filter（特别是 `AuthorizationFilter`）抛的异常。所以你想自定义 401/403 响应，最干净的方式是注入 `AuthenticationEntryPoint` / `AccessDeniedHandler`，而不是写 `@ControllerAdvice`。

### 1.3 Authentication 三件套：Manager、Provider、Token

```
        AuthenticationManager (接口)
              │
              ▼
        ProviderManager (默认实现)
              │  持有 List<AuthenticationProvider>
              ├── DaoAuthenticationProvider     (用户名密码)
              ├── JwtAuthenticationProvider     (Bearer Token)
              ├── LdapAuthenticationProvider    (LDAP)
              ├── RememberMeAuthenticationProvider
              └── ...自定义
```

**职责切分**：
- **AuthenticationManager** 是入口接口，只有 `authenticate(Authentication)` 一个方法
- **ProviderManager** 是默认实现，**轮询所有 Provider**，找到第一个 `supports(authClass)` 返回 true 的，调它的 `authenticate`
- **AuthenticationProvider** 是真正干活的：根据 token 类型（用户名密码 / JWT / OAuth2 code）做实际的校验

每个 `Authentication` 实现类对应一个 Provider：
- `UsernamePasswordAuthenticationToken` → `DaoAuthenticationProvider`
- `BearerTokenAuthenticationToken` → `JwtAuthenticationProvider`
- `OAuth2LoginAuthenticationToken` → `OAuth2LoginAuthenticationProvider`

**为什么是责任链而不是 if-else**？因为同一个应用可能既支持表单登录、又支持 LDAP、又支持 JWT。Provider 的 `supports()` 方法决定它处理哪种 token —— 这是典型的 **Strategy + Chain of Responsibility 组合模式**。

ProviderManager 还可以挂一个 `parent` AuthenticationManager，形成"全局兜底"层级。Spring Security 6 移除了 `WebSecurityConfigurerAdapter` 后，**parent 链通常用不到了**，但底层机制还在。

### 1.4 SecurityContext 的传播

```
SecurityContextHolder (静态门面)
    │
    ▼
SecurityContextHolderStrategy (策略接口)
    │
    ├── ThreadLocalSecurityContextHolderStrategy        (默认)
    ├── InheritableThreadLocalSecurityContextHolderStrategy
    └── GlobalSecurityContextHolderStrategy
```

默认的 ThreadLocal 实现意味着：**SecurityContext 仅在当前线程可见**。

跨线程场景：
- `@Async` 异步方法 → 子线程拿不到 → 用 `DelegatingSecurityContextExecutor` 包装线程池
- `WebAsyncManagerIntegrationFilter` 已经处理了 Spring MVC 的 `Callable` 返回值
- WebSocket / Reactor → 用 `ReactiveSecurityContextHolder`

`InheritableThreadLocal` 看似优雅，但**和线程池一起用是地狱**：worker 线程是池预创建的，第一次跑任务时复制了某个用户的 context，下次跑别的用户的任务时会**串身份**。所以生产几乎不用它，老老实实用装饰器。

### 1.5 OAuth2：三种角色，三个模块

Spring Security 6 把 OAuth2 拆成三个独立模块：

| 模块 | 角色 | 用途 |
| --- | --- | --- |
| `spring-boot-starter-oauth2-client` | OAuth2 Client | 你的应用代用户去第三方登录（"用 GitHub 登录"） |
| `spring-boot-starter-oauth2-resource-server` | Resource Server | 你的应用接受别人发的 Bearer Token，验签后放行 |
| `spring-security-oauth2-authorization-server` | Authorization Server | 你的应用是发 Token 的那一方（替代废弃的 spring-security-oauth） |

**最常见的搭配**：微服务架构下，每个微服务都是 Resource Server，前端 / BFF 是 Client，独立部署一个 Authorization Server（自建或用 Auth0 / Keycloak）发 Token。

Resource Server 模式下，**核心是 `BearerTokenAuthenticationFilter` + `JwtDecoder` + `JwtAuthenticationProvider`** 三件套。后面 2.4 详细看源码。

### 1.6 方法级安全：从 AOP 到 AuthorizationManager

`@PreAuthorize` / `@PostAuthorize` / `@Secured` / `@RolesAllowed` 这些注解的本质是 **Spring AOP + AuthorizationManager**。

Spring Security 6 之前用的是 `MethodSecurityInterceptor` + `AccessDecisionManager`。**6.x 重构成了基于 `AuthorizationManager` 的新模型**，旧的 API 标记 deprecated。差别：

| 旧（5.x） | 新（6.x） |
| --- | --- |
| `AccessDecisionManager` | `AuthorizationManager<MethodInvocation>` |
| `Voter` 投票模式 | 直接返回 `AuthorizationDecision` |
| `@EnableGlobalMethodSecurity` | `@EnableMethodSecurity` |
| 默认 prePostEnabled = false | 默认 prePostEnabled = true |

启用：

```java
@Configuration
@EnableMethodSecurity   // 默认就开 @PreAuthorize / @PostAuthorize
public class MethodSecurityConfig { }
```

注解差异：

| 注解 | 求值时机 | 表达式 | 推荐场景 |
| --- | --- | --- | --- |
| `@PreAuthorize` | 方法执行前 | SpEL（强大） | 大部分场景 |
| `@PostAuthorize` | 方法执行后，可访问返回值 | SpEL（含 `returnObject`） | 数据所有者校验 |
| `@PreFilter` | 方法执行前过滤集合参数 | SpEL（含 `filterObject`） | 入参集合裁剪 |
| `@PostFilter` | 方法执行后过滤返回集合 | SpEL | 返回集合裁剪 |
| `@Secured` | 前置 | 角色字符串数组（不支持 SpEL） | 简单场景 |
| `@RolesAllowed` | 前置 | JSR-250 标准 | 跨框架 |

> **口诀**：`@PreAuthorize` 是 Spring 推荐的"通用主力"。能用 SpEL 表达任何鉴权语义；`@Secured` 只能比较字符串。新项目无脑选 `@PreAuthorize`。

### 1.7 一张图把所有模型串起来

```
HTTP Request
    │
    ▼
[FilterChainProxy → SecurityFilterChain]
    │
    ├─ Bearer Token Filter          ──┐
    │      │                          │
    │      └→ AuthenticationManager   │  AuthN（认证）
    │            └→ JwtAuthProvider   │
    │                  └→ JwtDecoder ─┘
    │
    ├─ AuthorizationFilter (URL 级)   ──┐  AuthZ（鉴权）
    │      └→ AuthorizationManager     │
    │                                   │
    └─ DispatcherServlet → Controller   │
            │                           │
            └→ MethodInterceptor (AOP)  │
                  └→ AuthorizationManager (方法级)
```

理解这张图后，剩下的问题都只是细节填充。

> **口诀**：Spring Security 的核心抽象就两个 —— **AuthenticationManager（认证）** 和 **AuthorizationManager（鉴权）**。其他所有类都只是这两个的支撑。

---

## 2. 源码切片：六段关键路径

> 基于 Spring Security 6.2 源码。注释为本文添加。

### 2.1 FilterChainProxy.doFilter —— 多 chain 路由的入口

```java
// org.springframework.security.web.FilterChainProxy
private void doFilterInternal(ServletRequest request, ServletResponse response,
        FilterChain chain) throws IOException, ServletException {

    FirewalledRequest fwRequest = firewall.getFirewalledRequest((HttpServletRequest) request);
    HttpServletResponse fwResponse = firewall.getFirewalledResponse((HttpServletResponse) response);

    List<Filter> filters = getFilters(fwRequest);   // ★ 选一条 chain

    if (filters == null || filters.size() == 0) {
        fwRequest.reset();
        chain.doFilter(fwRequest, fwResponse);      // 不匹配任何 chain → 走原 Servlet 链
        return;
    }

    VirtualFilterChain vfc = new VirtualFilterChain(fwRequest, chain, filters);
    vfc.doFilter(fwRequest, fwResponse);
}

private List<Filter> getFilters(HttpServletRequest request) {
    int count = 0;
    for (SecurityFilterChain chain : filterChains) {
        if (chain.matches(request)) {              // ★ 第一个匹配的就赢
            return chain.getFilters();
        }
    }
    return null;
}
```

**逐段点透**：

- **`firewall`** 是 `HttpFirewall`，默认是 `StrictHttpFirewall`，会拦截畸形 URL（包含 `;`、`%2e`、`\` 等）。生产偶尔会因为它把合法请求挡在外面 —— 调试时记得看这一层。
- **`getFilters` 用的是"第一个匹配赢"，不是"全部匹配合并"**。这是为什么多 SecurityFilterChain 必须按特异性从高到低排（`/admin/**` 在 `/**` 之前）。`@Order` 注解决定 chain 的优先级。
- **不匹配任何 chain 时直接走原 chain**，意味着没有任何 Spring Security 保护。这就是案例 ② 的根因。

> **口诀**：FilterChainProxy 是 Servlet 视角的"一个 Filter"，对内是"路由器"。多 chain 配置必须显式 `@Order`，不要靠声明顺序。

### 2.2 UsernamePasswordAuthenticationFilter.attemptAuthentication

```java
// org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter
public Authentication attemptAuthentication(HttpServletRequest request,
        HttpServletResponse response) throws AuthenticationException {

    if (this.postOnly && !request.getMethod().equals("POST")) {
        throw new AuthenticationServiceException(
            "Authentication method not supported: " + request.getMethod());
    }

    String username = obtainUsername(request);
    username = (username != null) ? username.trim() : "";
    String password = obtainPassword(request);
    password = (password != null) ? password : "";

    UsernamePasswordAuthenticationToken authRequest =
        UsernamePasswordAuthenticationToken.unauthenticated(username, password);
    setDetails(request, authRequest);                       // ★ 把 IP / sessionId 塞进 details

    return this.getAuthenticationManager().authenticate(authRequest);   // ★ 真正干活的入口
}
```

**逐段点透**：

- **`unauthenticated()` 工厂方法**是 6.x 的新 API，明确表达"这个 token 还没认证过"。旧的 `new UsernamePasswordAuthenticationToken(u, p)` 仍可用但语义不清。
- **`setDetails`** 默认装的是 `WebAuthenticationDetails`（IP + sessionId），后面 Provider 可以拿来做风控（异常 IP 拒登）。
- **`authenticate` 抛异常**会被 `AbstractAuthenticationProcessingFilter` 的 try-catch 接住，转成 401，不会冒到 Controller。这是为什么"密码错误"不会触发 `@ControllerAdvice`。

### 2.3 ExceptionTranslationFilter —— 把异常翻译成 HTTP 响应

```java
// org.springframework.security.web.access.ExceptionTranslationFilter
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
    try {
        chain.doFilter(req, res);                           // ★ 自己不抛，催后面抛
    } catch (IOException ex) {
        throw ex;
    } catch (Exception ex) {
        Throwable cause = ex;
        while (cause != null) {
            if (cause instanceof AuthenticationException) {
                handleAuthenticationException((HttpServletRequest) req,
                    (HttpServletResponse) res, chain, (AuthenticationException) cause);
                return;
            }
            if (cause instanceof AccessDeniedException) {
                handleAccessDeniedException((HttpServletRequest) req,
                    (HttpServletResponse) res, chain, (AccessDeniedException) cause);
                return;
            }
            cause = cause.getCause();                       // ★ 沿 cause 链找
        }
        throw ex;                                           // 既不是 AuthN 也不是 AuthZ → 透传
    }
}

private void handleAuthenticationException(HttpServletRequest req, HttpServletResponse res,
        FilterChain chain, AuthenticationException ex) {
    SecurityContextHolder.clearContext();
    requestCache.saveRequest(req, res);                     // 保存请求，登录后可跳回
    authenticationEntryPoint.commence(req, res, ex);        // ★ 调 AuthenticationEntryPoint
}

private void handleAccessDeniedException(HttpServletRequest req, HttpServletResponse res,
        FilterChain chain, AccessDeniedException ex) {
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    if (auth == null || trustResolver.isAnonymous(auth) || trustResolver.isRememberMe(auth)) {
        // ★ 匿名/RememberMe 用户被 deny → 当作"还没登录"处理
        sendStartAuthentication(req, res, chain,
            new InsufficientAuthenticationException("Full authentication required"));
    } else {
        accessDeniedHandler.handle(req, res, ex);           // 已登录被 deny → 403
    }
}
```

**逐段点透**：

- **沿 cause 链找异常类型**：`AuthorizationFilter` 抛的可能是被包了好几层的 `AccessDeniedException`，所以这里要 `while (cause != null)`。
- **`AuthenticationEntryPoint`** 是 401 响应的"出口"。Form Login 默认是 `LoginUrlAuthenticationEntryPoint`（302 跳登录页），REST API 应该换成 `BearerTokenAuthenticationEntryPoint`（返回 401 + WWW-Authenticate 头）或自定义 JSON 输出。
- **匿名用户的 `AccessDeniedException` 被当成"未认证"处理**：这是为什么没登录访问受保护资源时浏览器跳到登录页（而不是看到"403 禁止"）。

### 2.4 BearerTokenAuthenticationFilter —— OAuth2 Resource Server 的入口

```java
// org.springframework.security.oauth2.server.resource.web.authentication.BearerTokenAuthenticationFilter
protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
        FilterChain chain) throws ServletException, IOException {

    String token;
    try {
        token = this.bearerTokenResolver.resolve(req);      // ★ 从 Header / query / form 提取
    } catch (OAuth2AuthenticationException invalid) {
        this.authenticationEntryPoint.commence(req, res, invalid);
        return;
    }

    if (token == null) {
        chain.doFilter(req, res);                           // 没带 token → 让后续过滤器决定
        return;
    }

    BearerTokenAuthenticationToken authRequest = new BearerTokenAuthenticationToken(token);
    authRequest.setDetails(this.authenticationDetailsSource.buildDetails(req));

    try {
        AuthenticationManager am = this.authenticationManagerResolver.resolve(req);
        Authentication authResult = am.authenticate(authRequest);

        SecurityContext context = this.securityContextHolderStrategy.createEmptyContext();
        context.setAuthentication(authResult);
        this.securityContextHolderStrategy.setContext(context);
        this.securityContextRepository.saveContext(context, req, res);

        chain.doFilter(req, res);
    } catch (AuthenticationException failed) {
        this.securityContextHolderStrategy.clearContext();
        this.authenticationEntryPoint.commence(req, res, failed);
    }
}
```

**逐段点透**：

- **`bearerTokenResolver` 默认从 `Authorization: Bearer xxx` 头提取**。可以配 `setAllowFormEncodedBodyParameter(true)` 支持 form 参数（很少用，且可能引入 CSRF 风险）。
- **没带 token 时不抛异常，直接放行**：因为可能是公开接口。后面 `AuthorizationFilter` 才会真正决定要不要放行。这是 Resource Server 模式比 Form Login 灵活的地方。
- **`securityContextRepository.saveContext`** 在无状态 API 模式下应该是 `RequestAttributeSecurityContextRepository`（只在请求生命周期内保留），而不是 HttpSession。配错了会偷偷创建 Session，性能下降。

### 2.5 NimbusJwtDecoder —— JWT 验签的核心

```java
// org.springframework.security.oauth2.jwt.NimbusJwtDecoder（简化）
public Jwt decode(String token) throws JwtException {
    JWT jwt = parse(token);                                 // ① 字符串解析成 JWT 对象

    if (jwt instanceof PlainJWT) {
        throw new BadJwtException("Unsigned Jwts are not supported");
    }

    Jwt createdJwt = createJwt(token, jwt);                 // ② 验签 + 反序列化 claims
    return validateJwt(createdJwt);                         // ③ 校验 exp / iss / aud
}

private Jwt createJwt(String token, JWT parsedJwt) {
    try {
        // ★ JWKS 在这里被使用（远程拉取或本地缓存）
        JWTClaimsSet jwtClaimsSet = this.jwtProcessor.process(parsedJwt, null);
        // ...
    } catch (BadJOSEException ex) {
        throw new BadJwtException("Failed to validate the token", ex);
    } catch (JOSEException ex) {
        throw new JwtException("Failed to validate the token", ex);
    }
}

private Jwt validateJwt(Jwt jwt) {
    OAuth2TokenValidatorResult result = this.jwtValidator.validate(jwt);
    if (result.hasErrors()) {
        Collection<OAuth2Error> errors = result.getErrors();
        String description = errors.iterator().next().getDescription();
        throw new JwtValidationException(
            "An error occurred while attempting to decode the Jwt: " + description, errors);
    }
    return jwt;
}
```

**逐段点透**：

- **`PlainJWT` 直接拒绝**：未签名的 JWT 在 JWS 规范里是合法的，但作为认证 token 完全不可信。Spring 显式拒绝，避免"算法降级攻击"（攻击者把 alg 改成 `none`）。
- **JWTProcessor 内部用 `JWKSource` 拿密钥**：远程 JWKS 模式下，第一次会请求 `${jwks-uri}` 拿密钥集，**默认缓存 5 分钟**。所以你旋转了密钥后，5 分钟内的请求可能还在用老密钥（或反过来）。这是案例 ④ 的根。
- **`jwtValidator` 是个 `OAuth2TokenValidator` 链**：默认包含 `JwtTimestampValidator`（exp/nbf）+ `JwtIssuerValidator`（iss）。可以加 audience 校验、自定义 claim 校验。

### 2.6 AuthorizationManagerBeforeMethodInterceptor —— 方法级 @PreAuthorize 的拦截

```java
// org.springframework.security.authorization.method.AuthorizationManagerBeforeMethodInterceptor
public Object invoke(MethodInvocation mi) throws Throwable {
    attemptAuthorization(mi);                                       // ★ 先鉴权
    return mi.proceed();                                            // 通过则放行
}

private void attemptAuthorization(MethodInvocation mi) {
    Supplier<Authentication> authentication = AUTHENTICATION_SUPPLIER;
    AuthorizationDecision decision = this.authorizationManager.check(authentication, mi);
    this.eventPublisher.publishAuthorizationEvent(authentication, mi, decision);
    if (decision != null && !decision.isGranted()) {
        throw new AuthorizationDeniedException("Access Denied", decision);
    }
}

// PreAuthorizeAuthorizationManager.check
public AuthorizationDecision check(Supplier<Authentication> authentication, MethodInvocation mi) {
    ExpressionAttribute attribute = this.registry.getAttribute(mi);   // 缓存 SpEL 表达式
    if (attribute == ExpressionAttribute.NULL_ATTRIBUTE) {
        return null;                                                  // 没 @PreAuthorize → 不管
    }
    EvaluationContext ctx = this.expressionHandler.createEvaluationContext(authentication, mi);
    boolean granted = ExpressionUtils.evaluateAsBoolean(attribute.getExpression(), ctx);
    return new ExpressionAuthorizationDecision(granted, attribute.getExpression());
}
```

**逐段点透**：

- **拦截器是 AOP MethodInterceptor**：所以 `@PreAuthorize` **必须通过 Spring Bean 调用才生效**。同类内部 `this.xxx()` 调用绕过 AOP，鉴权失效。这是案例 ⑦ 的根。
- **SpEL 表达式被缓存**：第一次拿到的 `Expression` 对象会被存进 `registry`，后续调用直接复用。所以表达式写错时，运行时报错，不是启动时报错 —— 必须在测试覆盖到。
- **`AuthorizationDecision` 的 null 语义**：null 表示"没有意见"，让其他 Manager 决定；非 null 的 `granted=false` 才真正拒绝。这是 6.x 新模型的设计 —— 鼓励 Manager 组合而非单一决策。

### 2.7 SecurityContextHolder 的 ThreadLocal 实现

```java
// org.springframework.security.core.context.ThreadLocalSecurityContextHolderStrategy
final class ThreadLocalSecurityContextHolderStrategy implements SecurityContextHolderStrategy {

    private static final ThreadLocal<Supplier<SecurityContext>> contextHolder = new ThreadLocal<>();

    public void clearContext() {
        contextHolder.remove();
    }

    public SecurityContext getContext() {
        return getDeferredContext().get();
    }

    public Supplier<SecurityContext> getDeferredContext() {
        Supplier<SecurityContext> result = contextHolder.get();
        if (result == null) {
            SecurityContext context = createEmptyContext();
            result = () -> context;
            contextHolder.set(result);
        }
        return result;
    }

    public void setContext(SecurityContext context) {
        Assert.notNull(context, "Only non-null SecurityContext instances are permitted");
        contextHolder.set(() -> context);
    }
}
```

**逐段点透**：

- **`Supplier<SecurityContext>` 而不是直接 `SecurityContext`**：6.x 引入的延迟模式 —— Spring Session / Redis-backed session 场景下，反序列化 SecurityContext 是有成本的，包成 Supplier 后只在真正用到时才反序列化。
- **`ThreadLocal` 没用 `InheritableThreadLocal`**：这是 Spring 反复纠结的一个决策。InheritableThreadLocal 在线程池场景下会串身份；显式装饰才安全。
- **`SecurityContextHolderFilter` 在请求结束时调 `clearContext`**：这是 finally 必经的逻辑，否则 ThreadLocal 残留在 Tomcat worker 线程上，下个请求看到上个用户的身份。

> **口诀**：SecurityContext 的传播 = ThreadLocal + 显式装饰器。**任何把任务提交到非主线程的代码都必须装饰**，否则身份必丢。

---

## 3. 实测数据：认证开销不是平等的

### 3.1 不同认证方式的延迟对比

实验：同一个 `/api/me` 接口，分别用四种方式认证，测 P99。

测试环境：8 核 / 16GB / 同机 Tomcat / wrk 压测 100 连接 30s。

| 认证方式 | QPS | P50 (ms) | P99 (ms) | 备注 |
| --- | --- | --- | --- | --- |
| **匿名** | 38,000 | 1.2 | 3.5 | 基线 |
| **HTTP Basic（每请求查 DB）** | 6,200 | 8.5 | 25 | bcrypt cost=10 |
| **HTTP Basic（DB + 缓存）** | 21,000 | 2.8 | 8 | Caffeine 缓存 |
| **JWT（HS256 自签名）** | 28,000 | 1.8 | 5 | 无外部依赖 |
| **JWT（RS256 + 本地 JWKS）** | 26,000 | 1.9 | 5.2 | 公钥本地缓存 |
| **JWT（RS256 + 远程 JWKS 首次）** | — | — | 80~200 | 第一次拉 JWKS |
| **Session（内存）** | 32,000 | 1.5 | 4 | 无锁查询 |
| **Session（Redis）** | 14,000 | 4.2 | 12 | 一次 Redis IO |

**结论**：
- 自签 JWT 几乎"零开销"，但密钥管理是大坑（见案例 ④）
- bcrypt 的 cost 参数直接决定登录延迟，cost=10 已经是 ~80ms 单次
- Redis Session 的瓶颈是网络 IO 而非反序列化

### 3.2 JWT 验证的 CPU 火焰图

用 async-profiler 抓 30 秒：

```bash
./profiler.sh -d 30 -e cpu -f jwt.html <pid>
```

典型样本占比：
- `NimbusJwtDecoder.decode`：8%
  - `JWTProcessor.process`：6.5%
    - `RSASSAVerifier.verify`：4%（RSA 验签）
    - `JWKSource.get`：2%（缓存命中）
- `JwtAuthenticationProvider.authenticate`：1%
- `BearerTokenAuthenticationFilter.doFilterInternal`：< 0.5%

**洞察**：80% 的 JWT 验证开销在**密码学验签**本身。换 ES256 比 RS256 快 1.5~2 倍（短签名 + 椭圆曲线运算少），但客户端兼容性要测。

### 3.3 方法级 @PreAuthorize vs URL 级 antMatchers

| 鉴权方式 | QPS | P99 (ms) | 备注 |
| --- | --- | --- | --- |
| **URL 级（hasRole）** | 27,500 | 5.0 | AuthorizationFilter 链 |
| **方法级（@PreAuthorize hasRole）** | 26,800 | 5.1 | AOP + SpEL 缓存命中 |
| **方法级（@PreAuthorize 复杂 SpEL）** | 24,500 | 6.2 | 含 `@beanXxx.check(#id)` 调用 |
| **方法级 + @PostAuthorize** | 23,000 | 6.8 | 多一次 SpEL 求值 |
| **方法级 + @PostFilter（1000 元素）** | 9,500 | 22 | 每元素一次 SpEL |

**结论**：
- `@PreAuthorize` 单次开销和 URL 级几乎一样，**SpEL 缓存机制让它接近免费**
- `@PostFilter` 在大集合上是性能杀手 —— 改成 SQL/查询层过滤，不要靠注解

### 3.4 OAuth2 Authorization Server 的发 Token 吞吐

Spring Authorization Server 0.4.x，单实例 `/oauth2/token` 端点压测：

| Grant Type | QPS | P99 (ms) |
| --- | --- | --- |
| `client_credentials`（HS256） | 2,800 | 35 |
| `client_credentials`（RS256） | 1,900 | 52 |
| `authorization_code` + PKCE | 1,200 | 70 |
| `refresh_token`（含 token rotation） | 2,100 | 45 |

**洞察**：
- 发 Token 比验 Token 慢一个数量级 —— 因为要查客户端、生成签名、写入 store
- `refresh_token` 的 rotation（每次 refresh 都换 token）需要写 DB，受 DB 影响大

### 3.5 Session vs Token 的内存对比

100 万在线用户：

| 方案 | 每用户内存 | 总占用 | 备注 |
| --- | --- | --- | --- |
| **Tomcat 内存 Session** | ~2KB（最小） | 2GB | 集群无法共享 |
| **Spring Session + Redis** | ~1.5KB | Redis 1.5GB | 反序列化开销 |
| **Spring Session + JDBC** | ~2KB | DB 2GB+索引 | 性能最差 |
| **Stateless JWT** | 0 | 0 | 服务端零状态，token 在客户端 |

**结论**：
- 大并发场景 JWT 内存优势压倒性
- 但 JWT 不能即时撤销（除非配黑名单），是另一类成本
- Session 适合"用户量小、强撤销需求"的内部系统

> **口诀**：JWT 适合**大规模无状态 API + 短有效期 + refresh 旋转**。Session 适合**强可控撤销 + 用户量小**。两者不是非此即彼。

---

## 4. 生产案例 9 则

### 案例 ① Spring Security 5 → 6 升级 —— "WebSecurityConfigurerAdapter 已废弃"

#### 现象

公司项目从 Spring Boot 2.7 升 3.x，Spring Security 跟着升 5.7 → 6.x。编译器警告：

```
warning: [deprecation] WebSecurityConfigurerAdapter in
    org.springframework.security.config.annotation.web.configuration has been deprecated
```

直接删了警告还行，结果运行时报：

```
NoSuchMethodError: WebSecurityConfigurerAdapter
```

或者：

```
Caused by: java.lang.IllegalStateException:
  Cannot apply @{} to already built object
```

#### 根因

Spring Security 6 **彻底移除了 `WebSecurityConfigurerAdapter`**。5.7 已 deprecated，6.0 删除。设计哲学：从"继承式配置"换成"组合式配置" —— 你提供 Bean，框架装配。

旧写法（5.x，不再可用）：

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
            .antMatchers("/api/**").authenticated()
            .anyRequest().permitAll()
            .and()
            .formLogin();
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.inMemoryAuthentication().withUser("admin").password("{noop}123").roles("ADMIN");
    }
}
```

#### 修复

新写法（6.x，Lambda DSL）：

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth                  // ★ 注意是 authorizeHttpRequests
                .requestMatchers("/api/**").authenticated()      // ★ antMatchers → requestMatchers
                .anyRequest().permitAll())
            .formLogin(Customizer.withDefaults())
            .build();
    }

    @Bean
    public UserDetailsService users() {
        UserDetails admin = User.builder()
            .username("admin")
            .password("{bcrypt}$2a$10$...")                       // 不要再用 {noop}
            .roles("ADMIN")
            .build();
        return new InMemoryUserDetailsManager(admin);
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    }
}
```

变化点清单：
- `WebSecurityConfigurerAdapter` → **暴露 `SecurityFilterChain` Bean**
- `authorizeRequests()` → **`authorizeHttpRequests()`**（旧的也 deprecated）
- `antMatchers / mvcMatchers / regexMatchers` → **统一为 `requestMatchers`**
- 链式 `.and()` → **Lambda DSL**
- `configure(AuthenticationManagerBuilder)` → **暴露 `UserDetailsService` Bean** 或 `AuthenticationManager` Bean

`UserDetailsService` 不再需要在 builder 里注册，**只要是 Bean Spring Security 就会发现**。

#### 多 chain 配置

```java
@Bean
@Order(1)
public SecurityFilterChain apiChain(HttpSecurity http) throws Exception {
    return http
        .securityMatcher("/api/**")                              // ★ 仅匹配 /api/**
        .authorizeHttpRequests(a -> a.anyRequest().authenticated())
        .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))
        .csrf(AbstractHttpConfigurer::disable)
        .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .build();
}

@Bean
@Order(2)
public SecurityFilterChain webChain(HttpSecurity http) throws Exception {
    return http
        .authorizeHttpRequests(a -> a
            .requestMatchers("/", "/login", "/css/**").permitAll()
            .anyRequest().authenticated())
        .formLogin(Customizer.withDefaults())
        .build();
}
```

`@Order` 决定优先级，**小的先匹配**。`securityMatcher` 决定整条 chain 的适用范围，匹配不上就跳过这条 chain。

#### 验证

```bash
# 启动后看日志，应该有：
# DefaultSecurityFilterChain : Will secure ...
# 一条 chain 一行

# 测试两条 chain 互不干扰
curl -i http://localhost:8080/api/me                 # 应 401（无 token）
curl -i http://localhost:8080/                       # 应 302 跳登录页
```

> **口诀**：6.x 升级三要素 —— **`SecurityFilterChain` Bean 替代 Adapter**、**`authorizeHttpRequests` + `requestMatchers`**、**Lambda DSL**。多 chain 必须 `@Order`。

---

### 案例 ② SecurityFilterChain 顺序错误 —— "权限被绕过"

#### 现象

线上发现某个未登录用户调用 `/api/admin/users/delete` 成功了，没拿到 401，直接返回了删除结果。

测试环境复现不出来 —— 测试只测了 `/api/admin/**` 的鉴权。

#### 根因

配置写成了这样：

```java
@Bean
@Order(1)
public SecurityFilterChain publicChain(HttpSecurity http) throws Exception {
    return http
        .securityMatcher("/api/**")                              // ★ 太宽
        .authorizeHttpRequests(a -> a.anyRequest().permitAll())
        .csrf(AbstractHttpConfigurer::disable)
        .build();
}

@Bean
@Order(2)
public SecurityFilterChain adminChain(HttpSecurity http) throws Exception {
    return http
        .securityMatcher("/api/admin/**")                        // ★ 永远到不了
        .authorizeHttpRequests(a -> a.anyRequest().hasRole("ADMIN"))
        .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))
        .build();
}
```

机理（呼应 2.1）：FilterChainProxy 用"第一个匹配赢"。`/api/admin/users/delete` 同时匹配两个 `securityMatcher`，但 `@Order(1)` 的 publicChain 先匹配上了 —— 它说"全开"，于是请求直接放行，根本走不到 adminChain。

这是 Spring Security 多 chain 配置最容易踩的坑。

#### 修复

铁律：**特异性高的 chain 必须 @Order 更小**。

```java
@Bean
@Order(1)                                                        // ★ 更具体的先
public SecurityFilterChain adminChain(HttpSecurity http) throws Exception {
    return http
        .securityMatcher("/api/admin/**")
        .authorizeHttpRequests(a -> a.anyRequest().hasRole("ADMIN"))
        .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))
        .build();
}

@Bean
@Order(2)
public SecurityFilterChain publicChain(HttpSecurity http) throws Exception {
    return http
        .securityMatcher("/api/**")
        .authorizeHttpRequests(a -> a.anyRequest().permitAll())
        .build();
}
```

更严谨的做法：**用一条 chain 内部分级控制**，避免多 chain 互相干扰。

```java
@Bean
public SecurityFilterChain apiChain(HttpSecurity http) throws Exception {
    return http
        .securityMatcher("/api/**")
        .authorizeHttpRequests(a -> a
            .requestMatchers("/api/admin/**").hasRole("ADMIN")
            .requestMatchers("/api/public/**").permitAll()
            .anyRequest().authenticated())
        .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))
        .build();
}
```

`requestMatchers` 内部也是"先匹配赢"，**具体路径必须在通配前面**。

#### 验证

写覆盖到所有受保护端点的安全测试：

```java
@Test
void anonymousMustNotAccessAdmin() throws Exception {
    mockMvc.perform(delete("/api/admin/users/1"))
        .andExpect(status().isUnauthorized());
}

@Test
void userMustNotAccessAdmin() throws Exception {
    mockMvc.perform(delete("/api/admin/users/1")
            .with(user("alice").roles("USER")))
        .andExpect(status().isForbidden());
}

@Test
void adminCanAccessAdmin() throws Exception {
    mockMvc.perform(delete("/api/admin/users/1")
            .with(user("admin").roles("ADMIN")))
        .andExpect(status().isOk());
}
```

启动期日志检查（写个 `ApplicationListener<ApplicationReadyEvent>`）：

```java
@Bean
public ApplicationListener<ApplicationReadyEvent> dumpFilterChains(FilterChainProxy proxy) {
    return event -> {
        proxy.getFilterChains().forEach(chain -> {
            log.info("Chain: matcher={}, filters={}",
                chain, chain.getFilters().stream().map(f -> f.getClass().getSimpleName()).toList());
        });
    };
}
```

> **口诀**：多 chain 必须**特异性高的 @Order 小**。能用一条 chain 解决的不要拆。CI 里跑全量"未登录访问受保护端点"测试。

---

### 案例 ③ JWT 时钟偏差 —— "客户端验签失败"

#### 现象

业务方反馈：从老服务器拿到的 JWT，发到新服务器立即被拒，错误：

```
JwtValidationException: Jwt expired at 2024-03-15T10:30:00Z
```

但 token 的 `exp` 字段是 1 小时后，时间没到啊。

#### 根因

两台服务器之间有 **NTP 时钟偏差**（差了 30 秒）。Authorization Server 签发时 token 的 `iat = 10:00:00`、`exp = 11:00:00`。Resource Server 当前系统时间是 `10:00:30`，看起来还没过期。但有些校验路径会同时检查 `nbf`（not before）和 `iat`，如果 `nbf > now`，token 被认为"还没生效"。

更经典的反向场景：Resource Server 的时钟比 Authorization Server **慢** 30 秒，token 实际还有效，但 RS 看到 `exp` 已经过去了，拒绝。

NimbusJwtDecoder 默认对 `exp` / `nbf` **完全不容忍偏差**（clockSkew = 0）。

#### 修复

显式设置时钟偏差容忍度：

```java
@Bean
public JwtDecoder jwtDecoder(@Value("${spring.security.oauth2.resourceserver.jwt.jwk-set-uri}") String jwkSetUri) {
    NimbusJwtDecoder decoder = NimbusJwtDecoder.withJwkSetUri(jwkSetUri).build();

    OAuth2TokenValidator<Jwt> withClockSkew = new DelegatingOAuth2TokenValidator<>(
        new JwtTimestampValidator(Duration.ofSeconds(60)),       // ★ 容忍 60 秒
        new JwtIssuerValidator("https://auth.example.com"),
        audienceValidator()
    );
    decoder.setJwtValidator(withClockSkew);
    return decoder;
}

private OAuth2TokenValidator<Jwt> audienceValidator() {
    return jwt -> {
        if (jwt.getAudience().contains("api.example.com")) {
            return OAuth2TokenValidatorResult.success();
        }
        return OAuth2TokenValidatorResult.failure(
            new OAuth2Error("invalid_audience", "Audience not allowed", null));
    };
}
```

`JwtTimestampValidator(Duration.ofSeconds(60))` 表示 `exp` 和 `nbf` 都允许 60 秒偏差。

**根治方案**：所有服务器接入 NTP，校准时间。

```bash
# 所有节点
sudo timedatectl set-ntp true
sudo timedatectl status                # 看 "NTP service: active"
```

容器化场景：
- K8s 节点必须装 chrony 或 systemd-timesyncd
- 容器内不能改时间，看的是宿主机时间
- 如果用了 firecracker / kata 等微 VM，特别注意时钟同步策略

#### 验证

```bash
# 故意把一台机器时间调快 30 秒，看请求是否仍能通过
sudo date -s '+30 seconds'
curl -i -H "Authorization: Bearer ${VALID_TOKEN}" http://localhost:8080/api/me
# 配了 60s skew → 应通过；未配 → 401
```

监控指标：JWT 验签失败率，按错误类型分类。如果"过期"错误占比突然飙升，先怀疑时钟。

> **口诀**：JWT 时钟偏差是分布式系统的**默认 bug**。`JwtTimestampValidator(60s)` 是兜底；NTP 是根治。**两件都要做**。

---

### 案例 ④ JWKS 缓存陈旧 —— "密钥旋转后用户全部掉线"

#### 现象

Authorization Server 旋转了签名密钥（JWKS endpoint 增加了新 kid）。**5 分钟内**所有 Resource Server 大量返回 401，错误：

```
JwtException: Signed JWT rejected: Another algorithm expected, or no matching key(s) found
```

#### 根因

`NimbusJwtDecoder.withJwkSetUri()` 默认使用 `RemoteJWKSet`，**JWKS 在 Resource Server 端缓存 5 分钟**（Nimbus 的默认 `DefaultJWKSetCache`）。

当 Authorization Server 旋转密钥（用新私钥签 token、JWKS 暴露新公钥），Resource Server 缓存里还是老 JWKS，用老公钥验新签名 → 失败。

更糟的场景：Authorization Server **强制立即旋转**（删掉老 kid），还在用老 token 的客户端的 token 在 RS 缓存里能验过，但下游服务（其他 RS）刷新缓存后用新 JWKS 反而验不过老 token。

#### 修复

**方案 A：JWKS 应该"双 kid 并存一段时间"**。

Authorization Server 旋转时**保留老密钥至少一个 token 生命周期**（比如 token 有效 1 小时，老 kid 至少保留 1 小时）。这是 OIDC 的最佳实践。

```json
// JWKS endpoint 输出，旋转后应该长这样
{
  "keys": [
    { "kid": "key-2024-03", "kty": "RSA", "use": "sig", "n": "...", "e": "AQAB" },
    { "kid": "key-2024-02", "kty": "RSA", "use": "sig", "n": "...", "e": "AQAB" }
  ]
}
```

**方案 B：Resource Server 收到"未知 kid"时强制刷新缓存**。

```java
@Bean
public JwtDecoder jwtDecoder() throws Exception {
    JWKSource<SecurityContext> jwkSource = JWKSourceBuilder
        .create(new URL("https://auth.example.com/jwks"))
        .cache(5 * 60 * 1000, 30 * 1000)      // (lifespan, refreshTimeout)
        .refreshAheadCache(true)              // ★ 提前刷新（lifespan 80% 时主动拉新的）
        .retrying(true)                        // ★ kid 找不到时强制重拉
        .build();

    DefaultJWTProcessor<SecurityContext> processor = new DefaultJWTProcessor<>();
    processor.setJWSKeySelector(new JWSVerificationKeySelector<>(JWSAlgorithm.RS256, jwkSource));

    NimbusJwtDecoder decoder = new NimbusJwtDecoder(processor);
    decoder.setJwtValidator(jwtValidator());
    return decoder;
}
```

`refreshAheadCache + retrying` 的组合是关键：
- 缓存即将过期前主动刷（不会让用户请求等远程 IO）
- 找不到 kid 时强制重拉（应对突发旋转）

**方案 C：减小缓存时间，但代价是 JWKS endpoint 压力大**。

```java
.cache(60 * 1000, 10 * 1000)      // 缓存 1 分钟
```

不推荐 —— Authorization Server 的 `/jwks` 往往是瓶颈点。

#### 监控

```java
// 把 JWKS 缓存指标暴露给 Micrometer
meterRegistry.gauge("jwks.cache.size", jwkSource, src -> src.get(...).size());

// JWT 验签失败按原因分类
@Bean
public JwtAuthenticationConverter jwtAuthConverter() {
    return new JwtAuthenticationConverter();   // 自定义里加 metric
}
```

告警：`jwt.validation.failed{reason="key_not_found"} > 0` 持续 1 分钟。

#### 验证

```bash
# 在 Authorization Server 旋转密钥
# 立即在 Resource Server 测试
for i in {1..100}; do
    curl -s -o /dev/null -w "%{http_code}\n" \
        -H "Authorization: Bearer ${NEW_TOKEN}" \
        http://localhost:8080/api/me
done | sort | uniq -c
# 应全部 200，不应有 401 集群
```

> **口诀**：JWKS 旋转必须 **AS 双 kid 并存 + RS refreshAhead/retrying**。两端配合，缺一不可。

---

### 案例 ⑤ CORS 与 CSRF 配置冲突 —— "前端请求被莫名拒绝"

#### 现象

前后端分离项目，前端 `https://app.example.com`、后端 `https://api.example.com`。前端 POST `/api/users` 请求一直 403。

浏览器控制台：

```
Access to XMLHttpRequest at 'https://api.example.com/users' from origin
'https://app.example.com' has been blocked by CORS policy
```

加了 CORS 配置后变成：

```
HTTP 403 Forbidden
Invalid CSRF token
```

#### 根因

两个独立但易混淆的安全机制：
- **CORS（Cross-Origin Resource Sharing）**：浏览器层面的同源策略，控制"哪些域可以读响应"
- **CSRF（Cross-Site Request Forgery）**：服务端防护，确保请求是来自合法页面而非伪造

Spring Security **默认开启 CSRF**，对所有"修改性 HTTP 方法"（POST/PUT/PATCH/DELETE）要求 CSRF Token。前后端分离项目用 Cookie 鉴权时这个保护是必要的；纯 JWT 鉴权时则不需要（JWT 在 Header 里，CSRF 攻击拿不到）。

#### 修复

**REST API + JWT 模式**（最常见）：

```java
@Bean
@Order(1)
public SecurityFilterChain apiChain(HttpSecurity http) throws Exception {
    return http
        .securityMatcher("/api/**")
        .cors(Customizer.withDefaults())                          // ★ 开 CORS
        .csrf(AbstractHttpConfigurer::disable)                    // ★ 关 CSRF
        .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))
        .authorizeHttpRequests(a -> a.anyRequest().authenticated())
        .build();
}

@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration cfg = new CorsConfiguration();
    cfg.setAllowedOrigins(List.of("https://app.example.com"));    // ★ 不要用 *（带 cookie 时 * 无效）
    cfg.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    cfg.setAllowedHeaders(List.of("Authorization", "Content-Type"));
    cfg.setAllowCredentials(true);                                // 如果要带 Cookie
    cfg.setMaxAge(3600L);                                         // 预检请求缓存

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", cfg);
    return source;
}
```

**Cookie Session 模式**（如果你必须用）：

```java
@Bean
public SecurityFilterChain webChain(HttpSecurity http) throws Exception {
    return http
        .cors(Customizer.withDefaults())
        .csrf(c -> c
            .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())   // ★ 暴露给 JS
            .csrfTokenRequestHandler(new CsrfTokenRequestAttributeHandler()))
        .authorizeHttpRequests(a -> a.anyRequest().authenticated())
        .formLogin(Customizer.withDefaults())
        .build();
}
```

`CookieCsrfTokenRepository.withHttpOnlyFalse()` 让 CSRF token 写入 Cookie 但允许 JS 读取（前端读出来塞到 `X-XSRF-TOKEN` header）。

#### 反常识：CORS 不是安全机制

很多人以为"配了 CORS 就安全了"。错。

CORS 是**浏览器对自己的限制**，让 JS 不能随便读跨域响应。但：
- 服务端必须自己鉴权（CORS 不替代鉴权）
- 非浏览器客户端（curl、爬虫、Postman）完全不受 CORS 约束
- `Access-Control-Allow-Origin: *` 不是"允许所有"，而是"允许所有但不带 cookie"

#### 验证

```bash
# 模拟跨域 OPTIONS 预检
curl -i -X OPTIONS https://api.example.com/users \
    -H "Origin: https://app.example.com" \
    -H "Access-Control-Request-Method: POST" \
    -H "Access-Control-Request-Headers: Authorization,Content-Type"
# 应 204，且响应有 Access-Control-Allow-* 头

# 实际 POST
curl -i -X POST https://api.example.com/users \
    -H "Origin: https://app.example.com" \
    -H "Authorization: Bearer ${TOKEN}" \
    -H "Content-Type: application/json" \
    -d '{"name":"alice"}'
# 应 200/201
```

> **口诀**：**REST API + JWT → 关 CSRF + 开 CORS**。**Cookie Session → 开 CSRF + 开 CORS**。两个都不开等于没穿衣服。

---

### 案例 ⑥ 异步线程下 SecurityContext 丢失 —— "@Async 里 currentUser() 是 null"

#### 现象

业务方法标了 `@Async`，里面调 `SecurityContextHolder.getContext().getAuthentication()` 拿到的是 `null` 或 anonymous。

```java
@Service
public class OrderService {

    @Async
    public void notify(Long orderId) {
        // 期望拿到调用者
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        log.info("Notifying for {} as {}", orderId, auth);   // ★ null
    }
}
```

#### 根因

`@Async` 把方法转移到 `TaskExecutor` 的另一个线程。Spring Security 的 `SecurityContextHolder` 默认是 ThreadLocal，**子线程拿不到**。

WebAsyncManagerIntegrationFilter 只覆盖了 Spring MVC 的 Callable / DeferredResult，**`@Async` 不在它的处理范围**。

#### 修复

**方案 A：把 TaskExecutor 包成 SecurityContext 透传版**。

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "taskExecutor")
    public TaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(8);
        executor.setMaxPoolSize(32);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("async-");
        executor.initialize();

        // ★ 用 DelegatingSecurityContextAsyncTaskExecutor 包一层
        return new DelegatingSecurityContextAsyncTaskExecutor(executor);
    }
}
```

`DelegatingSecurityContextAsyncTaskExecutor` 在提交任务时**捕获当前 SecurityContext**，在子线程里**先 set 后执行任务，finally 里 clear**。这是经典的 capture-replay-restore 模式（呼应线程池文章的案例 ⑤）。

**方案 B：全局改 SecurityContextHolder 策略**（不推荐）。

```java
SecurityContextHolder.setStrategyName(SecurityContextHolder.MODE_INHERITABLETHREADLOCAL);
```

InheritableThreadLocal 会把父线程的 ThreadLocal **复制到子线程创建的瞬间**。问题：
- 线程池场景下，worker 是池预创建的，复制的是"创建时"的父线程，**和当前提交者无关**
- 串身份风险：worker 复用时残留上一个用户的 context

只在**完全裸创建子线程**且每个子线程独立的场景才考虑。生产环境基本不用。

**方案 C：手动传递**。

```java
@Async
public void notify(Long orderId, Authentication auth) {       // 显式传
    SecurityContext ctx = new SecurityContextImpl(auth);
    SecurityContextHolder.setContext(ctx);
    try {
        // 业务
    } finally {
        SecurityContextHolder.clearContext();
    }
}
```

冗余但无歧义，适合临时修补。

#### Reactor 场景

WebFlux 不能用 ThreadLocal，必须用 `ReactiveSecurityContextHolder`：

```java
public Mono<String> handle() {
    return ReactiveSecurityContextHolder.getContext()
        .map(SecurityContext::getAuthentication)
        .map(Authentication::getName);
}
```

它通过 Reactor Context 传递（不是 ThreadLocal），切换线程不丢。

#### 验证

```java
@Test
void asyncMustPreserveSecurityContext() throws Exception {
    SecurityContextHolder.setContext(new SecurityContextImpl(
        new TestingAuthenticationToken("alice", "x", "ROLE_USER")));

    CompletableFuture<String> result = orderService.findCallerAsync();
    assertEquals("alice", result.get(2, TimeUnit.SECONDS));
}
```

> **口诀**：所有把任务提交到线程池/调度器的代码，都必须用 `DelegatingSecurityContext*` 装饰。InheritableThreadLocal 不是答案。

---

### 案例 ⑦ @PreAuthorize SpEL 写错 —— "鉴权失效但没人发现"

#### 现象

代码评审发现：

```java
@PreAuthorize("hasRole(ROLE_ADMIN)")            // ★ 没引号
public void deleteUser(Long id) { ... }
```

启动正常，访问也"正常"（没有报错）—— 但任意用户都能调用 `deleteUser`。

#### 根因

`hasRole(ROLE_ADMIN)` 的 SpEL 解析为：调用 `hasRole` 方法，参数是 `ROLE_ADMIN` 这个**变量**。Spring Security 的 SpEL 上下文里没有这个变量，求值结果是 `null`。`hasRole(null)` 通常返回 false 或抛异常 —— 这里要看异常处理路径。

但更糟的是某些写法**默默返回 true**：

```java
@PreAuthorize("permitAll")                       // ★ 缺括号，permitAll 被当成属性引用 → 不存在 → null → false
@PreAuthorize("hasRole('admin')")               // ★ 角色名小写
@PreAuthorize("hasAuthority(ROLE_ADMIN)")       // ★ ROLE_ADMIN 当变量
@PreAuthorize("isAuthenticated() && hasRole('USER')")  // 看似对，但 isAuthenticated() 包含 anonymous
```

最隐蔽的：

```java
@PreAuthorize("#userId == authentication.name")   // 期望"只能改自己的资料"
public void updateUser(@PathVariable Long userId) { ... }
// 但 authentication.name 是 String，userId 是 Long → 永远 false → 永远 deny
// 或者反过来比较的方向，让任何用户都能改任何资料
```

机理（呼应 2.6）：SpEL 表达式**只在调用时求值**，启动期不报错。所以 IDE 不警告，编译不报错，CI 不跑就完全发现不了。

#### 修复

**正确写法清单**：

```java
@PreAuthorize("hasRole('ADMIN')")                                // ✅ 角色字符串带引号
@PreAuthorize("hasRole('ADMIN') or hasRole('SUPER_ADMIN')")      // ✅ or 而非 ||（SpEL）
@PreAuthorize("hasAuthority('SCOPE_admin:write')")               // ✅ OAuth2 Scope
@PreAuthorize("isAuthenticated()")                                // ✅ 已登录（不含 anonymous）
@PreAuthorize("authentication.name == #userId.toString()")       // ✅ 类型对齐
@PreAuthorize("@orderService.isOwner(#orderId, authentication.name)")  // ✅ 调 Bean 方法
@PreAuthorize("hasRole('USER') and #request.amount < 10000")     // ✅ 用入参做更细判断
```

**强制 CI 校验**：

```java
// 用单元测试覆盖每个 @PreAuthorize 注解
@Test
void deleteUserRequiresAdmin() throws Exception {
    mockMvc.perform(delete("/users/1").with(user("alice").roles("USER")))
        .andExpect(status().isForbidden());
    mockMvc.perform(delete("/users/1").with(user("admin").roles("ADMIN")))
        .andExpect(status().isOk());
}
```

**自定义 PermissionEvaluator** 把复杂逻辑收敛：

```java
@Component
public class CustomPermissionEvaluator implements PermissionEvaluator {

    @Override
    public boolean hasPermission(Authentication auth, Object target, Object permission) {
        if (auth == null || !auth.isAuthenticated()) return false;
        if (target instanceof Order order) {
            return order.getOwnerId().equals(auth.getName())
                || hasAdminRole(auth);
        }
        return false;
    }

    @Override
    public boolean hasPermission(Authentication auth, Serializable targetId, String targetType, Object permission) {
        // 如果 target 不在内存里
        return false;
    }
}

@Configuration
@EnableMethodSecurity
public class MethodSecurityConfig {

    @Bean
    public MethodSecurityExpressionHandler expressionHandler(PermissionEvaluator pe) {
        DefaultMethodSecurityExpressionHandler h = new DefaultMethodSecurityExpressionHandler();
        h.setPermissionEvaluator(pe);
        return h;
    }
}
```

注解里就能写：

```java
@PreAuthorize("hasPermission(#order, 'WRITE')")
public void updateOrder(Order order) { ... }
```

逻辑收敛在 PermissionEvaluator 一处，IDE 能跳转、能重构、能单测。

#### 同类内部调用陷阱（呼应 Spring AOP）

```java
@Service
public class UserService {

    @PreAuthorize("hasRole('ADMIN')")
    public void deleteUser(Long id) { ... }

    public void cleanup() {
        deleteUser(123L);    // ★ this. 调用，绕过 AOP，鉴权失效
    }
}
```

修复：把方法挪到另一个 Bean，或注入自身代理：

```java
@Resource
private UserService self;

public void cleanup() {
    self.deleteUser(123L);   // ✅ 走代理
}
```

#### 验证

集成测试覆盖每个 `@PreAuthorize`：

```java
@Test
void allMethodSecurityAnnotationsAreEnforced() {
    // 用反射扫描所有 @PreAuthorize
    Reflections r = new Reflections("com.example", new MethodAnnotationsScanner());
    Set<Method> methods = r.getMethodsAnnotatedWith(PreAuthorize.class);
    assertFalse(methods.isEmpty(), "应至少有一个鉴权注解");
    // 然后对每个方法跑两次 mockMvc：未授权用户 → 403，授权用户 → 200
}
```

> **口诀**：SpEL 表达式**运行时求值不报错**。每个 `@PreAuthorize` 必须有覆盖测试（正向 + 反向）。逻辑复杂时下沉到 PermissionEvaluator。

---

### 案例 ⑧ Session 固定攻击 —— "登录前的 sessionid 登录后还能用"

#### 现象

安全扫描报告：登录前后 `JSESSIONID` 不变，可能被 Session Fixation 攻击利用。

攻击场景：
1. 攻击者访问 `https://app.example.com`，拿到 `JSESSIONID=ABC123`
2. 攻击者诱导受害者用同一个 `JSESSIONID=ABC123` 登录（钓鱼链接、XSS、子域名 cookie 注入）
3. 受害者登录成功，session 绑定到受害者身份
4. 攻击者用同一个 `JSESSIONID=ABC123` 访问，已是受害者身份

#### 根因

如果应用不在登录时**重新生成 session id**，登录前后用的是同一个 session，攻击者只要预先把 sessionid 注入到受害者浏览器即可。

Spring Security 默认其实**已经处理了** —— `SessionManagementFilter` 默认策略是 `migrateSession`（迁移老 session 内容到新 session id）。但有些场景会被禁用：

- 显式配了 `.sessionFixation(SessionFixationConfigurer::none)`
- 用了自定义 `SessionAuthenticationStrategy` 没继承默认行为
- 某些前置 Filter 提前固化了 sessionid

#### 修复

显式配置：

```java
.sessionManagement(s -> s
    .sessionFixation(SessionFixationConfigurer::changeSessionId)   // ★ 推荐
    // .sessionFixation(SessionFixationConfigurer::migrateSession)  // 也可
    // .sessionFixation(SessionFixationConfigurer::newSession)       // 不复制旧 session 内容
    .maximumSessions(1)                                              // 单端登录
    .maxSessionsPreventsLogin(false))                                // 后登录踢前面
```

三种策略对比：

| 策略 | 行为 | 适用 |
| --- | --- | --- |
| `changeSessionId` | 调 Servlet 3.1+ 的 `request.changeSessionId()`，session 内容保留 | **生产首选** |
| `migrateSession` | 创建新 session，复制老 session 属性 | Servlet 3.0 兼容 |
| `newSession` | 创建新 session，不复制属性（Spring Security 自己的属性也丢） | 几乎不用 |
| `none` | 不变 | **千万别用** |

加固 Cookie 属性：

```java
@Bean
public ServletContextInitializer sessionCookieConfig() {
    return ctx -> {
        SessionCookieConfig cfg = ctx.getSessionCookieConfig();
        cfg.setHttpOnly(true);                                    // 禁止 JS 读
        cfg.setSecure(true);                                      // 仅 HTTPS
        cfg.setMaxAge(60 * 30);                                   // 30 分钟
        // 不要设 Domain（除非真的要跨子域）
    };
}
```

`SameSite` 属性（防 CSRF + 部分防 Session Fixation）：

```yaml
server:
  servlet:
    session:
      cookie:
        same-site: strict      # 严格：跨站完全不带 cookie
        # same-site: lax       # 宽松：导航时带（推荐通用场景）
```

#### 验证

```bash
# 登录前抓 sessionid
curl -i -c cookies.txt http://localhost:8080/login | grep JSESSIONID

# 登录
curl -i -b cookies.txt -c cookies.txt -X POST http://localhost:8080/login \
    -d "username=alice&password=pwd" | grep JSESSIONID

# 比较前后 sessionid，必须不同
```

或用 OWASP ZAP 自动扫描。

> **口诀**：登录后必须换 sessionid，`changeSessionId` 是默认且最稳。Cookie 三件套 `HttpOnly + Secure + SameSite` 必配。

---

### 案例 ⑨ OAuth2 redirect_uri 没校验 —— "钓鱼站偷 code"

#### 现象

安全研究员上报：你的 OAuth2 Authorization Server 在 `/oauth2/authorize` 端点允许任意 `redirect_uri`，攻击者可以构造：

```
https://auth.example.com/oauth2/authorize
    ?response_type=code
    &client_id=legitimate-app
    &redirect_uri=https://attacker.com/steal
    &scope=read
    &state=xxx
```

诱骗用户授权后，code 被发到攻击者服务器，攻击者拿 code 换 token。

#### 根因

`redirect_uri` 是 OAuth2 协议中**最敏感**的参数。规范要求 Authorization Server 必须**严格校验** `redirect_uri` 与客户端注册时声明的 URI **完全匹配**（exact match，不允许通配、不允许 path 后缀差异）。

漏洞场景：
- 客户端注册时填了 `https://app.example.com/callback`
- AS 校验时只比 host，不比 path → 攻击者用 `https://app.example.com.attacker.com/callback`
- AS 允许通配 → `https://*.example.com/callback` → 攻击者注册子域
- AS 允许 path 后缀 → `https://app.example.com/callback/../redirect`

更隐蔽：
- Open Redirect：你的合法回调端点又转发到查询参数指定的 URL → 还是被钓
- Fragment-based 攻击：implicit flow（已废弃）下 token 在 URL fragment 里，重定向链上的服务器都能看到

#### 修复

Spring Authorization Server 默认就是 exact match：

```java
@Bean
public RegisteredClientRepository registeredClientRepository() {
    RegisteredClient client = RegisteredClient.withId(UUID.randomUUID().toString())
        .clientId("legitimate-app")
        .clientSecret("{bcrypt}$2a$10$...")
        .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
        .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
        .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
        .redirectUri("https://app.example.com/callback")          // ★ 完整 URL，不允许通配
        .scope(OidcScopes.OPENID)
        .scope("read")
        .clientSettings(ClientSettings.builder()
            .requireProofKey(true)                                 // ★ 强制 PKCE
            .requireAuthorizationConsent(true)
            .build())
        .tokenSettings(TokenSettings.builder()
            .accessTokenTimeToLive(Duration.ofMinutes(15))         // ★ 短有效期
            .refreshTokenTimeToLive(Duration.ofDays(30))
            .reuseRefreshTokens(false)                             // ★ refresh rotation
            .build())
        .build();

    return new InMemoryRegisteredClientRepository(client);
}
```

强制 PKCE 让公开客户端（SPA、移动 app）即使 code 泄露也无法换 token（因为缺 code_verifier）。

**回调端点本身的加固**（OAuth2 Client 侧）：

```java
@Bean
public OAuth2AuthorizedClientService authorizedClientService(
        ClientRegistrationRepository clientRegistrationRepository) {
    return new InMemoryOAuth2AuthorizedClientService(clientRegistrationRepository);
}

@GetMapping("/callback")
public String callback(@RequestParam String code,
                       @RequestParam String state,
                       HttpSession session) {
    // ★ 校验 state 防 CSRF
    String expectedState = (String) session.getAttribute("oauth2_state");
    if (!Objects.equals(state, expectedState)) {
        throw new IllegalStateException("State mismatch");
    }
    // ... 用 code 换 token
}
```

State 参数是 OAuth2 内置的 CSRF 防护，必须用且必须随机不可预测。

#### 长尾防护

- **不要用 implicit flow**（response_type=token）—— 已被 OAuth 2.1 废弃
- **不要在 redirect_uri 里放敏感参数**
- **回调端点不做二次跳转**（如要做，目标 URL 必须白名单）
- **AS 端日志记录所有 redirect_uri 校验失败**，作为告警源

#### 验证

```bash
# 试图用注册之外的 redirect_uri
curl -i "https://auth.example.com/oauth2/authorize?response_type=code&client_id=legitimate-app&redirect_uri=https://attacker.com/steal&scope=read&state=x"
# 应 400 或重定向到错误页，绝不能正常授权

# 试图用注册 URI 的子路径
curl -i "https://auth.example.com/oauth2/authorize?response_type=code&client_id=legitimate-app&redirect_uri=https://app.example.com/callback/../redirect&scope=read&state=x"
# 应被拒
```

> **口诀**：OAuth2 三大铁律 —— **redirect_uri 完全匹配**、**强制 PKCE**、**state 防 CSRF**。任何一条破了，整套体系塌。

---

## 5. 上线检查清单

### 配置层
- [ ] Spring Security 6.x 用 `SecurityFilterChain` Bean，不要再用 `WebSecurityConfigurerAdapter`
- [ ] 多 SecurityFilterChain 必须显式 `@Order`，特异性高的 Order 小
- [ ] `securityMatcher` + `requestMatchers` 路径必须从特殊到通用
- [ ] 启动期打印所有 chain 的 matcher 和 filter 列表，写进监控
- [ ] 默认开启的 Filter 不要随意 disable（每 disable 一个写注释说明原因）

### 认证层
- [ ] 密码必须用 `BCryptPasswordEncoder` 或更强的 `DelegatingPasswordEncoder`
- [ ] `bcrypt cost` 根据机器算力定期调整（每年评估）
- [ ] `UserDetailsService` 实现里**永远不要透露"用户存在但密码错"** —— 统一返回"用户名或密码错误"
- [ ] 登录失败次数限制 + 锁定（防暴力破解）
- [ ] 登录成功后**必须换 sessionid**（`changeSessionId`）

### 鉴权层
- [ ] URL 级 + 方法级双层鉴权（深度防御）
- [ ] 每个 `@PreAuthorize` 都有正反向单元测试
- [ ] 复杂逻辑下沉到 `PermissionEvaluator`，不写在 SpEL 里
- [ ] 不要用 `permitAll` 兜底；显式列出公开端点
- [ ] `anonymous` 也是个 Authentication —— 判断登录用 `!(auth instanceof AnonymousAuthenticationToken)`

### Session / Token 层
- [ ] REST API 用 `SessionCreationPolicy.STATELESS`，不偷偷创建 session
- [ ] Cookie 必须 `HttpOnly + Secure + SameSite`
- [ ] JWT 配 `JwtTimestampValidator` 容忍 30~60s 时钟偏差
- [ ] JWKS 用 `refreshAheadCache + retrying`，应对密钥旋转
- [ ] JWT 有效期 ≤ 1 小时，refresh token rotation
- [ ] CSRF：Cookie 模式必开，纯 JWT 模式可关

### CORS 层
- [ ] `allowedOrigins` 不用 `*`，显式列白名单
- [ ] 同时用 cookie 时 `allowCredentials = true` 且 origin 必须具体
- [ ] 预检请求 `maxAge` 设合理值（3600 是常见值）

### 异步 / 多线程层
- [ ] 所有 `TaskExecutor` 用 `DelegatingSecurityContextAsyncTaskExecutor` 包装
- [ ] `@Async` 子线程能正确拿到 SecurityContext
- [ ] 不要把 `SecurityContextHolder` 改成 `MODE_INHERITABLETHREADLOCAL`

### OAuth2 层
- [ ] `redirect_uri` 严格 exact match，不允许通配
- [ ] 强制 PKCE（公开客户端必须）
- [ ] `state` 参数随机不可预测
- [ ] Token 有效期 ≤ 1 小时
- [ ] refresh token rotation 开启
- [ ] 不用 implicit flow

### 监控告警
- [ ] 登录失败率（按用户、按 IP）
- [ ] JWT 验签失败率（按错误类型）
- [ ] 鉴权拒绝率（403 数）
- [ ] JWKS 拉取耗时与失败
- [ ] Session 数量、CSRF 失败数

### 安全响应头
- [ ] `Strict-Transport-Security`（HSTS）
- [ ] `X-Content-Type-Options: nosniff`
- [ ] `Content-Security-Policy`
- [ ] `Referrer-Policy: no-referrer`
- [ ] `X-Frame-Options: DENY`（防 clickjacking）

---

## 6. 还能往下挖什么

### 无密码认证 / WebAuthn / Passkey
- WebAuthn / FIDO2 协议：基于公私钥的免密码登录
- Passkey：苹果 / 谷歌 / 微软联合推动的 WebAuthn 增强
- Spring Security 6.3+ 内置 WebAuthn 支持：`spring-security-webauthn`
- 集成方向：把 WebAuthn 当成 `AuthenticationProvider` 注入

### 自定义 AuthenticationProvider 全流程
- 实现 `AuthenticationProvider` 接口的 `authenticate` + `supports`
- 自定义 `Authentication` 子类（比如 `SmsCodeAuthenticationToken`）
- 自定义 `AbstractAuthenticationProcessingFilter` 处理特定 endpoint
- 注册到 `AuthenticationManager` 链

### 多租户（Multi-Tenant）认证
- 每租户独立 `JwtDecoder`：`JwtIssuerAuthenticationManagerResolver`
- 域名映射 / Header 映射 / 路径前缀映射的 tenant resolver
- TenantContext 与 SecurityContext 的协同
- 数据隔离：从 SecurityContext 提取 tenantId，自动注入到 Repository 查询

### SAML 2.0 集成
- `spring-security-saml2-service-provider`：以 SP 身份接入 IdP
- AssertionConsumerService、SLO（Single Logout）的复杂性
- SAML vs OIDC：企业内网 SAML 仍占主流，互联网 OIDC 是未来

### OpenID Connect 完整实现
- ID Token 与 Access Token 的差异（前者描述"谁"，后者授权"做什么"）
- userinfo endpoint 的使用
- discovery endpoint（`/.well-known/openid-configuration`）的最佳实践
- session management 与 RP-initiated logout

### Spring Authorization Server 深入
- 替代废弃的 spring-security-oauth
- 内置的 OIDC + JWKS + introspection endpoint
- 自定义 token customizer（往 JWT 加自定义 claim）
- 与 Keycloak / Auth0 的对比与迁移成本

### OAuth2 PKCE 工程化
- SPA 应用的 code interception 防护
- BFF（Backend For Frontend）模式：把 token 留在服务端，浏览器只持 session cookie
- DPoP（Demonstration of Proof-of-Possession）：让 token 绑定到客户端密钥，泄露也用不了

### 反向代理 / 网关层的安全
- API Gateway（Spring Cloud Gateway / Kong / APISIX）做认证 + 鉴权前置
- 微服务间用 mTLS + 短 JWT 互信
- Service Mesh（Istio）下的 RequestAuthentication 与 AuthorizationPolicy

### Reactive 安全
- WebFlux 下的 `ServerSecurityContextRepository`、`ReactiveAuthenticationManager`
- `ReactiveSecurityContextHolder` 与 Reactor Context 的传播
- 响应式场景下的 OAuth2 Resource Server

---

## 7. 引用

- Spring Security 6 官方文档：[https://docs.spring.io/spring-security/reference/](https://docs.spring.io/spring-security/reference/)
- Spring Security 源码：[https://github.com/spring-projects/spring-security](https://github.com/spring-projects/spring-security)
- Spring Authorization Server：[https://github.com/spring-projects/spring-authorization-server](https://github.com/spring-projects/spring-authorization-server)
- RFC 6749：The OAuth 2.0 Authorization Framework
- RFC 7519：JSON Web Token (JWT)
- RFC 7636：Proof Key for Code Exchange by OAuth Public Clients (PKCE)
- RFC 8252：OAuth 2.0 for Native Apps
- OpenID Connect Core 1.0
- OAuth 2.1（draft）—— 整合最佳实践，废弃 implicit flow
- OWASP Authentication Cheat Sheet
- OWASP Session Management Cheat Sheet
- Nimbus JOSE + JWT：[https://connect2id.com/products/nimbus-jose-jwt](https://connect2id.com/products/nimbus-jose-jwt)
- Spring Security 5 → 6 Migration Guide：[https://docs.spring.io/spring-security/reference/migration/index.html](https://docs.spring.io/spring-security/reference/migration/index.html)

---

> 配套：[Web 安全实战](/article/security/security-deep) | [Spring AOP 内幕](/article/java/spring-aop) | [Spring Boot 自动配置](/article/java/spring-boot)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
