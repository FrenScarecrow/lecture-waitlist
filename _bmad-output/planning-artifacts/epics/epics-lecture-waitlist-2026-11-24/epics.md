---
status: final
stepsCompleted: [1, 2, 3, 4]
inputDocuments:
  - _bmad-output/planning-artifacts/prds/prd-lecture-waitlist-2026-11-22/prd.md
  - _bmad-output/planning-artifacts/architecture/architecture-lecture-waitlist-2026-11-22/architecture.md
  - _bmad-output/planning-artifacts/architecture/architecture-lecture-waitlist-2026-11-22/project-context.md
---

# 讲座报名与候补系统（lecture-waitlist）— Epic 拆解

## Overview

本文档把 PRD 的功能需求（FR-1…FR-14）、跨特性非功能需求（NFR-1…NFR-4）与架构决策（ADR-001…ADR-011）及 project-context 的 20 条硬性实现规则，拆解为 3 个业务史诗（共 12 条业务故事）+ 3 条支撑故事（工程脚手架、账号与角色、学生个人中心），供 Developer 直接实施。

- **产品核心**：并发正确性（不超卖、不双占）+ 候补状态机（按序、幂等、可审计）。
- **技术栈**：Vue 3 + Spring Boot 4 + MyBatis + MySQL 8.4 + Java 21（ADR-001…ADR-003）。
- **输出结构**：本文件为索引（需求清单、覆盖图、史诗列表）；每条故事一个独立文件，见 `stories/` 目录。

## Requirements Inventory

### Functional Requirements

- **FR-1**：账号密码登录——学生/管理员用用户名+密码登录，学生用户名即学号；失败给统一错误提示，不泄露账号存在与否。
- **FR-2**：两类角色——仅「学生」「管理员」，管理员即助教本人、单人直接生效；无审批流、无多人协作。
- **FR-3**：发布讲座——填标题、开始/结束时间、容量、报名开始时间（可选地点/主讲人/类型）；报名截止由系统派生为开始前 30 分钟；发布后状态「已发布」。
- **FR-4**：浏览列表与实时余量——学生浏览未结束讲座列表，实时看到剩余名额、满员标记与状态。
- **FR-5**：报名讲座——报名窗口内、有空位即报名成功占名额；满员拒绝；同场不双占；并发不超卖。
- **FR-6**：取消报名——报名窗口内取消已报名讲座，释放名额并触发补位（见 FR-8）。
- **FR-7**：满员提示与主动加入候补——满员时学生主动选择加入候补，按加入时间分配可见顺序号。
- **FR-8**：取消触发直接补位——取消产生空位，立即按候补顺序把队首转为已报名并站内通知。
- **FR-9**：补位幂等与重复触发防护——同一空位只补一人；重复触发不二次补位、不双占。
- **FR-10**：退出候补与截止清空——报名截止前可退出候补；截止后 1 分钟内系统清空候补并通知。
- **FR-11**：候补顺序审计——候补加入时间、顺序号、补位/退出/清空全程留痕，任意时点顺序可追溯还原。
- **FR-12**：容量调整——发布后至开始前 60 分钟可调容量；扩容即时补位，缩容按「后报名者先移出」转候补队首。
- **FR-13**：报名名单导出——导出 CSV（UTF-8 with BOM），字段含学号、姓名、报名时间、候补顺序。
- **FR-14**：关键操作日志——发布/报名/取消/补位/候补增删/调容量/清空全程记日志，可审计。

### NonFunctional Requirements

- **NFR-1 并发正确性（不超卖、不双占）**：并发报名/取消/补位下，最后一个名额只被一人占用；任一时刻有效报名数 ≤ 容量；同场不双占。
- **NFR-2 补位正确性（按序、幂等、可审计）**：按候补加入顺序补位，同一空位只补一人，重复触发不二次补位，全程可审计。
- **NFR-3 补位时延**：取消事务提交即完成补位（不依赖后台任务）。
- **NFR-4 部署与环境**：单机部署、零外部依赖（无 SMTP/短信/微信），电脑浏览器访问，部署运行门槛低。

