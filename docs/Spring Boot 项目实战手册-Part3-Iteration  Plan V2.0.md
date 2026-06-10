## 第三部分：实战工作手册 v2.0 阶段：输入校验 & 统一规范

> 覆盖 **v2.0 阶段的 C5、C6、C7** 三次提交

---

### 背景：为什么需要这个阶段？

在 v1.0 结束时，你的接口毫无防护，任何数据都能进来：

```bash
# 这些请求目前都能成功写入数据库！
curl -X POST http://localhost:8080/api/users \
  -d '{ "age": 999, "email": "not-an-email", "password": "ab" }'

# 接口抛出异常时，响应是这样的（Spring 默认错误格式）：
# { "timestamp": "2024-01-01T00:00:00.000+0000", "status": 500,
#   "error": "Internal Server Error", "message": "", "path": "/api/users" }
# 和 ResponseResult 格式完全不一致，前端无法统一处理！
```

v2.0 要做的就是在系统入口竖起两道防线：**参数校验拦截非法输入**，**全局异常兜底统一格式输出**。

---

### 📦 Commit 5 —— 建立参数校验体系

> `feat: 添加校验组（Validation Groups）并对 UserDTO 进行注解标注`

---

#### 🎯 本步目标

引入 Bean Validation 注解体系，为 `UserDTO` 的每个字段加上业务规则约束；同时用"分组校验"解决新增和更新场景下校验规则不同的问题；提供 `ValidatorUtils` 工具类供 Service 层手动触发校验。

---

#### 📂 目录变动

```
新增：
└── src/main/java/.../
    └── util/
        ├── InsertValidationGroup.java   ← 新增操作的校验分组标记接口
        ├── UpdateValidationGroup.java   ← 更新操作的校验分组标记接口
        └── ValidatorUtils.java          ← Service 层手动触发校验的工具类

修改：
└── src/main/java/.../
    └── domain/dto/UserDTO.java          ← 为字段添加校验注解
```

---

#### 💻 核心代码与详解

**① 先理解"为什么需要分组校验"**

新增和更新对同一个字段的要求是不同的：

```
新增用户时：username 必填（没有名字怎么注册？）
更新用户时：username 可以不传（我只想改密码，为什么要强制传用户名？）

新增用户时：version 不需要传（数据库还没有这条记录，哪来的版本号？）
更新用户时：version 必填（乐观锁需要它来判断数据是否被别人改过）
```

如果只有一套校验规则，要么新增时校验太松，要么更新时校验太严，两头为难。分组校验完美解决这个问题。

---

**② `InsertValidationGroup.java` & `UpdateValidationGroup.java` — 分组标记接口**

```java
/**
 * 新增操作的校验分组
 * 这是一个"标记接口"（Marker Interface）：接口体为空，
 * 它的唯一作用是作为一个"类型标签"，让校验框架识别当前是哪种场景
 * 类似 Java 中的 Serializable、Cloneable，本身没有方法，存在即意义
 */
public interface InsertValidationGroup {
}
```

```java
/**
 * 更新操作的校验分组
 * 同上，命名清晰即可，内容为空
 */
public interface UpdateValidationGroup {
}
```

> 💡 **标记接口的本质**
> Bean Validation 用接口的 `Class` 对象作为分组的唯一标识符。你完全可以用任意已有接口（比如 `java.io.Serializable`）作为分组，但那样语义不清。专门定义两个空接口，命名直接表达意图，是最佳实践。

---

**③ `UserDTO.java` — 为字段添加校验注解（核心）**

