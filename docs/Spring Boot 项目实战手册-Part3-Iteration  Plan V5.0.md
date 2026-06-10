## 第三部分：实战工作手册 v5.0 阶段：API 文档 & 工程收尾

> 覆盖 **v5.0 阶段的 C14、C15** 两次提交

---

### 背景：为什么需要这个阶段？

v4.0 结束后，所有核心功能已经完备。但回头看代码，还有几处"欠债"需要还清：

```
欠债1：UserDO 的系统字段从未被自动赋值
  → 每次 insert 都要手动 userDO.setCreated(LocalDateTime.now())
  → 每次 update 都要手动 userDO.setModified(LocalDateTime.now())
  → 数据库里 created/modified 全是 null（看 data.sql 就知道有多痛）

欠债2：并发更新没有保护
  → 用户A和用户B同时读到 version=1 的记录
  → 用户A改完先提交，version 变成 2
  → 用户B用旧的 version=1 提交，覆盖了用户A的修改
  → 数据静默丢失，没有任何报错

欠债3：前端同学问："你们的接口文档在哪里？"
  → 回答："你自己看 Controller 代码吧"
  → 前后端联调效率极低，字段名全靠口口相传
```

v5.0 就是来还这三笔技术债的，同时也是整个项目的**收官阶段**。

---

### 📦 Commit 14 —— 字段自动填充 & 乐观锁

> `feat: 为 UserDO 添加自动填充（Auto-fill）与乐观锁（Optimistic Lock）支持`

---

#### 🎯 本步目标

实现 MyBatis-Plus 的元数据自动填充机制，让 `created`、`modified`、`creator`、`operator`、`status`、`version` 六个系统字段在新增和更新时自动赋值，无需业务代码手动处理；同时启用乐观锁拦截器，通过 `version` 字段保护并发更新的数据安全。

---

#### 📂 目录变动

```
新增：
└── src/main/java/.../
    └── util/
        └── CommonMetaObjectHandler.java   ← 公共元数据处理器（自动填充逻辑）

修改：
└── src/main/java/.../
    ├── domain/entity/UserDO.java          ← 为字段添加 @TableField(fill) 和 @Version
    └── config/MybatisPlusConfig.java      ← 注册乐观锁拦截器
```

---

#### 💻 核心代码与详解

**① 先理解自动填充的工作原理**

```
MyBatis-Plus 自动填充流程：

insert(userDO) 被调用
    ↓
MyBatis-Plus 拦截，检查 UserDO 的字段注解
    ↓
发现 @TableField(fill = FieldFill.INSERT) 标注的字段
    ↓
调用 CommonMetaObjectHandler.insertFill(metaObject)
    ↓
在 metaObject 中找到对应字段，设置值
    ↓
最终执行的 SQL 中，这些字段已经有值了

效果：业务代码 userDO.setCreated(...) 可以完全删掉
      框架层面自动处理，业务层零感知
```

---

**② 更新 `UserDO.java` — 为系统字段添加注解**

```java
@Data
@TableName("user")
public class UserDO implements Serializable {

    // 业务字段：无需自动填充，由业务代码负责
    private String username;
    private String password;
    private String email;
    private Integer age;
    private String phone;

    @TableId(type = IdType.ASSIGN_ID)
    private Long id;

    /**
     * FieldFill.INSERT：只在 INSERT 时自动填充
     * 创建时间：只有新建时才设置，之后永不修改
     */
    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime created;

    /**
     * FieldFill.INSERT_UPDATE：INSERT 和 UPDATE 时都自动填充
     * 修改时间：新建时初始化，每次更新时刷新为当前时间
     */
    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime modified;

    /**
     * 创建者：只在新建时填充，记录"谁创建的"
     */
    @TableField(fill = FieldFill.INSERT)
    private String creator;

    /**
     * 最后操作者：新建和更新时都填充，记录"谁最后改的"
     */
    @TableField(fill = FieldFill.INSERT_UPDATE)
    private String operator;

    /**
     * 逻辑删除状态：只在新建时初始化为 0（有效）
     * 删除操作时手动将其设为 -1，而不是真正 DELETE
     */
    @TableField(fill = FieldFill.INSERT)
    private Integer status;

    /**
     * @Version：MyBatis-Plus 乐观锁注解
     * 标记这个字段是乐观锁的版本号
     * 每次成功更新后，MyBatis-Plus 自动将此字段值 +1
     *
     * @TableField(fill = FieldFill.INSERT)：新建时自动初始化为 1
     */
    @Version
    @TableField(fill = FieldFill.INSERT)
    private Long version;
}
```

