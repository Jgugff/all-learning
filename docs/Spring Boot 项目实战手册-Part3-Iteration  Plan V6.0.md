## 第三部分：实战工作手册 v6.0 阶段：认证与授权

> 覆盖 **v6.0 阶段的 C16、C17、C18、C19、C20** 两次提交

---

### 背景：为什么需要这个阶段？

v5.0 结束后，系统有一个根本性的安全漏洞：

```
任何人都可以调用任何接口：

curl -X DELETE http://localhost:8080/api/users/123
→ 删除成功！没有任何身份验证

curl -X POST http://localhost:8080/api/users ...
→ 新增成功！不知道是谁操作的

SELECT * FROM user WHERE username = 'admin'
→ creator = "system"  ← 所有操作都归属 "system"，完全无法审计
```

这个阶段要解决三个问题：

```
问题1：接口没有保护，任何人可以任意访问
  → 需要：身份认证（你是谁？）

问题2：不同用户应该有不同权限
  → 需要：授权（你能做什么？）

问题3：CommonMetaObjectHandler 里的 TODO 一直没填
  → 需要：从请求上下文中获取当前登录用户
```

---

### 迭代规划：v6.0 的 5 次提交

| # | Commit Message | 核心功能 |
|---|---------------|---------|
| C16 | `chore: 添加 Spring Security 与 JWT 依赖` | 引入依赖，理解自动配置 |
| C17 | `feat: 添加用户认证领域层（包括角色实体、Token 实体与 Repository 仓储接口）` | 角色表、用户角色关联、Token 实体 |
| C18 | `feat: 实现 JWT 工具类（Utils）与 Security 配置（Config）` | JWT 工具类、Security 核心配置 |
| C19 | `feat: 添加登录/注册接口（API）与 JWT 过滤器（Filter）` | 登录注册接口、Token 验证过滤器 |
| C20 | `refactor: 将当前用户引入元数据处理器并实现数据审计` | 替换 TODO、接口权限精细化控制 |

---

### 📦 Commit 16 —— 引入依赖，理解 Security 自动配置

> `chore: 添加 Spring Security 与 JWT 依赖`

---

#### 🎯 本步目标

引入 Spring Security 和 JWT 相关依赖，理解 Spring Security 加入后系统的变化，以及为什么所有接口会突然都需要登录才能访问。

---

#### 📂 目录变动

```
修改：
└── pom.xml    ← 新增 Spring Security 和 JWT 依赖
```

---

#### 💻 核心代码与详解

**① `pom.xml` — 新增依赖**

```xml
<!-- Spring Security：认证与授权框架
     引入后立即生效：所有接口自动受保护，需要登录才能访问
     默认用户名 user，密码在启动日志中随机生成 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- JWT（JSON Web Token）：无状态身份令牌
     jjwt-api：JWT 操作的接口定义
     jjwt-impl：JWT 的具体实现
     jjwt-jackson：使用 Jackson 处理 JWT 的 JSON 序列化 -->
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
```

---

**② 理解 Spring Security 引入后发生了什么**

```
引入依赖之前：
  所有接口完全开放，任何请求都直接到达 Controller

引入依赖之后（什么都不配置）：
  所有接口自动被 Spring Security 保护
  未登录的请求 → 302 重定向到 /login 页面（Security 自动生成的表单登录页）

启动日志中会出现：
  Using generated security password: a1b2c3d4-e5f6-7890-abcd-ef12345678
                                     ↑ 每次启动随机生成，开发调试用
```

> 💡 **Spring Security 的核心工作原理**
>
> ```
> HTTP 请求
>     ↓
> SecurityFilterChain（一组有序的 Security Filter）
>     ↓
> UsernamePasswordAuthenticationFilter  ← 处理表单登录
>     ↓
> BasicAuthenticationFilter             ← 处理 Basic 认证
>     ↓
> FilterSecurityInterceptor             ← 授权检查（你有没有权限访问这个接口）
>     ↓
> Controller
>
> 我们要做的：
> 1. 在这条链中插入自己的 JwtAuthenticationFilter（验证 JWT Token）
> 2. 告诉 Security 哪些接口需要保护，哪些接口公开（如 /api/auth/login）
> 3. 关闭默认的表单登录（我们用 JWT，不用 Session）
> ```

---

#### 🧪 运行与验证

引入依赖后什么都不改，直接启动项目：

```bash
# 之前能正常访问的接口，现在返回 302 或 401
curl "http://localhost:8080/api/users?pageNo=1&pageSize=10&username=username1"
# 返回：302 Found（重定向到 /login）
# 或：401 Unauthorized（取决于请求头）

# 查看启动日志中的临时密码
# Using generated security password: xxxx-xxxx-xxxx

# 用默认账号访问（Basic 认证）
curl -u user:xxxx-xxxx-xxxx \
  "http://localhost:8080/api/users?pageNo=1&pageSize=10&username=username1"
# 能正常返回数据
```

这证明 Spring Security 已经生效。接下来我们要用 JWT 替换这套默认机制。

---

### 📦 Commit 17 —— 认证授权领域模型

> `feat: 添加用户认证领域层（包括角色实体、Token 实体与 Repository 仓储接口）`

---

#### 🎯 本步目标

在数据库层面建立完整的权限体系：角色表、用户角色关联表；在 Java 层面建立对应的实体和 Mapper；扩展 `UserDO` 支持关联角色查询。

---

#### 📂 目录变动