```java
@Data
public class UserDTO implements Serializable {

    /**
     * groups 属性：指定这条规则在哪个分组下生效
     * InsertValidationGroup.class → 只在新增时校验"不能为空"
     * 更新时不传 username 完全合法
     */
    @NotBlank(
        message = "用户名不能为空！",
        groups = {InsertValidationGroup.class}
    )
    private String username;

    /**
     * 密码：新增时必填，且长度限制在新增和更新场景都要校验
     * 注意 @Length 没有指定 groups，意味着"默认分组（Default）"生效
     * 但因为我们触发校验时指定了分组，Default 组不会自动触发！
     * 所以这里要同时在两个分组都声明长度限制
     */
    @NotBlank(
        message = "密码不能为空！",
        groups = {InsertValidationGroup.class}
    )
    @Length(
        min = 6, max = 18,
        message = "密码长度不能少于6位，不能多于18位！",
        groups = {InsertValidationGroup.class, UpdateValidationGroup.class}
    )
    private String password;

    /**
     * @NotEmpty vs @NotBlank vs @NotNull 的区别，一次说清：
     *
     * @NotNull  : 不能是 null，但可以是 ""（空字符串）或 "  "（纯空格）
     * @NotEmpty : 不能是 null，也不能是 ""，但可以是 "  "（纯空格）
     * @NotBlank : 不能是 null，不能是 ""，也不能是 "  " ← 最严格，字符串字段推荐用这个
     *
     * 这里用 @NotEmpty 是源项目的写法，实际 @NotBlank 更合适
     */
    @NotEmpty(
        message = "邮箱不能为空！",
        groups = {InsertValidationGroup.class}
    )
    @Email(message = "必须为有效邮箱格式！")  // 没有 groups → 触发任何分组都会校验格式
    private String email;

    @NotNull(
        message = "年龄不能为空！",
        groups = {InsertValidationGroup.class}
    )
    @Max(value = 60, message = "年龄不能大于60岁！")
    @Min(value = 18, message = "年龄不能小于18岁！")
    private Integer age;

    @NotBlank(
        message = "手机号不能为空！",
        groups = {InsertValidationGroup.class}
    )
    private String phone;

    /**
     * version：只在更新时必填
     * 新增时不传（数据库还没这条记录，没有版本号）
     * 更新时必传（乐观锁使用，v5.0 深入讲解）
     */
    @NotNull(
        message = "版本号不能为空！",
        groups = {UpdateValidationGroup.class}
    )
    private Long version;

    private LocalDateTime created;
}
```

> ⚠️ **最容易犯的错：`@Email` 没有指定 `groups` 会怎样？**
>
> ```java
> // 触发 InsertValidationGroup 分组时：
> // @NotEmpty(groups={InsertValidationGroup.class}) → 生效 ✅
> // @Email                                          → 不生效 ❌（属于 Default 分组）
> 
> // 正确写法：如果希望格式校验在新增时也生效，必须加上 groups
> @Email(
>  message = "必须为有效邮箱格式！",
>  groups = {InsertValidationGroup.class, UpdateValidationGroup.class}
> )
> ```
>
> 这是分组校验最隐蔽的坑。**一旦你在某个注解上指定了 `groups`，没有指定 `groups` 的注解就只属于 `Default` 组，触发分组校验时不会执行。**

---

**④ `ValidatorUtils.java` — Service 层的手动校验工具**

```java
public class ValidatorUtils {

    /**
     * 用 JDK 的 Validation 工厂创建一个全局校验器
     * 设计为 static final：单例模式，整个应用生命周期只创建一次，避免重复开销
     */
    private static final Validator validator =
            Validation.buildDefaultValidatorFactory().getValidator();

    /**
     * 手动触发参数校验
     *
     * @param object  待校验的对象
     * @param groups  要触发的校验分组（不传则触发 Default 分组）
     */
    public static <T> void validate(T object, Class<?>... groups) {

        // validate() 返回所有校验失败的条目集合
        // 如果全部通过，返回空集合
        Set<ConstraintViolation<T>> violations =
                validator.validate(object, groups);

        if (!CollectionUtils.isEmpty(violations)) {
            // 把所有错误信息拼接成一条字符串抛出
            // 生产建议：返回 List<String> 或 Map，方便前端按字段显示错误
            StringBuilder msg = new StringBuilder();
            violations.forEach(v -> msg.append(v.getMessage()).append("; "));
            throw new RuntimeException(msg.toString());
        }
    }
}
```

