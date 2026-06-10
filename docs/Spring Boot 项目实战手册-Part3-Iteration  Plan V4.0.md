## 第三部分：实战工作手册 v4.0 阶段：文件上传 & 异步导出

> 覆盖 **v4.0 阶段的 C11、C12、C13** 三次提交

---

### 背景：为什么需要这个阶段？

v3.0 结束后，系统已经具备了缓存、限流、追踪三项横切能力。但业务上还缺少两个非常常见的功能场景：

```
场景1：用户要上传头像、上传合同文件
  → 直接把文件存数据库？BLOB 字段会让数据库体积爆炸，查询性能崩溃
  → 需要独立的文件存储服务

场景2：运营要导出 10 万行用户数据的 Excel
  → 同步导出：接口等待 30 秒，前端超时，用户以为失败了，反复点击，服务器压力翻倍
  → 一次性把 10 万行数据加载到内存：JVM 堆撑爆，触发 Full GC，整个服务卡顿

  正确姿势：
  → 接口立即返回"已受理"，后台异步处理
  → 数据分批加载，每批写入 Excel，不积压内存
```

这两个场景考察的是 **I/O 流的正确使用** 和 **异步任务的工程化设计**。

---

### 📦 Commit 11 —— 文件上传服务

> `feat: 添加文件上传服务（Service）与控制器（Controller）`

---

#### 🎯 本步目标

设计 `FileService` 接口屏蔽存储实现细节，实现基于本地磁盘的 `LocalFileServiceImpl`，对外暴露文件上传的 HTTP 接口；同时在 C10 的 `WebConfig` 中确认本地文件访问路径已就绪，实现文件的上传与下载闭环。

---

#### 📂 目录变动

```
新增：
├── uploads/
│   └── .gitkeep                         ← 空目录占位文件，让 Git 能追踪这个目录
└── src/main/java/.../
    ├── service/
    │   └── FileService.java             ← 文件服务接口
    ├── service/impl/
    │   └── LocalFileServiceImpl.java    ← 本地文件存储实现
    └── controller/
        └── FileController.java          ← 文件上传接口

修改：
└── src/main/resources/
    └── application.properties           ← 解除文件大小限制
```

---

#### 💻 核心代码与详解

**① 先理解接口与实现分离的设计价值**

```
今天：文件存本地磁盘（开发环境方便）
     ↓ 业务增长
明天：文件要存阿里云 OSS / 腾讯云 COS / AWS S3（生产环境要求）

如果 Controller 直接依赖 LocalFileServiceImpl：
  FileController → LocalFileServiceImpl  ← 写死了，换存储要改 Controller

正确做法：Controller 依赖接口，实现可以随时替换：
  FileController → FileService（接口）
                       ↑实现
              LocalFileServiceImpl   ← 本地存储
              OssFileServiceImpl     ← 将来替换，Controller 代码零改动
```

这就是经典的**依赖倒置原则（DIP）**，也是 Spring IoC 容器存在的核心意义之一。

---

**② `FileService.java` — 文件服务接口**

```java
public interface FileService {

    /**
     * 通过输入流上传文件
     * 适用场景：从 HTTP 请求中获取文件流，直接转存，不在内存中完整持有文件内容
     *
     * @param inputStream 文件内容的输入流
     * @param filename    目标文件名（含扩展名，如 "avatar.png"）
     */
    void upload(InputStream inputStream, String filename);

    /**
     * 通过 File 对象上传文件
     * 适用场景：服务器本地已有文件需要转存（如临时文件、生成的 Excel）
     *
     * @param file 本地文件对象
     */
    void upload(File file);
}
```

> 💡 **为什么提供两个重载而不是只有一个？**
>
> 两个上传入口对应两种不同的来源：
> - `upload(InputStream, String)`：文件来自**网络请求**（HTTP 上传），用流处理避免把整个文件加载到内存
> - `upload(File)`：文件来自**本地磁盘**（如 Excel 导出后生成的临时文件），直接传 File 对象更自然
>
> 在 `LocalFileServiceImpl` 中，`upload(File)` 会委托给 `upload(InputStream, String)` 来复用核心逻辑，遵循 DRY（Don't Repeat Yourself）原则。

---

**③ `LocalFileServiceImpl.java` — 本地文件存储实现**

