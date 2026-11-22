---
title: 讲座报名与候补系统 — 项目上下文 (project-context.md)
status: final
created: 2026-11-22
updated: 2026-11-22
companion: architecture.md
---

# 项目上下文

本文档供实现本系统的智能体 / 开发者作为**硬性约束**使用。架构决策见 `architecture.md`；本文只列「技术栈与版本」和「实现规则」——规则刻意只写**智能体猜不到、或容易自作主张写错**的东西。

## 技术栈与版本

| 层 | 选型 | 版本 |
| --- | --- | --- |
| 后端语言 | Java（Temurin） | 21 LTS |
| 后端框架 | Spring Boot | 4.1.x |
| 持久层 | MyBatis（XML Mapper）+ Flyway 迁移 | 以 `pom.xml` 锁定 |
| 认证 / 会话 | Spring Security + Spring Session JDBC | 同上 |
| 密码哈希 | BCrypt（`BCryptPasswordEncoder`） | 同上 |
| 数据库 | MySQL | 8.4 LTS |
| 前端 | Vue 3 + Vite + TypeScript | Vue 3.5.x |
| 构建 | Maven（后端）+ npm/Vite（前端） | Maven 3.9.x |

> **Java 版本**：Java 21 LTS（Temurin，支持到 2029，环境普及度最高）；Java 25 亦可，但代码不要用 22+ 独有 API。Spring Boot 4.1 要求 Java 17+。
>
> **Spring Boot 版本**：4.1.x（当前稳定；4.0 临近 2026-12-31 结束支持，3.5 已结束）。本项目的核心 API（`@Transactional`、Spring Security、Spring Session）与 3.x 基本一致。
>
> **Vue 版本**：3.5.x 为稳定版；3.6（Vapor 模式）仍在 RC，不要用。

## 工程结构

```text
lecture-waitlist/               # 仓库根目录
├── lecture-waitlist-front/     # Vue 3 + Vite + TypeScript（独立工程）
├── lecture-waitlist-backend/   # Spring Boot + MyBatis + MySQL（独立工程）
├── docker-compose.yml          # backend + mysql
└── scripts/bootstrap.sh         # 无 Docker 回退
```

后端三层分包：`controller/`（表现层）、`service/`（业务层，事务边界）、`mapper/`（持久层，MyBatis XML）。前端独立工程，通过 `/api` 与后端交互。

## 实现规则

### 规则 1 — 三层架构边界（强制）

- **Controller 只做**：取参、`@Valid` 校验、调 Service、返回响应。禁止在 Controller 写业务逻辑或 SQL。
- **Service 只做**：业务逻辑、`@Transactional` 事务边界、并发锁。禁止在 Service 里拼接 SQL。
- **Mapper 只做**：SQL（XML 或注解）。禁止在 Mapper 里写业务判断。
- 依赖只向下：Controller → Service → Mapper。禁止反向依赖、禁止 Controller 直接调 Mapper。

### 规则 2 — 事务边界（强制）

- 所有**写**操作必须在 `@Transactional` Service 方法内完成；禁止在事务外写库。
- 事务内只做 DB 读写；禁止在事务内做网络 / 文件 IO、`Thread.sleep` 等耗时操作（保持事务短，配合 P95≤1s）。
- `@Transactional` 经 Spring 代理才生效，禁止同类内 `this.xxx()` 自调用绕过事务代理。
- 只读请求不加事务。

### 规则 3 — 先锁后读，禁止「先查后改」

- 凡影响某讲座容量 / 报名 / 候补的写事务，**第一步**必须是锁该 `lecture` 行，MyBatis Mapper 写死 `FOR UPDATE`：

  ```xml
  <select id="lockLectureById" resultType="Lecture">
    SELECT * FROM lecture WHERE id = #{id} FOR UPDATE
  </select>
  ```

  锁完之后，才允许 `COUNT`、判断容量、写报名 / 候补。
- **禁止**「先 `COUNT` 判断未满、再 `INSERT`」的无锁写法——并发下必超卖。没有例外。

### 规则 4 — 状态迁移只能走指定 Service 方法

- `registration` 与 `waitlist_entry` 的**增删**只能通过 `service/` 下对应的唯一方法：`register()`、`cancelRegistration()`、`joinWaitlist()`、`leaveWaitlist()`、`promoteOne()`、`resizeCapacity()`。
- Controller / 其他层**禁止**直接调 Mapper 的 `delete` / `insert` 改这两张表。要改语义，改那个 Service 方法，不许另写。

### 规则 5 — 补位是唯一幂等例程

- `promoteOne()` 是**取消与扩容共用的唯一补位入口**：

  ```java
  // 取队首（FOR UPDATE 锁候补行）
  WaitlistEntry head = waitlistMapper.selectHeadForUpdate(lectureId); // ORDER BY cohort, sort_ts LIMIT 1 FOR UPDATE
  if (head == null) return;               // 无候补者，仅释放名额
  waitlistMapper.deleteById(head.getId()); // 硬删除原子消费
  registrationMapper.insert(new Registration(lecture, head.getStudentId(), "promoted"));
  ```

- 禁止在取消 / 扩容里各自实现一遍补位逻辑。

