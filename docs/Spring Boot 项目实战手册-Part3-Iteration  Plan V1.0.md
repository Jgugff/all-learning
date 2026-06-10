
## 第三部分：实战工作手册 v1.0 阶段:骨架搭建 & 数据持久化

> 覆盖 **v1.0 阶段的 C1、C2、C3、C4** 四次提交

---

### 📦 Commit 1 —— 工程初始化

> `chore: 初始化项目结构并添加核心依赖`

---

#### 🎯 本步目标

用 Spring Initializr 创建项目骨架，引入本项目全程所需的核心依赖，跑通第一次 `main` 方法启动。

---

#### 📂 目录变动

```
新增：
├── pom.xml                          ← 核心依赖声明
├── src/main/resources/
│   └── application.properties       ← 数据库连接配置
└── src/main/java/.../
    └── AllLearningApplication.java  ← Spring Boot 启动类
```

---

#### 💻 核心代码与详解

**① `pom.xml` — 只引入这一步真正需要的依赖**

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.2.5.RELEASE</version>
</parent>

<dependencies>
    <!-- Web 能力：让项目能接收 HTTP 请求 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- MySQL 驱动：运行时才需要，scope=runtime 不参与编译 -->
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <scope>runtime</scope>
    </dependency>

    <!-- MyBatis-Plus：封装了 MyBatis，省去写基础 SQL -->
    <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>mybatis-plus-boot-starter</artifactId>
        <version>3.3.1</version>
    </dependency>

    <!-- Lombok：用注解代替 getter/setter/构造器，减少样板代码 -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>  <!-- optional=true：下游项目不继承此依赖 -->
    </dependency>

    <!-- 测试支持 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

> ⚠️ **踩坑提示：渐进式引入原则**
> 注意此时我们**没有**引入 Swagger、EasyExcel、Guava、Caffeine。这些依赖在用到之前一律不引入。很多新手习惯一口气把所有依赖粘贴进来，结果启动就报一堆 `Bean` 冲突，完全不知道从哪里排查。

---

**② `application.properties` — 数据库连接配置**

```properties
## 指定 MySQL 8.x 驱动类（注意：旧版驱动类是 com.mysql.jdbc.Driver，已废弃）
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

## serverTimezone=UTC 是必须的，否则 MySQL 8 会报时区错误
spring.datasource.url=jdbc:mysql://localhost:3306/all-learning?characterEncoding=utf8&useSSL=false&serverTimezone=UTC

spring.datasource.username=root
spring.datasource.password=root
```

> 💡 **为什么要加 `serverTimezone=UTC`？**
> MySQL 8.0 的 JDBC 驱动强制要求指定时区。不加会在启动时抛出 `The server time zone value is unrecognized` 异常。UTC 是最通用的选择，生产环境可换成 `Asia/Shanghai`。

---

**③ `AllLearningApplication.java` — 启动类**

```java
@SpringBootApplication
// @ServletComponentScan 先不加！它的作用是扫描 @WebFilter 等 Servlet 注解
// 我们在 v3.0 加 TraceId Filter 时再引入，现在加了没有意义
public class AllLearningApplication {
    public static void main(String[] args) {
        SpringApplication.run(AllLearningApplication.class, args);
    }
}
```

> 💡 **`@SpringBootApplication` 做了什么？**
> 它是三个注解的组合：`@SpringBootConfiguration`（声明配置类）+ `@EnableAutoConfiguration`（开启自动装配，这是 Spring Boot "约定优于配置"的核心魔法）+ `@ComponentScan`（扫描当前包及子包下的所有组件）。

---

#### 🧪 运行与验证

**第一步：创建数据库**

```sql
CREATE DATABASE `all-learning` CHARACTER SET utf8mb4;
```

**第二步：启动项目，观察控制台**

启动成功的标志是看到以下日志（端口号默认 8080）：

```
Started AllLearningApplication in 2.3 seconds (JVM running for 3.1)
```