```java
@Slf4j
@Service("localFileServiceImpl")  // Bean 名称显式命名
                                   // 将来新增 OssFileServiceImpl 时，两个实现共存
                                   // Controller 通过 @Resource(name="localFileServiceImpl") 指定使用哪个
public class LocalFileServiceImpl implements FileService {

    /**
     * 文件存储的根目录
     * 对应项目根目录下的 uploads/ 文件夹
     * 通过 WebConfig 中的 addResourceHandlers 映射为 /uploads/** 可访问
     */
    private static final String BUCKET = "uploads";

    @Override
    public void upload(InputStream inputStream, String filename) {

        // 拼接完整的存储路径，如 "uploads/avatar.png"
        String storagePath = BUCKET + "/" + filename;

        /**
         * JDK 7+ 的 TWR（Try-With-Resources）语法：
         * 括号内声明的资源会在 try 块结束后自动调用 close()
         * 无论正常结束还是抛出异常，都保证流被关闭，不会造成资源泄漏
         *
         * 注意这里的一个细节：inputStream 来自外部（方法参数），
         * 我们把它赋值给 innerInputStream 再放入 TWR，
         * 是为了确保这个外部传入的流也能被正确关闭
         */
        try (
            InputStream innerInputStream = inputStream;
            FileOutputStream outputStream = new FileOutputStream(new File(storagePath))
        ) {
            // 缓冲区：每次从输入流读取 1KB 数据，写入输出流
            // 为什么是 1KB？在内存占用和 I/O 次数之间取得平衡
            // 太小（如 1 字节）：I/O 系统调用次数过多，性能差
            // 太大（如 100MB）：一次性占用大量堆内存，有 OOM 风险
            byte[] buffer = new byte[1024];
            int len;

            // 循环读写，直到流末尾（read 返回 -1）
            while ((len = innerInputStream.read(buffer)) > 0) {
                // 注意：write(buffer, 0, len) 而不是 write(buffer)
                // 最后一次读取时，buffer 可能只填充了部分数据（len < 1024）
                // 如果写整个 buffer，会在文件末尾追加随机垃圾字节，文件损坏！
                outputStream.write(buffer, 0, len);
            }

            // flush()：强制把 JVM 输出缓冲区的数据刷入操作系统缓冲区
            // 虽然 TWR 关闭流时也会 flush，但显式调用是好习惯
            outputStream.flush();

        } catch (Exception e) {
            log.error("文件写入失败，存储路径: {}", storagePath, e);
            throw new BusinessException(ErrorCodeEnum.FILE_UPLOAD_FAILURE, e);
        }
    }

    @Override
    public void upload(File file) {
        try {
            // 委托给 upload(InputStream, String)，复用核心逻辑
            // 这里 new FileInputStream(file) 如果失败会抛异常，
            // 在 catch 中包装后重新抛出，不会造成流泄漏（因为流还没成功打开）
            upload(new FileInputStream(file), file.getName());
        } catch (Exception e) {
            log.error("文件上传失败，文件名: {}", file.getName(), e);
            throw new BusinessException(ErrorCodeEnum.FILE_UPLOAD_FAILURE, e);
        }
    }
}
```

> ⚠️ **流操作中最常见的三个 bug，现在一次讲清：**
>
> | Bug | 错误代码 | 正确做法 |
> |-----|---------|---------|
> | 流未关闭（资源泄漏）| `try { ... }` 没有 finally 关闭 | 用 TWR `try (stream) { ... }` |
> | 最后一块数据写错 | `outputStream.write(buffer)` | `outputStream.write(buffer, 0, len)` |
> | 目录不存在直接创建文件 | `new FileOutputStream(path)` 报错 | 先检查并创建父目录 |
>
> 关于第三个 bug，生产级代码应该加目录检查：
> ```java
> File targetFile = new File(storagePath);
> // 如果父目录不存在，先创建（mkdirs 可创建多级目录）
> if (!targetFile.getParentFile().exists()) {
>     targetFile.getParentFile().mkdirs();
> }
> ```

---

**④ `FileController.java` — 文件上传接口**

