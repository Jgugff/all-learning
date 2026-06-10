# Spring Boot 项目实战工作手册

> **项目背景**：企业级综合学习平台后端系统，涵盖用户管理、文件存储、Excel 导出、限流、缓存、链路追踪等企业常见横切面能力。
> **核心技术栈**：Spring Boot 2.2.x · MyBatis-Plus 3.3.x · EasyExcel · Caffeine Cache · Guava RateLimiter · Swagger2 · MySQL

---

## 第一部分：项目迭代计划（Roadmap）

| 阶段 | 阶段名称 | 核心解决的痛点 | 涉及技术栈 | 预期达成效果 |
|------|----------|--------------|-----------|-------------|
| **v1.0** | 骨架搭建 & 数据持久化 | "我该怎么从零建一个能跑的 Spring Boot 项目，并连上数据库做增删改查？" | Spring Boot、MyBatis-Plus、MySQL、Lombok | 能通过 HTTP 接口对 User 表进行完整的 CRUD，理解 DO/DTO 分层设计 |
| **v2.0** | 输入校验 & 统一规范 | "接口参数没人校验、报错时前端拿到的是一堆 HTML 堆栈，怎么办？" | Bean Validation、统一响应体 `ResponseResult`、全局异常处理 `@ControllerAdvice` | 所有接口参数自动校验，所有报错统一格式返回，再也不怕前端抱怨 |
| **v3.0** | 缓存 & 限流 & 日志追踪 | "查询接口太慢、高并发会把数据库打挂、线上排查日志找不到同一次请求的上下文" | Caffeine 本地缓存、Guava RateLimiter、MDC TraceId Filter | 热点查询走缓存、全局 QPS 限制、每条日志都带 TraceId 可以串联 |
| **v4.0** | 文件上传 & 数据导出异步化 | "大文件上传怎么做？百万行 Excel 导出会撑爆内存、会让接口超时" | MultipartFile、EasyExcel 分批写入、`@Async` 线程池、`ByteArrayOutputStream` | 文件稳定上传落盘，Excel 导出后台异步执行、分页写入不 OOM |
| **v5.0** | API 文档 & 工程收尾 | "接口怎么给前端看？代码里还有一堆 TODO 没处理" | Swagger2/SpringFox、`MetaObjectHandler` 自动填充、`OptimisticLockerInterceptor` 乐观锁 | 自动生成可交互 API 文档，创建/更新时间自动填充，并发更新安全 |
| **v6.0** | 认证&鉴权 | "接口没有保护，任何人可以任意访问？不同用户应该有不同权限？ | Spring Security 、 JWT | 身份认证（你是谁？），授权（你能做什么？），从请求上下文中获取当前登录用户 |

---
