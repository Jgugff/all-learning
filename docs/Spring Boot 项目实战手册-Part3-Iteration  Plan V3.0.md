## 第三部分：实战工作手册 v3.0 阶段：缓存 & 限流 & 日志追踪

> 覆盖 **v3.0 阶段的 C8、C9、C10** 三次提交

---

### 背景：为什么需要这个阶段？

v2.0 结束后，你的系统在功能上已经完整，但面对真实流量时会暴露三个致命问题：

```
问题1：同一个用户列表被反复查询
  → 每次都打数据库，SELECT * FROM user WHERE username = ?
  → 数据库连接池被大量重复查询占满，真正的写请求被阻塞

问题2：恶意爬虫或 DDoS 攻击
  → 接口没有任何流量限制
  → 每秒 1000 个请求，数据库直接打挂，整个服务不可用

问题3：线上出了 bug，翻日志
  → 2024-01-01 10:00:00 用户A的请求日志
  → 2024-01-01 10:00:00 用户B的请求日志   ← 交织在一起
  → 2024-01-01 10:00:00 用户A的请求日志
  → 完全无法追踪一次请求的完整链路
```

这三个问题本质上都是**系统健壮性**问题，需要在业务逻辑之外建立横切能力。

---

### 📦 Commit 8 —— 为每个请求打上唯一追踪 ID

> `feat: 添加 TraceId 过滤器用于日志追踪。`

---

#### 🎯 本步目标

实现一个 Servlet Filter，在请求进入系统的第一时间为其分配全局唯一的 `TraceId`，并注入到 MDC（日志上下文）中，使该请求后续产生的所有日志都自动携带这个 ID，从而在海量日志中精准定位一次请求的完整链路。

---

#### 📂 目录变动

```
新增：
└── src/main/java/.../
    └── filter/
        └── TraceIdFilter.java          ← Servlet Filter，注入 TraceId

修改：
├── src/main/java/.../
│   └── AllLearningApplication.java     ← 加 @ServletComponentScan
└── src/main/resources/
    └── application.properties          ← 日志格式加入 TraceId 占位符
```

---

#### 💻 核心代码与详解

**① 先理解 MDC 是什么**

```
MDC（Mapped Diagnostic Context）：日志框架（Logback/Log4j2）提供的线程本地存储。

你可以把它理解为一个专属于当前线程的 Map<String, String>：
  - 放入：MDC.put("traceId", "abc-123")
  - 读取：MDC.get("traceId")
  - 清除：MDC.remove("traceId") 或 MDC.clear()

在日志配置中用 %X{traceId} 占位符，日志框架每次打印时会
自动从当前线程的 MDC 中取出这个值填进去。

效果：
  不用 MDC：2024-01-01 10:00:00 INFO  查询用户列表
  用了 MDC：2024-01-01 10:00:00 INFO  [abc-123] 查询用户列表
                                         ↑ 这个请求的所有日志都带这个 ID
```

---

**② `TraceIdFilter.java` — 请求追踪过滤器**

```java
@WebFilter(urlPatterns = "/*")  // 拦截所有请求
@Order(1)  // 过滤器执行顺序，数字越小越先执行
           // TraceId 要最先注入，后续所有日志才能带上它
public class TraceIdFilter implements Filter {

    private static final String TRACE_ID = "traceId";

    @Override
    public void doFilter(
            ServletRequest request,
            ServletResponse response,
            FilterChain chain) throws IOException, ServletException {

        try {
            // step1: 尝试从请求参数中获取 TraceId
            // 这样做的好处：上游系统（如网关）可以传入自己的 TraceId，
            // 实现跨服务的全链路追踪（分布式追踪的雏形）
            String traceId = request.getParameter(TRACE_ID);

            // step2: 如果上游没有传，自己生成一个
            if (StringUtils.isEmpty(traceId)) {
                traceId = UUID.randomUUID().toString()
                        .replace("-", ""); // 去掉横杠，缩短长度
            }

            // step3: 注入 MDC，后续所有日志自动携带
            MDC.put(TRACE_ID, traceId);

            // step4: 放行请求，继续执行后续的 Filter 和 Controller
            chain.doFilter(request, response);

        } finally {
            // ⚠️ 必须清除！这是最容易被遗忘的步骤
            // Tomcat 使用线程池复用线程，如果不清除 MDC，
            // 这个线程处理下一个请求时会带着上一个请求的 TraceId
            // 导致日志追踪完全混乱
            MDC.clear();
        }
    }
}
```

