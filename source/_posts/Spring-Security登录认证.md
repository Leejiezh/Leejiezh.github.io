---
title: Spring Security登录认证
top: false
cover: false
img: /medias/featureimages/SpringSecurity封面.png
toc: true
mathjax: true
date: 2026-08-26 09:24:07
password:
summary: 
tags: 
    - springSecurity
    - 登录认证鉴权
categories:
    - 后端
---

**前言：**在Spring Boot 3.x + Spring Security 6.x的时代，构建一个从0到1的认证鉴权系统，我们通常会抛弃传统的Session模式，转而采用**无状态**的JWT（JSON Web Token）模式，配合**RBAC（基于角色的访问控制）**模型。这也是当前构建前后端分离架构或微服务架构的标准范式。

---

### 第一阶段：架构设计与技术选型

在写代码之前，我们需要明确核心流程和技术栈：

1.  **认证流程**：
    *   用户提交账号密码 -> **认证过滤器** (未登录) -> **AuthenticationManager** 调用 **UserDetailsService** 查库 -> 成功生成 JWT 返回。
    *   后续请求携带 JWT -> **JWT过滤器** (校验Token) -> 设置 SecurityContext -> **鉴权过滤器** (访问资源)。

2.  **核心组件**：
    *   **Spring Security 6.x**：主要变化是弃用了 `WebSecurityConfigurerAdapter`，改为基于组件的配置（Bean）。
    *   **JJWT**：用于生成和解析Token。
    *   **RBAC模型**：用户 -> 角色 -> 权限（菜单/按钮标识）。

3.  **数据库表设计（简化版）**：
    *   `sys_user`：用户表
    *   `sys_role`：角色表
    *   `sys_menu`：菜单/权限表
    *   `sys_user_role`：用户角色关联
    *   `sys_role_menu`：角色权限关联

---

### 第二阶段：项目初始化与依赖

创建 Spring Boot 3 项目，引入以下核心依赖（Maven示例）：

```xml
<dependencies>
    <!-- Web Starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- Security Starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    
    <!-- MyBatis-Plus (简化数据库操作) -->
    <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>mybatis-plus-boot-starter</artifactId>
        <version>3.5.5</version>
    </dependency>
    
    <!-- JWT (JJWT) -->
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.11.5</version>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.11.5</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId>
        <version>0.11.5</version>
        <scope>runtime</scope>
    </dependency>
    
    <!-- Lombok -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

---

### 第三阶段：核心代码实现

#### 1. 工具类：JWT 处理

这是无状态认证的核心，负责Token的生成与解析。

```java
import io.jsonwebtoken.*;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Component;

import javax.crypto.SecretKey;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;

@Component
public class JwtTokenUtil {

    @Value("${jwt.secret:mySecretKeyThatIsAtLeast32BytesLongForHS256}")
    private String secret;

    @Value("${jwt.expiration:86400}") // 默认24小时
    private Long expiration;

    // 生成Key
    private SecretKey getSigningKey() {
        return Keys.hmacShaKeyFor(secret.getBytes());
    }

    // 生成Token
    public String generateToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();
        return doGenerateToken(claims, userDetails.getUsername());
    }

    private String doGenerateToken(Map<String, Object> claims, String subject) {
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + expiration * 1000);

        return Jwts.builder()
                .claims(claims)
                .subject(subject)
                .issuedAt(now)
                .expiration(expiryDate)
                .signWith(getSigningKey())
                .compact();
    }

    // 从Token中获取用户名
    public String getUsernameFromToken(String token) {
        return getClaimsFromToken(token).getSubject();
    }

    // 验证Token是否过期
    public Boolean isTokenExpired(String token) {
        final Date expiration = getClaimsFromToken(token).getExpiration();
        return expiration.before(new Date());
    }

    // 验证Token
    public Boolean validateToken(String token, UserDetails userDetails) {
        final String username = getUsernameFromToken(token);
        return (username.equals(userDetails.getUsername()) && !isTokenExpired(token));
    }

    private Claims getClaimsFromToken(String token) {
        return Jwts.parser()
                .verifyWith(getSigningKey())
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }
}
```

#### 2. 核心实体：UserDetails 实现

Spring Security 需要通过这个对象获取用户权限。

```java
import lombok.AllArgsConstructor;
import lombok.Data;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import java.util.Collection;
import java.util.List;
import java.util.stream.Collectors;