> 💡 **为什么 Service 层还需要手动校验？Controller 的 `@Validated` 不够吗？**
>
> Controller 层校验依赖 Spring MVC 拦截，但 Service 方法可能被以下方式调用：
>
> - 定时任务（`@Scheduled`）直接调用 Service，绕过 Controller
> - 单元测试直接 `new UserServiceImpl()` 调用
> - 消息队列的消费者调用 Service
>
> 在这些场景下，Controller 的校验完全失效。**Service 层的手动校验是最后一道保障**，两层校验互为补充，并不重复。

---

#### 🧪 运行与验证

这一步的校验注解还没有被触发（需要 C7 在 Controller 加 `@Validated`），但可以通过单元测试验证 `ValidatorUtils`：

```java
@Test
public void testValidatorUtils() {
    UserDTO userDTO = new UserDTO();
    userDTO.setAge(999);       // 超过 @Max(60) 限制
    userDTO.setEmail("bad-email"); // 不符合 @Email 格式

    // 触发 InsertValidationGroup 分组校验，预期抛出 RuntimeException
    Exception exception = Assertions.assertThrows(
        RuntimeException.class,
        () -> ValidatorUtils.validate(userDTO, InsertValidationGroup.class)
    );

    // 验证错误信息中包含关键字
    System.out.println("校验错误信息: " + exception.getMessage());
    // 输出类似：用户名不能为空！; 密码不能为空！; 年龄不能大于60岁！; ...
}
```

---

### 📦 Commit 6 —— 构建异常处理体系

> `feat: 添加全局异常处理器（Global Exception Handler）与错误码枚举（Error Code Enum）`

---

#### 🎯 本步目标

建立标准化的异常体系：用枚举统一管理所有错误码，定义业务异常类承载结构化错误信息，用 `@ControllerAdvice` 拦截全局异常并统一转换为 `ResponseResult` 格式响应。

---

#### 📂 目录变动

```
新增：
└── src/main/java/.../
    └── exception/
        ├── ErrorCodeEnum.java           ← 错误码枚举，集中管理所有错误类型
        ├── BusinessException.java       ← 业务异常类，携带结构化错误信息
        └── GlobalExceptionHandler.java  ← 全局异常拦截器

修改：
└── src/main/java/.../
    └── domain/common/ResponseResult.java  ← 新增 failure(ErrorCodeEnum) 重载方法
```

---

#### 💻 核心代码与详解

**① `ErrorCodeEnum.java` — 错误码集中管理**

```java
@AllArgsConstructor  // Lombok：生成包含所有字段的构造器
@Getter              // Lombok：生成所有字段的 getter
public enum ErrorCodeEnum {

    // 用注释划分错误码段，一目了然，方便排查问题
    // 约定：0*** 成功，1*** 参数异常，2*** 系统异常，3*** 业务异常
    SUCCESS("0000", "操作成功"),

    PARAM_ERROR("1001", "参数异常"),
    PARAM_NULL("1002", "参数为空"),
    PARAM_FORMAT_ERROR("1003", "参数格式不正确"),
    PARAM_VALUE_ERROR("1004", "参数值不正确"),

    SYSTEM_ERROR("2001", "服务异常"),
    UNKNOWN_ERROR("2002", "未知异常"),

    INSERT_FAILURE("3002", "新增失败"),
    UPDATE_FAILURE("3003", "更新失败"),
    DELETE_FAILURE("3004", "删除失败"),
    RATE_LIMIT_ERROR("3005", "限流异常"),
    FILE_UPLOAD_FAILURE("3006", "文件上传失败");

    private final String code;
    private final String message;
}
```