> ⚠️ **`finally` 块清除 MDC 是铁律，不可省略**
>
> ```
> 线程池场景下不清除 MDC 的后果：
>
> 请求A进来 → MDC.put("traceId", "aaa") → 处理完成
>                                              ↓ 线程回到线程池（未清除）
> 请求B进来 → 复用了同一个线程 → MDC 里还是 "aaa"
>          → 请求B的日志全部打印成 [aaa] ← 追踪完全错误！
> ```

---

**③ 更新 `AllLearningApplication.java` — 开启 Servlet 组件扫描**

```java
@SpringBootApplication
@ServletComponentScan  // 现在才加！让 Spring 扫描并注册 @WebFilter 注解
                       // 没有这个注解，TraceIdFilter 就是一个普通的 Java 类，完全不生效
public class AllLearningApplication {
    public static void main(String[] args) {
        SpringApplication.run(AllLearningApplication.class, args);
    }
}
```

> 💡 **`@WebFilter` 为什么需要 `@ServletComponentScan` 才能生效？**
>
> `@WebFilter` 是 Servlet 3.0 的标准注解，不属于 Spring 生态。Spring Boot 默认只扫描 Spring 自己的注解（`@Component`、`@Service` 等）。加上 `@ServletComponentScan` 后，Spring Boot 才会额外扫描 Servlet 规范的注解。
>
> 替代方案：用 `@Component` + 实现 `Filter` 接口，再通过 `FilterRegistrationBean` 注册，但代码更繁琐，不如 `@WebFilter` 简洁。

---

**④ 更新 `application.properties` — 日志格式加入 TraceId**

```properties
## 自定义日志输出格式
## %clr(...)  → 带颜色输出（仅控制台有效，日志文件会显示为纯文本）
## %d{...}    → 时间
## %5p        → 日志级别，固定5个字符宽度（如 " INFO"、"ERROR"）
## %X{traceId} → 从 MDC 中取 traceId 的值 ← 这是本次新增的关键部分
## %t          → 线程名
## %-40logger  → 类名，最多40字符
## %m%n        → 消息 + 换行

logging.pattern.console=%clr(%d{yyyy-MM-dd HH:mm:ss.SSS}){faint} \
  %clr(%5p) \
  %clr([%X{traceId}]){blue} \
  %clr(${PID}){magenta} \
  %clr(---){faint} \
  %clr([%15.15t]){faint} \
  %clr(%-40.40logger{39}){cyan} \
  %clr(:){faint} %m%n
```

---

#### 🧪 运行与验证

重启项目，发送任意请求，观察控制台日志：

```bash
curl "http://localhost:8080/api/users?pageNo=1&pageSize=10&username=username1"
```

**不带 TraceId 参数时**（系统自动生成）：
```
2024-01-01 10:00:00.123  INFO [a1b2c3d4e5f6] 12345 --- [nio-8080-exec-1] c.i.z.a.controller.UserController : 收到查询请求...
2024-01-01 10:00:00.125  INFO [a1b2c3d4e5f6] 12345 --- [nio-8080-exec-1] c.i.z.a.service.impl.UserServiceImpl    : 开始查询第1页数据
                               ↑ 同一个 TraceId，一眼看出是同一次请求
```

**带 TraceId 参数时**（前端或网关传入）：
```bash
curl "http://localhost:8080/api/users?pageNo=1&pageSize=10&username=username1&traceId=my-custom-trace-id"
# 日志中会显示 [my-custom-trace-id]
```

---

### 📦 Commit 9 —— 接入 Caffeine 本地缓存

> `feat: 添加 Caffeine 缓存用于用户查询`

---

#### 🎯 本步目标

引入 Caffeine 作为本地缓存，对用户查询接口的结果进行缓存，避免重复的数据库查询；同时配置写操作（新增/更新/删除）触发缓存驱逐，保证缓存数据与数据库的一致性。

---

#### 📂 目录变动

```
新增：
└── src/main/java/.../
    └── config/
        └── CaffeineCacheConfig.java    ← 定义缓存管理器和缓存策略

修改：
└── src/main/java/.../
    └── controller/UserController.java  ← 加 @Cacheable / @CacheEvict 注解

pom.xml ← 新增 Caffeine 和 spring-context-support 依赖
```

---

#### 💻 核心代码与详解

**① `pom.xml` — 新增缓存相关依赖**