```java
@RestController
@RequestMapping("/api/files")
@Slf4j
public class FileController {

    // 通过 Bean 名称指定注入本地实现
    // 将来换成 OSS 实现时，只改这里的 name 值即可
    @Resource(name = "localFileServiceImpl")
    private FileService fileService;

    /**
     * 文件上传接口
     * POST /api/files/upload
     * Content-Type: multipart/form-data
     *
     * @param file Spring 封装的多部分文件对象，从表单的 file 字段获取
     */
    @PostMapping("/upload")
    public ResponseResult<String> upload(
            @NotNull(message = "上传文件不能为空") MultipartFile file) {

        try {
            // getInputStream()：获取文件内容的流，不把整个文件加载到内存
            // getOriginalFilename()：客户端上传时的原始文件名（含扩展名）
            fileService.upload(
                    file.getInputStream(),
                    file.getOriginalFilename()
            );
        } catch (Exception e) {
            log.error("文件上传接口异常！文件名: {}", file.getOriginalFilename(), e);
            throw new BusinessException(ErrorCodeEnum.FILE_UPLOAD_FAILURE, e);
        }

        // 上传成功，返回可访问的文件名
        // 客户端可以通过 /uploads/{filename} 访问刚上传的文件
        return ResponseResult.success(file.getOriginalFilename());
    }
}
```

> ⚠️ **`getOriginalFilename()` 在生产环境有安全风险！**
>
> 直接使用客户端传来的文件名存在两类风险：
>
> ```java
> // 风险1：路径穿越攻击
> // 客户端上传文件名为 "../../etc/passwd"
> // storagePath = "uploads/../../etc/passwd" ← 覆盖系统文件！
>
> // 风险2：文件名重复覆盖
> // 两个用户都上传 "avatar.png"，后者覆盖前者
>
> // 生产环境正确做法：
> String safeFilename = UUID.randomUUID().toString()
>         + getExtension(file.getOriginalFilename()); // 只保留扩展名
> // 例如："a1b2c3d4-e5f6-7890-abcd-ef1234567890.png"
> ```
>
> 本项目为简化学习，保留原始文件名。实际项目务必做文件名安全处理。

---

**⑤ `application.properties` — 解除文件大小限制**

```properties
## Spring Boot 默认限制单个文件最大 1MB，超过会报 MaxUploadSizeExceededException
## 设为 -1 表示不限制（生产环境建议设一个合理上限，如 100MB）
spring.servlet.multipart.max-file-size=-1
spring.servlet.multipart.max-request-size=-1
```

---

#### 🧪 运行与验证

**确认 uploads 目录存在**（Git 不追踪空目录，`.gitkeep` 文件解决这个问题）：
```bash
ls uploads/
# 应该能看到 .gitkeep 文件
```

**测试文件上传**：
```bash
# 上传一个测试文件
curl -X POST http://localhost:8080/api/files/upload \
  -F "file=@/path/to/your/test.txt"

# 期望响应：
# { "success": true, "result": "test.txt" }
```

**验证文件可访问**（C10 中已配置静态资源映射）：
```bash
curl http://localhost:8080/uploads/test.txt
# 期望返回文件内容
```

---

### 📦 Commit 12 —— 定义异步导出线程池

> `feat: 添加异步线程池配置（Async Thread Pool Config）`

---

#### 🎯 本步目标

在实现异步导出之前，先单独配置一个**专用线程池**。这是工程化的重要实践：不同业务场景的异步任务应该使用隔离的线程池，防止一个业务的任务积压拖垮整个系统。

---

#### 📂 目录变动

```
新增：
└── src/main/java/.../
    └── config/
        └── ExecutorConfig.java    ← 导出服务专用线程池配置

pom.xml ← 无需新增依赖，Spring Boot 已内置线程池支持
```

---

#### 💻 核心代码与详解

**① 先理解为什么不能用默认线程池**

```
Spring @Async 默认使用 SimpleAsyncTaskExecutor：
  - 每次 @Async 方法调用都创建一个新线程
  - 没有线程复用，没有队列，没有上限
  - 100 个并发导出请求 → 创建 100 个线程 → 内存耗尽，系统崩溃

自定义线程池的作用：
  导出请求 → [核心线程处理] → 超过核心数放入[队列等待] → 超过队列容量[拒绝策略]
            ↑                                              ↑
         受控的并发度                              有序降级，不崩溃
```

---

**② `ExecutorConfig.java` — 线程池精细化配置**