```
新增：
├── src/main/resources/db/
│   └── auth_schema.sql                  ← 角色表和关联表的建表 SQL
└── src/main/java/.../
    ├── domain/entity/
    │   └── RoleDO.java                  ← 角色实体
    ├── domain/dto/
    │   ├── LoginDTO.java                ← 登录请求参数
    │   ├── RegisterDTO.java             ← 注册请求参数
    │   └── TokenDTO.java                ← 登录成功后返回的 Token 信息
    └── mapper/
        └── RoleMapper.java              ← 角色 Mapper（含根据 userId 查角色）

修改：
└── src/main/resources/db/
    └── schema.sql                       ← user 表新增 password_hash 字段说明
```

---

#### 💻 核心代码与详解

**① `auth_schema.sql` — 权限相关表结构**

```sql
-- 角色表：定义系统中有哪些角色
CREATE TABLE `role` (
  `id`          bigint(20)  NOT NULL AUTO_INCREMENT,
  `role_name`   varchar(50) NOT NULL COMMENT '角色名称，如 ROLE_ADMIN、ROLE_USER',
  `description` varchar(200) DEFAULT NULL COMMENT '角色描述',
  `created`     datetime    DEFAULT NULL,
  `status`      tinyint(2)  DEFAULT '1',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_role_name` (`role_name`)  -- 角色名唯一
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