如果看到 `Failed to configure a DataSource` 错误，说明数据库连接配置有问题，检查用户名/密码/数据库名是否正确。

**第三步：访问健康端点确认 Web 容器正常**

```bash
curl http://localhost:8080/
# 预期返回 404（因为还没有写任何接口），但这说明 Tomcat 已经启动成功
# 如果连接被拒绝（Connection refused），说明启动失败
```

---

### 📦 Commit 2 —— 建立领域模型分层

> `feat: 添加用户领域层（DO/DTO/VO）与数据库表结构（Schema）`

---

#### 🎯 本步目标

设计数据库表结构，并建立项目中贯穿全程的三层实体模型：`DO`（数据库映射）、`DTO`（业务传输）、`VO`（视图展示），从一开始就把"职责分离"的好习惯建立起来。

---

#### 📂 目录变动

```
新增：
├── src/main/resources/db/
│   ├── schema.sql                         ← 建表 SQL
│   └── data.sql                           ← 初始化测试数据
└── src/main/java/.../
    ├── domain/
    │   ├── entity/UserDO.java             ← 数据库映射实体
    │   ├── dto/UserDTO.java               ← 业务数据传输对象
    │   ├── dto/UserQueryDTO.java          ← 查询条件对象
    │   └── vo/UserVO.java                 ← 前端视图对象
```

---

#### 💻 核心代码与详解

**① `schema.sql` — 数据库表设计**

```sql
CREATE TABLE `user` (
  `id`       bigint(20)   NOT NULL AUTO_INCREMENT COMMENT '主键',
  `username` varchar(20)  DEFAULT NULL,
  `password` varchar(20)  DEFAULT NULL,
  `email`    varchar(100) DEFAULT NULL,
  `age`      int(11)      DEFAULT NULL,
  `phone`    varchar(20)  DEFAULT NULL,

  -- 以下是"系统级公共字段"，每张业务表都应该有
  `created`  datetime     DEFAULT NULL COMMENT '创建时间',
  `modified` datetime     DEFAULT NULL COMMENT '最后修改时间',
  `creator`  varchar(100) DEFAULT NULL COMMENT '创建者',
  `operator` varchar(100) DEFAULT NULL COMMENT '最后修改者',
  `status`   tinyint(2)   DEFAULT '1'  COMMENT '1:有效 -1:逻辑删除',
  `version`  bigint(20)   DEFAULT NULL COMMENT '乐观锁版本号',

  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

> 💡 **为什么要有 `version` 和 `status` 字段？**
> - `version`：乐观锁的基础，v5.0 中会用它防止并发更新时数据覆盖。
> - `status`：做"逻辑删除"用的，业务上删除只是把 `status` 设为 `-1`，数据不真的从数据库删除，方便审计和数据恢复。

---

**② `UserDO.java` — 数据库映射实体（这一步先只写字段，注解在 v5.0 再补全）**

```java
@Data           // Lombok：自动生成 getter/setter/toString/equals/hashCode
@TableName("user")  // 告诉 MyBatis-Plus 这个类映射到 "user" 表
public class UserDO implements Serializable {

    // 业务字段
    private String username;
    private String password;
    private String email;
    private Integer age;
    private String phone;

    // 系统字段
    @TableId(type = IdType.ASSIGN_ID) // 使用雪花算法生成全局唯一 ID，不依赖数据库自增
    private Long id;

    private LocalDateTime created;
    private LocalDateTime modified;
    private String creator;
    private String operator;
    private Integer status;
    private Long version;

    // 注意：@TableField(fill=...) 和 @Version 先不加
    // 那是 v5.0 自动填充和乐观锁的功能，现在加了也跑不起来
}
```

> ⚠️ **`IdType.ASSIGN_ID` vs `AUTO` 该选哪个？**
> - `AUTO`：依赖数据库自增，分库分表时 ID 会重复，扩展性差。
> - `ASSIGN_ID`（雪花算法）：趋势递增、全局唯一，是分布式系统的标准做法。本项目从一开始就用它，养成好习惯。

---

**③ 三层实体的"为什么"：一图胜千言**

```
HTTP请求 ──→ [Controller] ──→ [Service] ──→ [Mapper] ──→ 数据库
              接收 UserDTO      操作 UserDO     操作 UserDO
              返回 UserVO       ↑ BeanUtils.copyProperties() ↑
