
## 第二部分：Git 规范化提交规划

### v1.0 阶段（共 4 次提交）

| # | Commit Message | 新增/修改文件 | 核心功能点 |
|---|---------------|-------------|-----------|
| C1 | `chore: 初始化项目结构并添加核心依赖` | `pom.xml`、`application.properties`、`AllLearningApplication.java` | 初始化工程，引入 Spring Boot Web、MyBatis-Plus、MySQL 驱动、Lombok |
| C2 | `feat: 添加用户领域层（DO/DTO/VO）与数据库表结构（Schema）` | `UserDO.java`、`UserDTO.java`、`UserVO.java`、`UserQueryDTO.java`、`db/schema.sql`、`db/data.sql` | 建立实体分层模型，创建数据库表结构和测试数据 |
| C3 | `feat: 添加 UserMapper 与 MyBatis-Plus 相关配置` | `UserMapper.java`、`MybatisPlusConfig.java` | 接入 MyBatis-Plus BaseMapper，配置分页插件 |
| C4 | `feat: 添加 UserService 并暴露增删改查（CRUD）REST API` | `UserService.java`、`UserServiceImpl.java`、`UserController.java`、`PageQuery.java`、`PageResult.java`、`ResponseResult.java` | 完整 CRUD 业务逻辑 + REST 接口，统一分页响应结构 |

### v2.0 阶段（共 3 次提交）

| # | Commit Message | 新增/修改文件 | 核心功能点 |
|---|---------------|-------------|-----------|
| C5 | `feat: 添加校验组（Validation Groups）并对 UserDTO 进行注解标注` | `InsertValidationGroup.java`、`UpdateValidationGroup.java`、`UserDTO.java`（加注解）、`ValidatorUtils.java` | 引入分组校验，Service 层手动校验入口 |
| C6 | `feat: 添加全局异常处理器（Global Exception Handler）与错误码枚举（Error Code Enum）` | `ErrorCodeEnum.java`、`BusinessException.java`、`GlobalExceptionHandler.java` | 自定义业务异常体系，`@ControllerAdvice` 统一捕获所有异常 |
| C7 | `refactor: 在控制器（Controllers）中应用校验注解` | `UserController.java`（加 `@Validated`）| Controller 层开启声明式校验，与 Service 层校验形成双保险 |

### v3.0 阶段（共 3 次提交）

| # | Commit Message | 新增/修改文件 | 核心功能点 |
|---|---------------|-------------|-----------|
| C8 | `feat: 添加 TraceId 过滤器用于日志追踪。` | `TraceIdFilter.java`、`application.properties`（修改日志格式）| Servlet Filter 注入 MDC，日志格式加入 `[traceId]` 占位 |
| C9 | `feat: 添加 Caffeine 缓存用于用户查询` | `CaffeineCacheConfig.java`、`UserController.java`（加 `@Cacheable`/`@CacheEvict`）| 配置本地缓存管理器，查询走缓存，写操作驱逐缓存 |
| C10 | `feat: 添加全局限流拦截器（Global Rate Limiter Interceptor）` | `RateLimitInterceptor.java`、`WebConfig.java` | Guava 令牌桶限流，拦截器注册到 `/api/**` 路径 |

### v4.0 阶段（共 3 次提交）

| # | Commit Message | 新增/修改文件 | 核心功能点 |
|---|---------------|-------------|-----------|
| C11 | `feat: 添加文件上传服务（Service）与控制器（Controller）` | `FileService.java`、`LocalFileServiceImpl.java`、`FileController.java`、`uploads/.gitkeep` | 本地文件落盘，`InputStream` 流式写入，接口返回文件名 |
| C12 | `feat: 添加异步线程池配置（Async Thread Pool Config）` | `ExecutorConfig.java` | 定义导出专用线程池，核心数 = CPU 核数，最大数 = 2x，`AbortPolicy` 拒绝策略 |
| C13 | `feat: 添加支持异步的 EasyExcel 批量导出功能` | `UserExportDTO.java`、`LocalDateTimeStringConverter.java`、`ExcelExportService.java`、`ExcelExportServiceImpl.java`、`UserController.java`（加 export 接口）| 分页查询 + EasyExcel 分批写 Sheet，`@Async` 异步化，内存流转文件流上传 |

### v5.0 阶段（共 2 次提交）

| # | Commit Message | 新增/修改文件 | 核心功能点 |
|---|---------------|-------------|-----------|
| C14 | `feat: 为 UserDO 添加自动填充（Auto-fill）与乐观锁（Optimistic Lock）支持` | `CommonMetaObjectHandler.java`、`UserDO.java`（加 `@TableField(fill=...)`、`@Version`）、`MybatisPlusConfig.java`（加乐观锁拦截器）| 创建/更新时间自动填充，`version` 字段防并发覆写 |
| C15 | `feat: 集成 Swagger2 并添加 API 注解` | `Swagger2Config.java`、`WebConfig.java`（加静态资源映射）、`UserController.java`（加 `@Api` 注解）、`ResponseResult.java`（加 `@ApiModel`）| Swagger UI 可访问，核心接口有文档描述 |

### v6.0 阶段（ 共5 次提交）

| # | Commit Message | 新增/修改文件 | 核心功能 |
|---|---------------|---------|---|
| C16 | `chore: 添加 Spring Security 与 JWT 依赖` | pom.xml（新增 Spring Security 和 JWT 依赖） | 引入依赖，理解自动配置 |
| C17 | `feat: 添加用户认证领域层（包括角色实体、Token 实体与 Repository 仓储接口）` | RoleDO.java、LoginDTO.java、RegisterDTO.java、TokenDTO.java 、RoleMapper.java 、auth_schema.sql 、schema.sql （user 表新增 password_hash 字段说明） | 角色表、用户角色关联、Token 实体 |
| C18 | `feat: 实现 JWT 工具类（Utils）与 Security 配置（Config）` | JwtUtils.java、UserDetailsServiceImpl.java、SecurityConfig.java、application.properties （新增 JWT 配置项） |JWT 工具类、Security 核心配置  |
| C19 | `feat: 添加登录/注册接口（API）与 JWT 过滤器（Filter）` | JwtAuthenticationFilter.java、AuthService.java、AuthServiceImpl.java 、AuthController.java 、SecurityConfig.java（将 JWT Filter 插入过滤器链） | 登录注册接口、Token 验证过滤器 |
| C20 | `refactor: 将当前用户引入元数据处理器并实现数据审计` | CommonMetaObjectHandler.java、UserController.java | 替换 TODO、接口权限精细化控制 |

---