```java
@Configuration
@EnableAsync    // 开启 @Async 注解支持
                // 没有这个注解，@Async 标注的方法会同步执行，完全失效
@Slf4j
public class ExecutorConfig {

    /**
     * 导出服务专用线程池
     * Bean 名称 "exportServiceExecutor" 供 @Async("exportServiceExecutor") 引用
     */
    @Bean("exportServiceExecutor")
    public Executor exportServiceExecutor() {

        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();

        /**
         * 核心线程数：Runtime.getRuntime().availableProcessors()
         *
         * 为什么绑定 CPU 核心数？
         * Excel 导出是 CPU 密集型 + I/O 密集型混合任务：
         *   - CPU 密集型（数据转换、序列化）：线程数 = CPU 核心数时 CPU 利用率最高
         *   - I/O 密集型（数据库查询、文件写入）：线程可以更多，因为 I/O 等待时 CPU 空闲
         *
         * 实际场景下的经验公式：
         *   纯 CPU 密集：核心数
         *   纯 I/O 密集：核心数 × 2
         *   混合型（如导出）：核心数 × (1 + I/O等待时间/CPU计算时间)
         *   → 本项目取 核心数 作为保守估计，生产中根据压测数据调整
         */
        executor.setCorePoolSize(
                Runtime.getRuntime().availableProcessors()
        );

        /**
         * 最大线程数：核心数 × 2
         * 当队列满了，且任务还在涌入时，线程数从核心数扩展到最大数
         * 提供额外的突发处理能力，但上限是 2 倍核心数，防止线程过多导致上下文切换开销
         */
        executor.setMaxPoolSize(
                Runtime.getRuntime().availableProcessors() * 2
        );

        /**
         * 任务队列容量：Integer.MAX_VALUE（约 21 亿）
         *
         * 导出任务的特点：允许等待，不允许丢失
         *   用户提交了导出请求，宁愿等 10 分钟也不希望任务被丢弃
         *   所以用超大队列，确保所有任务都能排队等待执行
         *
         * 对比场景（实时性要求高，不允许积压）：
         *   如果是支付回调处理，队列满了应该立即告警，而不是无限等待
         *   那种场景应该用小队列 + 报警拒绝策略
         */
        executor.setQueueCapacity(Integer.MAX_VALUE);

        /**
         * 线程名前缀：方便在日志/线程 dump 中识别这是哪个线程池的线程
         * 不设置的话线程名是 "task-1"、"task-2"，无法区分业务归属
         * 设置后线程名是 "export-1"、"export-2"，一眼看出是导出任务
         */
        executor.setThreadNamePrefix("export-");

        /**
         * 拒绝策略：AbortPolicy（直接抛出异常）
         *
         * 四种拒绝策略对比：
         * AbortPolicy（默认）：抛出 RejectedExecutionException，让调用方感知到任务被拒绝
         * CallerRunsPolicy：由提交任务的线程自己执行，相当于降级为同步执行
         * DiscardPolicy：静默丢弃，危险！任务消失无踪，适合可丢失的日志等场景
         * DiscardOldestPolicy：丢弃队列最老的任务，让新任务进来
         *
         * 导出场景选 AbortPolicy：队列是"无限的"（Integer.MAX_VALUE），
         * 正常情况下永远不会触发拒绝策略
         * 如果真的触发了，说明系统有严重问题，应该立即报错让开发人员感知
         */
        executor.setRejectedExecutionHandler(
                new ThreadPoolExecutor.AbortPolicy()
        );

        // 初始化线程池，必须调用，否则上面的配置不生效
        executor.initialize();

        return executor;
    }
}
```

> 💡 **线程池参数设计的核心思路（一张图记住）**
>
> ```
> 任务提交
>    ↓
> 核心线程是否已满？
>    ├─ 否 → 创建核心线程执行                    ← 正常状态
>    └─ 是 → 队列是否已满？
>               ├─ 否 → 任务入队等待              ← 负载偏高
>               └─ 是 → 最大线程是否已满？
>                          ├─ 否 → 创建非核心线程  ← 高负载
>                          └─ 是 → 触发拒绝策略    ← 过载，需告警
> ```
>
> 设计线程池参数时要问自己三个问题：
> 1. **任务能等吗？** → 决定队列大小（能等→大队列，不能等→小队列）
> 2. **任务能丢吗？** → 决定拒绝策略（不能丢→AbortPolicy报错，能丢→DiscardPolicy）
> 3. **任务有多重？** → 决定核心线程数（CPU密集→少线程，I/O密集→多线程）