### Additional Requirements（Architecture）

技术约束（来自 architecture.md 与 project-context.md，影响实现与验收）：

- 技术栈固定：Vue 3 + Vite + TypeScript；Spring Boot 4 + MyBatis（XML）+ MySQL 8.4 + Java 21（ADR-001/002/003）。
- 绿场工程结构：`lecture-waitlist-front/` + `lecture-waitlist-backend/` 两独立工程，构建期前端产物复制进后端 `static/`，运行态单 jar 同源（ADR-001）。
- 三层架构 + 事务边界：Controller（取参/校验）→ Service（业务/事务/锁）→ Mapper（SQL），依赖只向下（ADR-002，规则 1/2）。
- 硬删除 + 普通 UNIQUE + 审计表留痕：取消/补位消费/退出/清空一律硬删除；`UNIQUE(lecture_id, student_id)` 防双占；历史靠 `audit_log`（ADR-004，规则 9）。
- 先锁后读：影响容量/报名/候补的写事务第一步 `SELECT … FOR UPDATE` 锁 `lecture` 行，再派生 `COUNT(*)` 比较容量；隔离级别 REPEATABLE READ（ADR-005，规则 3/6/17）。
- 补位同步触发 + 硬删除原子消费：补位与取消/扩容同事务；取队首 `DELETE` 影响行数为 1 才 `INSERT`（ADR-006，规则 5/18）。
- 候补顺序复合排序键 + 读时计算：排序键 `(cohort, sort_ts)`，`cohort` 数值优先级 front→0/normal→1；顺序号 `ROW_NUMBER()` 读时计算、不落库（ADR-007，规则 12/13）。
- 登录与会话：Spring Security + 服务端会话 cookie + Spring Session JDBC（MySQL）+ BCrypt；角色单列 `role`（ADR-008，规则 14）。
- 站内通知同事务写库：`notification` 表行与业务变更同事务（ADR-009，规则 10）。
- 部署形态：单 jar 同源 + Docker Compose 两容器 + `scripts/bootstrap.sh` 回退（ADR-010）。
- 时间读时派生 + 进程内定时清空：讲座状态读时派生；候补清空由 `@Scheduled(fixedDelay=60s)` 进程内执行（ADR-011，规则 16）。
- 时间一律 UTC：存储/边界判断用 UTC，展示层转 Asia/Shanghai（规则 7）。
- 错误码规范：统一 `{ code, message, data }` envelope；`DuplicateKeyException` 不得抛 500（规则 8）。
- 禁引入 Redis/消息队列/云 SDK/第二数据访问层（规则 11）。
- 测试与构建：JUnit 5 + Spring Boot Test、注入 `Clock`；前端 `npm ci && npm run build`、后端 `mvn clean package`（规则 19/20）。

### UX Design Requirements

（无 UX 设计契约输入，本节留空。）

## FR Coverage Map

| 需求 | 故事 | 简述 |
| --- | --- | --- |
| FR-1 | 支撑 F-2 | 账号密码登录 |
| FR-2 | 支撑 F-2、F-3 | 两类角色与鉴权、个人中心（通知与状态） |
| FR-3 | 1.1 | 发布讲座 |
| FR-4 | 1.2 | 浏览列表与实时余量 |
| FR-5 | 1.3 | 报名讲座 |
| FR-6 | 1.4 | 取消报名 |
| FR-7 | 2.1 | 满员提示与主动加入候补 |
| FR-8 | 2.2 | 取消触发直接补位 |
| FR-9 | 2.3 | 补位幂等与重复触发防护 |
| FR-10 | 2.4 | 退出候补与截止清空 |
| FR-11 | 2.5 | 候补顺序审计 |
| FR-12 | 3.1 | 容量调整 |
| FR-13 | 3.2 | 报名名单导出 |
| FR-14 | 3.3 | 关键操作日志 |

