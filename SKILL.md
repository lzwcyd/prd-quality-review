---
name: prd-quality-review
description: >-
  Review and score PRDs across weighted quality dimensions with P0/P1/P2
  findings, optionally using the target codebase to check field names, data
  models, APIs, validation, schema compatibility, integrations, and migration
  evidence. Use when the user asks for PRD 评估、评审、质量检查、打分、审核、验收、
  体检、复盘、PRD review、checklist、质量门禁, asks whether a requirement is clear,
  requests feedback/scoring/audit for a PRD file, or wants PRD vs codebase /
  PRD 与代码对齐 / 需求对照代码评审. Do not use to generate a PRD, create 系分文档,
  or write code; ask to choose 评审打分, 生成系分, or 直接写代码 when intent is
  ambiguous.
---

# PRD Quality Review（PRD 质量评审 Skill）

**Version:** 1.1 · **Logical id:** `prd_quality_review`

> 基于优秀 PRD 案例（5 份债转/账务/资管领域 PRD）+ 业界 PRD 评估最佳实践，对一份 PRD 从 10 个维度进行结构化打分与问题清单输出。每个维度内置 ≥10 个评估问题，按 **P0（致命）/ P1（重要）/ P2（建议）** 三级严重度标注。
>
> **v1.1 新增**：可选结合代码工程上下文（默认读取当前工作目录），交叉验证 PRD 字段、接口、校验逻辑、迁移脚本等是否与现有代码对齐，输出「PRD-代码冲突清单」与「待研发实现项」。

---

## Input

### 必填

- `prd`：PRD 正文文本 或 PRD 文件路径（`.md` / `.txt`，`.docx` 需先转 md/纯文本）

### 可选 - 评估行为

- `output_path`：评估报告输出路径，默认写入 PRD 所在目录的 `prd-review.<lang>.md`
- `lang`：`zh-CN` | `en`（默认按用户本轮语言）
- `dimensions`：限定只评某些维度，逗号分隔（如 `1,2,6,7`）；默认全部 10 个维度
- `strict_mode`：`on` | `off`（默认 `off`）。`on` 时所有 P0 问题命中即整体不通过
- `reference_dir`：优秀 PRD 参考样本目录（默认 `examples/good-prd-patterns.md`）
- `domain_hint`：领域提示（如 `账务/资金/风控/营销`），帮助选择更贴合的对照案例

### 可选 - 代码工程上下文（用于 PRD ↔ 代码对齐评估）

- `code_root`：代码工程根目录路径
  - **默认值**：当前工作目录（cwd）
  - 用途：让评估不仅看 PRD 文本，还结合现有代码进行交叉验证（字段是否已存在 / 接口是否已实现 / 校验是否已覆盖等）
  - 显式传入 `code_root=none` 可跳过代码扫描，回退到纯 PRD 静态评审
- `code_focus_paths`：限定扫描的子目录或文件列表（逗号分隔，相对 `code_root`）
  - 例：`src/main/java/com/foo/account/,src/main/resources/mapper/`
  - 默认：自动识别核心代码目录（见下方「代码工程识别」）
- `code_scan_depth`：扫描深度
  - `shallow`（默认）：仅扫描 model / DTO / controller / service 层、迁移脚本、关键配置；3 分钟以内
  - `deep`：进一步扫描业务逻辑实现、测试用例；5–10 分钟
  - `index`：仅建立文件索引不读代码内容；30 秒以内
- `code_languages`：限定语言（如 `java,sql,xml`），默认按工程自动识别
- `code_exclude`：排除路径模式，默认 `node_modules/,vendor/,target/,build/,dist/,.git/,*.min.js,*.lock`

## Output

1. **评估报告**（Markdown）：含总分、等级、维度分数表、按 P0→P1→P2 排序的问题清单、**代码佐证清单**（命中/反驳/缺口）、Top-N 改进建议
2. **可选 JSON 摘要**：`prd-review.<lang>.json`（机器可读评分结构，便于聚合统计）
3. **代码扫描索引**（可选）：`prd-review.code-index.json`，记录扫描过的关键代码符号与文件，便于二次评估复用

---