```

| 实体类 | 存在的意义 | 如果不用它会有什么问题 |
|--------|-----------|----------------------|
| `UserDO` | 与数据库表结构 1:1 映射 | 直接用它接收请求，数据库字段全暴露给前端（如 `password`、`status`、`version`）|
| `UserDTO` | Service 层的数据契约，携带业务校验注解 | 校验注解与数据库结构耦合，后续表结构变更牵一发动全身 |
| `UserVO` | 控制接口返回的字段（`password` 脱敏、`phone` 打码）| 敏感数据裸奔，引发安全问题 |

---

**④ `UserVO.java` — 只暴露前端需要的字段**

```java
@Data
public class UserVO implements Serializable {

    private String username;
    private String password;  // 在 Controller 中会被设置为 "******"
    private String email;
    private Integer age;
    private String phone;     // 在 Controller 中会做 138****8888 脱敏处理

    // 注意：没有 id、created、modified、version、status 等系统字段
    // 这些字段前端不需要，也不应该知道
}
```

---

**⑤ `UserQueryDTO.java` — 查询条件单独抽一个类**

```java
@Data
public class UserQueryDTO implements Serializable {

    // 目前只支持按用户名查询
    // 把查询条件单独放一个类，是为了以后扩展方便
    // 比如加个按邮箱、按年龄范围查询，只改这一个类，不影响 Controller 签名
    private String username;
}
```

> 💡 **为什么查询条件要单独建一个 DTO，而不是直接用若干个 `@RequestParam`？**
> 当查询条件超过 3 个参数时，方法签名会变得很长，测试和维护都很痛苦。用一个对象封装，配合 `@Valid` 还能加校验注解，是生产项目的标准做法。

---

#### 🧪 运行与验证

这一步没有新接口，验证方式是确认实体类设计是否正确：

**第一步：执行 SQL 文件**

在 MySQL 客户端（如 DBeaver、Navicat）中手动执行 `schema.sql` 建表，再执行 `data.sql` 插入 5 条测试数据。

**第二步：编写一个临时 Mapper 测试（v1.0 C3 会正式添加，这里作为超前预览）**

```java
// 这段代码 C3 之后才能运行，此处仅作理解用
@Test
public void testTableExists() {
    // 验证：数据库中能查到 5 条用户数据
    List<UserDO> users = userMapper.selectList(null);
    Assertions.assertEquals(5, users.size());
}
```

**第三步：重启应用，确认无报错**

MyBatis-Plus 在启动时会扫描 `@TableName` 注解并注册映射关系。如果实体类字段名与数据库列名不匹配（例如 Java 中用驼峰 `userName`，数据库用下划线 `user_name`），MyBatis-Plus 默认开启了驼峰转下划线，无需额外配置，但要保证命名规范统一。

---

> ✅ **v1.0 前两步小结**
>
> 完成 C1、C2 后，你建立了：
> 1. 一个能启动、能连库的 Spring Boot 骨架
> 2. 一套 DO → DTO → VO 的实体分层体系
>
> 这两步没有写任何业务逻辑，但这是最容易被新手跳过、事后最后悔没做好的部分。接下来的 C3（接入 Mapper）和 C4（完成 CRUD 接口）将在此基础上快速完成。


---

### 📦 Commit 3 —— 接入 MyBatis-Plus，打通数据库操作层

> `feat: 添加 UserMapper 与 MyBatis-Plus 相关配置`

---

#### 🎯 本步目标

创建 `UserMapper` 接口，让它继承 MyBatis-Plus 的 `BaseMapper`，从而"零 SQL"获得完整的单表 CRUD 能力；同时配置分页插件，为下一步的分页查询做好准备。

---

#### 📂 目录变动

```
新增：
└── src/main/java/.../
    ├── mapper/
    │   └── UserMapper.java          ← 继承 BaseMapper，获得基础 CRUD 方法
    └── config/
        └── MybatisPlusConfig.java   ← 注册分页插件