```xml
<!-- Spring Cache 抽象层：提供 @Cacheable 等注解的支持基础 -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context-support</artifactId>
    <!-- 版本由 spring-boot-starter-parent 统一管理，无需指定 -->
</dependency>

<!-- Caffeine：目前性能最好的 Java 本地缓存库
     基准测试中吞吐量比 Guava Cache 高出数倍
     Spring Boot 2.x 内置对它的自动配置支持 -->
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
    <!-- 版本同样由 parent 管理 -->
</dependency>
```

> 💡 **为什么选 Caffeine 而不是 Redis？**
>
> | 维度 | Caffeine（本地缓存）| Redis（分布式缓存）|
> |------|--------------------|--------------------|
> | 访问速度 | 纳秒级（JVM 堆内存）| 毫秒级（网络 I/O）|
> | 数据共享 | ❌ 单机，多实例不共享 | ✅ 多实例共享 |
> | 部署复杂度 | 零依赖 | 需要单独部署 Redis |
> | 适合场景 | 单机热点数据、不频繁变更的配置 | 分布式会话、跨服务共享数据 |
>
> 本项目是单体应用，Caffeine 完全够用。生产中的分布式系统通常两层缓存叠加：Caffeine（一级）+ Redis（二级）。

---

**② `CaffeineCacheConfig.java` — 缓存管理器配置**

```java
@Configuration
@EnableCaching  // 开启 Spring Cache 注解支持
                // 没有这个注解，@Cacheable 等注解完全不生效（是个常见坑）
@Slf4j
public class CaffeineCacheConfig {

    @Bean("cacheManager")
    public CacheManager cacheManager() {

        // SimpleCacheManager：Spring 提供的最简单的 CacheManager 实现
        // 它本身不缓存数据，只是管理一组 Cache 实例的容器
        SimpleCacheManager cacheManager = new SimpleCacheManager();

        ArrayList<CaffeineCache> caches = new ArrayList<>();

        // 为"用户查询"场景定制一个缓存实例
        // 每个业务场景可以有不同的缓存策略（大小、过期时间等）
        caches.add(new CaffeineCache(
            "users-cache",  // 缓存名称，@Cacheable(cacheNames="users-cache") 中引用

            Caffeine.newBuilder()

                // 最大缓存条目数：超过 1000 条时，Caffeine 会用 W-TinyLFU 算法
                // 自动淘汰"最不可能被再次访问"的条目
                // W-TinyLFU 比 LRU 有更高的缓存命中率
                .maximumSize(1000)

                // 基于访问时间的过期策略：
                // 最后一次读或写之后 120 秒过期
                // 适合读多写少的场景（用户列表变化不频繁）
                //
                // 对比另一个选项 expireAfterWrite（写入后过期）：
                // expireAfterAccess：只要有访问，过期时间就延续 ← 热点数据永不过期
                // expireAfterWrite：写入后固定时间过期，不管有没有访问 ← 数据时效性更强
                .expireAfterAccess(120, TimeUnit.SECONDS)

                .build()
        ));

        cacheManager.setCaches(caches);
        return cacheManager;
    }
}
```

> 💡 **`maximumSize` 应该设多大？**
>
> 一个简单的估算公式：
> ```
> 单条缓存数据大小（字节）× maximumSize ≤ 可用堆内存 × 缓存占比上限（建议 ≤ 20%）
>
> 例如：单条 UserDTO 约 500 字节
>       maximumSize = 1000
>       占用堆内存 ≈ 500 * 1000 = 500KB ← 远小于 20% 限制，安全
> ```
> 如果单条数据很大（如包含大文本字段），要相应降低 `maximumSize`，防止 OOM。

---

**③ 更新 `UserController.java` — 加缓存注解**