---

#### 🧪 运行与验证

线程池本身没有对外接口，通过启动日志验证 Bean 注册成功：

```bash
# 启动项目，观察日志中是否有线程池初始化的信息
# Spring 会打印 Bean 的创建过程（debug 级别）

# 也可以写一个简单的单元测试验证线程池 Bean 存在：
@SpringBootTest
public class ExecutorConfigTest {

    @Autowired
    @Qualifier("exportServiceExecutor")
    private Executor exportServiceExecutor;

    @Test
    public void testExecutorBeanExists() {
        Assertions.assertNotNull(exportServiceExecutor);
        System.out.println("线程池 Bean 注入成功: " + exportServiceExecutor.getClass().getName());
    }
}
```

---

### 📦 Commit 13 —— EasyExcel 分批导出与异步化

> `feat: 添加支持异步的 EasyExcel 批量导出功能`

---

#### 🎯 本步目标

实现完整的 Excel 导出功能：定义导出专用 DTO（与 EasyExcel 注解绑定），用分页查询分批加载数据写入不同 Sheet，通过内存流将生成的 Excel 转存到文件服务；最后用 `@Async` 将整个导出过程异步化，接口立即返回，后台静默完成。

---

#### 📂 目录变动

```
新增：
└── src/main/java/.../
    ├── domain/dto/
    │   └── UserExportDTO.java              ← Excel 导出专用实体
    ├── util/
    │   └── LocalDateTimeStringConverter.java ← LocalDateTime 转 Excel 字符串的转换器
    ├── service/
    │   └── ExcelExportService.java         ← 导出服务接口
    └── service/impl/
        └── ExcelExportServiceImpl.java     ← 导出服务实现（分批 + 异步）

修改：
└── src/main/java/.../
    └── controller/UserController.java      ← 新增 /export 接口

pom.xml ← 新增 EasyExcel 依赖
```

---

#### 💻 核心代码与详解

**① `pom.xml` — 新增 EasyExcel 依赖**

```xml
<!-- EasyExcel：阿里巴巴出品，基于 POI 的高性能 Excel 处理库
     核心优势：流式写入，不把整个 Excel 加载到内存
     百万行数据导出内存占用控制在几十 MB（POI 原生可能需要几 GB）-->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>easyexcel</artifactId>
    <version>2.1.6</version>
</dependency>

<!-- EasyExcel 依赖 ASM 做字节码增强，需要显式引入兼容版本 -->
<dependency>
    <groupId>org.ow2.asm</groupId>
    <artifactId>asm-all</artifactId>
    <version>5.2</version>
</dependency>
```

---

**② `UserExportDTO.java` — Excel 导出专用实体**

```java
@Data
public class UserExportDTO implements Serializable {

    /**
     * @ExcelProperty(value = "列名")
     * 指定该字段对应 Excel 中的列标题
     * EasyExcel 会自动把字段值填入对应列
     */
    @ExcelProperty(value = "用户名")
    private String username;

    @ExcelProperty(value = "年龄")
    private Integer age;

    @ExcelProperty(value = "版本号")
    private Long version;

    /**
     * LocalDateTime 类型的特殊处理：
     *
     * EasyExcel 原生不支持 LocalDateTime（只支持 java.util.Date）
     * 解决方案：提供自定义转换器 LocalDateTimeStringConverter
     *
     * @ExcelProperty 的 converter 参数指定使用哪个转换器
     * @DateTimeFormat 指定格式化模板（传给转换器使用）
     */
    @ExcelProperty(
        value = "创建时间",
        converter = LocalDateTimeStringConverter.class
    )
    @DateTimeFormat("yyyy年MM月dd日HH时mm分ss秒")
    private LocalDateTime created;
}
```

> 💡 **为什么不直接在 `UserDTO` 上加 `@ExcelProperty`，要单独建一个 `UserExportDTO`？**
>
> 职责分离原则：
> - `UserDTO`：承载业务参数，带 Bean Validation 校验注解
> - `UserExportDTO`：承载 Excel 导出格式，带 EasyExcel 注解
>
> 如果合并在一个类里，未来 Excel 格式调整（如加一列、改列名）会影响到业务传输对象，两个完全不同的关注点被耦合在一起。单独的导出 DTO 让两边变化互不影响。