-- 用户角色关联表：多对多关系（一个用户可以有多个角色）
CREATE TABLE `user_role` (
  `id`      bigint(20) NOT NULL AUTO_INCREMENT,
  `user_id` bigint(20) NOT NULL COMMENT '用户 ID',
  `role_id` bigint(20) NOT NULL COMMENT '角色 ID',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_user_role` (`user_id`, `role_id`)  -- 同一用户同一角色只关联一次
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

-- 初始化两个基础角色
INSERT INTO `role` (role_name, description) VALUES
  ('ROLE_ADMIN', '管理员，拥有所有权限'),
  ('ROLE_USER',  '普通用户，只有查询权限');

-- 注意：Spring Security 要求角色名必须以 ROLE_ 开头
-- hasRole("ADMIN") 在内部会自动补全为 ROLE_ADMIN
```

> 💡 **为什么角色名要以 `ROLE_` 开头？**
>
> 这是 Spring Security 的内部约定：
> ```java
> // @PreAuthorize 中这样写：
> @PreAuthorize("hasRole('ADMIN')")
>
> // Spring Security 内部会自动拼接前缀，实际检查的是：
> // 当前用户是否拥有 "ROLE_ADMIN" 这个权限
>
> // 如果你的角色名不加 ROLE_ 前缀，也可以用 hasAuthority() 代替：
> @PreAuthorize("hasAuthority('ADMIN')")
> // hasAuthority 不会自动拼接前缀，直接匹配原始字符串
> ```

---

**② `LoginDTO.java` & `RegisterDTO.java` & `TokenDTO.java`**

```java
// 登录请求参数
@Data
public class LoginDTO implements Serializable {

    @NotBlank(message = "用户名不能为空")
    private String username;

    @NotBlank(message = "密码不能为空")
    private String password;
}
```

```java
// 注册请求参数（复用 UserDTO，但需要确认密码）
@Data
public class RegisterDTO implements Serializable {

    @NotBlank(message = "用户名不能为空")
    private String username;

    @NotBlank(message = "密码不能为空")
    @Length(min = 6, max = 18, message = "密码长度 6-18 位")
    private String password;

    @NotBlank(message = "确认密码不能为空")
    private String confirmPassword;  // 前端二次输入，防止误操作

    @NotEmpty(message = "邮箱不能为空")
    @Email(message = "邮箱格式不正确")
    private String email;

    @NotNull(message = "年龄不能为空")
    @Min(value = 18) @Max(value = 60)
    private Integer age;

    @NotBlank(message = "手机号不能为空")
    private String phone;
}
```

```java
// 登录成功后返回给前端的 Token 信息
@Data
@Builder  // Lombok：提供链式构建器，方便构造复杂对象
public class TokenDTO implements Serializable {

    /** JWT Token 字符串，前端后续请求放在 Header：Authorization: Bearer {token} */
    private String accessToken;

    /** Token 类型，固定值 "Bearer"，是 OAuth2 规范的约定 */
    private String tokenType;

    /** Token 过期时间（Unix 时间戳，毫秒）前端可用于判断何时需要刷新 */
    private Long expiresAt;

    /** 当前登录用户名，前端用于显示欢迎信息 */
    private String username;

    /** 当前用户的角色列表，前端用于控制菜单/按钮的显示隐藏 */
    private List<String> roles;
}
```

---

**③ `RoleDO.java` — 角色实体**

```java
@Data
@TableName("role")
public class RoleDO implements Serializable {

    @TableId(type = IdType.AUTO)
    private Long id;

    private String roleName;    // 对应数据库 role_name（MyBatis-Plus 自动驼峰转换）
    private String description;
    private LocalDateTime created;
    private Integer status;
}
```

---

**④ `RoleMapper.java` — 角色 Mapper**

```java
@Mapper
public interface RoleMapper extends BaseMapper<RoleDO> {

    /**
     * 根据用户 ID 查询该用户拥有的所有角色
     * 需要关联 user_role 表，BaseMapper 没有这个方法，需要自定义
     * 对应 SQL 写在 RoleMapper.xml 中
     */
    List<RoleDO> selectRolesByUserId(@Param("userId") Long userId);
}
```

在 `src/main/resources/mapper/RoleMapper.xml` 中：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.imooc.zhangxiaoxi.alllearning.mapper.RoleMapper">

    <!--
        根据用户 ID 查角色：联表查询
        ur = user_role 关联表
        r  = role 角色表
    -->
    <select id="selectRolesByUserId" resultType="com.imooc.zhangxiaoxi.alllearning.domain.entity.RoleDO">
        SELECT r.*
        FROM role r
        INNER JOIN user_role ur ON r.id = ur.role_id
        WHERE ur.user_id = #{userId}
          AND r.status = 1
    </select>

</mapper>
```

---

#### 🧪 运行与验证

```bash
# 执行建表 SQL，并插入测试数据
# 然后手动给 data.sql 中的 username1 分配 ROLE_ADMIN 角色

# 在 MySQL 中执行：
INSERT INTO user_role (user_id, role_id)
VALUES (1220708537638920191, 1);  -- username1 → ROLE_ADMIN
```

---

### 📦 Commit 18 —— JWT 工具类 & Security 核心配置

> `feat: 实现 JWT 工具类（Utils）与 Security 配置（Config）`

---

#### 🎯 本步目标

实现 JWT 的生成和解析工具类；编写 Spring Security 的核心配置，定义哪些接口需要认证、哪些接口公开，以及如何加载用户信息。

---

#### 📂 目录变动

```
新增：
└── src/main/java/.../
    ├── util/
    │   └── JwtUtils.java                ← JWT 生成、解析、校验工具
    ├── service/impl/
    │   └── UserDetailsServiceImpl.java  ← 告诉 Security 如何加载用户信息
    └── config/
        └── SecurityConfig.java          ← Spring Security 核心配置

修改：
└── src/main/resources/
    └── application.properties           ← 新增 JWT 配置项
```

---

#### 💻 核心代码与详解

**① `application.properties` — JWT 配置**

```properties
## JWT 签名密钥：用于生成和验证 Token 的签名
## 生产环境必须是足够长的随机字符串（至少 256 位），
## 并且通过环境变量注入，绝对不能硬编码提交到 Git
jwt.secret=all-learning-super-secret-key-2024-must-be-very-long-for-hs256
## Token 有效期：86400000 毫秒 = 24 小时
jwt.expiration=86400000
```

---

**② `JwtUtils.java` — JWT 核心工具类**

```java
@Component  // 注册为 Bean，可被其他类注入使用
@Slf4j
public class JwtUtils {

    /**
     * 从配置文件注入 JWT 签名密钥
     * @Value：Spring 的属性注入注解，读取 application.properties 中的值
     */
    @Value("${jwt.secret}")
    private String secret;

    @Value("${jwt.expiration}")
    private Long expiration;

    /**
     * 生成 JWT Token
     *
     * JWT 结构：Header.Payload.Signature
     *   Header：{"alg":"HS256","typ":"JWT"}
     *   Payload：{"sub":"username","roles":["ROLE_ADMIN"],"iat":...,"exp":...}
     *   Signature：HMACSHA256(base64(Header) + "." + base64(Payload), secret)
     *
     * @param username  用户名（作为 JWT 的 subject）
     * @param roles     用户角色列表（自定义 Claim）
     */
    public String generateToken(String username, List<String> roles) {

        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + expiration);

        return Jwts.builder()
                // subject：JWT 的主体，通常是用户名或用户 ID
                .setSubject(username)

                // 自定义 Claim：在 Payload 中添加额外信息
                // roles 信息放入 Token，服务端解析后无需再查数据库
                .claim("roles", roles)

                // 签发时间
                .setIssuedAt(now)

                // 过期时间：超过此时间，Token 自动失效
                .setExpiration(expiryDate)

                // 签名算法：HS256（HMAC + SHA256）
                // 密钥：从配置文件读取的 secret
                .signWith(getSigningKey(), SignatureAlgorithm.HS256)

                .compact();  // 生成最终的 JWT 字符串
    }

    /**
     * 从 Token 中提取用户名
     */
    public String getUsernameFromToken(String token) {
        return parseClaims(token).getSubject();
    }

    /**
     * 从 Token 中提取角色列表
     */
    @SuppressWarnings("unchecked")
    public List<String> getRolesFromToken(String token) {
        return parseClaims(token).get("roles", List.class);
    }

    /**
     * 验证 Token 是否有效
     * 包含两项检查：用户名匹配 + 未过期
     */
    public boolean validateToken(String token, String username) {
        try {
            String tokenUsername = getUsernameFromToken(token);
            return tokenUsername.equals(username) && !isTokenExpired(token);
        } catch (JwtException e) {
            // JwtException 涵盖所有 JWT 相关异常：
            // 签名不匹配、Token 格式错误、Token 已过期等
            log.warn("JWT Token 验证失败: {}", e.getMessage());
            return false;
        }
    }

    /**
     * 检查 Token 是否已过期
     */
    public boolean isTokenExpired(String token) {
        return parseClaims(token).getExpiration().before(new Date());
    }

    /**
     * 解析 Token，获取 Payload 中的所有 Claims
     * 解析时会自动验证签名，签名不匹配会抛出 JwtException
     */
    private Claims parseClaims(String token) {
        return Jwts.parserBuilder()
                .setSigningKey(getSigningKey())
                .build()
                .parseClaimsJws(token)
                .getBody();
    }

    /**
     * 将配置的 secret 字符串转换为 HMAC 签名 Key 对象
     */
    private Key getSigningKey() {
        byte[] keyBytes = secret.getBytes(StandardCharsets.UTF_8);
        return Keys.hmacShaKeyFor(keyBytes);
    }
}
```

> 💡 **JWT 和 Session 的核心区别，为什么分布式系统选 JWT？**
>
> ```
> Session 方案（有状态）：
>   登录 → 服务端创建 Session → 返回 SessionId（Cookie）
>   请求 → 携带 Cookie → 服务端查 Session → 验证身份
>
>   问题：Session 存在服务端内存中
>         多实例部署时，A 服务器的 Session 在 B 服务器上查不到
>         需要 Session 共享（Redis），增加了复杂度
>
> JWT 方案（无状态）：
>   登录 → 服务端生成 JWT（包含用户信息）→ 返回 Token
>   请求 → 携带 Token → 服务端解析 Token → 验证签名 → 验证身份
>
>   优点：Token 包含完整信息，服务端无需存储任何状态
>         任意一台服务器都能验证，天然支持水平扩展
>
>   代价：Token 一旦签发，在过期前无法主动吊销（这是 JWT 的根本缺陷）
>         解决方案：短过期时间 + 刷新 Token 机制，或维护 Token 黑名单（Redis）
> ```

---

**③ `UserDetailsServiceImpl.java` — 告诉 Security 如何加载用户**

```java
/**
 * UserDetailsService：Spring Security 定义的接口
 * 只有一个方法：loadUserByUsername(String username)
 * Security 在验证身份时会调用此方法加载用户信息
 *
 * 我们的实现：从数据库中按用户名查用户，关联查角色
 */