@Data
@AllArgsConstructor
public class LoginUser implements UserDetails {

    private String username;
    private String password;
    // 存储权限列表，例如 ["user:add", "admin:delete"]
    private List<String> permissions;

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        // 将字符串权限转换为 Security 需要的 GrantedAuthority 对象
        return permissions.stream()
                .map(SimpleGrantedAuthority::new)
                .collect(Collectors.toList());
    }

    @Override
    public boolean isAccountNonExpired() { return true; }

    @Override
    public boolean isAccountNonLocked() { return true; }

    @Override
    public boolean isCredentialsNonExpired() { return true; }

    @Override
    public boolean isEnabled() { return true; }
}
```

#### 3. 核心逻辑：UserDetailsService

这里是连接数据库与Security的桥梁。你需要根据业务查询数据库组装 `LoginUser`。

```java
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;

import java.util.Collections;

@Service
public class UserDetailsServiceImpl implements UserDetailsService {

    // 模拟数据库查询，实际请注入 Mapper 调用 DB
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        // 1. 查数据库判断用户是否存在
        if (!"admin".equals(username)) {
            throw new UsernameNotFoundException("用户不存在");
        }
        
        // 2. 查询用户的权限列表 (模拟从关联表查询)
        // List<String> permissions = menuMapper.selectPermsByUserId(userId);
        List<String> permissions = Collections.singletonList("user:list", "user:add");

        // 3. 封装成 LoginUser 返回 (密码必须是加密后的，例如 {noop}123456 或 BCrypt加密串)
        // 为了演示，这里假设数据库存的密码已经是 BCrypt 加密的
        return new LoginUser(username, "$2a$10$N.zmdr9k7uOCQb376NoUnuTJ8iAt6Z5EHsM8lE9lBOsl7iAt6Z5EH", permissions);
    }·
}
```

#### 4. 过滤器：JWT 认证过滤器

这是最关键的一步，用于在用户请求时拦截并校验Header中的Token。

```java
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;

@Component
public class JwtAuthenticationTokenFilter extends OncePerRequestFilter {

    @Autowired
    private JwtTokenUtil jwtTokenUtil;

    @Autowired
    private UserDetailsServiceImpl userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {
        
        // 1. 获取 Header 中的 Token
        String tokenHeader = request.getHeader("Authorization");
        if (tokenHeader != null && tokenHeader.startsWith("Bearer ")) {
            String token = tokenHeader.substring(7);

            // 2. 解析 Token 获取用户名
            String username = jwtTokenUtil.getUsernameFromToken(token);

            // 3. 如果用户名不为空，且当前 SecurityContext 未认证（说明没登录）
            if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                // 4. 重新加载用户详情（为了获取最新的权限等信息）
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                // 5. 验证 Token 有效性
                if (jwtTokenUtil.validateToken(token, userDetails)) {
                    // 6. 手动设置 Authentication 对象到 SecurityContext
                    UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(
                            userDetails, null, userDetails.getAuthorities());
                    authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                    SecurityContextHolder.getContext().setAuthentication(authentication);
                }
            }
        }
        