## Language mode

1. `lang=zh-CN`：全量中文输出。
2. `lang=en`：全量英文输出。
3. 未指定 `lang`：按用户本轮语言自动识别，与 PRD 主体语言对齐。

---

## PRD 输入解析

1. `prd` 像路径且文件存在时按文件读取，否则按内联文本处理。
2. 优先支持 `.md`、`.txt`、可读文本文件。
3. 对 `.doc` / `.docx`：若无法可靠解析，**不得臆造内容**，要求用户转 `.md`/纯文本或直接粘贴。
4. 若 `prd` 未显式提供：
   - 自动在当前工作目录搜索 PRD 候选（如 `*PRD*.md`、`【PRD】*.md`）；
   - 仅 1 个候选时自动采用；
   - 多个候选时要求用户选择（编号列表）；
   - 无候选时要求用户提供 PRD。
5. 进入评估前必须有可解析 PRD；若 PRD 内容明显残缺（< 200 字），先提示用户确认是否继续。

---

## 代码工程上下文（Code Context）

> 当 `code_root` 指向有效代码工程时，本 skill 会读取关键代码以**反向佐证**或**反驳**PRD 的描述，让评估从「文档静态评审」升级到「文档 ↔ 代码对齐评审」。

### 1. 代码工程识别

按以下顺序判定 `code_root` 是否为代码工程，**至少命中 1 项即视为有效**：

| 工程类型 | 标志文件 / 目录 |
|---|---|
| Java / Maven | `pom.xml` |
| Java / Gradle | `build.gradle`, `build.gradle.kts`, `settings.gradle` |
| Node.js / TypeScript | `package.json`, `tsconfig.json` |
| Python | `pyproject.toml`, `requirements.txt`, `setup.py`, `Pipfile` |
| Go | `go.mod` |
| Rust | `Cargo.toml` |
| .NET | `*.csproj`, `*.sln` |
| 通用 | `.git/` 目录 + 任意源代码文件 |

**未识别为代码工程时**：在评估报告中明确标注「⚠️ 未识别到代码工程，本次为纯 PRD 静态评审」，跳过 CODE_SCAN 阶段，**不报错、不阻塞流程**。

### 2. 默认扫描范围（`code_focus_paths` 未指定时）

按工程类型自动选择「最有信号」的目录与文件，避免读全量源码：

| 类型 | 自动扫描的子集 |
|---|---|
| Java | `**/model/**`、`**/entity/**`、`**/dto/**`、`**/po/**`、`**/vo/**`、`**/controller/**`、`**/service/**` 的接口层、`**/mapper/**`、`**/resources/**/*.xml`（MyBatis）、`**/resources/db/migration/**`（Flyway/Liquibase）、`**/application*.yml`、`README.md` |
| Node/TS | `src/**/*.ts`（types/、models/、controllers/、routes/、services/ 接口）、`prisma/schema.prisma`、`migrations/`、`package.json`、`README.md` |
| Python | `**/models.py`、`**/schemas.py`、`**/serializers.py`、`**/views.py`、`**/urls.py`、`**/migrations/**`、`README.md` |
| Go | `**/model/**`、`**/handler/**`、`**/service/**`、`go.mod`、`README.md` |
| 通用补充 | 工程根 `README*`、`CHANGELOG*`、`docs/`、`schema*.sql`、`*.proto` |

### 3. 扫描方法（不耗 token 的最佳实践）

1. **优先 Glob + Grep**，不要全量 Read 大目录。
2. **关键词驱动扫描**：从 PRD 中提取业务关键词（中英文术语、字段名、枚举值、接口路径），在代码中 grep。
   - 示例：PRD 提到「资产包编号」「封包时间」→ grep `assetPackage`、`packageTime`、`asset_package_no`、`seal_time` 等可能命名变体。
3. **限定扫描预算**：单次扫描不超过 50 个文件 read、200 个 grep；超出后请求用户聚焦范围。
4. **优先读 schema 与 model**：数据库迁移脚本、ORM model、API DTO、proto 定义比业务实现更"性价比高"。
5. **README/CHANGELOG 必读**：通常包含工程定位、模块划分、关键改动史，是理解工程上下文的最快路径。