@Service
@Slf4j
public class UserDetailsServiceImpl implements UserDetailsService {

    @Autowired
    private UserMapper userMapper;

    @Autowired
    private RoleMapper roleMapper;

    @Override
    public UserDetails loadUserByUsername(String username)
            throws UsernameNotFoundException {

        // step1: 按用户名查用户
        QueryWrapper<UserDO> wrapper = new QueryWrapper<>();
        wrapper.eq("username", username)
               .eq("status", 1);  // 只查有效用户（status=1）

        UserDO userDO = userMapper.selectOne(wrapper);

        if (userDO == null) {
            log.warn("用户不存在: {}", username);
            // 必须抛出这个异常，Security 会据此返回 "用户名或密码错误"
            throw new UsernameNotFoundException("用户不存在: " + username);
        }

        // step2: 查询该用户的角色列表
        List<RoleDO> roles = roleMapper.selectRolesByUserId(userDO.getId());

        // step3: 将角色名转换为 Spring Security 的 GrantedAuthority 格式
        List<GrantedAuthority> authorities = roles.stream()
                .map(role -> new SimpleGrantedAuthority(role.getRoleName()))
                .collect(Collectors.toList());

        /**
         * User 是 Spring Security 提供的 UserDetails 标准实现
         * 参数：用户名、密码（BCrypt 哈希值）、权限列表
         *
         * 注意：数据库中存储的必须是 BCrypt 加密后的密码
         * Security 会用 BCryptPasswordEncoder 验证：
         *   BCryptPasswordEncoder.matches(输入的明文密码, 数据库中的哈希值)
         */
        return new org.springframework.security.core.userdetails.User(
                userDO.getUsername(),
                userDO.getPassword(),  // 数据库中存的 BCrypt 哈希
                authorities
        );
    }
}
```

---

**④ `SecurityConfig.java` — Spring Security 核心配置**

```java
@Configuration
@EnableWebSecurity   // 开启 Web 安全，接管默认配置
@EnableGlobalMethodSecurity(prePostEnabled = true)
// prePostEnabled = true：开启方法级别的安全注解
// 让 @PreAuthorize("hasRole('ADMIN')") 等注解生效
@Slf4j
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    private UserDetailsServiceImpl userDetailsService;

    /**
     * 密码编码器：BCrypt 算法
     * 注册为 Bean，供注册接口加密密码使用
     *
     * BCrypt 的特点：
     * 1. 每次加密同一个密码，结果都不同（内置随机 salt）
     * 2. 验证时：encoder.matches(明文, 密文) 返回 true/false
     * 3. 算法设计为"慢速"，暴力破解成本极高
     */
    @Bean
    public BCryptPasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    /**
     * 将 AuthenticationManager 暴露为 Bean
     * 登录接口需要它来执行用户名密码认证
     */
    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }

    /**
     * 核心：配置 HTTP 安全规则
     */
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            // 1. 关闭 CSRF 防护
            // CSRF 是针对 Cookie/Session 的攻击，JWT 不依赖 Cookie，不需要 CSRF 防护
            .csrf().disable()

            // 2. 关闭 Session（JWT 是无状态的，不使用 Session）
            .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()

            // 3. 定义接口访问规则
            .authorizeRequests()
                // 公开接口：无需认证即可访问
                .antMatchers(
                    "/api/auth/login",     // 登录
                    "/api/auth/register",  // 注册
                    "/swagger-ui.html",    // Swagger 文档
                    "/webjars/**",
                    "/v2/api-docs",
                    "/swagger-resources/**"
                ).permitAll()

                // 管理员专属接口
                .antMatchers(
                    HttpMethod.DELETE, "/api/users/**"  // 只有管理员能删除用户
                ).hasRole("ADMIN")

                // 其余所有 /api/** 接口：登录后即可访问（不限角色）
                .antMatchers("/api/**").authenticated()

                // 其他路径（如静态资源）：完全开放
                .anyRequest().permitAll()
            .and()

            // 4. 关闭默认的表单登录（我们用 JWT 接口登录）
            .formLogin().disable()

            // 5. 关闭默认的 HTTP Basic 认证
            .httpBasic().disable();

        // 6. 后续 C19 中在这里插入 JwtAuthenticationFilter
    }

    /**
     * 配置认证方式：告诉 Security 用我们的 UserDetailsService
     * 和 BCrypt 来验证用户名密码
     */
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService)
            .passwordEncoder(passwordEncoder());
    }
}
```

---

#### 🧪 运行与验证

```bash
# 启动项目，现在 /api/auth/login 可以不登录访问（返回 404，因为还没实现）
curl http://localhost:8080/api/auth/login
# 返回 404（接口不存在），而不是 401（未授权）
# 说明 permitAll() 配置生效