> 💡 **为什么要用枚举管理错误码，而不是直接写字符串？**
>
> 直接写字符串的问题：
>
> ```java
> // 散落在各处，无法统一查阅
> return ResponseResult.failure("3002", "新增失败");  // Controller A
> return ResponseResult.failure("3002", "插入失败");  // Controller B ← 同一错误，消息不一致！
> return ResponseResult.failure("3003", "新增失败");  // Controller C ← 错误码用错了！
> ```
>
> 用枚举后：
>
> ```java
> return ResponseResult.failure(ErrorCodeEnum.INSERT_FAILURE); // 唯一来源，不可能出错
> ```
>
> 枚举本质上是"有限集合"，强制所有错误码在同一个地方声明，**编译期就能发现错误**，这是比文档更可靠的约束。

---

**② `BusinessException.java` — 业务异常类**

```java
public class BusinessException extends RuntimeException {

    /**
     * 为什么继承 RuntimeException 而不是 Exception？
     *
     * Exception（受检异常）：调用方必须 try-catch 或 throws 声明，
     *   导致业务代码里充斥着 try-catch，严重影响可读性
     *
     * RuntimeException（非受检异常）：调用方无需强制处理，
     *   由最外层的 GlobalExceptionHandler 统一兜底，业务代码保持干净
     *
     * Spring 的 @Transactional 也默认只回滚 RuntimeException，
     * 所以业务异常继承 RuntimeException 是 Spring 生态的标准做法
     */

    @Getter
    private final String code;  // 从 ErrorCodeEnum 中来的错误码

    /** 最常用：直接传枚举，code 和 message 一起带过来 */
    public BusinessException(ErrorCodeEnum error) {
        super(error.getMessage());
        this.code = error.getCode();
    }

    /** 需要自定义消息时：用枚举的 code，但消息可以更具体 */
    public BusinessException(ErrorCodeEnum error, String message) {
        super(message);
        this.code = error.getCode();
    }

    /** 包装底层异常时：保留原始异常的堆栈，方便排查根因 */
    public BusinessException(ErrorCodeEnum error, Throwable cause) {
        super(cause);
        this.code = error.getCode();
    }
}
```

> 💡 **异常链（Exception Chaining）为什么重要？**
>
> ```java
> // ❌ 错误做法：吞掉原始异常
> try {
>  fileService.upload(inputStream, filename);
> } catch (Exception e) {
>  throw new BusinessException(ErrorCodeEnum.FILE_UPLOAD_FAILURE);
>  // 原始的 IOException 消失了！运维查日志不知道真正原因
> }
> 
> // ✅ 正确做法：用 cause 参数保留异常链
> } catch (Exception e) {
>  throw new BusinessException(ErrorCodeEnum.FILE_UPLOAD_FAILURE, e);
>  // 日志中会打印：BusinessException → caused by IOException → ...
>  // 完整的调用链，一目了然
> }
> ```

---

**③ `GlobalExceptionHandler.java` — 三层防护网**

```java
@ControllerAdvice  // 拦截所有 @Controller 抛出的异常（AOP 实现，不侵入业务代码）
@Slf4j
public class GlobalExceptionHandler {

    /**
     * 第一层：拦截业务异常
     * 优先级最高，精确匹配 BusinessException
     * 这类异常是"预期内的"，日志记录为 error 但不需要告警
     */
    @ResponseBody
    @ExceptionHandler(value = BusinessException.class)
    public ResponseResult handleBusinessException(BusinessException e) {
        log.error("业务异常: code={}, message={}", e.getCode(), e.getMessage());
        // 直接用 BusinessException 里的 code 和 message 构造响应
        return ResponseResult.failure(e.getCode(), e.getMessage());
    }

    /**
     * 第二层：拦截运行时异常
     * 匹配所有 RuntimeException（BusinessException 已被上面拦截，不会到这里）
     * 这类异常是"非预期的"，通常是代码 bug，需要重点关注
     */
    @ResponseBody
    @ExceptionHandler(value = RuntimeException.class)
    public ResponseResult handleRuntimeException(RuntimeException e) {
        log.error("运行时异常: ", e);  // 打印完整堆栈，排查 bug 用
        return ResponseResult.failure(
            ErrorCodeEnum.UNKNOWN_ERROR.getCode(),
            e.getMessage()
        );
    }

    /**
     * 第三层：兜底所有 Throwable
     * 包括 Error（如 OutOfMemoryError）、非 RuntimeException 的 Exception
     * 有了这一层，任何情况下响应格式都是 ResponseResult，前端永远不会拿到 HTML 错误页
     */
    @ResponseBody
    @ExceptionHandler(value = Throwable.class)
    public ResponseResult handleThrowable(Throwable th) {
        log.error("系统异常: ", th);
        return ResponseResult.failure(
            ErrorCodeEnum.SYSTEM_ERROR.getCode(),
            th.getMessage()
        );
    }
}
```

