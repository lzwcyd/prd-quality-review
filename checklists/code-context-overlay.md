# 代码上下文叠加规则（Code Context Overlay）

> 当 `code_root` 指向有效代码工程时，本文件定义每个评估维度如何叠加「代码佐证」来调整判定结果。**这是叠加规则，不是新维度**。

---

## 总体规则（适用所有维度）

| 代码佐证状态 | 标记 | 对原判定的影响 |
|---|---|---|
| 代码已实现且与 PRD 一致 | ✅ 一致 | 该题判「未命中」（已满足），加正面证据 |
| 代码已实现但与 PRD 冲突 | ⚠️ 冲突 | **强制升级为 P0**，单独高亮在「PRD-代码冲突」章节 |
| 代码已实现但 PRD 缺失 | 🟡 PRD 缺 / 代码有 | 原命中**降级一级**（P0→P1，P1→P2，P2→不扣分），改进建议改为"PRD 应补描述对齐代码" |
| 代码未实现且 PRD 已写 | 🔵 PRD 有 / 代码无 | 不影响 PRD 评分，但作为「待研发实现项」单列，标注预计工作量提示 |
| 代码未实现且 PRD 也无 | ⚪ 双缺 | 按原 PRD 评分规则正常命中 |
| 未扫描或不可判 | ❓ 未扫描 | 不调级，但报告中标注未扫描原因 |

> **注**：仅 ⚠️ 与 🟡 会改变扣分；其他状态仅作信息披露。

---

## D1 基本信息与变更管理 - 代码佐证规则

| 问题 ID | 代码扫描动作 | 佐证判定 |
|---|---|---|
| **D1-Q4** 关联上游文档 | grep `README.md`、`CHANGELOG.md` 中是否提及该 PRD / 项目编号 | 命中 → ✅ |
| **D1-Q5** 操作记录表 | 查看 git log（仅当用户授权 `code_scan_depth=deep`），统计近期 commit 数 | 仅信息展示，不调分 |

> 本维度多数题为 PRD 元数据，**代码佐证作用有限**。

---

## D2 需求背景 - 代码佐证规则

| 问题 ID | 代码扫描动作 | 佐证判定 |
|---|---|---|
| **D2-Q4** 业务流程图 | 查找 `docs/`、`*.puml`、`*.mmd`、`README.md` 是否含流程图 | 命中且 PRD 缺图 → 🟡 |
| **D2-Q8** 名词解释 | 查找 `*.md` 中是否含 glossary / 术语表 | 命中且 PRD 缺 → 🟡 |

---

## D3 需求价值 - 代码佐证规则

> 价值是业务侧描述，**代码无法佐证**，本维度不进行代码扫描。

---

## D4 业务流程与场景 - 代码佐证规则

| 问题 ID | 代码扫描动作 | 佐证判定 |
|---|---|---|
| **D4-Q3** 角色职责 | 查找 `enum Role`、`@PreAuthorize`、权限相关常量 | 命中且 PRD 未提 → 🟡 |
| **D4-Q6** 状态机 | 查找 `enum *Status`、`*State`、状态机框架（Spring StateMachine、XState 等） | 命中且 PRD 未列 → 🟡；已列但枚举值与代码不一致 → ⚠️ |
| **D4-Q7** 状态跃迁条件 | 在状态机代码中查找 `transition`、`from-to` 配置 | 命中代码有但 PRD 未写跃迁 → 🟡 |

---

## D5 功能规格与字段定义 - 代码佐证规则（**重点维度**）