### 4. 安全约束（必须遵守）

- ❌ **不修改任何代码文件**（read-only 评估）
- ❌ **不执行任何代码或脚本**（不跑测试、不启服务）
- ❌ **不读密钥/敏感文件**：跳过 `.env`、`*.pem`、`*.key`、`secrets/`、`credentials*`
- ❌ **不下载远程依赖**：仅读已有文件
- ✅ 仅静态读取代码文本，所有"代码佐证"必须可被人复查（注明文件路径 + 行号）

### 5. 代码佐证如何影响评分

代码佐证规则详见 [checklists/code-context-overlay.md](checklists/code-context-overlay.md)。核心规则：

| 代码佐证结果 | 对评估的影响 |
|---|---|
| **PRD 缺失 + 代码已实现** | 该问题降级（如 P0→P1 或 P1→P2），但仍要在报告中提醒 PRD 应补充描述以对齐 |
| **PRD 描述 + 代码已实现且一致** | 不命中（已满足），并在报告中加正面证据 |
| **PRD 描述 + 代码已实现但冲突** | 升级为 **P0**（无论原本严重度），单独高亮在「⚠️ PRD-代码冲突」章节 |
| **PRD 描述 + 代码未实现** | 不影响 PRD 本身评分，但作为「研发风险」单列（提示研发工作量） |
| **PRD 缺失 + 代码也无** | 按原有 PRD 评分规则正常命中 |

---

## 评估维度（10 个）

| # | 维度 | 文件 | 权重 |
|---|---|---|---|
| 1 | 基本信息与变更管理 | [checklists/01-metadata.md](checklists/01-metadata.md) | 8% |
| 2 | 需求背景 | [checklists/02-background.md](checklists/02-background.md) | 10% |
| 3 | 需求价值 | [checklists/03-value.md](checklists/03-value.md) | 8% |
| 4 | 业务流程与场景 | [checklists/04-process.md](checklists/04-process.md) | 12% |
| 5 | 功能规格与字段定义 | [checklists/05-functional-spec.md](checklists/05-functional-spec.md) | 14% |
| 6 | 校验规则与异常处理 | [checklists/06-validation.md](checklists/06-validation.md) | 12% |
| 7 | 数据与历史兼容 | [checklists/07-data-history.md](checklists/07-data-history.md) | 10% |
| 8 | 系统集成与上下游 | [checklists/08-integration.md](checklists/08-integration.md) | 10% |
| 9 | 非功能性需求 | [checklists/09-non-functional.md](checklists/09-non-functional.md) | 8% |
| 10 | 可测试性与验收标准 | [checklists/10-testability.md](checklists/10-testability.md) | 8% |

权重合计 = 100%。维度 4/5/6 权重最高，因为业务流程、字段规格、校验规则是 PRD 决定研发成败的核心。

---

## 严重程度定义（P0 / P1 / P2）

| 等级 | 定义 | 触发条件 | 评分影响 |
|---|---|---|---|
| **P0（致命/阻断）** | 缺失会导致**无法开发/无法上线**或引发**资金/合规/数据安全事故** | 必备字段未定义、校验规则缺失、历史数据无方案、上下游接口字段不明 | 单题命中扣 10 分；`strict_mode=on` 时整体降为「不通过」 |
| **P1（重要/返工）** | 缺失会导致**理解偏差、研发返工**或上线后**重要功能体验问题** | 错误提示文案缺失、流程图缺失、状态机不全、灰度方案缺失 | 单题命中扣 5 分 |
| **P2（建议/优化）** | 不影响主流程，但**降低 PRD 专业度/可维护性** | 操作记录不全、命名不规范、缺名词解释、缺指标度量 | 单题命中扣 2 分 |

> 单维度得分下限为 0（不会出现负分）。

---

## 评分与等级

### 维度评分公式

```
单维度得分 = max(0, 100 - Σ(P0命中数×10 + P1命中数×5 + P2命中数×2))
加权总分 = Σ(单维度得分 × 维度权重)
```

### 等级映射