---

**③ `LocalDateTimeStringConverter.java` — 自定义类型转换器**

```java
@Slf4j
public class LocalDateTimeStringConverter
        implements Converter<LocalDateTime> {

    // 告诉 EasyExcel：这个转换器处理 Java 端的 LocalDateTime 类型
    @Override
    public Class supportJavaTypeKey() {
        return LocalDateTime.class;
    }

    // 告诉 EasyExcel：转换后 Excel 单元格存储为字符串类型
    @Override
    public CellDataTypeEnum supportExcelTypeKey() {
        return CellDataTypeEnum.STRING;
    }

    /**
     * Excel 导入时使用（Excel → Java）
     * 本项目暂不实现导入功能，返回 null 即可
     */
    @Override
    public LocalDateTime convertToJavaData(
            CellData cellData,
            ExcelContentProperty contentProperty,
            GlobalConfiguration globalConfiguration) {
        return null;
    }

    /**
     * Excel 导出时使用（Java → Excel）
     * 将 LocalDateTime 转换为格式化的字符串写入单元格
     */
    @Override
    public CellData convertToExcelData(
            LocalDateTime localDateTime,
            ExcelContentProperty contentProperty,
            GlobalConfiguration globalConfiguration) {

        // 优先使用 @DateTimeFormat 注解中指定的格式
        if (contentProperty != null
                && contentProperty.getDateTimeFormatProperty() != null) {

            String pattern = contentProperty
                    .getDateTimeFormatProperty().getFormat();

            return new CellData(
                DateTimeFormatter.ofPattern(pattern).format(localDateTime)
            );
        }

        // 没有指定格式时，使用 ISO 标准格式（如 "2024-01-01T10:00:00"）
        return new CellData(
            DateTimeFormatter.ISO_LOCAL_DATE_TIME.format(localDateTime)
        );
    }
}
```

---

**④ `ExcelExportServiceImpl.java` — 分批导出的核心逻辑**