```

---

#### 💻 核心代码与详解

**① `UserMapper.java` — 只需一行继承，获得十余个内置方法**

```java
@Mapper  // 告诉 Spring：这是一个 MyBatis 的 Mapper 接口，需要生成代理对象注入容器
public interface UserMapper extends BaseMapper<UserDO> {

    // 目前什么都不用写！
    // BaseMapper<UserDO> 已经为我们提供了：
    //   insert(UserDO)                  → 插入一条记录
    //   deleteById(Serializable)        → 按主键删除
    //   updateById(UserDO)              → 按主键更新（只更新非null字段）
    //   selectById(Serializable)        → 按主键查询
    //   selectList(QueryWrapper)        → 条件查询列表
    //   selectPage(Page, QueryWrapper)  → 分页查询（需要分页插件）
    //   ... 共 17 个内置方法

    // 如果将来需要写自定义 SQL（如多表联查），在这里加方法，
    // 同时在 resources/mapper/UserMapper.xml 里写对应的 SQL
}
```

> 💡 **`@Mapper` vs 在启动类上加 `@MapperScan` 哪个更好？**
>
> | 方式 | 优点 | 缺点 |
> |------|------|------|
> | `@Mapper`（每个接口单独标注）| 意图明确，IDE 跳转友好 | 每个 Mapper 都要写 |
> | `@MapperScan`（启动类统一扫描）| 写一次，全局生效 | 包路径写错会导致所有 Mapper 失效，排查困难 |
>
> **本项目选择 `@Mapper`**，单个文件职责清晰，更适合学习阶段理解原理。生产项目中两种方式都很常见，团队统一即可。

---

**② `MybatisPlusConfig.java` — 注册分页插件**

```java
@Configuration  // 声明这是一个配置类，等价于 Spring 的 XML 配置文件
public class MybatisPlusConfig {

    /**
     * 分页插件：不注册它，调用 selectPage() 时会直接忽略分页参数，
     * 返回全表数据！这是新手最常踩的坑之一。
     */
    @Bean
    public PaginationInterceptor paginationInterceptor() {
        PaginationInterceptor interceptor = new PaginationInterceptor();

        // 明确指定数据库类型，MyBatis-Plus 会据此生成正确方言的分页 SQL
        // MySQL 的分页是 LIMIT offset, size
        // Oracle 的分页是 ROWNUM，语法完全不同
        interceptor.setDbType(DbType.MYSQL);

        return interceptor;
    }

    // 注意：OptimisticLockerInterceptor（乐观锁插件）先不加
    // 乐观锁需要 UserDO 上的 @Version 注解配合，那是 v5.0 的内容
    // 现在加上去，启动不报错，但会给读者造成"这是干嘛的"的困惑
}
```

> ⚠️ **不注册分页插件的后果演示**
>
> ```java
> // 假设数据库有 100 条数据，你期望查第1页，每页10条
> Page<UserDO> page = new Page<>(1, 10);
> userMapper.selectPage(page, null);
>
> // 没有分页插件时，实际执行的 SQL 是：
> // SELECT * FROM user          ← 全表扫描！返回 100 条！
>
> // 注册插件后，实际执行的 SQL 是：
> // SELECT * FROM user LIMIT 0, 10  ← 正确！
> ```

---

**③ 验证 Mapper 的快速单元测试**

```java
@SpringBootTest
@Slf4j
public class UserMapperTest {

    @Autowired
    private UserMapper userMapper;