> 💡 **`@TableField(fill)` 和直接在代码里 `setXxx()` 有什么本质区别？**
>
> 表面上结果一样，但关注点分离：
> ```java
> // ❌ 业务代码中手动填充（职责混乱）
> public int save(UserDTO userDTO) {
>     UserDO userDO = new UserDO();
>     BeanUtils.copyProperties(userDTO, userDO);
>     userDO.setCreated(LocalDateTime.now());   // 这是系统关注点
>     userDO.setModified(LocalDateTime.now());  // 不应该出现在业务逻辑里
>     userDO.setStatus(1);
>     userDO.setVersion(1L);
>     return userMapper.insert(userDO);
> }
>
> // ✅ 框架自动填充（业务代码干净）
> public int save(UserDTO userDTO) {
>     UserDO userDO = new UserDO();
>     BeanUtils.copyProperties(userDTO, userDO);
>     return userMapper.insert(userDO);  // 系统字段由框架层处理
> }
> ```
>
> 更重要的是：假设系统有 50 张表，每张表都有这些系统字段，手动填充意味着 50 处代码要同步修改。自动填充只需改 `CommonMetaObjectHandler` 一处。

---

**③ `CommonMetaObjectHandler.java` — 元数据自动填充处理器**

```java
@Component  // 注册为 Spring Bean，MyBatis-Plus 会自动发现并使用它
@Slf4j
public class CommonMetaObjectHandler implements MetaObjectHandler {

    /**
     * INSERT 操作时触发
     * 对应 @TableField(fill = FieldFill.INSERT) 和
     *      @TableField(fill = FieldFill.INSERT_UPDATE) 的字段
     */
    @Override
    public void insertFill(MetaObject metaObject) {
        log.info("触发 insertFill，开始填充系统字段");

        /**
         * strictInsertFill 的"严格模式"含义：
         * 只有当字段当前值为 null 时，才进行填充
         * 如果业务代码已经手动 set 了值，不会覆盖
         *
         * 对比 setFieldValByName（非严格模式）：
         * 无论字段是否有值，强制覆盖
         *
         * 结论：strictInsertFill 更安全，是推荐用法
         *
         * 参数说明：
         * 参数1：metaObject（当前操作的实体对象的元数据包装）
         * 参数2：fieldName（Java 字段名，不是数据库列名）
         * 参数3：fieldType（字段类型，用于类型安全检查）
         * 参数4：fieldVal（要填充的值）
         */
        this.strictInsertFill(
            metaObject, "created", LocalDateTime.class, LocalDateTime.now()
        );
        this.strictInsertFill(
            metaObject, "modified", LocalDateTime.class, LocalDateTime.now()
        );

        /**
         * creator 和 operator：应该从当前登录用户的上下文中获取
         * 真实项目中：SecurityContextHolder.getContext().getAuthentication().getName()
         * 本项目尚未集成 Spring Security，用占位符代替
         * v6.0（如果有的话）接入 Security 后，这里改成真实取值
         */
        this.strictInsertFill(
            metaObject, "creator", String.class, getCurrentUser()
        );
        this.strictInsertFill(
            metaObject, "operator", String.class, getCurrentUser()
        );

        // 逻辑删除初始值：0 表示有效记录
        this.strictInsertFill(
            metaObject, "status", Integer.class, 0
        );

        // 乐观锁版本号初始值：从 1 开始（0 在语义上容易与"未初始化"混淆）
        this.strictInsertFill(
            metaObject, "version", Long.class, 1L
        );
    }

    /**
     * UPDATE 操作时触发
     * 对应 @TableField(fill = FieldFill.INSERT_UPDATE) 的字段
     */
    @Override
    public void updateFill(MetaObject metaObject) {
        log.info("触发 updateFill，刷新修改时间和操作人");

        this.strictUpdateFill(
            metaObject, "modified", LocalDateTime.class, LocalDateTime.now()
        );
        this.strictUpdateFill(
            metaObject, "operator", String.class, getCurrentUser()
        );
    }

    /**
     * 获取当前操作用户
     * TODO：集成 Spring Security 后替换为真实实现
     * 例如：return SecurityContextHolder.getContext()
     *              .getAuthentication().getName();
     */
    private String getCurrentUser() {
        return "system";
    }
}
```