### 规则 6 — 计数禁止冗余列

- `lecture` 表**不设** `active_count` / `remaining` 计数列。
- 「剩余名额 / 满员」一律由 `SELECT COUNT(*) FROM registration WHERE lecture_id=?` 对 `capacity` 派生（配合 ADR-004 硬删除，`COUNT(*)` 即有效数）。禁止新增或维护计数列。

### 规则 7 — 时间一律存 UTC

- 时间统一用 Java `Instant` / `OffsetDateTime`（JDBC 存 `DATETIME`，值一律 UTC）。
- 边界判断（报名窗口、退出候补最晚 min(截止, 开始前 30min)、调容量窗口、讲座状态派生）**一律用 UTC 比较**；只在展示层（Vue 端）转 Asia/Shanghai。
- 禁止用 `LocalDateTime.now()` 默认时区 + 字符串比较做边界判断。

### 规则 8 — 错误码规范

- 接口统一 envelope：`{ code, message, data }`（成功按需带 `data`）。
- 业务拒绝抛业务异常，全局 `@RestControllerAdvice` 统一映射错误码与 HTTP 状态：

  | 场景 | code | HTTP |
  | --- | --- | --- |
  | 满员 | `LECTURE_FULL` | 409 |
  | 报名窗口外 | `REG_WINDOW_CLOSED` | 409 |
  | 重复报名 | `ALREADY_REGISTERED` | 409 |
  | 重复候补 | `DUPLICATE_WAITLIST` | 409 |
  | 调容量窗口外 | `CAPACITY_WINDOW_CLOSED` | 409 |
  | 越权 | `FORBIDDEN` | 403 |
  | 输入校验失败 | `VALIDATION_ERROR` | 422 |

- **禁止**把 `DuplicateKeyException`（唯一键冲突）直接抛成 500——必须捕获并映射为 `ALREADY_REGISTERED` / `DUPLICATE_WAITLIST`。Bean Validation（`@Valid`）负责输入校验（422）。

### 规则 9 — 唯一约束是最后防线

- Flyway 迁移里**必须**建（缺一不可）：

  ```sql
  ALTER TABLE registration ADD CONSTRAINT uq_registration UNIQUE (lecture_id, student_id);
  ALTER TABLE waitlist_entry ADD CONSTRAINT uq_waitlist UNIQUE (lecture_id, student_id);
  ```

- 即使应用层已查重，也必须保留该唯一约束兜底（并发最后防线）。

### 规则 10 — 通知与审计同事务

- 任何业务变更产生通知 / 审计时，通知行与审计行必须写在与该变更**同一个 `@Transactional`** 里。
- 禁止「先提交变更、再异步写通知 / 审计」的分两步写法。

### 规则 11 — 禁止引入的库 / 服务（硬约束）

- 禁止：Redis 及任何缓存库、消息队列（Kafka / RabbitMQ / RocketMQ / …）、任何云 SDK（阿里云 / AWS / …）。
- 禁止除 MyBatis 外的第二个数据访问层（不得同时引入 Spring Data JPA / MyBatis-Plus / JdbcTemplate 直接绕过 MyBatis）。
- 会话存储**必须** Spring Session JDBC（MySQL），禁止内存 / Redis session。

### 规则 12 — 候补顺序号禁止落库

- 顺序号 = 读时计算：

  ```sql
  SELECT …, ROW_NUMBER() OVER (
    PARTITION BY lecture_id ORDER BY cohort, sort_ts
  ) AS seq FROM waitlist_entry WHERE lecture_id = ?
  ```

- 禁止把顺序号作为可变字段存储；`joined_at` 只是审计时间戳，**不参与排序**（排序键是 `cohort + sort_ts`，见规则 13）。

### 规则 13 — 缩容移入者的排序语义

- 缩容把超出的报名者转候补时，其排序键必须是「队首、内部按原报名先后」：
  - `cohort = 'front'`（排最前）
  - `sort_ts = 原 registration.registered_at`
- 禁止简单用当前时间 join 到队尾。

### 规则 14 — 鉴权与角色

- 统一 Spring Security 鉴权；从 session 解析当前用户。
- 角色只用 `user.role` 单列两值（`student` / `admin`）判别；禁止用用户名 / 邮箱前缀等猜角色。
- 学生访问管理接口、管理员访问学生敏感接口，一律 403（`FORBIDDEN`），不重定向、不泄露信息。

### 规则 15 — 前后端边界（同源，无需 Nginx）

- 前端 Vue 只经 **`/api/**`** 调后端 REST，禁止前端直连数据库或旁路后端。
- **部署同源**：构建期把 Vue 产物复制进后端 `src/main/resources/static/`，由后端单端口同时托管前端页面与 `/api`（同一域、零 CORS）；开发期 Vite dev server 把 `/api` 代理到后端。**禁止前端 5173 跨域直连后端 8080**（否则 CORS + 会话 cookie SameSite 问题）。
- **前端路由用 hash 模式**（`createWebHashHistory()`），避免 history 模式下刷新子路由 404（后端不配 SPA fallback 时）。
- 前端展示用 UTC→Asia/Shanghai 转换；业务边界判断一律由后端（规则 7）完成，前端不做业务时间数学。