| 总分区间 | 等级 | 含义 |
|---|---|---|
| ≥ 90 | **A（优秀）** | 可直接进入研发，几乎无需补齐 |
| 80 – 89 | **B（良好）** | 主体可用，需补齐少量 P1/P2 项 |
| 70 – 79 | **C（合格）** | 需补齐部分 P0/P1 项后才可进入研发 |
| 60 – 69 | **D（待改进）** | 存在多个关键缺口，建议返工后再评审 |
| < 60 | **F（不通过）** | 缺口过多，必须重写关键章节 |

### `strict_mode=on` 附加规则

- 任意 P0 问题命中 → 总评直接降为 **F（不通过）**
- 适用于「上线前最终质量门禁」场景

---

## 工作流程（6 阶段）

### Phase 1: INTAKE（接收与解析）
1. 解析 `prd` 输入（路径 / 内联文本）。
2. 识别 PRD 语言、长度、章节结构（H1/H2/H3）。
3. 解析 `code_root`（默认 cwd）；若用户显式 `code_root=none` 则跳过 Phase 3 (CODE_SCAN)。
4. 输出一行摘要给用户：`已识别 PRD：<标题>，约 <X> 字，<Y> 个二级章节；代码工程根目录：<code_root>`，请求确认开始。

### Phase 2: SCAN（PRD 章节扫描与关键词抽取）
1. 按维度对应的关键词扫描 PRD 章节，建立「PRD 章节 ↔ 评估维度」的初步映射。
2. 关键词参考（节选）：
   - 维度 1：「云效 / 操作记录 / 操作时间 / 操作人 / 关联需求 / BRD」
   - 维度 2：「需求背景 / 现状 / 痛点 / 业务流程图」
   - 维度 4：「mermaid / 时序图 / 状态 / 流程」
   - 维度 6：「校验 / 提示 / 报错 / 不允许 / 弹窗」
   - 维度 7：「历史 / 存量 / 增量 / 灰度 / 补齐 / 刷数」
   - 维度 8：「接口 / 字段 / 上下游 / 监听 / 消息」
3. **抽取 PRD 中的业务关键词集合**（中英文术语、字段名、枚举值、接口路径、表名），写入临时索引供 Phase 3 使用。
4. 未识别到对应章节的维度，初步记为「该维度高风险，需重点核查」。

### Phase 3: CODE_SCAN（代码工程上下文扫描，可选）
> 仅当 `code_root` 指向有效代码工程时执行；否则跳过并在报告标注「⚠️ 未识别到代码工程」。

1. **工程识别**：按「代码工程上下文 § 1」判定 `code_root` 是否有效。
2. **范围确定**：按「§ 2」选择默认扫描子集，或使用用户传入的 `code_focus_paths`。
3. **README 必读**：先读 `code_root/README*` 获取工程定位与模块划分。
4. **关键词驱动 grep**：用 Phase 2 抽取的业务关键词在代码中检索：
   - 字段名 → model / DTO / table column
   - 接口路径 → controller / route
   - 枚举值 → enum / constants
   - 表名 → migration scripts / SQL / mapper xml
5. **schema/model 优先**：读取数据库迁移脚本、ORM model、proto 文件，建立「PRD 字段 ↔ 代码字段」对照表。
6. **校验逻辑扫描**：搜索 `@Valid`、`assert`、`throw` 异常类、`if (...) error` 等模式，定位现有校验。
7. **预算控制**：累计文件 read ≤ 50、grep 调用 ≤ 200；超出时停止并告知用户「代码扫描预算已用完，建议传入 `code_focus_paths` 聚焦范围」。
8. **输出**：构建「代码佐证索引」，每条形如：
   - `{prd_concept: "资产包编号", code_evidence: "src/.../AssetPackage.java:L15 字段 packageNo", match: "命名一致|命名不一致|未找到"}`