> ⚠️ **`strictInsertFill` 中 fieldName 必须是 Java 字段名，常见错误：**
>
> ```java
> // ❌ 错误：使用数据库列名
> this.strictInsertFill(metaObject, "create_time", ...);
>
> // ✅ 正确：使用 Java 字段名
> this.strictInsertFill(metaObject, "created", ...);
>
> // 如果字段名写错，不会报错，只是静默不填充！
> // 排查方法：开启 debug 日志，看 MyBatis-Plus 的元数据处理日志
> ```

---

**④ 更新 `MybatisPlusConfig.java` — 注册乐观锁拦截器**

```java
@Configuration
public class MybatisPlusConfig {

    @Bean
    public PaginationInterceptor paginationInterceptor() {
        PaginationInterceptor interceptor = new PaginationInterceptor();
        interceptor.setDbType(DbType.MYSQL);
        return interceptor;
    }

    /**
     * 乐观锁拦截器
     *
     * 工作原理（以 update 为例）：
     *
     * 你的代码：userMapper.updateById(userDO)  其中 userDO.version = 1
     *
     * 没有乐观锁拦截器，执行的 SQL：
     *   UPDATE user SET password=?, modified=? WHERE id=?
     *
     * 有乐观锁拦截器，执行的 SQL：
     *   UPDATE user SET password=?, modified=?, version=2 WHERE id=? AND version=1
     *                                            ↑自动+1        ↑自动加条件
     *
     * 如果 WHERE id=? AND version=1 没有匹配到任何行（说明 version 已被别人改为 2）：
     *   updateById 返回 0（影响行数为 0）
     *   业务代码判断返回值为 0 → 告知用户"数据已被他人修改，请刷新后重试"
     */
    @Bean
    public OptimisticLockerInterceptor optimisticLockerInterceptor() {
        return new OptimisticLockerInterceptor();
    }
}
```

---

**⑤ 理解乐观锁的完整并发场景**

```
时间线：

T1：用户A 查询记录  → 得到 {id:1, password:"old", version:1}
T2：用户B 查询记录  → 得到 {id:1, password:"old", version:1}

T3：用户A 提交更新  → UPDATE ... WHERE id=1 AND version=1
                    → 匹配到 1 行，更新成功，version 变为 2
                    → 数据库：{id:1, password:"A改的", version:2}

T4：用户B 提交更新  → UPDATE ... WHERE id=1 AND version=1
                    → version 已经是 2，WHERE 条件不匹配，影响行数 = 0
                    → updateById 返回 0
                    → UserServiceImpl 判断返回值为 0
                    → 返回"更新失败"给用户B
                    → 用户B 收到提示："数据已被他人修改，请刷新后重试"

结果：用户A的修改被保留，用户B的修改被安全拒绝，数据没有丢失 ✅
```

> 💡 **乐观锁 vs 悲观锁的选择依据**
>
> | 维度 | 乐观锁（CAS + version）| 悲观锁（SELECT FOR UPDATE）|
> |------|----------------------|--------------------------|
> | 假设 | 冲突很少发生 | 冲突经常发生 |
> | 加锁时机 | 提交更新时检查 | 读取数据时就加锁 |
> | 并发性能 | 高（读操作无锁）| 低（读操作会阻塞） |
> | 适合场景 | 读多写少（用户信息、商品详情）| 写多读少（库存扣减、账户转账）|
> | 冲突处理 | 返回失败，让用户重试 | 等待锁释放，自动重试 |
>
> 用户信息更新是典型的读多写少场景，乐观锁是正确选择。

---

#### 🧪 运行与验证

**① 验证自动填充**

```bash
# 新增一个用户（不传任何系统字段）
curl -X POST http://localhost:8080/api/users \
  -H "Content-Type: application/json" \
  -d '{
    "username": "autotest",
    "password": "123456",
    "email": "auto@test.com",
    "age": 25,
    "phone": "13700137000"
  }'
```

直接查数据库验证：