# /api/users 需要认证
curl "http://localhost:8080/api/users?pageNo=1&pageSize=10&username=username1"
# 返回 403 Forbidden 或 401 Unauthorized
# 说明接口保护已生效
```

---

### 📦 Commit 19 —— 登录注册接口 & JWT 验证过滤器

> `feat: 添加登录/注册接口（API）与 JWT 过滤器（Filter）`

---

#### 🎯 本步目标

实现登录和注册接口，颁发 JWT Token；实现 `JwtAuthenticationFilter` 在每次请求时从 Header 中提取并验证 Token，将认证信息注入 Security 上下文。

---

#### 📂 目录变动

```
新增：
└── src/main/java/.../
    ├── filter/
    │   └── JwtAuthenticationFilter.java  ← JWT Token 验证过滤器
    ├── service/
    │   └── AuthService.java              ← 认证服务接口
    ├── service/impl/
    │   └── AuthServiceImpl.java          ← 登录、注册业务逻辑
    └── controller/
        └── AuthController.java           ← 登录、注册 HTTP 接口

修改：
└── src/main/java/.../
    └── config/SecurityConfig.java        ← 将 JWT Filter 插入过滤器链
```

---

#### 💻 核心代码与详解

**① `JwtAuthenticationFilter.java` — 每次请求都执行的 Token 验证**

```java
/**
 * OncePerRequestFilter：保证每个请求只执行一次的过滤器基类
 * 避免在 Forward 等场景下被重复执行
 */
@Component
@Slf4j
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Autowired
    private JwtUtils jwtUtils;

    @Autowired
    private UserDetailsServiceImpl userDetailsService;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        try {
            // step1: 从请求 Header 中提取 Token
            String token = extractTokenFromRequest(request);

            // step2: 验证 Token 并将认证信息注入 Security 上下文
            if (token != null && validateAndAuthenticate(token)) {
                log.debug("JWT 认证成功，用户: {}",
                    jwtUtils.getUsernameFromToken(token));
            }

        } catch (Exception e) {
            // 过滤器中的异常不能让它冒泡，否则会返回 500
            // 只记录日志，让后续的 Security 过滤器去返回 401
            log.warn("JWT 认证过程中发生异常: {}", e.getMessage());
        }

        // 无论认证成功还是失败，都放行给后续过滤器
        // 后续 FilterSecurityInterceptor 会检查认证状态，
        // 如果未认证则返回 401
        filterChain.doFilter(request, response);
    }

    /**
     * 从 Authorization Header 中提取 Token
     * 标准格式：Authorization: Bearer eyJhbGci...
     */
    private String extractTokenFromRequest(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");

        // 检查 Header 是否存在且以 "Bearer " 开头
        if (StringUtils.hasText(bearerToken)
                && bearerToken.startsWith("Bearer ")) {
            // 截取 "Bearer " 之后的部分（7个字符）
            return bearerToken.substring(7);
        }
        return null;
    }

    /**
     * 验证 Token 并将认证信息写入 SecurityContext
     */
    private boolean validateAndAuthenticate(String token) {

        // 从 Token 中提取用户名
        String username = jwtUtils.getUsernameFromToken(token);

        /**
         * 关键检查：SecurityContextHolder.getContext().getAuthentication() == null
         * 意思是：当前请求还没有被认证过
         *
         * 为什么要检查这个？
         * 在同一个请求的生命周期内，可能有多个过滤器执行，
         * 如果已经认证过了，不需要再重复认证
         */
        if (username != null
                && SecurityContextHolder.getContext()
                        .getAuthentication() == null) {

            // 从数据库加载用户完整信息（包括权限）
            UserDetails userDetails =
                    userDetailsService.loadUserByUsername(username);

            // 验证 Token 有效性（签名正确 + 未过期 + 用户名匹配）
            if (jwtUtils.validateToken(token, userDetails.getUsername())) {

                /**
                 * 创建认证对象，写入 SecurityContext
                 * UsernamePasswordAuthenticationToken 三参数构造器表示"已认证"
                 * 参数1：principal（当前用户，UserDetails 对象）
                 * 参数2：credentials（密码，认证后不需要了，传 null）
                 * 参数3：authorities（权限列表，从 UserDetails 中获取）
                 */
                UsernamePasswordAuthenticationToken authentication =
                        new UsernamePasswordAuthenticationToken(
                                userDetails,
                                null,
                                userDetails.getAuthorities()
                        );

                // 附加请求信息（IP、Session 等），便于审计
                authentication.setDetails(
                        new WebAuthenticationDetailsSource()
                                .buildDetails(request)
                );

                // 将认证信息存入当前线程的 Security 上下文
                // 后续 Controller 可以通过 SecurityContextHolder 获取当前用户
                SecurityContextHolder.getContext()
                        .setAuthentication(authentication);

                return true;
            }
        }
        return false;
    }
}
```

---

**② `AuthServiceImpl.java` — 登录和注册的业务逻辑**

```java
@Service
@Slf4j
public class AuthServiceImpl implements AuthService {

    @Autowired
    private AuthenticationManager authenticationManager;

    @Autowired
    private UserDetailsServiceImpl userDetailsService;

    @Autowired
    private UserMapper userMapper;

    @Autowired
    private RoleMapper roleMapper;

    @Autowired
    private JwtUtils jwtUtils;

    @Autowired
    private BCryptPasswordEncoder passwordEncoder;

