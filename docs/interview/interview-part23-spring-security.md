# Spring Security 安全框架面试题（第23期）

> 适用岗位：Java后端开发 / 安全工程师 / 架构师
> 覆盖范围：Spring Security 5.x～6.x 核心组件、认证授权、OAuth2、JWT、分布式Session

---

## 1. Spring Security 核心架构：FilterChain、Security Filter、DelegatingFilterProxy

### 题目
请描述 Spring Security 的核心架构，特别是 `FilterChain`、`Security Filter` 和 `DelegatingFilterProxy` 三者的关系与分工。

### 核心答案

Spring Security 本质上是一组 **Servlet Filter 链**，采用责任链模式层层叠加。

| 组件 | 职责 | 关键点 |
|------|------|--------|
| **DelegatingFilterProxy** | 桥接 Servlet 容器与 Spring 容器 | 它是 Servlet Filter，在 `web.xml`/Java Config 中注册；持有 `ApplicationContext` 引用，将请求**委托**给真正的 Filter（Bean）处理 |
| **FilterChainProxy（SecurityFilterChain）** | 统一管理所有 Security Filter | 内部持有一个 `List<Filter>` 列表（`SecurityFilterChain`），是所有安全过滤器的统一入口 |
| **Security Filter** | 真正执行安全逻辑 | 如 `UsernamePasswordAuthenticationFilter`、`SessionManagementFilter`、`CsrfFilter` 等，各司其职 |

**请求处理链路：**

```
请求 → DelegatingFilterProxy → FilterChainProxy → [Security Filters按顺序执行] → 目标资源
```

`DelegatingFilterProxy` 通过 Servlet 容器标准 Filter 机制注册，但实际逻辑委托给 Spring Bean `FilterChainProxy`。`FilterChainProxy` 根据请求路径匹配对应的 `SecurityFilterChain`（可配置多条链），然后依次执行链中每个 Filter。

### 追问方向
- `FilterChain` 与 `SecurityFilterChain` 的区别是什么？
- 如何通过 Java Config 配置多条 `SecurityFilterChain` 并实现请求路径分流？
- `DelegatingFilterProxy` 的 `targetBeanName` 属性何时需要手动指定？

### 避坑提示
- 常见混淆：`FilterChain` 是 Servlet API 的接口（`javax.servlet.FilterChain`），而 `SecurityFilterChain` 是 Spring Security 的配置类，两者不是同一个东西。
- `DelegatingFilterProxy` 不是 FilterChain 的终点，它只是 delegation 模式，Filter 处理逻辑全在下游。

---

## 2. 认证流程：UsernamePasswordAuthenticationFilter、AuthenticationManager

### 题目
描述 Spring Security 中一次完整用户名/密码登录的认证流程，核心Filter和组件是如何协作的？

### 核心答案

**认证流程（以表单登录为例）：**

```
1. 用户提交 POST /login {username, password}
2. DelegatingFilterProxy → FilterChainProxy → UsernamePasswordAuthenticationFilter
3. Filter 拦截请求，提取 username/password → 组装 UsernamePasswordAuthenticationToken (unauthenticated)
4. Filter 调用 AuthenticationManager.authenticate(token)
5. AuthenticationManager 遍历 Provider 列表，找到支持 UsernamePasswordAuthenticationToken 的 Provider
6. Provider 调用 UserDetailsService.loadUserByUsername() 查询用户
7. Provider 调用 PasswordEncoder.matches() 比对密码
8. 认证成功 → 返回 Authentication (authenticated) → 存入 SecurityContext
9. Filter 执行 SecurityContextRepository.saveContext()（默认是 SaveContextOnUpdateAuthenticationErrorHandler）
10. 认证失败 → 抛出 AuthenticationException → 触发 AuthenticationEntryPoint（重定向到登录页）
```

**核心组件：**

- **UsernamePasswordAuthenticationFilter**：负责从请求中提取用户名密码，构建 `UsernamePasswordAuthenticationToken`（未认证状态）。
- **AuthenticationManager**：接口，仅定义 `authenticate()` 方法。默认实现是 `ProviderManager`，内部持有 `List<AuthenticationProvider>`。
- **AuthenticationProvider**：具体认证逻辑，`DaoAuthenticationProvider` 是最常用的实现，组合了 `UserDetailsService` + `PasswordEncoder`。

### 追问方向
- `Authentication` 接口的 `getAuthorities()` 返回的是什么？何时填充的？
- `ProviderManager.authenticate()` 循环遍历 Provider 时，如果第一个 Provider 投了"不支持"票（返回 null），流程继续吗？
- 自定义登录逻辑时，为什么建议**替换**而不是**继承** `UsernamePasswordAuthenticationFilter`？

### 避坑提示
- 认证成功后 `SecurityContext` 默认存储在 `ThreadLocal` 中，请求结束前必须显式保存（通过 `SecurityContextHolderFilter`），否则后续请求可能丢失。
- `ProviderManager` 有一个重要的 `eraseCredentials()` 机制：认证成功后会自动将 `Credentials`（密码）置空，防止意外泄露。

---

## 3. 授权流程：AccessDecisionManager、WebSecurityConfigurerAdapter

### 题目
Spring Security 的授权（访问控制）流程是怎样的？`AccessDecisionManager` 和 `WebSecurityConfigurerAdapter`（或 `SecurityFilterChain`）在其中的角色是什么？

### 核心答案

**授权流程：**

```
请求到达 → FilterSecurityInterceptor（在 Security Filter 链末端）
  → 读取 @SecurityConfig 中的配置（哪些url需要什么角色）
  → AccessDecisionManager.decide(authentication, object, config)
    → 遍历 AccessDecisionVoter 列表逐一投票
    → 投票结果：ACCESS_GRANTED / ACCESS_DENIED / ACCESS_ABSTAIN
    → AccessDecisionManager 根据策略（默认 AffirmativeBased）统计结果
  → 允许通过 / 抛出 AccessDeniedException
```

**核心组件：**

| 组件 | 角色 |
|------|------|
| **FilterSecurityInterceptor** | 拦截器入口，读取配置_attrs，触发决策 |
| **AccessDecisionManager** | 决策策略实现（`AffirmativeBased`/`ConsensusBased`/`UnanimousBased`） |
| **AccessDecisionVoter** | 投票器，`RoleVoter`、`AuthenticatedVoter`、`WebExpressionVoter` 等 |
| **SecurityMetadataSource** | 提供"资源→访问配置"的元数据（由 `HttpSecurity` 配置填充） |

**`WebSecurityConfigurerAdapter`（已废弃，推荐 `SecurityFilterChain` @Bean）：**

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .antMatchers("/admin/**").hasRole("ADMIN")
            .antMatchers("/user/**").hasAnyRole("USER", "ADMIN")
            .anyRequest().authenticated()
        )
        .formLogin(Customizer.withDefaults());
    return http.build();
}
```

### 追问方向
- `AccessDecisionVoter` 投 `ACCESS_ABSTAIN` 时，算通过还是拒绝？
- `AffirmativeBased`（一票通过）、`ConsensusBased`（多数通过）、`UnanimousBased`（全票通过）三种策略分别适用于什么场景？
- 如何实现自定义投票器，比如基于 IP 地址的访问控制？

### 避坑提示
- `WebSecurityConfigurerAdapter` 在 Spring Security 5.7+ 已废弃，强烈建议迁移到 `@Bean SecurityFilterChain` 的方式。
- 配置授权时 `.anyRequest().authenticated()` 要放在**最后**，因为匹配规则是**按顺序**的，一旦匹配到就短路。

---

## 4. 会话管理：Session Fixation 攻击、Session 并发控制、@SessionAttributes

### 题目
Spring Security 如何防护 Session Fixation 攻击？如何实现会话并发控制？`@SessionAttributes` 与安全框架的关系是什么？

### 核心答案

**Session Fixation（会话固定）防护：**

攻击者先获取一个有效 session ID，诱骗用户使用该 ID 登录，登录后攻击者凭借该 ID 劫持用户会话。

Spring Security 提供的防护策略（`sessionFixation()` 配置）：

| 策略 | 行为 |
|------|------|
| `migrateSession`（默认） | 登录后创建新 session，迁移旧 session 属性 |
| `changeSessionId` | 不创建新 session，但 Servlet 3.1+ 会改变 session ID（容器原生防护） |
| `newSession` | 登录后创建全新空白 session，不迁移旧属性 |
| `none` | **不防护**，保持旧 session（已废弃） |

**会话并发控制：**

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .sessionManagement(session -> session
            .maximumSessions(1)                    // 最多一个活跃session
            .maxSessionsPreventsLogin(true)       // 达到上限后阻止新登录（而非踢掉旧）
            .expiredUrl("/session-expired")       // session过期重定向
        );
}
```