```sql
SELECT id, username, created, modified, creator, operator, status, version
FROM user
WHERE username = 'autotest';

-- 期望结果（系统字段全部自动填充）：
-- id       | username  | created             | modified            | creator | status | version
-- 1234...  | autotest  | 2024-01-01 10:00:00 | 2024-01-01 10:00:00 | system  | 0      | 1
--                         ↑ 自动填充           ↑ 自动填充             ↑ 自动   ↑ 自动  ↑ 自动
```

**② 验证乐观锁**

```bash
# step1：查询刚新增的用户，确认 version=1
# （从数据库直接取 id 和 version）

# step2：第一次更新（携带 version=1），应该成功
curl -X PUT http://localhost:8080/api/users/{刚才新增的id} \
  -H "Content-Type: application/json" \
  -d '{ "password": "newpass1", "version": 1 }'
# 期望：{ "success": true, "result": "更新成功！" }
# 此时数据库 version 已变为 2

# step3：用旧的 version=1 再次更新，应该失败
curl -X PUT http://localhost:8080/api/users/{同一个id} \
  -H "Content-Type: application/json" \
  -d '{ "password": "newpass2", "version": 1 }'
# 期望：{ "success": false, "code": "3003", "message": "更新失败" }
# 说明乐观锁生效，version 不匹配时更新被拒绝
```

查看控制台 SQL 日志，验证乐观锁拦截器自动修改了 SQL：

```sql
-- 实际执行的 SQL（乐观锁拦截器自动加了 version 条件和 version+1）：
UPDATE user SET password=?, modified=?, version=2 WHERE id=? AND version=1
```

---

### 📦 Commit 15 —— 集成 Swagger2，生成交互式 API 文档

> `feat: 集成 Swagger2 并添加 API 注解`

---

#### 🎯 本步目标

集成 Swagger2/SpringFox，通过配置类自动扫描 Controller 生成 API 文档，并为核心接口和返回实体添加语义化注解，最终提供一个可在浏览器中查看和调试的交互式 API 文档页面。

---

#### 📂 目录变动

```
新增：
└── src/main/java/.../
    └── config/
        └── Swagger2Config.java              ← Swagger2 配置类

修改：
└── src/main/java/.../
    ├── controller/UserController.java       ← 加 @Api、@ApiOperation 等注解
    └── domain/common/ResponseResult.java    ← 加 @ApiModel、@ApiModelProperty 注解

pom.xml ← 新增 Swagger2 依赖
（WebConfig.java 中的静态资源映射在 C10 已提前配置好，此处无需修改）
```

---

#### 💻 核心代码与详解

**① `pom.xml` — 新增 Swagger2 依赖**

```xml
<!-- Swagger2 核心：提供注解和 API 分析能力 -->
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.9.2</version>
</dependency>

<!-- Swagger UI：将 Swagger2 分析的结果渲染成可交互的网页 -->
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.9.2</version>
</dependency>
```

> ⚠️ **Swagger2（2.9.2）与 Spring Boot 2.6+ 的兼容性问题**
>
> 本项目使用 Spring Boot 2.2.5，没有问题。但如果你的项目升级到 Spring Boot 2.6+，会遇到启动报错：
> ```
> Failed to start bean 'documentationPluginsBootstrapper'
> ```
>
> 解决方案（二选一）：
> ```properties
> # 方案1：在 application.properties 中关闭新的路径匹配策略
> spring.mvc.pathmatch.matching-strategy=ant_path_matcher
>
> # 方案2：升级到 Swagger3（springdoc-openapi），API 更现代
> # 换成 springdoc-openapi-ui 依赖，注解体系略有不同
> ```

---

**② `Swagger2Config.java` — Swagger2 配置类**

```java
@Configuration
@EnableSwagger2  // 开启 Swagger2 功能，让 SpringFox 开始扫描和分析接口
public class Swagger2Config {

    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)

                // API 基本信息：显示在文档页面顶部
                .apiInfo(buildApiInfo())

                // 接口扫描配置
                .select()
                // 只扫描这个包下的 Controller，排除框架自带的接口
                .apis(RequestHandlerSelectors
                        .basePackage("com.imooc.zhangxiaoxi.alllearning.controller"))
                // 暴露所有路径
                .paths(PathSelectors.any())
                .build();
    }

    private ApiInfo buildApiInfo() {
        return new ApiInfoBuilder()
                .title("ALL-LEARNING 项目接口文档")
                .description("Spring Boot 综合实战项目，包含用户管理、文件上传、数据导出等功能")
                .contact(new Contact(
                    "开发团队",        // 联系人名称
                    "https://github.com",  // 联系主页
                    "dev@example.com"  // 联系邮箱
                ))
                .version("1.0.0")
                .build();
    }
}
```