    @Override
    public TokenDTO login(LoginDTO loginDTO) {

        try {
            /**
             * authenticationManager.authenticate()：
             * 执行完整的用户名密码认证流程：
             * 1. 调用 UserDetailsService.loadUserByUsername()
             * 2. 用 BCryptPasswordEncoder.matches() 验证密码
             * 3. 验证通过 → 返回认证成功的 Authentication 对象
             * 4. 验证失败 → 抛出 BadCredentialsException
             */
            Authentication authentication = authenticationManager.authenticate(
                    new UsernamePasswordAuthenticationToken(
                            loginDTO.getUsername(),
                            loginDTO.getPassword()
                    )
            );

            // 认证成功后，将认证信息存入 Security 上下文
            SecurityContextHolder.getContext()
                    .setAuthentication(authentication);

        } catch (BadCredentialsException e) {
            // 统一返回"用户名或密码错误"，不区分是用户名错误还是密码错误
            // 这样做是为了防止"用户名枚举攻击"（攻击者通过错误信息判断用户名是否存在）
            throw new BusinessException(
                    ErrorCodeEnum.PARAM_ERROR, "用户名或密码错误"
            );
        }

        // 查询用户角色，生成 Token
        UserDO userDO = getUserByUsername(loginDTO.getUsername());
        List<String> roles = getRoleNames(userDO.getId());
        String token = jwtUtils.generateToken(loginDTO.getUsername(), roles);

        log.info("用户登录成功: {}, 角色: {}", loginDTO.getUsername(), roles);

        return TokenDTO.builder()
                .accessToken(token)
                .tokenType("Bearer")
                .expiresAt(System.currentTimeMillis() + 86400000L)
                .username(loginDTO.getUsername())
                .roles(roles)
                .build();
    }

    @Override
    public void register(RegisterDTO registerDTO) {

        // step1: 验证两次密码是否一致
        if (!registerDTO.getPassword()
                .equals(registerDTO.getConfirmPassword())) {
            throw new BusinessException(
                    ErrorCodeEnum.PARAM_ERROR, "两次密码输入不一致"
            );
        }

        // step2: 检查用户名是否已存在
        QueryWrapper<UserDO> wrapper = new QueryWrapper<>();
        wrapper.eq("username", registerDTO.getUsername());
        if (userMapper.selectCount(wrapper) > 0) {
            throw new BusinessException(
                    ErrorCodeEnum.PARAM_ERROR, "用户名已存在"
            );
        }

        // step3: 创建用户，密码必须 BCrypt 加密存储
        UserDO userDO = new UserDO();
        BeanUtils.copyProperties(registerDTO, userDO);

        /**
         * BCrypt 加密：永远不要明文存储密码！
         * encode() 每次调用结果都不同（内置随机 salt）
         * 存入数据库的是加密后的哈希值，如：
         * $2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy
         */
        userDO.setPassword(passwordEncoder.encode(registerDTO.getPassword()));

        userMapper.insert(userDO);

        // step4: 分配默认角色（普通用户 ROLE_USER）
        assignDefaultRole(userDO.getId());

        log.info("新用户注册成功: {}", registerDTO.getUsername());
    }

    private void assignDefaultRole(Long userId) {
        // 查找 ROLE_USER 角色的 ID
        QueryWrapper<RoleDO> wrapper = new QueryWrapper<>();
        wrapper.eq("role_name", "ROLE_USER");
        RoleDO roleUser = roleMapper.selectOne(wrapper);

        if (roleUser != null) {
            // 插入 user_role 关联记录
            // （此处简化，实际应有 UserRoleMapper）
        }
    }

    private UserDO getUserByUsername(String username) {
        QueryWrapper<UserDO> wrapper = new QueryWrapper<>();
        wrapper.eq("username", username);
        return userMapper.selectOne(wrapper);
    }

    private List<String> getRoleNames(Long userId) {
        return roleMapper.selectRolesByUserId(userId)
                .stream()
                .map(RoleDO::getRoleName)
                .collect(Collectors.toList());
    }
}
```

---

**③ `AuthController.java` — 对外暴露认证接口**

```java
@RestController
@RequestMapping("/api/auth")
@Slf4j
@Api(tags = {"认证管理"})
public class AuthController {

    @Autowired
    private AuthService authService;

    /**
     * 登录接口：返回 JWT Token
     * POST /api/auth/login
     */
    @PostMapping("/login")
    @ApiOperation(value = "用户登录", notes = "返回 JWT Token，后续请求在 Header 中携带：Authorization: Bearer {token}")
    public ResponseResult<TokenDTO> login(
            @Validated @RequestBody LoginDTO loginDTO) {

        TokenDTO tokenDTO = authService.login(loginDTO);
        return ResponseResult.success(tokenDTO);
    }

    /**
     * 注册接口
     * POST /api/auth/register
     */
    @PostMapping("/register")
    @ApiOperation(value = "用户注册")
    public ResponseResult<String> register(
            @Validated @RequestBody RegisterDTO registerDTO) {

        authService.register(registerDTO);
        return ResponseResult.success("注册成功！");
    }

    /**
     * 获取当前登录用户信息（需要认证）
     * GET /api/auth/me
     */
    @GetMapping("/me")
    @ApiOperation(value = "获取当前登录用户信息")
    public ResponseResult<TokenDTO> getCurrentUser() {

        // 从 Security 上下文获取当前用户
        Authentication authentication =
                SecurityContextHolder.getContext().getAuthentication();

        String username = authentication.getName();
        List<String> roles = authentication.getAuthorities()
                .stream()
                .map(GrantedAuthority::getAuthority)
                .collect(Collectors.toList());

        return ResponseResult.success(
                TokenDTO.builder()
                        .username(username)
                        .roles(roles)
                        .build()
        );
    }
}
```

---

**④ 更新 `SecurityConfig.java` — 插入 JWT Filter**

```java
@Autowired
private JwtAuthenticationFilter jwtAuthenticationFilter;