    @Test
    public void testSelectByMap() {
        // selectByMap：传入 Map 作为 WHERE 条件，适合简单等值查询
        HashMap<String, Object> param = new HashMap<>();
        param.put("username", "username1");  // key 必须是数据库列名，不是 Java 字段名

        List<UserDO> result = userMapper.selectByMap(param);

        // 断言：应该查到 1 条数据（data.sql 里有）
        Assertions.assertEquals(1, result.size());
        log.info("查询结果: {}", result);
    }
}
```

---

#### 🧪 运行与验证

**运行单元测试，观察控制台 SQL 日志**

为了能在控制台看到 MyBatis-Plus 实际执行的 SQL（方便调试），在 `application.properties` 中临时加一行：

```properties
# 开启 SQL 日志，开发环境调试用，生产环境务必关闭（性能损耗 + 可能打印敏感数据）
mybatis-plus.configuration.log-impl=org.apache.ibatis.logging.stdout.StdOutImpl
```

运行 `UserMapperTest#testSelectByMap`，控制台应出现：

```
==>  Preparing: SELECT id,username,password,... FROM user WHERE username = ?
==> Parameters: username1(String)
<==      Total: 1
```

看到这个输出，说明 Mapper 层已经完全打通。

---

### 📦 Commit 4 —— 完成完整 CRUD 业务接口

> `feat: 添加 UserService 并暴露增删改查（CRUD）REST API`

---

#### 🎯 本步目标

实现 `UserService` 业务层和 `UserController` 接入层，将分层实体串联起来，对外暴露标准的 REST 接口（POST 新增、PUT 更新、DELETE 删除、GET 分页查询），完成 v1.0 的核心目标。

---

#### 📂 目录变动

```
新增：
└── src/main/java/.../
    ├── domain/common/
    │   ├── PageQuery.java           ← 通用分页查询入参封装
    │   ├── PageResult.java          ← 通用分页查询结果封装
    │   └── ResponseResult.java      ← 统一 HTTP 响应体
    ├── service/
    │   └── UserService.java         ← 业务接口定义
    ├── service/impl/
    │   └── UserServiceImpl.java     ← 业务逻辑实现
    └── controller/
        └── UserController.java      ← REST 接口入口
```

---

#### 💻 核心代码与详解

**① `ResponseResult.java` — 统一响应体，先搭架子**

```java
@Data
public class ResponseResult<T> implements Serializable {

    private Boolean success;   // true/false
    private String  code;      // 业务错误码，成功时为 null
    private String  message;   // 错误描述，成功时为 null
    private T       result;    // 真正的业务数据，用泛型承载

    // 静态工厂方法：成功时调用
    public static <T> ResponseResult<T> success(T result) {
        ResponseResult<T> r = new ResponseResult<>();
        r.setSuccess(Boolean.TRUE);
        r.setResult(result);
        return r;
    }

    // 静态工厂方法：失败时调用（错误码和消息先用字符串，v2.0 改为枚举）
    public static <T> ResponseResult<T> failure(String code, String message) {
        ResponseResult<T> r = new ResponseResult<>();
        r.setSuccess(Boolean.FALSE);
        r.setCode(code);
        r.setMessage(message);
        return r;
    }
}
```

> 💡 **为什么要有统一响应体？不直接返回业务对象行吗？**
>
> 直接返回业务对象时：
> ```json
> // 成功：{ "id": 1, "username": "tom" }
> // 失败：{ "timestamp": "...", "status": 500, "error": "Internal Server Error" }  ← Spring 默认格式
> ```
> 前端需要判断 HTTP Status Code，且失败格式千奇百怪。
>
> 统一响应体后：
> ```json
> // 成功：{ "success": true, "result": { "id": 1, "username": "tom" } }
> // 失败：{ "success": false, "code": "3002", "message": "新增失败" }
> ```
> 前端只需判断 `success` 字段，格式永远一致，联调效率提升数倍。

---

**② `PageQuery.java` & `PageResult.java` — 分页的入参和出参**

