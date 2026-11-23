---
story_key: "1.3"
title: "报名讲座"
epic: "史诗一 讲座发布与报名"
fr: ["FR-5"]
adr: ["ADR-004", "ADR-005"]
rules: ["规则 3", "规则 8", "规则 17", "规则 19"]
status: draft
---

# Story 1.3：报名讲座

**作为** 学生，
**我希望** 对报名窗口内且未满的讲座报名，
**以便** 占用一个名额，并在并发下保证绝不超卖、不双占。

## 验收标准

1. 有空位时报名成功，报名数 +1（FR-5）。
2. 达到容量后新报名被拒绝，返回 `LECTURE_FULL`（409）并提示满员（FR-5，规则 8）。
3. 同一学号对同一讲座重复报名被拒绝，返回 `ALREADY_REGISTERED`（409）（FR-5）。
4. 报名开始前、报名截止后，或讲座已开始/已结束后，报名被拒绝，返回 `REG_WINDOW_CLOSED`（409）（FR-5）。
5. 报名写事务第一步 `SELECT … FOR UPDATE` 锁 `lecture` 行，锁后才 `COUNT` 判断容量，禁止「先查后改」（ADR-005，规则 3/17）。
6. 并发 AC：100 个并发请求抢最后一个名额，成功数恰好 1，`registration` 行数不超过容量（NFR-1，规则 19）。
7. `DuplicateKeyException`（唯一键冲突）被捕获并映射为 `ALREADY_REGISTERED`，不抛 500（规则 8）。
8. 报名成功写入 `registration` 行，`first_entered_at = registered_at`（ADR-004 数据模型）。