底层使用 `SessionRegistry`（默认 `SpringSessionBackedSessionRegistry`）维护活跃 session 映射。

**`@SessionAttributes` 与安全：**

`@SessionAttributes("user")` 是 Spring MVC 注解，将模型数据存入 **HTTP Session**。它与 Spring Security 的 `SecurityContext` 是**两套独立的 session 属性**。`@SessionAttributes` 不会自动参与 Spring Security 的会话安全管理，不会因并发控制被失效，需手动处理。

### 追问方向
- `maximumSessions` 限制的是同用户还是同 session ID？集群环境下如何共享 `SessionRegistry`？
- `changeSessionId` 和 `migrateSession` 在会话固定防护效果上有何本质区别？
- 如果使用 Spring Session + Redis，`SessionRegistry` 是如何实现的？

### 避坑提示
- 启用并发控制后，默认实现中**旧 session 会被 invalidate**，用户被踢下线。如果希望新登录失败而非踢人，必须配置 `.maxSessionsPreventsLogin(true)`。
- `@SessionAttributes` 存储的 session 属性在并发 session 被 invalidate 时**也会被清除**，可能导致业务数据丢失，需要评估风险。

---

## 5. CSRF 防护：Token 存储、Synchronizer Token Pattern

### 题目
Spring Security 的 CSRF 防护机制是什么？Token 如何存储和使用？Synchronizer Token Pattern 的核心原理是什么？

### 核心答案

**Spring Security CSRF 策略：**

Spring Security 5.5+ 默认启用 CSRF 防护（之前的版本可选）。使用 **Synchronizer Token Pattern（同步器令牌模式）**。

**Token 存储方式：**

| 存储位置 | 实现 |
|----------|------|
| **HTTP Session** | 默认，`CsrfToken` 存入 `HttpSession`，请求时从 session 取出比对 |
| **Cookie** | `CookieCsrfTokenRepository`（浏览器可自动读取 `XSRF-TOKEN` cookie 并回传为 `X-XSRF-TOKEN` header） |

**Token 结构（`CsrfToken` 接口）：**

```java
public interface CsrfToken extends Serializable {
    String getHeaderName();    // 默认 "X-XSRF-TOKEN"
    String getParameterName(); // 默认 "_csrf"
    String getToken();         // 随机令牌值
}
```

**核心工作流程：**

```
服务端渲染表单时：
  CsrfFilter 生成 CsrfToken → 写入 HttpSession → 通过页面hidden或JS变量暴露给前端

表单提交时（POST/PUT/DELETE）：
  客户端必须携带 _csrf 参数或 X-XSRF-TOKEN header
  CsrfFilter 从 session 取出 token → 比对请求中的 token
    ✓ 匹配 → 放行
    ✗ 不匹配/缺失 → 403 Forbidden
```

**Cookie 存储模式（适合前后端分离 + SPA）：**

```java
CsrfTokenRepository repository = CookieCsrfTokenRepository.withHttpOnlyFalse();
// 响应头：Set-Cookie: XSRF-TOKEN=<value>; Path=/; Secure; SameSite=Lax
// 前端JS读取document.cookie中的XSRF-TOKEN，axios等库自动加到header
```

### 追问方向
- 为什么 `CookieCsrfTokenRepository` 要设置 `httpOnlyFalse`？这对 XSS 有何影响？
- 前后端分离场景下，CORS 预检请求（OPTIONS）需要携带 CSRF token 吗？
- 禁用 CSRF 防护 `.csrf().disable()` 在什么场景下可以接受？微服务内部调用可以禁用吗？

### 避坑提示
- CSRF token 是**每个用户会话级别**的随机值，不是全局的，必须与用户 session 绑定——不可使用固定 token 或全局 token，否则失去防护意义。
- 敏感业务（银行转账/密码修改）禁用 CSRF 前必须确认有其他等效防护（如旧密码确认 / 二次验证）。

---

## 6. 密码加密：BCryptPasswordEncoder、Argon2、密码哈希加盐

### 题目
Spring Security 如何实现安全的密码存储？`BCryptPasswordEncoder`、`Argon2PasswordEncoder` 的原理是什么？加盐的作用是什么？

### 核心答案

**加盐（Salting）的必要性：**

相同密码用同一算法哈希后结果固定。攻击者可用彩虹表（预计算哈希表）或字典攻击批量破解。**加盐**是在哈希前拼接随机盐值，使相同密码每次生成不同哈希值，破坏彩虹表攻击的前提。

**Spring Security 密码编码器体系：**

| 实现 | 算法 | 特点 |
|------|------|------|
| `BCryptPasswordEncoder` | BCrypt（基于 Blowfish） | **自适应**，可配置 cost factor（默认10），自带随机盐，Spring 默认推荐 |
| `Argon2PasswordEncoder` | Argon2id（2015年Password Hashing Competition冠军） | 内存hard to implement，2023年后推荐替代BCrypt |
| `SCryptPasswordEncoder` | Scrypt | 内存hard，适用于高安全场景 |
| `PBKDF2PasswordEncoder` | PBKDF2 | NIST 批准，但相对慢 |

**BCrypt 工作原理：**

```java
PasswordEncoder encoder = new BCryptPasswordEncoder();
// 存储：用encoder.encode(rawPassword) → "$2a$10$..."格式（含盐值和cost）
String hashed = encoder.encode("password123"); // 每次结果不同（随机盐）

// 验证：
encoder.matches(rawPassword, hashed); // 自动提取盐再哈希比对
```

`$2a$10$N9qo8uLOickgx2ZMRZoMye.IxDqZ5p3LY5D5D3...` 格式：`$2a$`算法标识 → `$10$`cost因子 → 22字符盐 → 31字符哈希。

**Argon2（Spring Security 5.8+）：**

```java
Argon2PasswordEncoder encoder = new Argon2PasswordEncoder(16, 32, 1, 1<<16, 3);
// 参数：saltLength, hashLength, parallelism, memory, iterations
```

### 追问方向
- BCrypt 的 cost factor（计算强度）如何影响性能和安全？生产环境推荐值是多少？
- 旧系统用 MD5/SHA-1 存储密码，如何在不强制用户改密码的前提下迁移到 BCrypt？
- 什么是"慢哈希"（slow hash）？BCrypt/Argon2 为什么要慢？

### 避坑提示
- 不要自己实现盐生成或拼接逻辑，使用框架内置的 `PasswordEncoder`，它内置安全随机盐生成。
- 多实例部署时所有节点必须使用**相同的 `PasswordEncoder` 实例**（或兼容的编码器），否则同一密码不同实例验证结果不一致。
- Argon2 需要 `spring-security-crypto` 额外依赖或 JNI 库，初次使用前确认依赖完整引入。

---

## 7. OAuth2.0：四种授权模式、授权码模式流程

### 题目
OAuth 2.0 的四种授权模式是什么？请详细描述授权码模式（Authorization Code Grant）的完整流程。

### 核心答案

**四种授权模式：**