```java
// 分页查询入参：用泛型 T 承载动态的查询条件
@Data
public class PageQuery<T> implements Serializable {

    private Integer pageNo   = 1;   // 当前页，默认第1页
    private Integer pageSize = 20;  // 每页条数，默认20条

    // 动态查询条件：这里的 T 将来会是 UserQueryDTO、OrderQueryDTO 等
    // 这样这个类就是真正"通用"的，不和任何业务绑定
    private T query;
}
```

```java
// 分页查询出参：同样用泛型 T 承载动态的数据列表
@Data
public class PageResult<T> implements Serializable {

    private Integer pageNo;    // 当前页号
    private Integer pageSize;  // 每页行数
    private Long    total;     // 总记录数（前端用来渲染"共 XX 条"）
    private Long    pageNum;   // 总页数（前端用来渲染分页器）
    private T       data;      // 实际数据列表
}
```

> 💡 **`PageQuery<T>` 和 `PageResult<T>` 为什么都要用泛型？**
>
> 假设你的系统有 10 张业务表都需要分页查询，如果不用泛型，你需要写：
> `UserPageQuery`、`OrderPageQuery`、`ProductPageQuery`... 10 个类，代码膨胀且逻辑重复。
> 用泛型后，一套分页结构全系统复用，这是框架设计的基本思维。

---

**③ `UserServiceImpl.java` — 核心业务逻辑**

```java
@Service
public class UserServiceImpl implements UserService {

    @Autowired
    private UserMapper userMapper;

    @Override
    public int save(UserDTO userDTO) {
        UserDO userDO = new UserDO();
        // BeanUtils.copyProperties：按字段名匹配做浅拷贝
        // 注意参数顺序：copyProperties(源对象, 目标对象)，很多人记反
        BeanUtils.copyProperties(userDTO, userDO);
        return userMapper.insert(userDO);
    }

    @Override
    public int update(Long id, UserDTO userDTO) {
        UserDO userDO = new UserDO();
        BeanUtils.copyProperties(userDTO, userDO);
        userDO.setId(id);  // 必须设置 id，否则 updateById 不知道更新哪条

        // updateById 只更新 userDO 中非 null 的字段
        // 例如：userDTO 只传了 password，那就只更新 password 列，其他列不动
        return userMapper.updateById(userDO);
    }

    @Override
    public int delete(Long id) {
        // 注意：这里是物理删除！v5.0 加上 status 字段后改为逻辑删除
        return userMapper.deleteById(id);
    }

    @Override
    public PageResult<List<UserDTO>> query(PageQuery<UserQueryDTO> pageQuery) {

        // step1: 构造 MyBatis-Plus 的分页对象
        Page<UserDO> page = new Page<>(
            pageQuery.getPageNo(),
            pageQuery.getPageSize()
        );

        // step2: 构造查询条件
        // QueryWrapper 用法：new QueryWrapper<>(entity) 会把 entity 中非 null 的字段
        // 自动转成 WHERE 条件（等值匹配），简单查询不用手写 SQL
        UserDO queryDO = new UserDO();
        BeanUtils.copyProperties(pageQuery.getQuery(), queryDO);
        QueryWrapper<UserDO> queryWrapper = new QueryWrapper<>(queryDO);

        // step3: 执行分页查询
        IPage<UserDO> pageResult = userMapper.selectPage(page, queryWrapper);

        // step4: 封装返回结果，DO 转 DTO
        PageResult<List<UserDTO>> result = new PageResult<>();
        result.setPageNo((int) pageResult.getCurrent());
        result.setPageSize((int) pageResult.getSize());
        result.setTotal(pageResult.getTotal());
        result.setPageNum(pageResult.getPages());

        // 用 Stream 流将 List<UserDO> 转换为 List<UserDTO>
        List<UserDTO> userDTOList = Optional
                .ofNullable(pageResult.getRecords())
                .map(List::stream)
                .orElseGet(Stream::empty)  // 防止 records 为 null 时 NPE
                .map(userDO -> {
                    UserDTO userDTO = new UserDTO();
                    BeanUtils.copyProperties(userDO, userDTO);
                    return userDTO;
                })
                .collect(Collectors.toList());

        result.setData(userDTOList);
        return result;
    }
}
```