```java
@RestController
@RequestMapping("/api/users")
@Validated
@Slf4j
public class UserController {

    /**
     * @CacheEvict：驱逐缓存
     * cacheNames：指定操作哪个缓存区
     * allEntries = true：清空 "users-cache" 下的所有条目
     *
     * 为什么新增时要清缓存？
     * 因为新增了一个用户，之前缓存的"查询 username=tom 的结果"可能已经过期
     * 最简单的策略就是全部清掉，下次查询重新从数据库加载
     */
    @CacheEvict(cacheNames = "users-cache", allEntries = true)
    @PostMapping
    public ResponseResult save(
            @Validated(InsertValidationGroup.class) @RequestBody UserDTO userDTO) {
        // ... 代码不变
    }

    /**
     * 更新和删除同样需要驱逐缓存
     * 原因：数据已变更，缓存的旧数据不能继续使用
     */
    @CacheEvict(cacheNames = "users-cache", allEntries = true)
    @PutMapping("/{id}")
    public ResponseResult update(/* ... */) {
        // ... 代码不变
    }

    @CacheEvict(cacheNames = "users-cache", allEntries = true)
    @DeleteMapping("/{id}")
    public ResponseResult delete(/* ... */) {
        // ... 代码不变
    }

    /**
     * @Cacheable：缓存查询结果
     *
     * 执行逻辑（Spring AOP 代理）：
     * 1. 计算缓存 Key（默认用所有方法参数组合生成）
     * 2. 在 "users-cache" 中查找这个 Key
     * 3. 命中 → 直接返回缓存值，方法体完全不执行（数据库查询被跳过）
     * 4. 未命中 → 执行方法体，将返回值存入缓存，再返回给调用方
     */
    @Cacheable(cacheNames = "users-cache")
    @GetMapping
    public ResponseResult<PageResult> query(
            @NotNull Integer pageNo,
            @NotNull Integer pageSize,
            @Validated UserQueryDTO query) {

        // 只有缓存未命中时这行日志才会打印
        // 如果命中缓存，这行日志不会出现 ← 可以用这个特性验证缓存是否生效
        log.info("缓存未命中，查询数据库！pageNo={}, username={}", pageNo, query.getUsername());

        // ... 查询逻辑不变
    }
}
```

> ⚠️ **`@Cacheable` 的三个典型陷阱**
>
> **陷阱1：同类内部调用缓存不生效**
> ```java
> // UserController 内部这样调用：
> public void someMethod() {
>     this.query(1, 10, queryDTO);  // ❌ 缓存不生效！
> }
> // 原因：@Cacheable 基于 Spring AOP 代理实现，
> // 同类内部调用绕过了代理对象，直接调用原始方法
> // 解决：通过 Spring 容器获取代理对象再调用，或重构代码
> ```
>
> **陷阱2：方法返回 null 时缓存行为**
> ```java
> // 如果查询结果为 null，默认会缓存 null 值
> // 下次同样的查询直接返回 null（缓存穿透的变种）
> // 解决：@Cacheable(unless = "#result == null") 不缓存 null 结果
> ```
>
> **陷阱3：缓存 Key 冲突**
> ```java
> // 默认 Key 是所有参数的组合
> // query(1, 10, queryDTO{username="tom"}) 的 Key 是 [1, 10, UserQueryDTO{username=tom}]
> // 如果 UserQueryDTO 没有正确实现 equals/hashCode，缓存 Key 可能不准确
> // Lombok 的 @Data 已经自动生成 equals/hashCode，本项目没有这个问题
> ```

---

#### 🧪 运行与验证

**验证缓存命中**

```bash
# 第一次查询（冷启动，缓存未命中）
curl "http://localhost:8080/api/users?pageNo=1&pageSize=10&username=username1"
# 控制台打印："缓存未命中，查询数据库！"
# SQL 日志："SELECT * FROM user WHERE username = ?"

# 第二次相同查询（缓存命中）
curl "http://localhost:8080/api/users?pageNo=1&pageSize=10&username=username1"
# 控制台：没有任何日志（方法体完全没有执行）
# 没有 SQL 日志
```

**验证缓存驱逐**

```bash
# 执行一次新增，触发 @CacheEvict
curl -X POST http://localhost:8080/api/users \
  -H "Content-Type: application/json" \
  -d '{ "username": "newuser", "password": "123456", "email": "new@test.com", "age": 20, "phone": "13900139000" }'

# 再次执行相同查询，缓存已被清空，重新查数据库
curl "http://localhost:8080/api/users?pageNo=1&pageSize=10&username=username1"
# 控制台再次打印："缓存未命中，查询数据库！"
```

---

### 📦 Commit 10 —— 全局限流拦截器

> `feat: 添加全局限流拦截器（Global Rate Limiter Interceptor）`

---

#### 🎯 本步目标

引入 Guava 的令牌桶算法实现全局 QPS 限流，将限流逻辑封装在 `HandlerInterceptor` 中，注册到 Spring MVC 的拦截器链上，对所有 `/api/**` 接口进行统一流量控制，超限时返回标准的限流错误响应。

---

#### 📂 目录变动

```
新增：
└── src/main/java/.../
    ├── interceptor/
    │   └── RateLimitInterceptor.java   ← 限流拦截器
    └── config/
        └── WebConfig.java              ← 注册拦截器到 Spring MVC

pom.xml ← 新增 Guava 依赖
```

---