@Override
protected void configure(HttpSecurity http) throws Exception {
    http
        .csrf().disable()
        .sessionManagement()
            .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
        .and()
        .authorizeRequests()
            .antMatchers("/api/auth/**", "/swagger-ui.html",
                         "/webjars/**", "/v2/api-docs",
                         "/swagger-resources/**").permitAll()
            .antMatchers(HttpMethod.DELETE, "/api/users/**").hasRole("ADMIN")
            .antMatchers("/api/**").authenticated()
            .anyRequest().permitAll()
        .and()
        .formLogin().disable()
        .httpBasic().disable();

    /**
     * 将 JwtAuthenticationFilter 插入过滤器链
     * addFilterBefore(A, B)：在 B 执行之前执行 A
     *
     * UsernamePasswordAuthenticationFilter 是 Security 处理表单登录的过滤器
     * 我们要在它之前先执行 JWT 验证，让已持有 Token 的请求直接通过
     */
    http.addFilterBefore(
        jwtAuthenticationFilter,
        UsernamePasswordAuthenticationFilter.class
    );
}
```

---

#### 🧪 运行与验证

```bash
# step1：注册新用户
curl -X POST http://localhost:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "jwtuser",
    "password": "secure123",
    "confirmPassword": "secure123",
    "email": "jwt@test.com",
    "age": 25,
    "phone": "13500135000"
  }'
# 期望：{ "success": true, "result": "注册成功！" }

# step2：登录，获取 Token
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{ "username": "jwtuser", "password": "secure123" }'
# 期望：
# {
#   "success": true,
#   "result": {
#     "accessToken": "eyJhbGci...",
#     "tokenType": "Bearer",
#     "username": "jwtuser",
#     "roles": ["ROLE_USER"]
#   }
# }

# 保存 Token（实际操作中复制 accessToken 的值）
TOKEN="eyJhbGci..."

# step3：携带 Token 访问受保护接口
curl "http://localhost:8080/api/users?pageNo=1&pageSize=10&username=username1" \
  -H "Authorization: Bearer $TOKEN"
# 期望：正常返回用户列表

# step4：不携带 Token 访问受保护接口
curl "http://localhost:8080/api/users?pageNo=1&pageSize=10&username=username1"
# 期望：403 或 401

# step5：普通用户尝试删除（无 ADMIN 角色）
curl -X DELETE http://localhost:8080/api/users/123 \
  -H "Authorization: Bearer $TOKEN"
# 期望：403 Forbidden（权限不足）
```

---

### 📦 Commit 20 —— 替换 TODO，接口权限精细化

> `refactor: 将当前用户引入元数据处理器并实现数据审计`

---

#### 🎯 本步目标

终于可以替换 `CommonMetaObjectHandler` 里的 `TODO` 了：从 Spring Security 上下文中获取当前登录用户，填入 `creator` 和 `operator` 字段；同时为关键接口添加方法级别的权限控制。

---

#### 📂 目录变动

```
修改：
└── src/main/java/.../
    ├── util/CommonMetaObjectHandler.java  ← 替换 TODO，读取当前登录用户
    └── controller/UserController.java     ← 加方法级权限注解
```

---

#### 💻 核心代码与详解

**① 更新 `CommonMetaObjectHandler.java` — 终于填上 TODO**

```java
@Component
@Slf4j
public class CommonMetaObjectHandler implements MetaObjectHandler {

    @Override
    public void insertFill(MetaObject metaObject) {
        String currentUser = getCurrentLoginUser();
        log.info("insertFill，当前操作用户: {}", currentUser);

        this.strictInsertFill(metaObject, "created",
                LocalDateTime.class, LocalDateTime.now());
        this.strictInsertFill(metaObject, "modified",
                LocalDateTime.class, LocalDateTime.now());
        this.strictInsertFill(metaObject, "creator",
                String.class, currentUser);   // ← TODO 已替换
        this.strictInsertFill(metaObject, "operator",
                String.class, currentUser);   // ← TODO 已替换
        this.strictInsertFill(metaObject, "status", Integer.class, 1);
        this.strictInsertFill(metaObject, "version", Long.class, 1L);
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        String currentUser = getCurrentLoginUser();
        log.info("updateFill，当前操作用户: {}", currentUser);

        this.strictUpdateFill(metaObject, "modified",
                LocalDateTime.class, LocalDateTime.now());
        this.strictUpdateFill(metaObject, "operator",
                String.class, currentUser);   // ← TODO 已替换
    }

    /**
     * 从 Spring Security 上下文中获取当前登录用户名
     * 这就是 TODO 了这么久等待的那行代码
     */
    private String getCurrentLoginUser() {
        try {
            Authentication authentication =
                    SecurityContextHolder.getContext().getAuthentication();

            // 未认证或匿名访问时（如公开接口），返回 "system"
            if (authentication == null
                    || !authentication.isAuthenticated()
                    || authentication instanceof AnonymousAuthenticationToken) {
                return "system";
            }

            // 已认证：返回当前登录用户名
            return authentication.getName();

        } catch (Exception e) {
            // 防御性处理：异步线程中 SecurityContext 可能为空
            // 例如：@Async 导出任务，线程切换后 SecurityContext 丢失
            log.warn("获取当前用户失败（可能在异步线程中）: {}", e.getMessage());
            return "system";
        }
    }
}
```

> ⚠️ **`@Async` 方法中 SecurityContext 会丢失！**
>
> ```
> 主线程：SecurityContext 有当前用户信息
>     ↓ @Async 提交任务到线程池
> 子线程：SecurityContext 是空的！（默认不传递）
>
> 解决方案：在 ExecutorConfig 中配置 SecurityContext 传递
> ```
>
> ```java
> // ExecutorConfig.java 中增加：
> @Bean("exportServiceExecutor")
> public Executor exportServiceExecutor() {
>     ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
>     // ... 原有配置不变
>
>     // 用 DelegatingSecurityContextExecutor 包装，
>     // 自动将主线程的 SecurityContext 复制到子线程
>     executor.initialize();
>     return new DelegatingSecurityContextExecutorService(
>             executor.getThreadPoolExecutor()
>     );
> }
> ```

---

**② 更新 `UserController.java` — 方法级权限控制**

```java
@RestController
@RequestMapping("/api/users")
@Validated
@Slf4j
@Api(tags = {"用户管理"})
public class UserController {