> ⚠️ **`BeanUtils.copyProperties` 的三大陷阱，现在就要知道：**
>
> | 陷阱 | 说明 | 解决方案 |
> |------|------|---------|
> | 参数顺序 | Spring 的是 `(src, dest)`，Apache Commons 的是 `(dest, src)`，完全相反！ | 统一用 Spring 的，记住"源在前目标在后" |
> | 浅拷贝 | 引用类型字段（如 `List`、自定义对象）只复制引用，修改会相互影响 | 引用类型字段手动处理，或使用 MapStruct |
> | 字段名必须完全一致 | `userName` 和 `username` 不会匹配 | 不一致的字段在拷贝后手动 `set` |

---

**④ `UserController.java` — REST 接口，此阶段的简化版**

```java
@RestController          // = @Controller + @ResponseBody，方法返回值自动序列化为 JSON
@RequestMapping("/api/users")
@Slf4j
public class UserController {

    @Autowired
    private UserService userService;

    /** 新增用户：POST /api/users */
    @PostMapping
    public ResponseResult save(@RequestBody UserDTO userDTO) {
        // @RequestBody：从请求体中解析 JSON -> UserDTO 对象
        // 注意：这一步先不加 @Validated 校验，v2.0 专门讲这个
        int rows = userService.save(userDTO);
        return rows == 1
                ? ResponseResult.success("新增成功！")
                : ResponseResult.failure("3002", "新增失败");
    }

    /** 更新用户：PUT /api/users/{id} */
    @PutMapping("/{id}")
    public ResponseResult update(
            @PathVariable("id") Long id,      // 从 URL 路径中取 id
            @RequestBody UserDTO userDTO) {
        int rows = userService.update(id, userDTO);
        return rows == 1
                ? ResponseResult.success("更新成功！")
                : ResponseResult.failure("3003", "更新失败");
    }

    /** 删除用户：DELETE /api/users/{id} */
    @DeleteMapping("/{id}")
    public ResponseResult delete(@PathVariable("id") Long id) {
        int rows = userService.delete(id);
        return rows == 1
                ? ResponseResult.success("删除成功！")
                : ResponseResult.failure("3004", "删除失败");
    }

    /** 分页查询：GET /api/users?pageNo=1&pageSize=10&username=xxx */
    @GetMapping
    public ResponseResult<PageResult> query(
            @RequestParam Integer pageNo,      // @RequestParam：从 URL 查询参数中取值
            @RequestParam Integer pageSize,
            UserQueryDTO query) {              // 复杂对象不加注解，Spring 自动从参数中绑定

        log.info("收到查询请求，pageNo={}, pageSize={}, query={}", pageNo, pageSize, query);

        PageQuery<UserQueryDTO> pageQuery = new PageQuery<>();
        pageQuery.setPageNo(pageNo);
        pageQuery.setPageSize(pageSize);
        pageQuery.setQuery(query);

        PageResult<List<UserDTO>> pageResult = userService.query(pageQuery);

        // 将 UserDTO 列表转换为 UserVO 列表（脱敏处理）
        List<UserVO> voList = Optional
                .ofNullable(pageResult.getData())
                .map(List::stream)
                .orElseGet(Stream::empty)
                .map(userDTO -> {
                    UserVO vo = new UserVO();
                    BeanUtils.copyProperties(userDTO, vo);

                    // 密码脱敏：不管数据库存什么，对外永远显示 ******
                    vo.setPassword("******");

                    // 手机号脱敏：138****8888
                    if (!StringUtils.isEmpty(userDTO.getPhone())) {
                        vo.setPhone(userDTO.getPhone()
                            .replaceAll("(\\d{3})\\d{4}(\\d{4})", "$1****$2"));
                    }
                    return vo;
                })
                .collect(Collectors.toList());

        // 复用 PageResult 结构，只替换 data 字段的类型
        PageResult<List<UserVO>> voPageResult = new PageResult<>();
        BeanUtils.copyProperties(pageResult, voPageResult);
        voPageResult.setData(voList);

        return ResponseResult.success(voPageResult);
    }
}
```