#### 💻 核心代码与详解

**① `pom.xml` — 新增 Guava 依赖**

```xml
<!-- Guava：Google 出品的 Java 工具库
     我们只用它的 RateLimiter（令牌桶限流）
     它还提供：不可变集合、字符串处理、缓存、函数式工具等 -->
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>28.0-jre</version>
</dependency>
```

---

**② 先理解令牌桶算法**

```
令牌桶算法图解：

系统以固定速率（如 10 QPS）往桶里放令牌：
  每 100ms 放 1 个令牌（1秒放10个）

每个请求来了，先从桶里拿 1 个令牌：
  拿到令牌 → 放行请求 ✅
  桶里没有令牌 → 限流，拒绝请求 ❌

优点：
  1. 允许一定程度的突发流量（桶里存积了令牌时）
  2. 长期来看吞吐量不超过设定的 QPS

对比漏桶算法：
  漏桶：严格匀速，完全不允许突发（更适合保护下游系统）
  令牌桶：允许短暂突发（更适合应对正常流量波动）
```

---

**③ `RateLimitInterceptor.java` — 限流拦截器**

```java
@Slf4j
@Component  // 注册为 Spring Bean，才能被 WebConfig 注入使用
public class RateLimitInterceptor implements HandlerInterceptor {

    /**
     * 全局共享的令牌桶限流器
     *
     * static final：单例，整个应用只有一个限流器实例
     * 如果每个请求 new 一个，限流就完全失效了
     *
     * RateLimiter.create(1)：每秒生成 1 个令牌，即 QPS 上限为 1
     * 注意：这里设为 1 是为了方便本地测试时能快速触发限流
     * 生产环境应根据压测结果设置合理的值（如 500、1000）
     */
    private static final RateLimiter rateLimiter = RateLimiter.create(1);

    /**
     * preHandle：在 Controller 方法执行之前调用
     * 返回 true  → 放行，继续执行后续的拦截器和 Controller
     * 返回 false → 拦截，请求到此为止（需要在此处写入响应）
     * 抛出异常  → 被 GlobalExceptionHandler 捕获（我们选这种方式，响应格式统一）
     */
    @Override
    public boolean preHandle(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler) throws Exception {

        /**
         * tryAcquire()：非阻塞尝试获取令牌
         *   成功获取 → 返回 true
         *   令牌不足 → 立即返回 false（不等待）
         *
         * 对比 acquire()：会阻塞等待直到拿到令牌
         *   acquire() 不适合 Web 场景：线程被阻塞，Tomcat 线程池会被耗尽
         *   tryAcquire() 快速失败，线程立即释放，系统稳定性更好
         */
        if (!rateLimiter.tryAcquire()) {
            log.warn("触发限流！请求路径：{}，IP：{}",
                    request.getRequestURI(),
                    request.getRemoteAddr());

            // 抛出业务异常，由 GlobalExceptionHandler 统一处理
            // 这样限流响应也是标准的 ResponseResult 格式
            throw new BusinessException(ErrorCodeEnum.RATE_LIMIT_ERROR);
        }

        return true;
    }
}
```

> 💡 **为什么限流要用 `throw` 而不是直接写 `response.getWriter().write(...)`？**
>
> 直接写响应的问题：
> ```java
> // ❌ 手动写响应
> response.setContentType("application/json");
> response.getWriter().write("{\"success\":false,\"message\":\"限流\"}");
> return false;
> // 这段 JSON 是硬编码的，格式和 ResponseResult 不一致
> // 将来 ResponseResult 结构调整了，这里漏改就会出 bug
> ```
>
> 抛出异常的好处：
> ```java
> // ✅ 抛异常，让 GlobalExceptionHandler 统一处理
> throw new BusinessException(ErrorCodeEnum.RATE_LIMIT_ERROR);
> // 自动走 GlobalExceptionHandler，格式永远和其他接口一致
> // ErrorCodeEnum 改了消息，这里自动跟着变
> ```

---

**④ `WebConfig.java` — 注册拦截器和静态资源**