**NFR 覆盖**：

- NFR-1（并发不超卖/不双占）→ 故事 1.3、2.3、3.1（贯穿报名/补位/扩容的并发 AC）。
- NFR-2（补位按序/幂等/可审计）→ 故事 2.2、2.3。
- NFR-3（取消事务提交即补位）→ 故事 2.2。
- NFR-4（单机、零外部依赖）→ 支撑 F-1（单 jar 同源 + Docker Compose），全故事遵循「禁引入外部服务」（规则 11）。

## Epic List

> 支撑故事（Foundation）为业务史诗的前置，不计入 12 条业务故事。

### 支撑故事（Foundation）

- **F-1 工程脚手架** — 建立前后端分离工程骨架、Flyway 基线、一键构建/启动与前端基础布局。见 `stories/F-1-project-scaffold.md`
- **F-2 账号与角色**（FR-1、FR-2）— 账号密码登录 + 两类角色鉴权。见 `stories/F-2-account-and-roles.md`
- **F-3 学生个人中心**（FR-2）— 查看报名/候补列表（含顺序号）与站内通知、未读标记。见 `stories/F-3-student-center.md`

### 史诗一：讲座发布与报名（FR-3、FR-4、FR-5、FR-6）

用户成果：管理员能发布讲座，学生能浏览列表、看余量、报名、取消。

- **1.1 发布讲座**（FR-3）— `stories/1-1-publish-lecture.md`
- **1.2 浏览列表与实时余量**（FR-4）— `stories/1-2-browse-list-availability.md`
- **1.3 报名讲座**（FR-5）— `stories/1-3-register-lecture.md`
- **1.4 取消报名**（FR-6）— `stories/1-4-cancel-registration.md`

### 史诗二：候补与自动补位（FR-7、FR-8、FR-9、FR-10、FR-11）

用户成果：满员可加入候补并拿到顺序号；取消/扩容自动补位，公平可审计。

- **2.1 满员提示与主动加入候补**（FR-7）— `stories/2-1-join-waitlist.md`
- **2.2 取消触发直接补位**（FR-8）— `stories/2-2-cancel-promotion.md`
- **2.3 补位幂等与重复触发防护**（FR-9）— `stories/2-3-promotion-idempotency.md`
- **2.4 退出候补与截止清空**（FR-10）— `stories/2-4-leave-waitlist-deadline-clearing.md`
- **2.5 候补顺序审计**（FR-11）— `stories/2-5-waitlist-order-audit.md`

### 史诗三：运营与治理（FR-12、FR-13、FR-14）

用户成果：管理员能调容量（即时补位/缩容插队）、导名单、查看操作日志。

- **3.1 容量调整**（FR-12）— `stories/3-1-capacity-adjustment.md`
- **3.2 报名名单导出**（FR-13）— `stories/3-2-export-csv.md`
- **3.3 关键操作日志**（FR-14）— `stories/3-3-audit-log.md`

## 实现说明（跨故事约束）

- `audit_log`、`notification` 等表由支撑故事 F-1 的 Flyway 基线按最终数据模型一次性建好（架构数据模型已定稿）；审计/通知的**写入**遵循规则 10「同事务」，在各业务故事实现对应操作时一并接入。故事 2.5 交付「候补顺序可重构」的查询口径，故事 3.3 交付「管理员日志视图 + 全量操作类型审计覆盖校验」。
- 补位唯一入口 `promoteOne()` 在故事 2.2 引入，故事 2.3 加固幂等，故事 3.1（扩容）复用同一入口（规则 5）。
- 故事 1.4（取消）先交付「释放名额」，补位接入在故事 2.2 对取消事务做增量修改（取消存在候补者时调用 `promoteOne()`，报名数保持不变）。