> 💡 **REST 动词选择的最佳实践**
>
> | 操作 | HTTP 方法 | URL 设计 | ❌ 反面例子 |
> |------|----------|---------|-----------|
> | 新增 | `POST` | `/api/users` | `/api/addUser` |
> | 更新全部字段 | `PUT` | `/api/users/{id}` | `/api/updateUser?id=1` |
> | 更新部分字段 | `PATCH` | `/api/users/{id}` | （本项目简化，统一用 PUT）|
> | 删除 | `DELETE` | `/api/users/{id}` | `/api/deleteUser/{id}` |
> | 查询列表 | `GET` | `/api/users` | `/api/getUsers` |
>
> **核心原则**：URL 表示"资源"（名词），HTTP 方法表示"动作"（动词），二者不要混用。

---

#### 🧪 运行与验证

启动项目，使用以下 `curl` 命令或导入 Postman 逐一验证：

**① 新增用户**

```bash
curl -X POST http://localhost:8080/api/users \
  -H "Content-Type: application/json" \
  -d '{
    "username": "testuser",
    "password": "123456",
    "email": "test@test.com",
    "age": 25,
    "phone": "13800138000"
  }'
```

期望响应：
```json
{ "success": true, "result": "新增成功！" }
```

**② 分页查询**

```bash
curl "http://localhost:8080/api/users?pageNo=1&pageSize=10&username=username1"
```

期望响应（注意 `password` 脱敏、`phone` 打码）：
```json
{
  "success": true,
  "result": {
    "pageNo": 1, "pageSize": 10, "total": 1, "pageNum": 1,
    "data": [{
      "username": "username1",
      "password": "******",
      "phone": "158****1111",
      "email": "email1@email.com",
      "age": 18
    }]
  }
}
```

**③ 更新用户**

先从数据库取一个真实存在的 `id`（如 `1220708537638920191`），然后：

```bash
curl -X PUT http://localhost:8080/api/users/1220708537638920191 \
  -H "Content-Type: application/json" \
  -d '{ "password": "newpassword" }'
```

期望响应：
```json
{ "success": true, "result": "更新成功！" }
```

**④ 删除用户**

```bash
curl -X DELETE http://localhost:8080/api/users/1220708537638920191
```

期望响应：
```json
{ "success": true, "result": "删除成功！" }
```

---

###  ✅ **v1.0 阶段完整回顾**
>
> 经过 4 次提交，你从零完成了：
>
> ```
> C1: 工程骨架 + 依赖管理
>   ↓
> C2: 数据库表 + DO/DTO/VO 三层实体
>   ↓
> C3: Mapper 层 + 分页插件（数据读写能力）
>   ↓
> C4: Service 业务逻辑 + Controller REST 接口（对外服务能力）
> ```
>
> 此时你的项目已经是一个**麻雀虽小五脏俱全**的 CRUD 服务。但你一定发现了几个问题：
> - 传一个 `age=999` 的用户进来，也能存进数据库 → **这是 v2.0 要解决的：参数校验**
> - 接口抛出异常时，返回的不是 `ResponseResult` 格式 → **这是 v2.0 要解决的：全局异常处理**
> - 每次查询都要打数据库，性能差 → **这是 v3.0 要解决的：缓存**
>
> 每一个"发现的问题"，都是下一个迭代版本存在的理由。这就是真实项目演进的方式。

---

### 📌 **下期预告 —— v2.0：输入校验 & 统一规范（C5 ~ C7）**
>
> 下一阶段将实现：
> - 用 `@NotBlank`、`@Email`、`@Max` 等注解自动拦截非法参数
> - 新增/更新走不同校验规则（分组校验）
> - 用 `@ControllerAdvice` 兜底所有异常，保证响应格式永远是 `ResponseResult`
> - Service 层也能校验参数（`ValidatorUtils` 手动触发）