### Phase 4: EVALUATE（逐维度评估，叠加代码佐证）
1. **顺序依次** 读取 `checklists/01-metadata.md` … `checklists/10-testability.md`。
2. 同时读取 [checklists/code-context-overlay.md](checklists/code-context-overlay.md)，对每个维度应用「代码佐证叠加规则」。
3. 每个维度按「问题列表」逐题判定，每题输出：
   - `问题 ID`（如 `D6-Q3`）
   - `严重度`（P0/P1/P2，可能被代码佐证调级）
   - `判定结果`（命中 / 未命中 / N/A）
   - `PRD 证据`（PRD 中相关原文片段，≤50 字 或行号）
   - **`代码佐证`**（可选；含文件路径 + 行号 + 一句话结论：✅ 已实现一致 / ⚠️ 已实现但冲突 / ⚪ 代码无 / ❌ 未扫描）
   - `改进建议`（一句话，可执行）
   - `优秀案例对照`（可选，引用 `examples/good-prd-patterns.md` 中的 pattern 编号）
4. 计算单维度得分（代码佐证按规则可调整严重度，再算分）。

### Phase 5: REPORT（汇总报告）
1. 加权计算总分与等级。
2. 按 [templates/report-template.md](templates/report-template.md) 生成结构化报告。
3. 报告必须包含：
   - 总分 + 等级 + 一句话总评
   - 维度分数表（10 行，含权重、原始分、加权分）
   - 问题清单（按 P0→P1→P2→维度顺序排序），每条附「代码佐证」列
   - **Top 5 改进建议**（按"修复后分数提升幅度"降序）
   - **风险提示**（资金/合规/安全相关 P0 单独高亮）
   - **⚠️ PRD-代码冲突清单**（若有 CODE_SCAN 且检出冲突）
   - **代码工程信息**（root path、识别的工程类型、扫描文件数、关键命中数）

### Phase 6: DELIVER（交付）
1. 写入报告文件到 `output_path`（默认与 PRD 同目录）。
2. 在对话中输出「精简版报告」（总分 + 等级 + Top 5 + 风险提示 + PRD-代码冲突），完整版指向报告文件。
3. 主动询问用户是否需要：
   - 针对某个 P0/P1 问题给出「示例修订片段」
   - 针对 PRD-代码冲突给出对齐建议（修 PRD 还是改代码）
   - 输出 JSON 摘要用于团队聚合
   - 输出代码扫描索引 `prd-review.code-index.json` 便于二次评估

---

## N/A 判定原则

不是所有问题都适用所有 PRD。判定 N/A 的标准：

1. **领域不适用**：例如纯前端样式 PRD 不评「资金安全」相关问题。
2. **范围明确不涉及**：PRD 已显式说明"本期不做 XX"，则相关问题判 N/A。
3. **N/A 不计入分母**：单维度评分公式中，N/A 题既不命中扣分也不参与上限计算（不影响 100 分基线）。
4. **N/A 必须给出理由**：报告中 N/A 题需注明「N/A 理由」，避免漏评。

---

## 评估准则（避免误判）

1. **看证据，不臆测**：每个问题的判定必须能在 PRD 原文（或代码原文）中找到证据。
2. **优先级让位结构**：先判 P0，再判 P1，最后判 P2，不允许"漏 P0 但提 P2"。
3. **重复内容只扣一次**：同一缺失被多个维度命中时，按主维度扣分，其他维度引用即可。
4. **领域适配**：金融类 PRD 默认提高「资金安全 / 合规 / 数据兼容」权重；通用 SaaS PRD 默认提高「用户体验 / 性能」权重。
5. **杜绝水分提示**：改进建议必须可执行（"建议补充 XX 字段及其来源"），禁止"建议加强描述"这类空话。
6. **代码佐证不可造假**：代码引用必须含具体文件路径 + 行号；找不到时一律标 ⚪「代码无」，**严禁臆造代码片段**。
7. **PRD-代码冲突优先**：若代码与 PRD 存在冲突（命名、字段类型、枚举值不一致），**无论 PRD 本题原本是什么严重度，统一升级为 P0** 并单独高亮。
8. **不替产品决策**：当 PRD 与代码不一致时，只指出冲突 + 提供"修 PRD" 与"改代码"两种方向的建议，**不替用户做选择**。

---

## 优秀 PRD 案例对照