| 模式 | 适用场景 | 关键特征 |
|------|----------|----------|
| **Authorization Code（授权码）** | 有后端服务器的Web应用 | 前后端分离，使用 code 换 token，code 仅一次使用 |
| **Implicit（隐式）** | SPA等纯前端应用 | 直接返回 access_token，已不推荐（简化版+安全性差） |
| **Resource Owner Password Credentials（密码凭证）** | 高度可信的老系统迁移 | 用户直接把账号密码给客户端，适合内部系统 |
| **Client Credentials（客户端凭证）** | 服务间通信（M2M） | 无用户参与，客户端以自己身份访问资源 |

**授权码模式完整流程（RFC 6749）：**

```
┌─────────┐                    ┌──────────────┐                    ┌─────────────┐
│  User   │                    │   Auth Server │                    │  Resource   │
│ Browser │                    │   (Authorization Server) │                    │   Server    │
└────┬────┘                    └───────┬──────┘                    └──────┬──────┘
     │                                  │                                  │
     │  1. 访问受保护资源               │                                  │
     │─────────────────────────────────>│                                  │
     │                                  │                                  │
     │  2. 重定向到登录/授权页           │                                  │
     │<─────────────────────────────────│                                  │
     │                                  │                                  │
     │  3. 用户登录并点击"授权"          │                                  │
     │─────────────────────────────────>│                                  │
     │                                  │                                  │
     │  4. 生成 Authorization Code，重定向│                                  │
     │<─────────────────────────────────│                                  │
     │                                  │                                  │
     │  5. 用 Code 回调到客户端后端       │                                  │
     │─────────────────────────────────>│                                  │
     │                                  │                                  │
     │  6. 后端用 ClientId+ClientSecret+Code│                                │
     │     换 AccessToken (+ RefreshToken)│                                │
     │─────────────────────────────────>│                                  │
     │                                  │                                  │
     │  7. 返回 AccessToken              │                                  │
     │<─────────────────────────────────│                                  │
     │                                  │                                  │
     │  8. 用 AccessToken 请求资源        │                                  │
     │───────────────────────────────────────────────────────────────>│
     │                                  │                                  │
     │  9. 返回受保护资源                │                                  │
     │<───────────────────────────────────────────────────────────────│
```

**核心请求示例：**

授权请求：
```
GET /authorize?response_type=code&client_id=APP_ID&redirect_uri=https://APP/callback&scope=read
```

Token 请求（后端到后端）：
```
POST /token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&code=AUTH_CODE&redirect_uri=https://APP/callback
&client_id=APP_ID&client_secret=APP_SECRET
```

### 追问方向
- 授权码模式为什么要用 code 换 token，而不是直接返回 token？中间多一步的意义是什么？
- `redirect_uri` 的校验策略是什么？为什么不允许完全通配的 `*`？
- Refresh Token 的有效期一般设置多长？如何实现 Refresh Token 的撤销（revocation）？

### 避坑提示
- **Implicit 模式已废弃**（OAuth 2.1 规范中移除），SPA 应使用 Authorization Code + PKCE。
- `client_secret` 是**后端到后端**通信时使用的，绝不能暴露在前端代码中（所以纯前端 SPA 必须使用 PKCE 而非普通的 Authorization Code）。
- 授权服务器必须对 `redirect_uri` 做精确匹配或白名单校验，否则存在授权码劫持风险。

---

## 8. JWT：Token 结构（Header/Payload/Signature）、无状态认证

### 题目
JWT 的结构是什么？每个部分分别存储什么内容？请说明 JWT 在无状态认证中的工作原理及优缺点。

### 核心答案

**JWT（JSON Web Token）结构：**

```
Header.Payload.Signature
```

三部分均为 **Base64URL** 编码，用 `.` 拼接。

**Header（头部）：**
```json
{
  "alg": "RS256",       // 算法：HMAC-SHA256 / RS256 / ES256
  "typ": "JWT"          // 类型
}
```

**Payload（载荷）：**
```json
{
  "iss": "auth-server",     // issuer：签发者
  "sub": "user123",         // subject：用户标识
  "aud": "my-api",          // audience：受众
  "exp": 1715000000,        // expiration time
  "iat": 1714996400,        // issued at
  "nbf": 1714996400,        // not before
  "jti": "unique-id",       // JWT ID：防重放
  "roles": ["ADMIN"],
  "permissions": ["read", "write"]
}
```

**Signature（签名）：**

以 RS256 为例：
```
Signature = RS256(
  Base64URL(Header) + "." + Base64URL(Payload),
  私钥
)
```

**无状态认证流程：**

```
登录成功 → 服务端用私钥签发 JWT → 返回给客户端
客户端请求时携带 Header: Authorization: Bearer <JWT>
服务端用公钥验证 JWT 签名 → 解析 Payload → 提取用户身份和权限
  ✓ 验证通过 → 直接放行（无需查数据库/Redis）
  ✗ 验证失败 → 401 Unauthorized
```

**优缺点：**

| 优点 | 缺点 |
|------|------|
| 无状态，可水平扩展 | 签发后无法撤销（必须依赖黑名单或短有效期） |
| 减少存储压力（无需 Session Store） | Payload 过大影响网络开销 |
| 天然支持跨域认证 | Token 泄露后难以控制 |

### 追问方向
- JWT 的 `exp`、`nbf`、`iat` 时间戳分别是什么含义？时间不同步（时钟漂移）时会有什么影响？
- JWS（带签名）和 JWE（加密）的区别是什么？什么场景用 JWE？
- "JWT 无状态" 意味着完全不需要服务端存储吗？Refresh Token 的场景呢？

### 避坑提示
- JWT 的 **Payload 是 Base64URL 编码但未加密**，敏感信息（如密码）不应放入 JWT Payload。
- JWT 的签名算法 `none`（不签名）和 `HS256`（对称签名）的密钥如果配置错误，可能被伪造。建议使用 RS256/ES256 等非对称算法。
- Access Token 建议有效期控制在 **5分钟～1小时**，配合 Refresh Token 实现续期，避免 Token 泄露后影响时间过长。

---

## 9. Spring Security + JWT：自定义 Filter、Token 验证

### 题目
如何在 Spring Security 中集成 JWT 自定义认证 Filter？请描述完整的实现思路，包括 Filter 位置、Token 解析与验证流程。

### 核心答案

**整体架构：**

```
JWT Filter（在 UsernamePasswordAuthenticationFilter 之前）→ 解析 Token → 验证签名 → 设置 SecurityContext → 继续 Filter 链
```

**实现步骤：**

**Step 1：创建 JWT 工具类**

```java
@Service
public class JwtService {
    @Value("${jwt.secret}")
    private String secret;

    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    public boolean isTokenValid(String token) {
        try {
            Jwts.parser()
                .verifyWith(Keys.hmacShaKeyFor(secret.getBytes()))
                .build()
                .parseSignedClaims(token);
            return !isTokenExpired(token);
        } catch (JwtException e) {
            return false;
        }
    }

    private Claims extractAllClaims(String token) {
        return Jwts.parser()
            .verifyWith(Keys.hmacShaKeyFor(secret.getBytes()))
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }
}
```

**Step 2：自定义 Filter**

```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain
    ) throws ServletException, IOException {
        final String authHeader = request.getHeader("Authorization");
        final String jwt;
        final String username;

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response); // 放行，由后续Filter处理
            return;
        }

        jwt = authHeader.substring(7);
        username = jwtService.extractUsername(jwt);

        // 已认证则跳过（避免重复认证）
        if (username != null &&
            SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails user = userDetailsService.loadUserByUsername(username);
            if (jwtService.isTokenValid(jwt, user)) {
                UsernamePasswordAuthenticationToken authToken =
                    new UsernamePasswordAuthenticationToken(
                        user, null, user.getAuthorities()
                    );
                authToken.setDetails(
                    new WebAuthenticationDetailsSource().buildDetails(request)
                );
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }
        filterChain.doFilter(request, response);
    }
}
```