```java
@Service
@Slf4j
public class ExcelExportServiceImpl implements ExcelExportService {

    @Resource(name = "localFileServiceImpl")
    private FileService fileService;

    @Autowired
    private UserService userService;

    /**
     * 核心私有方法：执行分批查询和 Excel 写入
     * 设计为私有方法是因为它只是一个内部实现步骤，不对外暴露
     *
     * @param outputStream 数据写入的目标流（由调用方提供）
     * @param query        查询条件
     */
    private void doExport(OutputStream outputStream, UserQueryDTO query) {

        /**
         * EasyExcelFactory.write()：创建 ExcelWriter
         * 参数1：数据写入的目标（这里是内存流，也可以是 HttpServletResponse 的流）
         * 参数2：数据实体类（EasyExcel 通过反射读取 @ExcelProperty 注解生成列头）
         */
        ExcelWriter excelWriter = EasyExcelFactory
                .write(outputStream, UserExportDTO.class)
                .build();

        // 分页查询配置：每批处理 2 条（演示用，生产建议 500~2000 条）
        PageQuery<UserQueryDTO> pageQuery = new PageQuery<>();
        pageQuery.setQuery(query);
        pageQuery.setPageSize(2);

        int pageNo = 0;
        PageResult<List<UserDTO>> pageResult;

        do {
            // 先自增再使用（pageNo 从 1 开始）
            pageQuery.setPageNo(++pageNo);

            log.info("开始导出第 [{}] 页数据", pageNo);

            // 查询一批数据
            pageResult = userService.query(pageQuery);

            // UserDTO → UserExportDTO 转换
            List<UserExportDTO> exportList = Optional
                    .ofNullable(pageResult.getData())
                    .map(List::stream)
                    .orElseGet(Stream::empty)
                    .map(userDTO -> {
                        UserExportDTO exportDTO = new UserExportDTO();
                        BeanUtils.copyProperties(userDTO, exportDTO);
                        return exportDTO;
                    })
                    .collect(Collectors.toList());

            /**
             * 为每一批数据创建一个独立的 Sheet 页
             * writerSheet(sheetNo, sheetName)
             *   sheetNo：Sheet 序号（从 0 开始，但我们用 pageNo 从 1 开始，视觉上更友好）
             *   sheetName：Sheet 标签名，显示在 Excel 底部标签栏
             *
             * 为什么每批一个 Sheet 而不是全部写在一个 Sheet？
             * 因为单个 Sheet 超过 100 万行 Excel 会报错（xlsx 格式上限）
             * 分 Sheet 可以突破这个限制，每个 Sheet 最多 100 万行
             */
            WriteSheet writeSheet = EasyExcelFactory
                    .writerSheet(pageNo, "第" + pageNo + "页")
                    .build();

            excelWriter.write(exportList, writeSheet);

            log.info("第 [{}] 页数据导出完成，本批 {} 条", pageNo, exportList.size());

        // 循环条件：总页数 > 当前页号，说明还有下一页
        } while (pageResult.getPageNum() > pageNo);

        /**
         * finish()：必须调用！
         * 作用：将 ExcelWriter 内部缓冲的数据全部刷入 outputStream，并关闭文件
         * 不调用的话：outputStream 中的数据可能是不完整的，文件损坏无法打开
         */
        excelWriter.finish();

        log.info("Excel 导出完成，共 {} 页", pageNo);
    }

    /**
     * 同步导出（供测试和对比使用）
     * 整个过程在调用线程中完成，接口会阻塞直到导出完毕
     */
    @Override
    public void export(UserQueryDTO query, String filename) {

        // 内存输出流：Excel 数据先写入内存，再整体上传
        // 适合中小文件（几十 MB 以内）
        ByteArrayOutputStream outputStream = new ByteArrayOutputStream();

        // step1：把 Excel 数据写入内存流
        doExport(outputStream, query);

        // step2：把内存流转为输入流，上传到文件存储
        // ByteArrayInputStream 是对 byte[] 的包装，不会产生额外的内存拷贝
        ByteArrayInputStream inputStream =
                new ByteArrayInputStream(outputStream.toByteArray());

        fileService.upload(inputStream, filename);
    }

    /**
     * 异步导出（生产使用）
     *
     * @Async("exportServiceExecutor")：
     *   此方法被 Spring AOP 代理，调用时不在当前线程执行，
     *   而是提交到 "exportServiceExecutor" 线程池异步执行
     *   调用方立即获得控制权（方法立即返回），不等待导出完成
     */
    @Async("exportServiceExecutor")
    @Override
    public void asyncExport(UserQueryDTO query, String filename) {

        log.info("异步导出任务开始，线程名: {}, 文件名: {}",
                Thread.currentThread().getName(), filename);

        // 直接复用同步导出逻辑
        export(query, filename);

        log.info("异步导出任务完成，文件名: {}", filename);
    }
}
```

> ⚠️ **`@Async` 失效的三大原因，必须知道：**
>
> ```java
> // 原因1：同类内部调用（和 @Cacheable 一样的 AOP 代理问题）
> public void someMethod() {
>     this.asyncExport(query, filename);  // ❌ 同步执行，@Async 失效
> }
> // 解决：从 Spring 容器中获取代理对象再调用，或拆到另一个 Bean
>
> // 原因2：@EnableAsync 没有加在配置类上
> // 解决：确认 ExecutorConfig 上有 @EnableAsync（C12 已加）
>
> // 原因3：@Async 方法是 private 的
> // Spring AOP 无法代理 private 方法
> // 解决：必须是 public 方法
> @Async("exportServiceExecutor")
> public void asyncExport(...) { ... }  // ✅ public，可以被代理
> ```

---

**⑤ `UserController.java` — 新增导出接口**

```java
@Autowired
private ExcelExportService excelExportService;

/**
 * 数据导出接口
 * GET /api/users/export?username=xxx&filename=users.xlsx
 *
 * 设计要点：接口立即返回 true，导出在后台异步进行
 * 前端收到响应后可以轮询"导出任务状态"接口（本项目简化，不实现状态查询）
 */
@GetMapping("/export")
public ResponseResult<Boolean> export(
        @Validated UserQueryDTO query,
        @NotEmpty(message = "导出文件名不能为空") String filename) {

    log.info("收到导出请求，文件名: {}", filename);

    // 异步执行，立即返回
    // 调用的是 @Async 方法，此行代码执行完毕时，导出任务刚刚被提交到线程池
    // 导出实际上还没开始，但接口已经返回了
    excelExportService.asyncExport(query, filename);

    return ResponseResult.success(Boolean.TRUE);
}
```