参考 [examples/good-prd-patterns.md](examples/good-prd-patterns.md)，内含 12 个从真实优秀 PRD 中提炼的 pattern（如 P-04「Mermaid 时序图描绘多系统协作」、P-07「校验规则带具体提示文案」），评估时可在「改进建议」中引用 pattern ID，让产品同学有具体参照。

---

## 报告模板

详见 [templates/report-template.md](templates/report-template.md)。

---

## 与其他 skill 的边界

| 用户意图 | 应触发的 skill |
|---|---|
| 评估/评审现有 PRD 质量 | **本 skill：prd-quality-review** |
| 从 PRD 生成系分文档 | `prd-to-design` |
| 从 PRD 直接生成代码 | `prd-to-code` |
| 从已有系分文档生成代码 | `design-to-code` |
| 从零写一份 PRD | （非本 skill 范围） |

歧义时（"我有一份 PRD"），先问用户：**A=评审打分 / B=生成系分 / C=直接写代码**。

---

## 快速使用示例

### 示例 1：纯 PRD 静态评审（无代码工程）

**用户：** "帮我评一下 `~/Desktop/prd资料/【PRD】资产债转平台一期.md` 这份 PRD"

**Skill 行为：**
1. 解析 PRD 文件；判断 cwd（`~/Desktop/prd资料`）非代码工程 → 跳过 CODE_SCAN。
2. 输出："已识别 PRD：资产债转平台一期，约 6800 字，11 个二级章节；⚠️ 当前目录非代码工程，本次为纯 PRD 静态评审。"
3. 顺序读取 10 个 checklist 逐题判定。
4. 输出报告到 `~/Desktop/prd资料/prd-review.zh-CN.md`。
5. 对话中给出精简版：总分 87 / B（良好）/ Top 5 改进 / 1 个 P0 风险。

### 示例 2：PRD ↔ 代码工程对齐评审（默认使用 cwd）

**用户：** （在 `~/code/github/oac-server` 项目目录下）"用 prd-quality-review 评审这份 PRD：`/path/to/【PRD】xxx.md`"

**Skill 行为：**
1. `code_root` 默认取 cwd = `~/code/github/oac-server`，识别为 Java/Maven 工程（存在 pom.xml）。
2. 输出："已识别 PRD：xxx，约 4500 字；代码工程：Java/Maven，将扫描 model/entity/dto/controller/mapper 与 db/migration 子集。"
3. CODE_SCAN：从 PRD 抽取关键词（资产包编号、封包时间、债转模式等），grep 代码确认现有实现。
4. EVALUATE：每条问题叠加代码佐证。例如 D5-Q3「字段来源」原本判 P0 命中（PRD 未说），扫码后发现该字段已在 `AssetPackage.java:L23` 定义且与 PRD 一致 → 降级为 P2，并提示"建议 PRD 补一句'取自 oac.asset_package.package_no'对齐"。
5. 输出报告，包含 ⚠️ PRD-代码冲突章节（如发现 PRD 写"枚举：A/B/C" 但代码 enum 是 `A/B/C/D`，标 P0 升级冲突）。

### 示例 3：显式指定代码目录与聚焦范围

**用户：** "评估 `xxx.md`，代码工程在 `~/code/github/oac-server`，只看 `account` 模块"

**Skill 行为：**
1. `code_root=~/code/github/oac-server`，`code_focus_paths=src/main/java/com/foo/account/,src/main/resources/mapper/account/`
2. 仅在该范围内 grep / read，避免扫整个工程。
3. 其他流程同示例 2。

### 示例 4：明确跳过代码扫描

**用户：** "评估 `xxx.md`，跳过代码扫描"

**Skill 行为：** 设置 `code_root=none`，行为同示例 1（纯 PRD 静态评审）。

---

## 维护说明

- 新增问题：在对应 `checklists/0X-*.md` 末尾追加 `Qn` 编号，标注严重度与判定标准。
- 调整权重：修改本文件「评估维度」表中的「权重」列，确保合计 = 100%。
- 沉淀新 pattern：从优秀 PRD 中提炼后追加到 `examples/good-prd-patterns.md`。
- 版本更新：修改本文件顶部 `Version` 字段并在 `README.md` 的变更记录中追加。