> 💡 **`@ExceptionHandler` 的匹配规则**
>
> Spring 会选择**最精确匹配**的 `@ExceptionHandler`。继承链越靠下（越具体）的异常类，匹配优先级越高：
>
> ```
> Throwable              ← 第三层兜底
> └── Exception
>      └── RuntimeException    ← 第二层
>            └── BusinessException  ← 第一层（优先匹配）
> ```
>
> 所以三个方法的声明顺序不重要，Spring 会自动选择最合适的那个。

---

**④ 更新 `ResponseResult.java` — 新增枚举重载**

```java
// 在 ResponseResult 中新增这个重载方法，让调用更简洁
public static <T> ResponseResult<T> failure(ErrorCodeEnum codeEnum) {
    // 委托给已有的 failure(String, String) 方法，避免重复逻辑
    return failure(codeEnum.getCode(), codeEnum.getMessage());
}
```

同时更新 `UserController` 中的硬编码字符串：

```java
// v1.0 的写法（硬编码，容易出错）：
return ResponseResult.failure("3002", "新增失败");

// v2.0 更新后（枚举，安全且语义清晰）：
return ResponseResult.failure(ErrorCodeEnum.INSERT_FAILURE);
```

---

#### 🧪 运行与验证

**验证全局异常处理效果：手动触发一个异常**

在 `UserController` 的 `save` 方法中临时加一行测试代码：

```java
@PostMapping
public ResponseResult save(@RequestBody UserDTO userDTO) {
    // 临时测试：直接抛出业务异常
    if ("test".equals(userDTO.getUsername())) {
        throw new BusinessException(ErrorCodeEnum.INSERT_FAILURE, "用户名不允许为test");
    }
    // ... 正常逻辑
}
```

发送请求：

```bash
curl -X POST http://localhost:8080/api/users \
  -H "Content-Type: application/json" \
  -d '{ "username": "test", "password": "123456", "email": "a@b.com", "age": 20, "phone": "13800138000" }'
```

期望响应（不再是 Spring 默认的 HTML 错误页）：

```json
{
  "success": false,
  "code": "3002",
  "message": "用户名不允许为test",
  "result": null
}
```

测试完成后删除临时代码。

---

### 📦 Commit 7 —— 在 Controller 层激活声明式校验

> `refactor: 在控制器（Controllers）中应用校验注解`

---

#### 🎯 本步目标

在 Controller 层通过 `@Validated` 注解激活分组校验，让 C5 中定义的校验规则真正在 HTTP 请求入口生效；同时将 `MethodArgumentNotValidException` 等校验异常纳入全局异常处理器，返回友好的错误信息。

---

#### 📂 目录变动

```
修改：
└── src/main/java/.../
    ├── controller/UserController.java       ← 加 @Validated 分组注解
    └── exception/GlobalExceptionHandler.java ← 新增校验异常的处理方法
```

---

#### 💻 核心代码与详解

**① 更新 `UserController.java` — 激活分组校验**