---

**③ 更新 `UserController.java` — 添加接口级 Swagger 注解**

```java
@RestController
@RequestMapping("/api/users")
@Validated
@Slf4j

/**
 * @Api：类级别注解，描述整个 Controller 的功能
 * value：简短描述（某些 UI 版本展示）
 * tags：标签，用于对接口分组，同一 tags 的接口会归为一组显示
 */
@Api(
    value = "用户管理接口",
    tags = {"用户管理"},
    protocols = "http, https"
)
public class UserController {

    // save、delete、query 方法暂不加注解，重点展示 update 方法的完整注解写法

    /**
     * @ApiOperation：方法级别注解，描述单个接口
     * value：接口的简短描述（显示在接口标题上）
     * notes：详细说明（点开接口后显示的备注）
     * response：接口返回值的类型（帮助 Swagger 生成准确的响应示例）
     * httpMethod：HTTP 方法（冗余但明确）
     */
    @PutMapping("/{id}")
    @ApiOperation(
        value = "更新用户信息",
        notes = "根据用户 ID 更新用户信息，支持部分字段更新（只更新传入的非空字段）。" +
                "version 字段必传，用于乐观锁控制并发更新。",
        response = ResponseResult.class,
        httpMethod = "PUT"
    )

    /**
     * @ApiImplicitParams：描述方法参数（尤其是路径参数和请求头，
     * 因为这些无法从方法签名直接推断）
     */
    @ApiImplicitParams({
        @ApiImplicitParam(
            name = "id",
            value = "用户主键 ID",
            required = true,
            paramType = "path",   // path/query/body/header/form
            dataType = "Long",
            example = "1220708537638920191"
        ),
        @ApiImplicitParam(
            name = "userDTO",
            value = "用户更新信息（version 必填，其他字段选填）",
            required = true,
            paramType = "body",
            dataTypeClass = UserDTO.class  // 指定类后 Swagger 会展示该类的字段结构
        )
    })

    /**
     * @ApiResponses：描述可能的响应状态
     * 帮助前端了解不同业务结果对应的响应内容
     */
    @ApiResponses({
        @ApiResponse(code = 200, message = "更新成功，result 为 '更新成功！'"),
        @ApiResponse(code = 200, message = "更新失败（如乐观锁冲突），result 为 null，code 为 3003")
    })
    public ResponseResult update(
            @NotNull @PathVariable("id") Long id,
            @Validated(UpdateValidationGroup.class) @RequestBody UserDTO userDTO) {
        int rows = userService.update(id, userDTO);
        return rows == 1
                ? ResponseResult.success("更新成功！")
                : ResponseResult.failure(ErrorCodeEnum.UPDATE_FAILURE);
    }

    // ... 其他方法
}
```

---

**④ 更新 `ResponseResult.java` — 添加模型级 Swagger 注解**

```java
/**
 * @ApiModel：类级别注解，描述这个类在 Swagger 文档中的展示信息
 * 影响 Swagger UI 的 "Models" 区域和接口响应示例的展示
 */
@Data
@ApiModel(
    value = "统一响应结果",
    description = "所有接口的统一返回格式，前端通过 success 字段判断请求是否成功"
)
public class ResponseResult<T> implements Serializable {

    /**
     * @ApiModelProperty：字段级别注解，描述字段在文档中的信息
     * name：字段名（通常和 Java 字段名一致）
     * value：字段的中文描述
     * required：是否必须
     * example：示例值（帮助前端理解字段含义）
     */
    @ApiModelProperty(
        value = "是否成功：true 成功 / false 失败",
        required = true,
        example = "true"
    )
    private Boolean success;

    @ApiModelProperty(
        value = "业务错误码，成功时为 null，失败时参考 ErrorCodeEnum",
        required = false,
        example = "3002"
    )
    private String code;

    @ApiModelProperty(
        value = "错误描述信息，成功时为 null",
        example = "新增失败"
    )
    private String message;

    @ApiModelProperty(
        value = "业务数据，泛型类型，不同接口返回不同结构"
    )
    private T result;

    // ... 静态工厂方法不变
}
```