    /**
     * @PreAuthorize：方法执行前检查权限
     * 需要 SecurityConfig 上有 @EnableGlobalMethodSecurity(prePostEnabled = true)
     *
     * hasRole('ADMIN')：检查当前用户是否有 ROLE_ADMIN 角色
     * 比 SecurityConfig 中的路径匹配更灵活：可以根据方法参数动态判断
     */
    @PreAuthorize("hasRole('ADMIN')")
    @CacheEvict(cacheNames = "users-cache", allEntries = true)
    @PostMapping
    public ResponseResult save(
            @Validated(InsertValidationGroup.class) @RequestBody UserDTO userDTO) {
        // ... 逻辑不变
    }

    /**
     * hasAnyRole：拥有任意一个指定角色即可
     * 更新操作：管理员或普通用户都能更新（但普通用户只能改自己的，更细的控制见下面）
     */
    @PreAuthorize("hasAnyRole('ADMIN', 'USER')")
    @CacheEvict(cacheNames = "users-cache", allEntries = true)
    @PutMapping("/{id}")
    public ResponseResult update(
            @NotNull @PathVariable("id") Long id,
            @Validated(UpdateValidationGroup.class) @RequestBody UserDTO userDTO) {
        // ... 逻辑不变
    }

    /**
     * 只有管理员才能删除用户
     * 双重保护：SecurityConfig 的路径规则 + @PreAuthorize 方法注解
     */
    @PreAuthorize("hasRole('ADMIN')")
    @CacheEvict(cacheNames = "users-cache", allEntries = true)
    @DeleteMapping("/{id}")
    public ResponseResult delete(
            @NotNull @PathVariable("id") Long id) {
        // ... 逻辑不变
    }

    /**
     * 查询接口：登录后所有人都能访问
     * isAuthenticated()：检查当前用户是否已认证（比 hasRole 宽松）
     */
    @PreAuthorize("isAuthenticated()")
    @Cacheable(cacheNames = "users-cache")
    @GetMapping
    public ResponseResult<PageResult> query(/* ... */) {
        // ... 逻辑不变
    }
}
```

---

#### 🧪 运行与验证：完整认证授权流程

```bash
# ========== 场景1：完整登录和访问流程 ==========

# 1. 用管理员账号登录（需要先在数据库给某个用户分配 ROLE_ADMIN）
ADMIN_TOKEN=$(curl -s -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"username1","password":"password1"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['result']['accessToken'])")

# 2. 管理员可以删除用户
curl -X DELETE http://localhost:8080/api/users/123 \
  -H "Authorization: Bearer $ADMIN_TOKEN"
# 期望：{ "success": true, "result": "删除成功！" }

# ========== 场景2：权限不足测试 ==========

# 1. 普通用户登录
USER_TOKEN=$(curl -s -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"jwtuser","password":"secure123"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['result']['accessToken'])")

# 2. 普通用户尝试删除 → 权限不足
curl -X DELETE http://localhost:8080/api/users/123 \
  -H "Authorization: Bearer $USER_TOKEN"
# 期望：403 Forbidden

# ========== 场景3：验证审计字段 ==========

# 1. 管理员登录并新增用户
curl -X POST http://localhost:8080/api/users \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"username":"audituser","password":"audit123","email":"audit@test.com","age":28,"phone":"13400134000"}'

# 2. 查数据库，验证 creator 字段是当前管理员的用户名
# SELECT username, creator, operator FROM user WHERE username = 'audituser';
# 期望：creator = "username1"（管理员的用户名，不再是 "system"）
```

---

### ✅ **v6.0 阶段完整回顾**
>
> 五次提交，系统获得了完整的认证授权能力：
>
> ```
> C16：引入依赖，理解 Security 自动保护所有接口的原理
>   ↓
> C17：建立权限数据模型（角色表、关联表、认证 DTO）
>   ↓
> C18：JWT 工具类 + Security 核心配置（接口规则、密码编码器）
>   ↓
> C19：登录注册接口 + JwtAuthenticationFilter（Token → SecurityContext）
>   ↓
> C20：替换所有 TODO + 方法级权限控制
>
> 完整请求链路：
> 请求进来
>   → TraceIdFilter（C8）：分配追踪 ID
>   → JwtAuthenticationFilter（C19）：解析 Token → 写入 SecurityContext
>   → RateLimitInterceptor（C10）：限流检查
>   → FilterSecurityInterceptor（Security 内置）：权限检查
>   → @PreAuthorize（C20）：方法级权限检查
>   → Controller → Service → Mapper
>   → CommonMetaObjectHandler（C20）：从 SecurityContext 读当前用户填充审计字段
> ```

---

###  📌 **整个教程的学习路径回顾**
>
> ```
> v1.0  骨架 + CRUD          → 从零跑起来
> v2.0  校验 + 异常           → 入口防护
> v3.0  缓存 + 限流 + 追踪    → 生产健壮性
> v4.0  文件 + 异步导出        → 常见业务场景
> v5.0  自动填充 + 乐观锁 + 文档 → 工程收尾
> v6.0  认证 + 授权 + 审计     → 安全基线
> ```
>
> 至此，这个项目已经具备了一个中小型生产系统的完整技术基线。你可以在这个基础上继续扩展：Redis 分布式缓存、消息队列、分库分表、微服务拆分……每一个方向都是一个新的旅程。