```java
@RestController
@RequestMapping("/api/users")
@Validated  // ① 类级别加这个注解，才能激活方法参数上的校验（单个参数校验必须）
@Slf4j
public class UserController {

    @PostMapping
    public ResponseResult save(
            // ② @Validated(InsertValidationGroup.class)：
            //    触发 UserDTO 中 groups 包含 InsertValidationGroup 的所有校验规则
            //    @RequestBody：从请求体解析 JSON
            @Validated(InsertValidationGroup.class) @RequestBody UserDTO userDTO) {

        int rows = userService.save(userDTO);
        return rows == 1
                ? ResponseResult.success("新增成功！")
                : ResponseResult.failure(ErrorCodeEnum.INSERT_FAILURE);
    }

    @PutMapping("/{id}")
    public ResponseResult update(
            // ③ 路径参数的非空校验：@NotNull 是单个参数校验
            //    类上必须有 @Validated 才能触发（Spring 对单参数校验的特殊要求）
            @NotNull(message = "用户ID不能为空") @PathVariable("id") Long id,

            // ④ 更新时触发 UpdateValidationGroup：
            //    只有 version 是必填的，其他字段都可以不传
            @Validated(UpdateValidationGroup.class) @RequestBody UserDTO userDTO) {

        int rows = userService.update(id, userDTO);
        return rows == 1
                ? ResponseResult.success("更新成功！")
                : ResponseResult.failure(ErrorCodeEnum.UPDATE_FAILURE);
    }

    @DeleteMapping("/{id}")
    public ResponseResult delete(
            @NotNull(message = "用户ID不能为空！") @PathVariable("id") Long id) {
        int rows = userService.delete(id);
        return rows == 1
                ? ResponseResult.success("删除成功！")
                : ResponseResult.failure(ErrorCodeEnum.DELETE_FAILURE);
    }

    // ... query 方法保持不变
}
```

> ⚠️ **`@Validated` 和 `@Valid` 的区别，必须搞清楚：**
>
> | 注解         | 来源                               | 支持分组 | 适用位置                   |
> | ------------ | ---------------------------------- | -------- | -------------------------- |
> | `@Valid`     | JSR-303 标准（`javax.validation`） | ❌ 不支持 | 方法参数、字段（级联校验） |
> | `@Validated` | Spring 扩展                        | ✅ 支持   | 类、方法参数               |
>
> **结论**：
>
> - 需要分组校验 → 用 `@Validated(XxxGroup.class)`
> - 嵌套对象级联校验（对象里面有对象）→ 用 `@Valid`
> - 两者可以混用，如 `PageQuery` 的 `query` 字段上加 `@Valid` 触发嵌套校验

---

**② 更新 `GlobalExceptionHandler.java` — 处理校验异常**

```java
@ControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    /**
     * 新增：处理 @RequestBody 参数校验失败的异常
     * 当 @Validated 校验不通过时，Spring 抛出 MethodArgumentNotValidException
     */
    @ResponseBody
    @ExceptionHandler(value = MethodArgumentNotValidException.class)
    public ResponseResult handleValidationException(
            MethodArgumentNotValidException e) {

        // getBindingResult() 包含所有校验失败的详情
        // getFieldErrors() 获取字段级别的错误列表
        String errorMessage = e.getBindingResult()
                .getFieldErrors()
                .stream()
                // 每个 FieldError 包含：字段名、拒绝值、错误消息
                .map(fieldError ->
                    fieldError.getField() + ": " + fieldError.getDefaultMessage()
                )
                .collect(Collectors.joining("; "));

        log.warn("参数校验失败: {}", errorMessage);

        return ResponseResult.failure(
                ErrorCodeEnum.PARAM_ERROR.getCode(),
                errorMessage
        );
    }

    /**
     * 新增：处理 @RequestParam / @PathVariable 参数校验失败的异常
     * 当类上有 @Validated 且单个参数校验不通过时，抛出 ConstraintViolationException
     */
    @ResponseBody
    @ExceptionHandler(value = ConstraintViolationException.class)
    public ResponseResult handleConstraintViolationException(
            ConstraintViolationException e) {

        String errorMessage = e.getConstraintViolations()
                .stream()
                .map(ConstraintViolation::getMessage)
                .collect(Collectors.joining("; "));

        log.warn("参数约束违反: {}", errorMessage);

        return ResponseResult.failure(
                ErrorCodeEnum.PARAM_ERROR.getCode(),
                errorMessage
        );
    }

    // ... 之前的三个异常处理方法保持不变
}
```