> 💡 **Swagger 注解有必要全都写吗？**
>
> 遵循"按价值写注解"的原则：
>
> | 注解 | 写的价值 | 建议 |
> |------|---------|------|
> | `@Api(tags)` | 接口分组，文档结构清晰 | ✅ 必写 |
> | `@ApiOperation(value)` | 每个接口的一句话说明 | ✅ 必写 |
> | `@ApiOperation(notes)` | 详细说明、使用注意事项 | 复杂接口写，简单接口可省 |
> | `@ApiImplicitParam` | 路径参数、请求头说明 | ✅ 路径参数必写 |
> | `@ApiModelProperty` | 实体字段说明 | 共享实体类写，内部 DTO 可省 |
> | `@ApiResponse` | 各种响应码说明 | 有多种业务结果的接口写 |
>
> **不要为了"文档完整"而写一堆没有意义的注解**，`@ApiOperation(value="save", notes="save")` 这种注解比不写更糟糕。

---

#### 🧪 运行与验证

**① 启动项目，访问 Swagger UI**

```
浏览器打开：http://localhost:8080/swagger-ui.html
```

正常情况下应该看到：

```
ALL-LEARNING 项目接口文档
Spring Boot 综合实战项目，包含用户管理、文件上传、数据导出等功能

[用户管理]  ← 折叠的接口分组，点击展开
  POST  /api/users          新增用户
  GET   /api/users          查询用户列表（分页）
  GET   /api/users/export   导出用户数据
  DELETE /api/users/{id}    删除用户
  PUT   /api/users/{id}     更新用户信息

[文件管理]
  POST  /api/files/upload   文件上传
```

**② 在 Swagger UI 中在线调试接口**

以"更新用户信息"为例：
1. 点击 `PUT /api/users/{id}` 展开
2. 点击右上角 `Try it out` 按钮
3. 在 `id` 输入框填入真实的用户 ID
4. 在 `userDTO` 的 JSON 编辑器中填入：
   ```json
   { "password": "swagger-test", "version": 1 }
   ```
5. 点击 `Execute`
6. 观察 `Responses` 区域的返回结果

**③ 验证整个项目的"完整性检查"**

按照以下顺序执行一遍完整的业务流程，验证所有功能正常协作：

```bash
# 1. 新增用户（触发 @CacheEvict + 自动填充）
curl -X POST http://localhost:8080/api/users \
  -H "Content-Type: application/json" \
  -d '{"username":"finaltest","password":"123456","email":"final@test.com","age":30,"phone":"13600136000"}'

# 2. 查询用户（第一次查数据库，结果存入缓存）
curl "http://localhost:8080/api/users?pageNo=1&pageSize=10&username=finaltest"

# 3. 再次查询（命中缓存，不打数据库，控制台无 SQL 日志）
curl "http://localhost:8080/api/users?pageNo=1&pageSize=10&username=finaltest"

# 4. 更新用户（携带 version=1，乐观锁检查通过，触发 @CacheEvict）
curl -X PUT http://localhost:8080/api/users/{上一步得到的id} \
  -H "Content-Type: application/json" \
  -d '{"password":"newpass","version":1}'

# 5. 用旧 version 再次更新（乐观锁拦截，更新失败）
curl -X PUT http://localhost:8080/api/users/{同上id} \
  -H "Content-Type: application/json" \
  -d '{"password":"hackpass","version":1}'
# 期望：{ "success": false, "code": "3003", "message": "更新失败" }

# 6. 上传文件
curl -X POST http://localhost:8080/api/files/upload -F "file=@README.md"

# 7. 触发异步导出（立即返回，后台处理）
curl "http://localhost:8080/api/users/export?username=finaltest&filename=final.xlsx"

# 8. 等待 2 秒后检查导出文件是否生成
sleep 2 && ls uploads/
```

---

> ✅ **v5.0 阶段完整回顾，暨整个项目收官**
>
> 两次提交，还清了三笔技术债：
>
> ```
> 技术债1（系统字段手动赋值）→ CommonMetaObjectHandler 自动填充
> 技术债2（并发更新数据丢失）→ @Version + OptimisticLockerInterceptor
> 技术债3（接口文档靠口口相传）→ Swagger2 自动生成交互式文档
> ```