---

#### 🧪 运行与验证

**① 验证异步导出接口**

```bash
curl "http://localhost:8080/api/users/export?username=username1&filename=users.xlsx"

# 接口应该立即返回（不超过 100ms）：
# { "success": true, "result": true }
```

**② 观察控制台日志，确认异步执行**

```
# 接口返回后，后台线程才开始打印导出日志
# 注意线程名是 "export-1"（ExecutorConfig 中配置的前缀）

2024-01-01 10:00:00.100 [traceId-xxx] --- [  export-1] ExcelExportServiceImpl : 异步导出任务开始，线程名: export-1
2024-01-01 10:00:00.120 [traceId-xxx] --- [  export-1] ExcelExportServiceImpl : 开始导出第 [1] 页数据
2024-01-01 10:00:00.135 [traceId-xxx] --- [  export-1] ExcelExportServiceImpl : 第 [1] 页数据导出完成，本批 2 条
2024-01-01 10:00:00.150 [traceId-xxx] --- [  export-1] ExcelExportServiceImpl : Excel 导出完成，共 1 页
2024-01-01 10:00:00.160 [traceId-xxx] --- [  export-1] ExcelExportServiceImpl : 异步导出任务完成，文件名: users.xlsx
```

**③ 验证导出文件**

```bash
# 导出完成后，在 uploads/ 目录下应该能看到文件
ls uploads/
# users.xlsx  test.txt  .gitkeep

# 用 Excel 打开 users.xlsx，应该能看到：
# - 列头：用户名 | 年龄 | 版本号 | 创建时间
# - 数据按 2 条一个 Sheet 分页（因为 pageSize=2）
# - 创建时间格式：2020年01月24日08时01分50秒
```

**④ 对比同步和异步的响应时间差异**

临时修改 Controller 调用同步接口做对比：

```java
// 改为同步调用（测试完记得改回来）
excelExportService.export(query, filename);  // 同步
// excelExportService.asyncExport(query, filename);  // 异步
```

```bash
# 同步调用，接口需要等待导出完成才返回（可能需要几百毫秒到几秒）
time curl "http://localhost:8080/api/users/export?username=username1&filename=sync.xlsx"
# real    0m0.856s  ← 等待了 856ms

# 切回异步调用
time curl "http://localhost:8080/api/users/export?username=username1&filename=async.xlsx"
# real    0m0.023s  ← 立即返回，只用了 23ms
```

---

### ✅**v4.0 阶段完整回顾**
>
> 三次提交，完成了两个生产级功能模块：
>
> ```
> 文件上传链路：
> HTTP multipart/form-data
>     ↓
> FileController.upload()          C11：接收 MultipartFile，获取流
>     ↓
> LocalFileServiceImpl.upload()    C11：1KB 缓冲区循环写，TWR 保证流关闭
>     ↓
> uploads/ 目录落盘
>     ↓
> /uploads/{filename} HTTP 访问    C10（WebConfig）：静态资源映射
>
> Excel 异步导出链路：
> GET /api/users/export
>     ↓
> UserController.export()          C13：立即返回 true
>     ↓（提交到线程池）
> ExcelExportServiceImpl           C12：export- 前缀线程池
>     ↓（循环分页）
> userService.query(pageNo++)      每批 N 条，不积压内存
>     ↓
> EasyExcel.write(sheet)           每批写一个 Sheet
>     ↓
> excelWriter.finish()             C13：刷盘，关闭文件
>     ↓
> fileService.upload()             C11：内存流 → 文件存储
> ```
>
> 至此 v4.0 的核心挑战——**大文件不 OOM、导出不超时**——已经被完整解决。

---

###  📌 **下期预告 —— v5.0：API 文档 & 工程收尾（C14 ~ C15）**
>
> 最后阶段将实现：
> - `CommonMetaObjectHandler`：`created`、`modified`、`creator`、`version` 等系统字段在新增/更新时自动填充，彻底消灭 Service 层的手动赋值代码
> - `OptimisticLockerInterceptor`：乐观锁拦截器，配合 `@Version` 字段，并发更新时自动检测冲突，防止"最后写入者获胜"导致的数据覆盖
> - `Swagger2Config`：一行配置生成可交互的 API 文档，所有接口自动展示，支持在线调试