> 💡 **为什么校验失败要用 `log.warn` 而不是 `log.error`？**
>
> 日志级别应该反映问题的严重程度：
>
> - `error`：系统出现了不该出现的错误，需要立即处理（触发告警）
> - `warn`：系统正常运行，但有不符合预期的输入（参数校验失败属于正常业务流程的一部分）
> - `info`：正常的业务流程节点记录
>
> 把参数校验失败记录为 `error` 会导致告警系统被大量正常流量触发，运维人员会产生"狼来了"的疲惫感，真正的错误反而被淹没。

---

#### 🧪 运行与验证

**① 验证新增接口的参数校验**

```bash
# 测试1：缺少必填字段（username 为空）
curl -X POST http://localhost:8080/api/users \
  -H "Content-Type: application/json" \
  -d '{ "password": "123456", "email": "test@test.com", "age": 25, "phone": "13800138000" }'

# 期望响应：
# { "success": false, "code": "1001", "message": "username: 用户名不能为空！" }
```

```bash
# 测试2：字段格式不合法
curl -X POST http://localhost:8080/api/users \
  -H "Content-Type: application/json" \
  -d '{
    "username": "tom",
    "password": "ab",
    "email": "not-an-email",
    "age": 999,
    "phone": "13800138000"
  }'

# 期望响应（多个校验错误同时返回）：
# {
#   "success": false,
#   "code": "1001",
#   "message": "password: 密码长度不能少于6位，不能多于18位！; email: 必须为有效邮箱格式！; age: 年龄不能大于60岁！"
# }
```

**② 验证更新接口的分组校验（只要求 version）**

```bash
# 不传 username（更新时合法），但不传 version（更新时必填）
curl -X PUT http://localhost:8080/api/users/1220708537638920192 \
  -H "Content-Type: application/json" \
  -d '{ "password": "newpassword123" }'

# 期望响应：
# { "success": false, "code": "1001", "message": "version: 版本号不能为空！" }
```

```bash
# 带上 version，更新成功
curl -X PUT http://localhost:8080/api/users/1220708537638920192 \
  -H "Content-Type: application/json" \
  -d '{ "password": "newpassword123", "version": 1 }'

# 期望响应：
# { "success": true, "result": "更新成功！" }
```

---

###  ✅ **v2.0 阶段完整回顾**
>
> 经过 3 次提交，你的系统建立了完整的"输入-处理-输出"防护体系：
>
> ```
> HTTP 请求
>  ↓
> [Controller @Validated] ─── 校验失败 ──→ [GlobalExceptionHandler] ──→ ResponseResult(failure)
>  ↓ 校验通过
> [Service ValidatorUtils] ── 校验失败 ──→ RuntimeException ──→ [GlobalExceptionHandler]
>  ↓ 校验通过
> [Mapper 数据库操作] ──────── 操作失败 ──→ BusinessException ──→ [GlobalExceptionHandler]
>  ↓ 操作成功
> ResponseResult(success)
> ```
>
> 现在你的接口有了双重校验保护，任何异常都会被兜底处理，响应格式永远一致。
>
> 但你可能又发现了新问题：
>
> - 每次查询都打数据库，相同的查询条件反复执行相同的 SQL → **v3.0：Caffeine 缓存**
> - 压测时数据库连接耗尽，系统直接崩溃 → **v3.0：Guava 限流**
> - 线上出了问题，日志里全是不相关的请求交织在一起，无法定位 → **v3.0：TraceId 追踪**

---

###  📌 **下期预告 —— v3.0：缓存 & 限流 & 日志追踪（C8 ~ C10）**
>
> 下一阶段将实现：
>
> - `TraceIdFilter`：每个请求分配唯一 ID，所有日志自动带上这个 ID，串联完整链路
> - `CaffeineCacheConfig`：热点数据缓存，写操作自动驱逐，查询响应从毫秒级降到微秒级
> - `RateLimitInterceptor`：令牌桶算法，QPS 超限时返回限流提示而不是让数据库崩溃