---

## 项目阶段学习总结

历经 **15 次 Git 提交**、**5 个迭代版本**，你从零构建了一个生产可用的 Spring Boot 综合项目。下面用两张图做最终的全局回顾：

---

### 🗺️ 完整架构鸟瞰图

```
┌─────────────────────────────────────────────────────────────────┐
│                         HTTP 请求                                │
└──────────────────────────┬──────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────────────┐
│  Servlet Filter 层（C8）                                         │
│  TraceIdFilter：分配 TraceId → MDC → 所有日志自动携带            │
└──────────────────────────┬───────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────────────┐
│  Spring MVC 拦截器层（C10）                                      │
│  RateLimitInterceptor：令牌桶限流，超限抛 BusinessException       │
└──────────────────────────┬───────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────────────┐
│  Controller 层（C4、C7、C11、C13、C15）                          │
│  @Validated 分组校验 → 业务分发 → @Cacheable/@CacheEvict         │
│  Swagger2 注解 → 自动生成 API 文档                               │
└──────────┬────────────────────────────────┬─────────────────────┘
           ↓ 同步                            ↓ 异步（@Async C13）
┌──────────────────────┐      ┌─────────────────────────────────┐
│  Service 层（C4、C5）│      │  ExcelExportService（C13）      │
│  ValidatorUtils 校验  │      │  export- 线程池（C12）          │
│  BeanUtils 实体转换   │      │  EasyExcel 分批写入             │
└──────────┬───────────┘      └──────────────┬──────────────────┘
           ↓                                  ↓
┌──────────────────────┐      ┌─────────────────────────────────┐
│  Mapper 层（C3）     │      │  FileService（C11）             │
│  MyBatis-Plus        │      │  LocalFileServiceImpl           │
│  BaseMapper CRUD     │      │  1KB 缓冲流式写入               │
│  自动填充（C14）      │      └──────────────┬──────────────────┘
│  乐观锁（C14）        │                     ↓
└──────────┬───────────┘              uploads/ 目录
           ↓
┌──────────────────────┐
│  MySQL 数据库（C1）  │
│  user 表（C2）       │
│  分页插件（C3）       │
└──────────────────────┘

┌─────────────────────────────────────────────────────┐
│  横切关注点（贯穿所有层）                             │
│  GlobalExceptionHandler（C6）：统一异常 → ResponseResult│
│  CaffeineCacheConfig（C9）：查询缓存，写操作驱逐       │
│  MDC TraceId（C8）：所有日志自动带追踪 ID              │
└─────────────────────────────────────────────────────┘
```

---

### 📋 15 次提交全景回顾

| 版本 | Commit | 核心产出 | 解决的问题 |
|------|--------|---------|-----------|
| v1.0 | C1 | 工程骨架 + 依赖 | 从零启动一个 Spring Boot 项目 |
| | C2 | DO / DTO / VO 三层实体 | 职责分离，防止数据裸奔 |
| | C3 | UserMapper + 分页插件 | 零 SQL 获得 CRUD 能力 |
| | C4 | UserService + UserController | 完整 CRUD REST 接口上线 |
| v2.0 | C5 | 分组校验 + ValidatorUtils | 非法参数无法写入系统 |
| | C6 | ErrorCodeEnum + GlobalExceptionHandler | 异常响应格式永远统一 |
| | C7 | Controller 激活 @Validated | HTTP 入口双重防护 |
| v3.0 | C8 | TraceIdFilter + MDC | 海量日志中精准定位一次请求 |
| | C9 | CaffeineCacheConfig + @Cacheable | 热点查询不打数据库 |
| | C10 | RateLimitInterceptor + WebConfig | 系统不被流量打挂 |
| v4.0 | C11 | FileService + LocalFileServiceImpl | 文件稳定上传落盘 |
| | C12 | ExecutorConfig 线程池 | 异步任务有序执行不崩溃 |
| | C13 | EasyExcel 分批导出 + @Async | 百万行导出不 OOM 不超时 |
| v5.0 | C14 | CommonMetaObjectHandler + 乐观锁 | 系统字段自动填充，并发更新安全 |
| | C15 | Swagger2 + API 注解 | 接口文档自动生成，前后端高效联调 |

---