| 问题 ID | 代码扫描动作 | 佐证判定 |
|---|---|---|
| **D5-Q2** 字段含义 | 抽 PRD 字段名 → grep model/entity/DTO/SQL；对比代码注释、字段定义 | 字段在代码已存在且有注释 → ✅；存在但 PRD 描述与字段类型/语义冲突 → ⚠️ |
| **D5-Q3** 字段来源 | 同上；同时检查 mapper xml / repository / service 中该字段的查询来源 | 代码已有清晰来源链路 → 🟡 (PRD 应补) |
| **D5-Q4** 字段类型/枚举 | 对比 PRD 字段类型与代码字段类型；对比 PRD 枚举值与代码 enum 取值 | 类型/枚举值不一致 → **⚠️ 强制 P0** |
| **D5-Q5** 必填/选填 | 检查 `@NotNull`、`required`、DDL 中的 `NOT NULL`，对比 PRD 描述 | 不一致 → ⚠️ |
| **D5-Q11** 字段命名规范 | 抽工程现有字段命名风格（驼峰/下划线/前缀），对比 PRD 命名 | PRD 命名风格不符现有 → 🟡 (PRD 应对齐) |
| **D5-Q12** 字段长度约束 | 检查 DDL/Bean Validation 中的 length / max / min | 代码有约束但 PRD 未写 → 🟡 |
| **D5-Q13** 对内/对外字段 | 检查 controller DTO 与内部 model 的字段差异 | 代码已分层但 PRD 混淆 → 🟡 |

> 本维度是**代码佐证收益最大的维度**：金融账务类需求 80% 的字段都可在 model / mapper xml 中找到对照，能大幅提高评估精度。

---

## D6 校验规则与异常处理 - 代码佐证规则

| 问题 ID | 代码扫描动作 | 佐证判定 |
|---|---|---|
| **D6-Q1** 业务校验规则 | grep `@Valid`、`@Validated`、`assert`、自定义异常类、`if (...) throw` 模式 | 代码已有但 PRD 未列 → 🟡 |
| **D6-Q2** 错误提示文案 | grep `throw new BusinessException("...")`、message constants、i18n 文件 | 代码已有完整文案 → 🟡 (PRD 应补)；PRD 文案与代码不一致 → ⚠️ |
| **D6-Q3** 前后端校验分层 | 检查前端表单 validator 与后端 validator 是否一致 | 不一致 → ⚠️ |
| **D6-Q4** 边界场景 | grep 单元测试中的边界用例（empty、null、max、duplicated） | 代码测试已覆盖 → 信息项展示 |
| **D6-Q11** 校验留痕 | grep `audit_log`、`@AuditLog`、操作日志表 | 代码已有但 PRD 未提 → 🟡 |

---

## D7 数据与历史兼容 - 代码佐证规则（**重点维度**）

| 问题 ID | 代码扫描动作 | 佐证判定 |
|---|---|---|
| **D7-Q1** 历史数据处理 | 查找 `migration/`、`flyway/`、`liquibase/`、`*.sql`、`scripts/migrate*` | 代码已有迁移脚本 → 🟡 (PRD 应描述策略)；脚本逻辑与 PRD 描述冲突 → ⚠️ |
| **D7-Q2** 迁移执行方式 | 检查迁移脚本的执行方式（自动/手动 / job） | 不一致 → ⚠️ |
| **D7-Q3** 存量增量差异 | 检查代码中是否有 `historical` / `legacy` / `if (createTime < X)` 之类分支 | 代码已分支但 PRD 未区分 → 🟡 |
| **D7-Q6** 灰度策略 | grep `feature flag`、灰度配置、`@ConditionalOnProperty` | 代码已有但 PRD 未提 → 🟡 |
| **D7-Q7** 回滚方案 | 查找迁移脚本的 `down()` / `rollback` 部分 | 代码有 → 🟡；代码无 → ⚪ 双缺 |
| **D7-Q9** 时间分界点 | grep 代码中的硬编码日期 / 配置项 | 代码有 → 🟡 |

---

## D8 系统集成与上下游 - 代码佐证规则（**重点维度**）