```java
@Configuration
@EnableWebMvc   // 开启 Spring MVC 的完整配置，允许我们自定义 MVC 行为
                // ⚠️ 加了 @EnableWebMvc 后，Spring Boot 的 MVC 自动配置会退出
                //    你需要手动配置静态资源映射，否则 /swagger-ui.html 等访问不了
@Slf4j
public class WebConfig implements WebMvcConfigurer {

    @Resource
    private RateLimitInterceptor rateLimitInterceptor;

    /**
     * 注册拦截器：把 rateLimitInterceptor 加入 Spring MVC 的拦截器链
     */
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(rateLimitInterceptor)
                // 只拦截 /api/ 开头的业务接口
                // 不拦截 /swagger-ui.html、/uploads/ 等静态资源
                .addPathPatterns("/api/**");
    }

    /**
     * 静态资源映射：让 /uploads/** 的 URL 访问到本地文件夹
     * 这是文件下载功能的基础，v4.0 上传文件后通过这里访问
     */
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {

        // 将 /uploads/** 映射到项目根目录下的 uploads/ 文件夹
        // file: 前缀表示本地文件系统路径
        registry.addResourceHandler("/uploads/**")
                .addResourceLocations("file:uploads/");

        // Swagger2 的静态资源映射（v5.0 添加 Swagger 时需要）
        // 这里提前加上，避免 v5.0 时再改这个文件
        registry.addResourceHandler("/swagger-ui.html")
                .addResourceLocations("classpath:/META-INF/resources/");
        registry.addResourceHandler("/webjars/**")
                .addResourceLocations("classpath:/META-INF/resources/webjars/");
    }
}
```

> ⚠️ **`@EnableWebMvc` 加了之后 Spring Boot 自动配置失效，这是高频踩坑点**
>
> ```
> 不加 @EnableWebMvc：
>   Spring Boot 自动配置 MVC，/static、/public 等目录自动作为静态资源，
>   Jackson 自动配置，MessageConverter 自动注册
>
> 加了 @EnableWebMvc：
>   你完全接管 MVC 配置，所有自动配置退出
>   需要在 WebMvcConfigurer 的方法中手动配置一切
>
> 本项目加它的原因：需要自定义静态资源路径（文件上传目录）
> 如果你的项目不需要深度定制 MVC，可以不加 @EnableWebMvc，
> 只在 application.properties 中配置即可
> ```

---

#### 🧪 运行与验证

**快速触发限流（QPS 设为 1，连续发送两个请求）**

```bash
# 连续快速发送两次请求（在同一秒内）
curl "http://localhost:8080/api/users?pageNo=1&pageSize=10&username=username1" &
curl "http://localhost:8080/api/users?pageNo=1&pageSize=10&username=username1" &

# 第一个请求：正常响应
# { "success": true, "result": { ... } }

# 第二个请求：限流响应（格式与普通响应完全一致）
# { "success": false, "code": "3005", "message": "限流异常", "result": null }
```

控制台日志（带 TraceId，C8 的成果）：
```
2024-01-01 10:00:00.100  WARN [a1b2c3d4] --- [exec-1] ...RateLimitInterceptor : 触发限流！请求路径：/api/users，IP：127.0.0.1
```

**验证静态资源映射（为 v4.0 文件上传做准备）**

```bash
# 在项目根目录的 uploads/ 文件夹下放一个测试文件 test.txt
echo "hello" > uploads/test.txt

# 通过 HTTP 访问
curl http://localhost:8080/uploads/test.txt
# 期望返回：hello
```

---

###  ✅ **v3.0 阶段完整回顾**
>
> 三次提交，系统获得了生产级的横切能力：
>
> ```
> HTTP 请求
>     ↓
> [TraceIdFilter]           C8：分配唯一 TraceId，注入 MDC
>     ↓
> [RateLimitInterceptor]    C10：令牌桶限流，超限抛 BusinessException
>     ↓
> [Controller @Cacheable]   C9：查询先走缓存，命中直接返回
>     ↓（未命中时）
> [Service + Mapper]        查数据库，结果存入缓存
>     ↓
> 响应（所有日志自动带 TraceId）
> ```
>
> 三个能力之间有清晰的优先级关系：
> - 限流在缓存**之前**：被限流的请求甚至不需要查缓存，直接拒绝
> - 缓存在数据库**之前**：缓存命中则完全不访问数据库
> - TraceId 贯穿**全程**：无论在哪个环节产生的日志，都能被追踪

---

###  📌 **下期预告 —— v4.0：文件上传 & 异步导出（C11 ~ C13）**
>
> 下一阶段将实现：
> - `LocalFileServiceImpl`：用 `InputStream` 流式写文件，不把文件加载到内存
> - `ExecutorConfig`：定义导出专用线程池，核心数绑定 CPU 核心数，防止线程爆炸
> - `ExcelExportServiceImpl`：EasyExcel 分页写入 + `@Async` 异步化，让百万行导出不 OOM、不超时