**Step 3：注册 Filter（指定顺序）**

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/auth/**").permitAll()
                .anyRequest().authenticated()
            )
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .addFilterBefore(
                jwtAuthFilter,
                UsernamePasswordAuthenticationFilter.class  // 在表单登录Filter之前
            );
        return http.build();
    }
}
```

### 追问方向
- 为什么要在 `UsernamePasswordAuthenticationFilter` **之前**添加 JWT Filter？
- JWT Token 过期时，如何区分"无效 Token"和"过期 Token"，并返回不同的 401 信息？
- 如果使用了 Spring Security OAuth2 Resource Server 方式，JWT 验证方式有什么不同？

### 避坑提示
- 记住 `filterChain.doFilter(request, response)` **之后**仍有逻辑要执行（后置处理），但更常见的问题是忘记调用它导致请求被卡住。
- `SessionCreationPolicy.STATELESS` 不会阻止 `SecurityContext` 被存入 session，只是不创建新 session——实际 Stateless JWT 认证不需要任何 session 存储。
- Token 黑名单/撤销在纯 JWT 无状态模式下无法完美实现，只能用短有效期或引入 Redis 黑名单折中处理。

---

## 10. 权限表达式：hasRole/hasAuthority/permitAll/spel 表达式

### 题目
Spring Security 的权限表达式有哪些？`hasRole` 和 `hasAuthority` 有何区别？SPEL 表达式能做哪些复杂判断？

### 核心答案

**常见安全表达式：**

| 表达式 | 含义 | 示例 |
|--------|------|------|
| `permitAll` | 允许所有（已认证+未认证） | `.antMatchers("/public/**").permitAll()` |
| `denyAll` | 拒绝所有 | - |
| `anonymous` | 匿名用户 | `.antMatchers("/guest/**").anonymous()` |
| `authenticated` | 已认证用户 | `.anyRequest().authenticated()` |
| `hasRole('ROLE_USER')` | 拥有指定角色 | `.antMatchers("/admin/**").hasRole("ADMIN")` |
| `hasAuthority('ROLE_USER')` | 拥有指定权限 | 同上，效果等价 |
| `hasAnyRole('ADMIN','USER')` | 拥有任一角色 | - |
| `hasIpAddress('192.168.1.0/24')` | 来自指定IP | `.antMatchers("/internal/**").hasIpAddress("127.0.0.1")` |
| `rememberMe()` | 通过记住我登录 | - |
| `fullyAuthenticated()` | 非 rememberMe 的完整认证 | - |

**hasRole vs hasAuthority 的区别：**

```java
// 配置中写（底层都存为 Authority）：
.hasRole("ADMIN")          // 自动加前缀 ROLE_ → 存为 "ROLE_ADMIN"
.hasAuthority("ADMIN")    // 直接存为 "ADMIN"
```

```java
// 代码中比较：
authentication.getAuthorities() // Collection<GrantedAuthority>
                             // 遍历比较字符串 "ROLE_ADMIN"
```

两者本质上**完全相同**，都是 `GrantedAuthority` 接口的实现，区别仅在于：
- `hasRole()` 会自动拼接 `ROLE_` 前缀
- `hasAuthority()` 直接使用原值

**SPEL 表达式（@PreAuthorize 等注解中）：**

```java
// 基本用法
@PreAuthorize("hasRole('ADMIN')")
public void deleteUser(Long id);

// 复杂表达式
@PreAuthorize("hasRole('ADMIN') or (hasRole('USER') and #ownerId == authentication.principal.userId)")
public void updateOwnData(Long ownerId, Data data);

// 内置方法
@PreAuthorize("hasAuthority('SCOPE_read') and @permissionService.hasPermission(#docId, 'WRITE')")
public void editDocument(Long docId);

// 组合条件
@PreAuthorize("#amount <= 10000 and T(com.utils.SecurityUtils).isInternalNetwork()")
public void transferMoney(BigDecimal amount);
```

### 追问方向
- `@PostAuthorize` 和 `@PreAuthorize` 的执行时机有何不同？在返回后做权限校验有什么应用场景？
- `hasRole("ADMIN")` 和 `hasAuthority("ROLE_ADMIN")` 完全等价吗？有没有边界情况？
- 如何注册自定义的 SPEL 安全方法（`@PreAuthorize` 注解中使用 `@beanName` 引用 Spring Bean）？

### 避坑提示
- 在 `@PreAuthorize` 中写 SPEL 时，`#paramName` 直接引用方法参数，但如果参数是对象属性（如 `#user.id`），需要确保 SPEL 解析器能正确访问（通常能）。
- `permitAll()` 和 `anonymous()` 在 Filter 链中属于**早期处理**，如果需要做业务层鉴权，不要依赖它们。

---

## 11. 方法级安全：@PreAuthorize/@Secured/JSR250

### 题目
Spring Security 支持哪几种方法级安全注解？`@PreAuthorize`、`@Secured`、`@RolesAllowed`（JSR250）各有何特点？

### 核心答案

**三种方法级安全注解对比：**

| 特性 | `@PreAuthorize` | `@Secured` | `@RolesAllowed`（JSR250） |
|------|-----------------|------------|---------------------------|
| 包位置 | `spring-security-test` | Spring 自带 | `javax.annotation-security`（Java EE） |
| SPEL 支持 | ✅ 完整 SPEL | ❌ 不支持 | ❌ 不支持 |
| 表达式类型 | 任意布尔表达式 | 角色列表 | 角色列表 |
| 举例 | `@PreAuthorize("hasRole('ADMIN') and #amount > 0")` | `@Secured({"ROLE_ADMIN", "ROLE_USER"})` | `@RolesAllowed({"ROLE_ADMIN"})` |
| 默认前缀 | 无自动前缀 | 无自动前缀（需手动写 `ROLE_`） | 无自动前缀 |

**启用方法安全注解：**

```java
@EnableMethodSecurity   // Spring Security 5.x/6.x 推荐方式（替代 @EnableGlobalMethodSecurity）
public class SecurityConfig { ... }
```

**`@PreAuthorize` 完整示例：**

```java
@PreAuthorize("isAuthenticated()")
public String getUserInfo(String userId) { ... }

@PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.userId")
public void updateEmail(@Param("userId") String userId, String newEmail) { ... }

@PreAuthorize("#document.owner == authentication.principal.username")
public void deleteDocument(Document document) { ... }
```

**`@PostAuthorize`（后置检查）：**

```java
@PostAuthorize("returnObject.owner == authentication.principal.username or hasRole('ADMIN')")
public Document getDocument(Long id) { ... }
```

**`@Secured` 示例：**

```java
@Secured({"ROLE_MANAGER", "ROLE_ADMIN"})
public void approveExpense(Expense expense) { ... }
```

### 追问方向
- `@EnableMethodSecurity` 和旧的 `@EnableGlobalMethodSecurity(prePostEnabled=true)` 有何不同？`@EnableGlobalMethodSecurity` 在 Spring Security 6.x 中还能用吗？
- 方法级注解在 Spring AOP 代理下的限制是什么？内部方法调用（`this.xxx()`）会拦截吗？
- 如何将 `@PreAuthorize` 和自定义 `PermissionEvaluator` 结合，实现基于领域对象的权限控制（ACL）？

### 避坑提示
- 方法安全注解默认作用于 **Spring AOP 代理**，同一类内部 `this.method()` 调用不会触发拦截——这是 AOP 代理的固有限制。解决方案是注入自身或使用 AspectJ。
- `@Secured` 不支持 SPEL，如果需要表达式判断，只能用 `@PreAuthorize`。
- Spring Security 6.x 中 `@EnableGlobalMethodSecurity` 已废弃，迁移到 `@EnableMethodSecurity`。

---

## 12. Spring Security OAuth2 Resource Server：JWT 验签

### 题目
Spring Security OAuth2 Resource Server 如何验证 JWT Token？JWT 验签的完整流程是什么？

### 核心答案

**引入依赖：**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

**最小配置：**

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(Customizer.withDefaults())  // 启用 JWT 验证
            );
        return http.build();
    }
}
```

**JWT 验签完整流程：**

```
1. 请求携带 Authorization: Bearer <JWT>
2. JwtAuthenticationFilter（在Security Filter链中）拦截
3. 调用 JwtDecoder.decode(jwtString) 解析 Token（不验签）
4. 调用 JwtDecoder.verify(signature) 验签：
   a. 从 JOSE Header 读取 alg（如 RS256）
   b. 从 Authorization Server 的 jwks_uri 获取公钥（或配置中的公钥）
   c. 使用公钥验证签名
5. 验签通过 → 解析 Claims（sub/exp/iat/...）
6. 构建 JwtAuthenticationToken（principal=Claims, authorities从scope/authorities提取）
7. 存入 SecurityContext → 继续 Filter 链
```

**自定义 JWT Decoder（使用 JWK Set）：**

```java
@Bean
public JwtDecoder jwtDecoder() {
    return JwtDecoders.fromIssuerLocation("https://your-auth-server/.well-known/openid-configuration");
    // 或直接用 JWK Set URI：
    // return JwtDecoders.fromJwkSetUri("https://auth-server/.well-known/jwks.json");
}
```

**自定义 Claim 映射：**

```java
@Configuration
public class JwtConfig {
    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter grantedAuthConverter = new JwtGrantedAuthoritiesConverter();
        grantedAuthConverter.setAuthoritiesClaimName("permissions");
        grantedAuthConverter.setAuthorityPrefix("ROLE_");

        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(grantedAuthConverter);
        return converter;
    }
}
```

### 追问方向
- `spring-security-oauth2-jose` 和 `spring-boot-starter-oauth2-resource-server` 的关系是什么？
- JWT 验证失败（签名无效/过期/格式错误）时分别抛出什么异常？Spring Security 如何处理这些异常并返回 401？
- 如何在 Resource Server 中实现多租户 JWT（不同 tenant 对应不同 issuer）的验证？

### 避坑提示
- Resource Server 验证 JWT 时，**默认使用 Nimbus 实现**，它会缓存已解析的 JWK Key，缓存时间默认 5 分钟（可配置 `jwkSetCacheDuration`）。
- 如果 Authorization Server 使用自签名证书或非标准 CA，需要配置 `SslConfiguration` 或在启动时导入对应证书，否则 JWT 验签会报 SSL 相关错误。
- Stateless 模式下 `SecurityContext` 不会持久化（默认 `HttpSessionSecurityContextRepository`），每次请求都重新从 JWT 解析。

---

## 13. 常见安全漏洞：SQL 注入、XSS、CSRF、越权访问

### 题目
Spring/Spring Security 项目中如何防护 SQL 注入、XSS、CSRF、越权访问这四类常见安全漏洞？

### 核心答案

| 漏洞 | 原理 | Spring/Spring Security 防护方案 |
|------|------|-------------------------------|
| **SQL 注入** | 用户输入拼接到 SQL 语句中被执行 | 使用 `JdbcTemplate` / JPA `EntityManager` / MyBatis `#{}`（参数化查询），禁止字符串拼接；Hibernate 5+ 默认禁用原生 SQL |
| **XSS** | 攻击者在页面注入恶意脚本 | Spring MVC `@Controller` 默认启用 HTML 转义；使用 Thymeleaf/Lay Templates（默认转义）；`@CrossOrigin` 配合 CSP 响应头 |
| **CSRF** | 攻击者诱骗用户向目标站点提交伪造请求 | Spring Security 的 `CsrfFilter` + Synchronizer Token Pattern；或使用 CORS 限制 `origin` 白名单 |
| **越权访问** | 用户访问了本无权访问的资源（水平/垂直越权） | `@PreAuthorize` 方法级鉴权；Filter 链中的 `FilterSecurityInterceptor`；数据层使用用户 ID 做关联查询 |

**SQL 注入防御示例：**

```java
// ✅ 安全：使用参数化查询
List<User> users = jdbcTemplate.query(
    "SELECT * FROM users WHERE email = ?",
    userRowMapper,
    email
);

// ❌ 危险：字符串拼接
String sql = "SELECT * FROM users WHERE email = '" + email + "'";
```

**XSS 防护示例：**

```java
// Spring MVC 控制器转义（默认已启用）
// 使用 @RequestParam @PathVariable 时框架做基本校验
// Content-Type 限制
http.securityMatcher("/api/**")
    .headers(headers -> headers
        .contentSecurityPolicy(policy -> policy
            .policyDirectives("default-src 'self'; script-src 'self'")
        )
    );
```

**越权访问防御示例：**

```java
// 水平越权：在查询时强制加用户ID条件
@PreAuthorize("#userId == authentication.principal.userId")
public User getUserProfile(Long userId);

// 垂直越权：使用角色层级
@PreAuthorize("hasRole('ADMIN')")
public void deleteAnyUser(Long userId);
```

### 追问方向
- 水平越权和垂直越权的定义是什么？各有何典型场景？
- CSRF 和 XSS 的关系是什么？两者如何相互利用？
- "参数化查询"能完全防止 SQL 注入吗？有没有参数化查询无法防御的场景（如 ORDER BY 子句）？

### 避坑提示
- 防范 CSRF 的 token 放在 `Cookie` 中（`CookieCsrfTokenRepository`）需要注意：Cookie 中的 token 可被 JavaScript 读取，存在 XSS 窃取风险（SameSite Cookie 是缓解手段但不完美）。
- 防止越权访问的黄金法则：**所有数据查询必须带当前用户 ID 条件**，不能依赖客户端传来的 `userId` 而不做校验直接查询。
- 仅仅配置 `.antMatchers().hasRole()` 不等于完全防越权——API 层可以返回数据库中任意用户的数据，需配合方法级注解和数据层校验。

---

## 14. CORS 配置：@CrossOrigin、WebMvcConfigurer 全局配置

### 题目
在 Spring Security 环境下如何正确配置 CORS？`@CrossOrigin` 注解和 `WebMvcConfigurer` 全局配置有何区别？为什么 CORS 配置必须和 Spring Security 配合？

### 核心答案

**CORS 原理（预检请求）：**

浏览器对跨域 `PUT/DELETE/POST`（带自定义头）先发 **OPTIONS 预检**，服务器响应 `Access-Control-Allow-Origin` 等头，浏览器确认后才发真实请求。

**方案一：`@CrossOrigin` 注解（方法/类级别）**

```java
@RestController
@RequestMapping("/api")
public class UserController {
    @CrossOrigin(
        origins = "https://example.com",
        allowedHeaders = {"Authorization", "Content-Type"},
        exposedHeaders = {"X-Total-Count"},
        methods = {RequestMethod.GET, RequestMethod.POST},
        allowCredentials = "true",
        maxAge = 3600
    )
    @GetMapping("/users")
    public List<User> getUsers() { return users; }
}
```

**方案二：`WebMvcConfigurer` 全局配置**

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://example.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowedHeaders("*")
            .exposedHeaders("X-Total-Count")
            .allowCredentials(true)
            .maxAge(3600);
    }
}
```

**Spring Security 集成（必须）：**

若不配置，Spring Security 默认**禁用** CORS，导致预检请求被拦截：

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .cors(cors -> cors
                .configurationSource(corsConfigurationSource())  // 关键！
            )
            .csrf(AbstractHttpConfigurer::disable)  // 前后端分离常禁用CSRF
            // ...
        return http.build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://example.com"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("*"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

**三层配置的关系：**

```
@CrossOrigin（细粒度） > WebMvcConfigurer（全局限流） > Spring Security CorsConfigurationSource（安全层）
Spring Security 的 CORS Filter（Security Filter 链中）最先处理预检请求
```

### 追问方向
- 为什么 Spring Security 必须配合 CORS 配置，不能单独使用 `@CrossOrigin`？
- `Access-Control-Allow-Credentials: true` 与 `Access-Control-Allow-Origin: *` 能同时使用吗？浏览器兼容性问题是什么？
- CORS 预检请求的缓存 `maxAge` 设置过长/过短各有什么问题？

### 避坑提示
- `allowedOrigins("*")` 与 `allowCredentials(true)` **互斥**，同时设置会导致浏览器拒绝跨域。生产环境必须指定具体域名。
- Spring Security 的 CORS Filter 位于 `FilterSecurityInterceptor` **之前**，这是设计好的——预检请求在鉴权之前处理，避免未认证用户的预检被拒绝。
- `allowedHeaders("*")` 配合 `allowCredentials(true)` 时，如果浏览器发送了 `Authorization` 头，`allowedHeaders` 必须明确包含它（`*` 不包含 `Authorization`）。

---

## 15. 过滤器链顺序：UsernamePasswordAuthenticationFilter 位置、自定义 Filter 插入位置

### 题目
Spring Security 默认 Filter 链中 `UsernamePasswordAuthenticationFilter` 的位置是什么？自定义 Filter 应该如何选择插入位置？ Filter 顺序错误的常见后果是什么？

### 核心答案

**Spring Security 6.x 默认 Filter 链顺序（从外到内，数字越小越先执行）：**

| 顺序 | Filter 名称 | 职责 |
|------|-------------|------|
| -100 | `ChannelProcessingFilter` | HTTP/HTTPS 通道处理 |
| -50 | `WebAsyncManagerIntegrationFilter` | 异步上下文 |
| -40 | `SecurityContextPersistenceFilter` | 加载/保存 SecurityContext |
| -30 | `HeaderWriterFilter` | 写入安全响应头 |
| -20 | `CorsFilter` | CORS 预检处理 |
| -10 | `CsrfFilter` | CSRF Token 验证 |
| 0 | `LogoutFilter` | 登出处理 |
| 100 | `**UsernamePasswordAuthenticationFilter**` | 表单登录认证 |
| 200 | `RequestCacheAwareFilter` | 请求缓存 |
| 300 | `SecurityContextHolderAwareRequestFilter` | 包装请求（支持 `HttpServletRequest.getRemoteUser()` 等） |
| 400 | `AnonymousAuthenticationFilter` | 匿名用户匿名 Token |
| 500 | `SessionManagementFilter` | Session fixation 防护 |
| 600 | `ExceptionTranslationFilter` | 翻译 AccessDeniedException / AuthenticationException |
| 700 | `FilterSecurityInterceptor` | 权限决策（调用 AccessDecisionManager） |

**自定义 Filter 插入位置选择：**

```java
http.addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);  // 之前
http.addFilterAfter(correlationIdFilter, HeaderWriterFilter.class);             // 之后
http.addFilterAt(customFilter, UsernamePasswordAuthenticationFilter.class);   // 替换
```

**插入策略建议：**

| Filter 目的 | 建议位置 |
|-------------|----------|
| JWT/Token 认证 | `UsernamePasswordAuthenticationFilter` **之前**（因为要尽早识别用户身份） |
| 请求日志/TraceId | `HeaderWriterFilter` 之后 |
| 特殊业务校验 | `FilterSecurityInterceptor` 之前 |

**顺序错误的后果：**

- JWT Filter 放在 `UsernamePasswordAuthenticationFilter` **之后** → 已通过表单登录的用户走 JWT，未登录用户直接被 `AnonymousAuthenticationFilter` 处理，JWT 永远不生效。
- `CorsFilter` 放在 `CsrfFilter` **之后** → 预检请求被 CSRF 拦截，返回 403。

### 追问方向
- `FilterSecurityInterceptor` 抛出 `AccessDeniedException` 后，`ExceptionTranslationFilter` 是如何把它转成 HTTP 响应的？
- 如何通过 `@Order` 或 `FilterRegistrationBean` 调整非 Spring Security Filter 的顺序？
- `AnonymousAuthenticationFilter` 在链中处于什么位置？它的存在意义是什么？

### 避坑提示
- 使用 `addFilterAt` 替换已有的 Filter（如 `UsernamePasswordAuthenticationFilter`）时，被替换的 Filter **仍然存在但不再被调用**，需谨慎。
- `UsernamePasswordAuthenticationFilter` 默认只处理 **POST** `/login` 请求，如果登录 URL 不同，需要修改 `.formLogin().loginProcessingUrl("/login")`。

---

## 16. SecurityContext：ThreadLocal 存储、SecurityContextHolder 策略

### 题目
Spring Security 的 `SecurityContext` 是什么？如何通过 `ThreadLocal` 实现线程隔离？`SecurityContextHolder` 提供了哪几种存储策略？

### 核心答案

**SecurityContext 接口：**

```java
public interface SecurityContext extends Serializable {
    Authentication getAuthentication();
    void setAuthentication(Authentication authentication);
}
```

**ThreadLocal 存储机制：**

```java
// SecurityContext 默认实现：SecurityContextImpl
public class SecurityContextImpl implements SecurityContext {
    private final ThreadLocal<SecurityContext> contextHolder =
        new ThreadLocal<>();  // 关键！每个线程独立副本
}
```

同一用户的两个并发请求进入不同线程，`SecurityContext` 互不影响——这就是**线程隔离**。

**SecurityContextHolder 三种存储策略：**

| 策略 | 模式 | 配置 | 适用场景 |
|------|------|------|----------|
| **THREADLOCAL**（默认） | ThreadLocal | 默认 | 单线程同步处理 |
| **InheritableThreadLocal** | InheritableThreadLocal | `SecurityContextHolder.setStrategyName("MODE_INHERITABLETHREADLOCAL")` | 异步场景，子线程继承父线程 SecurityContext |
| **Global** | 静态变量 | `MODE_GLOBAL` | 不推荐，并发安全问题 |

**典型使用：**

```java
// 获取当前认证用户（Controller/Service 层）
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
String username = auth.getName();
Collection<? extends GrantedAuthority> roles = auth.getAuthorities();

// 手动设置（Filter 中）
SecurityContext context = SecurityContextHolder.createEmptyContext();
context.setAuthentication(token);
SecurityContextHolder.setContext(context);

// 显式清除（logout 时）
SecurityContextHolder.clearContext();
```

**Filter 链中的存储流程：**

```
请求到达
  → SecurityContextPersistenceFilter
       → 创建空 Context 或从 Session 恢复 → 存入 ThreadLocal
  → [业务Filter操作ThreadLocal中的Context]
  → FilterSecurityInterceptor 处理完
  → SecurityContextPersistenceFilter
       → 将 ThreadLocal Context 保存到 Session（HttpSessionSecurityContextRepository）
       → 清除 ThreadLocal（防止内存泄漏）
```

### 追问方向
- 异步多线程场景（`@Async`、`CompletableFuture`）中，`SecurityContext` 为什么默认**不会**传递到子线程？如何让它传递？
- 使用 `InheritableThreadLocal` 传递 SecurityContext 有何局限性？什么场景下会失效？
- `SecurityContextPersistenceFilter` 在 Spring Security 6.x 中是否仍然存在？其功能是否被其他组件替代？

### 避坑提示
- `SecurityContextHolder.getContext().setAuthentication(...)` 之前必须先调用 `SecurityContextHolder.createEmptyContext()`（不继承父线程 context），否则可能意外继承不该继承的认证信息。
- 线程池处理请求时，`InheritableThreadLocal` 的继承只在**线程创建时**发生——已存在的线程池 worker 不会继承。生产环境异步处理推荐使用 `DelegatingSecurityContextRunnable`（Spring Security 提供）。

---

## 17. 记住我功能：TokenRepository、持久化令牌

### 题目
Spring Security 的 "Remember Me"（记住我）功能是如何实现的？`TokenRepository` 的作用是什么？持久化令牌（Persistent Token）相比内存令牌有何优势？

### 核心答案

**两种 Remember Me 策略：**

| 策略 | 实现类 | 存储位置 | 安全性 |
|------|--------|----------|--------|
| **简单令牌**（哈希签名） | `TokenBasedRememberMeServices` | Cookie（用户名+过期时间+签名） | 较低 |
| **持久化令牌**（推荐） | `PersistentTokenRepository` | 数据库/Redis | 高 |

**持久化令牌工作流程：**

```
登录时（勾选"记住我"）：
  1. 生成随机 Series ID（64字节，UUID）
  2. 生成随机 Token Value（64字节）
  3. 记录 token 到数据库（Series ID + Token Value + 用户名 + 日期）
  4. 写入 Cookie: remember-me = SeriesID:TokenValue

后续请求：
  1. 读取 Cookie 中的 SeriesID + TokenValue
  2. TokenRepository 根据 SeriesID 查找记录
  3. 比对 TokenValue（完全匹配）
  4. 更新 TokenValue（新随机值，重放保护）→ 写回数据库
  5. 自动登录成功
```

**数据库表结构（默认 JdbcTokenRepository）：**

```sql
CREATE TABLE persistent_logins (
    series VARCHAR(64) PRIMARY KEY,
    username VARCHAR(64) NOT NULL,
    token VARCHAR(64) NOT NULL,
    last_used TIMESTAMP NOT NULL
);
```

**配置持久化令牌：**

```java
@Bean
public PersistentTokenRepository persistentTokenRepository(DataSource dataSource) {
    JdbcTokenRepositoryImpl repo = new JdbcTokenRepositoryImpl();
    repo.setDataSource(dataSource);
    repo.setCreateTableOnStartup(false);  // 建议手动建表
    return repo;
}

@Configuration
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .rememberMe(remember -> remember
                .tokenRepository(persistentTokenRepository(null))  // 注入上面Bean
                .tokenValiditySeconds(14 * 24 * 60 * 60)           // 14天
                .useSecureCookie(true)                             // HTTPS only
                .sameSiteCookie("Strict")                         // CSRF缓解
            );
        return http.build();
    }
}
```

### 追问方向
- 持久化令牌中 Series ID 和 Token Value 的作用分别是什么？为什么 Token Value 每次请求都要更新？
- 攻击者获取了用户的 remember-me cookie，能否通过重放攻击登录？如何防护？
- `useSecureCookie(true)` 在 HTTP 环境会怎样？同一域名下同时有 HTTP 和 HTTPS 时需要注意什么？

### 避坑提示
- Remember Me cookie 必须设置 `httpOnly`（防 XSS）、`secure`（HTTPS-only）、`sameSite`（防 CSRF）属性，缺一不可。
- `tokenValiditySeconds` 设置过长（如 30 天）是便利性与安全性的权衡，如果 Token 泄露影响窗口大，需结合 IP/User-Agent 绑定等二次验证。
- `PersistentTokenRepository` 默认建表 SQL 由 `JdbcTokenRepositoryImpl` 提供，但 Spring Boot 2.x+ 不会自动建表（除非 `createTableOnStartup=true`），生产环境建议手动建表。

---

## 18. 分布式 Session：Spring Session、Redis 存储 Session

### 题目
如何实现 Spring Security 的分布式 Session？Spring Session + Redis 的工作原理是什么？有哪些坑需要避免？

### 核心答案

**为什么需要分布式 Session？**

多实例部署时，用户请求可能被分发到不同实例，每个实例的 `ThreadLocal` 是独立的。共享 Session（集中存储到 Redis）确保任意实例都能获取同一用户的认证状态。

**Spring Session + Redis 架构：**

```
用户请求 → 实例A → Session不存在
  → 创建 Session → 序列化为 JSON → 存入 Redis（key: spring:session:sessions:<sessionId>）
  → 返回 Cookie: SESSION=sessionId

后续请求 → 实例B → 从 Redis 读取 Session → 反序列化 → 恢复认证状态
```

**核心依赖：**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
```

**最小配置：**

```java
// Spring Boot 2.x 起，自动配置生效，只需指定 namespace
@Configuration
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)  // 30分钟
public class SessionConfig {}

// application.yml
spring:
  session:
    store-type: redis
  redis:
    host: localhost
    port: 6379
```

**Session 序列化（Spring Session 2.x/3.x 区别）：**

Spring Session 2.x 使用 **Java 序列化**（`JavaRedisHttpSessionConfiguration`），必须实现 `Serializable`。Spring Session 3.x 改用 **JSON 序列化**（Jackson），无需实现 `Serializable`。

**Redis Session 的 Key 结构：**

```
spring:session:sessions:<sessionId>
  ├─ attr:map<string, Object>     // Session 属性（含 SecurityContext）
  ├─ creationTime                  // 创建时间
  ├─ maxInactiveInterval          // 过期间隔
  └─ lastAccessedTime             // 最后访问时间
```

### 追问方向
- `maxInactiveIntervalInSeconds` 和 Redis TTL 的关系是什么？为什么要设置 `spring.session.redis.namespace`？
- 集群环境下 `HttpServletRequest.getSession()` 获取的是同一个 Session 吗？实例重启后 Session 会丢失吗？
- Session 序列化/反序列化过程中，`SecurityContext` 中的 `UserDetails` 对象需要满足什么条件才能正确还原？

### 避坑提示
- Spring Session 2.x 默认 Java 序列化有**安全风险**：如果 Redis 被未授权访问，`UserDetails` 会被反序列化，可能导致远程代码执行（RCE）。**必须升级到 Spring Session 3.x** 使用 JSON 序列化，或显式配置 Jackson 序列化。
- `SecurityContext` 存入 Redis 时，`UserDetails`（密码字段应为空 `BCrypt` 编码后存为 `""`）需要实现正确序列化。
- 集群部署时，如果使用 `spring.session.store-type: redis`，`@EnableRedisHttpSession` 注解可以被省略（Boot 自动配置生效），但必须确保 Redis 连接配置正确。

---

## 19. 双因素认证：TOTP、Google Authenticator

### 题目
双因素认证（2FA）是什么？TOTP（基于时间的一次性密码）算法原理是什么？如何在 Spring Security 中集成 Google Authenticator 风格的 2FA？

### 核心答案

**双因素认证定义：**

身份认证的三个因素：Something you **know**（密码）、Something you **have**（手机/硬件Key）、Something you **are**（指纹/面部）。2FA 要求同时验证**两个不同类别**的因素。

**TOTP 算法原理（RFC 6238）：**

```
TOTP = HMAC-SHA1(SecretKey, TimeCounter)
TimeCounter = floor(Unix_timestamp / Step)
Step = 30秒（Google Authenticator 默认）

最终密码 = TOTP 取前6位数字（也可取更长）
```

验证双方各自用相同时间片计算 TOTP，比对前6位数字。由于时钟可能有漂移，服务器端通常允许 **前后1个时间片** 的容差。

**TOTP 密钥的存储与格式：**

- SecretKey 通常编码为 **Base32**（方便人工读）
- URI 格式（Google Authenticator 扫码）：`otpauth://totp/Issuer:user@email.com?secret=JBSWY3DPEHPK3PXP&issuer=Issuer&algorithm=SHA1&digits=6&period=30`
- SecretKey 必须**安全存储**（加密存储在数据库），不可明文

**Spring Security 集成 TOTP 思路（不使用 spring-security-oauth2，仅自研/库）：**

```java
// 核心库：de.taimos:totp-spring-boot 或 com.warrenstrange:googleauth
@Service
public class TotpService {
    private final GoogleAuthenticator gAuth = new GoogleAuthenticator();

    // 生成密钥（用户首次开启2FA时调用）
    public String generateSecret() {
        return gAuth.createCredentials().getKey();
    }

    // 获取TOTP URI（用于生成二维码）
    public String getUri(String secret, String user) {
        return GoogleAuthenticatorQRGenerator.getOtpAuthUri(
            "Issuer", user, secret, "issuer", "SHA1", 6, 30
        );
    }

    // 验证TOTP code
    public boolean verify(String secret, int code) {
        return gAuth.authorize(secret, code);
    }
}
```

**认证流程（登录 + 2FA）：**

```
Step1: 用户名+密码认证 → 成功 → 检查用户是否启用2FA
  → 启用 → 生成随机 2FA session key → 跳转到2FA验证页
Step2: 用户输入6位TOTP code → 服务端验证
  → 成功 → 清除2FA session key → 完成登录，写入SecurityContext
  → 失败 → 返回错误，session记录失败次数（防暴力破解）
```

### 追问方向
- TOTP 为什么选择 **Base32** 而不是 Base64 编码 Secret？熵是否相同？
- TOTP 的时间窗口容差（前后1个Step）会不会降低安全性？攻击者如何利用？
- 除了 TOTP，还有哪些 2FA 方法（HOTP/U2F/推送通知）？各有何优劣？

### 避坑提示
- 用户首次开启 2FA 时生成的 Secret 必须一次性显示给用户（用于绑定 Authenticator APP），**之后不再重复显示**。如果用户丢失设备，需要通过备用码（backup codes）或其他身份验证方式恢复。
- TOTP 验证应加入**速率限制**（rate limiting）：暴力破解6位数字仅有 100 万种组合，攻击者可在短时间内尝试完毕。建议验证失败后加入指数退避或临时封禁。
- `TimeCounter` 基于 **UTC 时间**，服务器和用户手机的时钟必须同步（误差不超过Step/2）。时区无关（都是UTC），但时钟漂移有关。

---

## 20. 注解式权限：@PreAuthorize 的 SPEL 表达式应用

### 题目
`@PreAuthorize` 的 SPEL 表达式能做哪些复杂权限判断？请举例说明常见的 SPEL 安全表达式用法，以及如何自定义 SPEL 安全方法。

### 核心答案

**@PreAuthorize SPEL 表达式基础：**

```java
@PreAuthorize("isAuthenticated()")                   // 已认证
@PreAuthorize("isAnonymous()")                      // 未认证（匿名）
@PreAuthorize("hasRole('USER')")                    // 有角色
@PreAuthorize("hasAuthority('SCOPE_read')")        // 有OAuth2 scope
@PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")     // 任一角色
@PreAuthorize("permitAll()")                        // 允许所有人
@PreAuthorize("denyAll()")                          // 拒绝所有人
@PreAuthorize("#userId == authentication.principal.id")  // 参数与当前用户比对
```

**参数引用语法：**

| 语法 | 含义 |
|------|------|
| `#paramName` | 方法参数 |
| `#object.field` | 参数对象的属性 |
| `#object.@method()` | 参数对象的方法 |
| `authentication` | 当前 `Authentication` 对象 |
| `principal` | 当前用户主体（通常是 `UserDetails`） |
| `request` | `HttpServletRequest` |
| `hasPermission(targetId, permission)` | 自定义权限方法 |

**常见复杂表达式示例：**

```java
// 1. 只有资源所有者或管理员可访问
@PreAuthorize("#doc.owner == authentication.principal.username or hasRole('ADMIN')")
public void deleteDocument(Document doc);

// 2. 金额阈值控制
@PreAuthorize("hasRole('ADMIN') or #amount <= 10000")
public void transfer(BigDecimal amount);

// 3. 时间窗口限制（仅工作时间）
@PreAuthorize("hasRole('CLERK') and " +
    "T(java.time.LocalTime).now().isAfter(T(java.time.LocalTime).of(9,0)) and " +
    "T(java.time.LocalTime).now().isBefore(T(java.time.LocalTime).of(18,0)))")
public void processOrder(Order order);

// 4. 组合权限（AND/OR/NOT）
@PreAuthorize("hasAuthority('READ') and (hasAuthority('WRITE') or hasRole('EDITOR'))")
public void editResource(Resource r);

// 5. 集合参数遍历
@PreAuthorize("@documentService.isOwner(#docIds, authentication.principal.id)")
public void batchDeleteDocuments(List<Long> docIds);
```

**自定义 SPEL 安全方法（@bean引用）：**

```java
@Service("documentService")
public class DocumentService {
    // 在 @PreAuthorize 中通过 @documentService 引用
    public boolean isOwner(List<Long> docIds, String userId) {
        return documentRepo.countByIdInAndOwnerIdNot(docIds, userId) == 0;
    }
}
```

```java
@PreAuthorize("@documentService.isOwner(#docId, authentication.principal.id)")
public Document getById(Long docId);
```

**自定义 PermissionEvaluator（ACL 级别）：**

```java
@Component
public class CustomPermissionEvaluator implements PermissionEvaluator {
    public boolean hasPermission(Authentication auth, Object targetDomainObject, Object permission) {
        // targetDomainObject: 被访问的对象
        // permission: 字符串权限标识如 "READ"
    }

    public boolean hasPermission(Authentication auth, Serializable targetId,
                                  String targetType, Object permission) {
        // targetId + targetType: 对象标识
    }
}
```

```java
@EnableMethodSecurity
public class SecurityConfig {
    @Bean
    public DefaultMethodSecurityExpressionHandler expressionHandler(
            CustomPermissionEvaluator evaluator) {
        DefaultMethodSecurityExpressionHandler handler =
            new DefaultMethodSecurityExpressionHandler();
        handler.setPermissionEvaluator(evaluator);
        return handler;
    }
}
```

```java
@PreAuthorize("hasPermission(#doc, 'READ')")
public Document getDocument(Document doc);
```

### 追问方向
- `@PreAuthorize` 中访问方法参数的属性（如 `#user.id`）时，如果参数为 null 会抛出什么异常？如何用 `?.` 安全导航运算符处理？
- `@EnableMethodSecurity` 的 `prePostEnabled = true` 与默认行为有何不同？
- `@PostAuthorize` 的 `returnObject` 如何使用？如果方法返回 `null`，`returnObject` 还能访问吗？

### 避坑提示
- SPEL 中调用方法时，方法需是 **public** 的才能被安全表达式求值器访问（Spring EL 解析器基于反射）。
- `authentication.principal` 的类型取决于认证方式：密码登录是 `UserDetails`，JWT 认证是 `Claims`（Spring Security OAuth2 Resource Server）。
- 在复杂 SPEL 表达式中使用太多逻辑会降低可读性和可维护性。**推荐将复杂逻辑提取到 `@Bean` 方法**中，保持注解简洁。

---

## 附录：Spring Security 6.x 重要变化（面试加分项）

| 变化 | 说明 |
|------|------|
| `WebSecurityConfigurerAdapter` 废弃 | 改用 `@Bean SecurityFilterChain` |
| `@EnableGlobalMethodSecurity` 废弃 | 改用 `@EnableMethodSecurity` |
| `Authorization` 替代 `Authentication` | Spring Security 6.x 引入新的 `AuthorizationManager` API |
| `ServerHttpSecurity` | 在 Spring Security 6.x 中更倾向使用 `AuthorizationManager` |
| SessionCreationPolicy.STATELESS | 明确不再依赖 HttpSession |
| Lambda DSL | 配置风格从 `.and()` 链式转向 Lambda DSL |

---

> **面试建议**：Spring Security 面试的核心在于"从请求进来到响应出去"的完整链路理解。建议按以下线索复习：① FilterChain 请求流向 → ② 认证（Authentication）→ ③ 授权（Authorization）→ ④ 会话管理 → ⑤ OAuth2/JWT 扩展 → ⑥ 微服务安全集成。每个环节都能深挖到底层原理和源码细节。