        // 放行，继续执行后续过滤器
        filterChain.doFilter(request, response);
    }
}
```

#### 5. 配置类：SecurityConfig

Spring Security 6 的标准配置方式，使用 Lambda DSL 配置链路。

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Autowired
    private JwtAuthenticationTokenFilter jwtAuthenticationTokenFilter;

    // 密码编码器
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    // 认证管理器（用于登录接口校验）
    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    // 配置过滤器链
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // 1. 关闭 CSRF
            .csrf(csrf -> csrf.disable())
            // 2. 无状态 Session
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            // 3. 授权配置
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/auth/login").permitAll() // 放行登录接口
                .requestMatchers("/test/admin").hasAuthority("admin:delete") // 需特定权限
                .anyRequest().authenticated() // 其他请求均需认证
            )
            // 4. 添加自定义 JWT 过滤器（在 UsernamePasswordFilter 之前）
            .addFilterBefore(jwtAuthenticationTokenFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

---

### 第四阶段：业务接口落地

#### 1. 登录接口

用户调用此接口获取Token。

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.AuthenticationException;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.HashMap;
import java.util.Map;

@RestController
@RequestMapping("/auth")
public class AuthController {

    @Autowired
    private AuthenticationManager authenticationManager;

    @Autowired
    private JwtTokenUtil jwtTokenUtil;

    @PostMapping("/login")
    public Map<String, Object> login(@RequestBody Map<String, String> request) {
        String username = request.get("username");
        String password = request.get("password");

        // 1. 使用 AuthenticationManager 进行认证
        // 内部会调用 UserDetailsService
        try {
            Authentication authentication = authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(username, password)
            );
            
            // 2. 认证成功，生成 Token
            String token = jwtTokenUtil.generateToken((LoginUser) authentication.getPrincipal());
            
            Map<String, Object> result = new HashMap<>();
            result.put("code", 200);
            result.put("token", token);
            return result;
            
        } catch (AuthenticationException e) {
            Map<String, Object> result = new HashMap<>();
            result.put("code", 401);
            result.put("msg", "用户名或密码错误");
            return result;
        }
    }
}
```

#### 2. 测试资源接口

```java
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class TestController {

    // 需登录即可访问
    @GetMapping("/hello")
    public String hello() {
        return "Hello, authenticated user!";
    }

    // 需要特定权限 user:list (在 UserDetails 中配置)
    @GetMapping("/test/user")
    @PreAuthorize("hasAuthority('user:list')")
    public String user() {
        return "User Info Access Granted";
    }
    
    // 需要特定权限 admin:delete (如果用户没有，将返回 403)
    @GetMapping("/test/admin")
    @PreAuthorize("hasAuthority('admin:delete')")
    public String admin() {
        return "Admin Access Granted";
    }
}
```

---

### 第五阶段：架构师的思考与优化建议

作为架构师，完成功能只是第一步，以下是在生产环境中必须考虑的**风险与优化点**：

1.  **性能优化 - 缓存**：
    *   **问题**：上述代码中，每次请求接口（`JwtAuthenticationTokenFilter`）都会调用 `UserDetailsService.loadUserByUsername` 查询数据库获取权限，这在大并发下是不可接受的。
    *   **方案**：引入 Redis。在生成 Token 时，将 `LoginUser` 对象存入 Redis（Key为Token的一部分或UUID）。在 Filter 中解析 Token 后，直接从 Redis 获取用户信息，而不再查库。

2.  **安全性 - Token 刷新机制**：
    *   **问题**：JWT一旦签发，在过期前即使用户修改了密码或被注销，Token依然有效。
    *   **方案**：实现双Token机制或黑名单机制。
        *   *黑名单*：用户修改密码/注销时，将 Token 存入 Redis 黑名单，Filter 校验时先查黑名单。
        *   *刷新Token*：Access Token 有效期短（如30分钟），Refresh Token 有效期长（如7天）。Access 过期后用 Refresh 换新的 Access。

3.  **异常处理 - 全局捕获**：
    *   Spring Security 的异常（如 `AccessDeniedException`）通常不会直接被 `@RestControllerAdvice` 捕获，需要自定义 `AccessDeniedHandler` 和 `AuthenticationEntryPoint` 并在 `SecurityConfig` 中配置，以便统一返回 JSON 格式的 403/401 信息。

4.  **扩展性 - OAuth2**：
    *   如果未来需要接入第三方登录（微信、GitHub）或单点登录（SSO），上述自定义 JWT 模式可能需要重构。建议一开始就调研 Spring Authorization Server，但学习曲线较陡，中小型项目上述方案足够。

5.  **数据一致性**：
    *   在分布式系统中，确保权限变更后 Redis 缓存的及时更新是难点，需要通过 Canal 监听 MySQL Binlog 或者在业务代码修改权限时主动删除缓存。

### 总结

通过以上步骤，我们构建了一个标准的、无状态的 Spring Security + JWT 认证鉴权体系。核心在于**理解过滤器链的执行顺序**以及**UserDetails与SecurityContext的交互机制**。希望这个方案对你的项目开发有所帮助。