| 问题 ID | 代码扫描动作 | 佐证判定 |
|---|---|---|
| **D8-Q1** 上下游系统列表 | 检查 `application*.yml` 中的下游 URL/MQ topic、Feign client、RPC 引用 | 代码引用的下游 PRD 未列 → ⚠️ (遗漏)；PRD 列了代码无 → 🔵 (待实现) |
| **D8-Q3** 接口字段定义 | 查找 controller / RPC interface / proto / openapi 文件 | 接口已存在且字段一致 → ✅；字段不一致 → ⚠️ |
| **D8-Q4** 交互方式 | 区分 REST controller / MQ listener / scheduled job / Feign | 不一致 → ⚠️ |
| **D8-Q5** 入参出参字段 | 对比 controller 方法签名 / proto / openapi 定义 | 不一致 → ⚠️；缺字段 → 🟡 |
| **D8-Q6** 消息幂等 | grep `idempotent`、`@Idempotent`、唯一键约束、消息去重表 | 代码已有 → 🟡；MQ 但代码无 → 🔵 风险提示 |
| **D8-Q8** 新增 vs 改造 | 通过 git blame（`code_scan_depth=deep` 时）判断接口新旧 | 仅信息展示 |

---

## D9 非功能性需求 - 代码佐证规则

| 问题 ID | 代码扫描动作 | 佐证判定 |
|---|---|---|
| **D9-Q2** 权限控制 | grep `@PreAuthorize`、`@RolesAllowed`、`hasRole`、`shiro` 注解 | 代码已有但 PRD 未提 → 🟡 |
| **D9-Q6** 操作日志 | grep `audit`、`OperationLog`、`@Audit` | 代码已有但 PRD 未提 → 🟡 |
| **D9-Q7** 敏感信息处理 | grep `@Sensitive`、`encrypt`、`mask`、脱敏工具类 | 代码已有但 PRD 未提 → 🟡 |
| **D9-Q8** 并发控制 | grep `@Lock`、`synchronized`、`Redisson`、唯一索引、乐观锁 `@Version` | 代码已有但 PRD 未提 → 🟡 |
| **D9-Q9** 监控告警 | grep `Metrics`、`@Counted`、`Prometheus`、`alert*` 配置 | 代码已有但 PRD 未提 → 🟡 |
| **D9-Q11** 可观测性 | 同上 | 同上 |

---

## D10 可测试性与验收标准 - 代码佐证规则

| 问题 ID | 代码扫描动作 | 佐证判定 |
|---|---|---|
| **D10-Q1** 场景可独立验证 | 查找现有 unit / integration tests 中类似场景 | 代码已有覆盖 → ✅ 信息展示 |
| **D10-Q4** 测试数据样例 | 查找 `src/test/resources/`、`*.json`、`*.yml` 测试 fixture | 代码已有 → 🟡 |
| **D10-Q5** 测试环境 | 检查 `docker-compose.test.yml`、`testcontainers` 等 | 代码已有 → 信息展示 |
| **D10-Q9** 边界用例 | 同 D6-Q4 | 信息展示 |

---

## 扫描预算分配建议

> 当 `code_scan_depth=shallow`（默认），按以下顺序消耗预算（read 50 文件 / grep 200 次）：

| 优先级 | 扫描内容 | 预算分配 |
|---|---|---|
| 1 (必扫) | README + CHANGELOG | 2 read |
| 2 (必扫) | DB migration + schema + proto + openapi | 5–10 read |
| 3 (必扫) | model / entity / DTO 顶层 | 10–15 read |
| 4 (按需) | 关键 controller / service interface | 10–15 read |
| 5 (按需) | mapper xml / repository | 5–10 read |
| 6 (按需) | 测试 fixture | 3–5 read |
| - | grep 关键词查询（贯穿全程） | 200 次 |

预算耗尽后立刻停止并出报告，**不要为了完整性无限扫描**。

---

## ⚠️ 反模式（禁止行为）

1. ❌ 不要把一个 grep 失败就当"代码无该实现"。先尝试 2-3 种命名变体（驼峰/下划线/缩写）再判定。
2. ❌ 不要在没有代码佐证时凭空补"代码佐证"列内容（必须 ❓ 未扫描）。
3. ❌ 不要因为代码完整就把所有 PRD 缺口都降级；元数据类（D1）、价值类（D3）问题代码无法佐证。
4. ❌ 不要修改任何代码文件 / 执行任何代码。
5. ❌ 不要读 `.env`、密钥、`secrets/` 等敏感文件。
