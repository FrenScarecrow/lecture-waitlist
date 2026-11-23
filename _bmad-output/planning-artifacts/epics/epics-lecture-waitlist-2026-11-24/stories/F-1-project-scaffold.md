---
story_key: "F-1"
title: "工程脚手架"
epic: "支撑故事（Foundation）"
fr: []
adr: ["ADR-001", "ADR-002", "ADR-003", "ADR-010"]
rules: ["规则 6", "规则 8", "规则 9", "规则 11", "规则 15", "规则 19", "规则 20"]
status: draft
---

# Story F-1：工程脚手架

**作为** 开发者与不懂运维的学生助教，
**我希望** 项目具备前后端分离的工程骨架、一键构建与一条命令启动的能力，
**以便** 后续所有故事都能在稳定、可评审、零外部依赖的环境上迭代。

## 验收标准

1. 仓库根目录含 `lecture-waitlist-front/`（Vue 3 + Vite + TypeScript）与 `lecture-waitlist-backend/`（Spring Boot 4 + MyBatis + MySQL）两个独立工程；构建期前端产物复制进后端 `src/main/resources/static/`（ADR-001）。
2. 根目录提供 `docker-compose.yml`（`backend` + `mysql` 两容器）；`docker compose up -d --build` 一条命令起服务，包含 Flyway 迁移、管理员账号初始化，并 seed 2–3 个学生账号；README 写明初始账号与密码，供评审机登录演示（ADR-010）。
3. 提供无 Docker 回退脚本 `scripts/bootstrap.sh`（ADR-010）。
4. 根目录提供 `build.sh` 与 `build.cmd`；`npm ci && npm run build`（前端）与 `mvn clean package`（后端）均成功产出，README 只写这一条构建命令（规则 20）。
5. Flyway 基线迁移按最终数据模型建 6 张表：`user`、`lecture`、`registration`、`waitlist_entry`、`notification`、`audit_log`（ADR-004）。
6. Flyway 迁移建唯一约束 `UNIQUE(lecture_id, student_id)` on `registration` 与 `waitlist_entry`，缺一不可（规则 9）。
7. `lecture` 表不设 `active_count`/`remaining` 计数列；剩余名额一律由报名数对 `capacity` 派生（规则 6）。
8. 后端按三层分包 `controller/`、`service/`、`mapper/`（ADR-002）；提供统一响应 envelope `{ code, message, data }` 与全局 `@RestControllerAdvice` 异常映射（规则 8）。
9. 前端提供基础布局与登录页外壳（登录表单可见，实际鉴权在 Story F-2）；路由采用 hash 模式 `createWebHashHistory()`；开发期 Vite dev server 将 `/api` 代理到后端（规则 15）。
10. 未引入 Redis / 消息队列 / 云 SDK / 第二数据访问层（规则 11）。
11. 测试脚手架就位：JUnit 5 + Spring Boot Test，时间相关逻辑注入 `java.time.Clock`（规则 19）。
