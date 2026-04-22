# prd-quality-review

> Cursor / Claude 用的 Agent Skill：对一份 PRD 进行结构化打分与问题清单输出，**10 个评估维度、123 个问题、P0/P1/P2 三级严重度**，并可选**结合代码工程上下文**做 PRD ↔ 代码对齐评审。

---

## 这是什么

这是一个 **PRD 质量评估 Skill**，做三件事：

1. **诊断** —— 按 10 个维度逐一扫描你的 PRD，找出缺失项和风险点。
2. **打分 + 改进建议** —— 给出加权总分（0-100）、等级（A-F）、按严重度排序的问题清单，以及 Top N 改进建议。
3. **代码交叉验证（可选）** —— 自动读取代码工程（默认当前工作目录），交叉检查字段命名、数据模型、现有接口、校验逻辑、迁移脚本等，输出 **PRD-代码冲突清单**与**待研发实现项**。

设计目标：让 PRD 的质量门禁可以像 code review 一样**结构化、可追溯、可量化**，并且**贴合实际代码工程**而不是空对空评审。

---

## 为什么需要它

- **PRD 质量参差不齐**：同一团队、同一产品同学写出的 PRD，关键章节缺失程度差异巨大；
- **凭"感觉"评审**：传统评审主要靠经验，没有量化标准，难以沉淀；
- **常见雷区固定**：金融/账务类 PRD 反复在「字段来源、校验提示文案、历史数据兼容、灰度回滚」上踩坑；
- **新人难快速上手**：新产品同学不知道一份"合格 PRD"长什么样。

本 Skill 把团队 5 份**真实优秀 PRD**（债转/账务/资管领域）的共同特征提炼成 122 道评估题 + 12 个 pattern 模板，让评审有据可依。

---

## 快速开始

### 1. 让 Cursor / Claude 识别这个 Skill

确保 Skill 能被 Cursor 自动加载，常见做法二选一：

**方式 A：软链到 Cursor skills 目录**

```bash
ln -s ~/code/github/prd-quality-review ~/.cursor/skills/prd-quality-review
```

**方式 B：项目内引用**（仅当你在某个项目里用）

```bash
mkdir -p .cursor/skills
ln -s ~/code/github/prd-quality-review .cursor/skills/prd-quality-review
```

### 2. 触发评估

#### 模式 A：纯 PRD 静态评审（无代码工程时）

在 Cursor / Claude 对话中说：

```
帮我评估一下 ~/Desktop/prd资料/【PRD】xxxxx.md
```

或：

```
PRD 体检 / PRD 打分 / PRD 评审 / PRD review / 评估 PRD 质量 / 给这份需求打分
```

#### 模式 B：PRD ↔ 代码对齐评审（默认行为）

在**代码工程目录下**触发：

```
cd ~/code/github/oac-server
# 在 Cursor 对话里：
帮我评估一下 /path/to/【PRD】xxx.md
```

Skill 会自动用当前工作目录作为 `code_root`，识别为代码工程后启动 CODE_SCAN 阶段，做 PRD ↔ 代码对齐评审。

#### 模式 C：显式指定代码工程路径

```
评估这份 PRD：xxx.md，代码工程在 ~/code/github/oac-server
```

或限定扫描范围：

```
评估 xxx.md，代码工程在 ~/code/github/oac-server，只看 account 模块
```

#### 模式 D：跳过代码扫描

```
评估 xxx.md，跳过代码扫描
```

#### 全部支持的触发关键词

| 类型 | 关键词 |
|---|---|
| **基础评估** | PRD 评估 / PRD 评审 / PRD 质量 / PRD 打分 / PRD 检查 / PRD 审核 / PRD 验收 / PRD 体检 / PRD 复盘 / PRD review / PRD 质量门禁 / PRD checklist |
| **口语触发** | 评估这份 PRD / 评估这份需求 / 评分需求 / 给 PRD 打个分 / 看下这个 PRD 写得怎么样 / 帮我评一下这个需求 / 这份 PRD 怎么样 / 这个需求写得清楚吗 |
| **需求口径** | 需求质量 / 需求评审 / 需求审核 / 需求体检 / 需求复盘 / 需求 review / 产品文档评审 |
| **代码对齐** | PRD vs codebase / PRD 与代码对齐 / 需求对照代码评审 / PRD 代码一致性 / 评估代码与 PRD 一致性 / 结合代码评估 PRD / 看 PRD 和代码是否一致 |
| **直接 ID** | prd-quality-review / quality gate / rubric |

### 3. 查看输出

Skill 会在 PRD 同目录生成 `prd-review.zh-CN.md`，并在对话中给出精简版报告（总分 + 等级 + Top 5 + P0 风险 + PRD-代码冲突）。

---

## 目录结构

```
prd-quality-review/
├── SKILL.md                       # 主入口：触发条件、工作流、评分规则、严重度定义、代码上下文
├── README.md                      # 本文件
├── checklists/                    # 10 个评估维度的详细问题清单 + 1 个代码佐证叠加
│   ├── 01-metadata.md                 # 基本信息与变更管理（11 题）
│   ├── 02-background.md               # 需求背景（11 题）
│   ├── 03-value.md                    # 需求价值（11 题）
│   ├── 04-process.md                  # 业务流程与场景（12 题）
│   ├── 05-functional-spec.md          # 功能规格与字段定义（13 题）
│   ├── 06-validation.md               # 校验规则与异常处理（13 题）
│   ├── 07-data-history.md             # 数据与历史兼容（13 题）
│   ├── 08-integration.md              # 系统集成与上下游（13 题）
│   ├── 09-non-functional.md           # 非功能性需求（13 题）
│   ├── 10-testability.md              # 可测试性与验收标准（13 题）
│   └── code-context-overlay.md        # 代码佐证叠加规则（每维度的代码扫描动作 + 调级规则）
├── templates/
│   └── report-template.md         # 评估报告输出模板（含 PRD-代码冲突清单与代码工程信息）
└── examples/
    └── good-prd-patterns.md       # 12 个优秀 PRD pattern（P-01 ~ P-12）
```

合计 **123 道评估问题** + **代码佐证叠加规则**，覆盖 PRD 全生命周期。

---

## 评分体系

### 维度权重

| # | 维度 | 权重 |
|---|---|---|
| 1 | 基本信息与变更管理 | 8% |
| 2 | 需求背景 | 10% |
| 3 | 需求价值 | 8% |
| 4 | 业务流程与场景 | 12% |
| 5 | 功能规格与字段定义 | **14%** |
| 6 | 校验规则与异常处理 | 12% |
| 7 | 数据与历史兼容 | 10% |
| 8 | 系统集成与上下游 | 10% |
| 9 | 非功能性需求 | 8% |
| 10 | 可测试性与验收标准 | 8% |

> 维度 4/5/6 权重最高，因为业务流程、字段规格、校验规则是 PRD 决定研发成败的核心。

### 严重度

| 等级 | 含义 | 单题扣分 |
|---|---|---|
| **P0** | 致命/阻断（无法开发或引发资金/合规事故） | -10 |
| **P1** | 重要/返工（理解偏差或体验问题） | -5 |
| **P2** | 建议/优化（专业度/可维护性） | -2 |

### 等级映射

| 总分 | 等级 | 含义 |
|---|---|---|
| ≥ 90 | **A** | 优秀，可直接进入研发 |
| 80-89 | **B** | 良好，补齐少量 P1/P2 |
| 70-79 | **C** | 合格，需补齐 P0/P1 后进入研发 |
| 60-69 | **D** | 待改进，建议返工后再评审 |
| < 60 | **F** | 不通过，必须重写关键章节 |

`strict_mode=on` 时，任意 P0 命中即整体降为 F（用于上线前最终质量门禁）。

---

## 设计参考

本 Skill 的评估问题来源于三部分：

1. **业界 PRD 评估最佳实践**：完整性（Completeness）、清晰性（Clarity）、一致性（Consistency）、可测试性（Testability）、可行性（Feasibility）、可追溯性（Traceability）、场景覆盖、风险识别、变更控制等。
2. **真实优秀 PRD 共性**：从团队 5 份金融账务领域 PRD（债转/资管/账务）中提炼的 12 个高质量 pattern，参见 [examples/good-prd-patterns.md](examples/good-prd-patterns.md)。
3. **代码工程对齐**：通过扫描 model / DTO / mapper / migration / config 等关键代码区域，识别 PRD 与代码的不一致，详见 [checklists/code-context-overlay.md](checklists/code-context-overlay.md)。

## 代码扫描原则

| 项 | 说明 |
|---|---|
| 默认行为 | 自动以当前工作目录为 `code_root`，识别失败则回退到纯 PRD 静态评审 |
| 默认深度 | `shallow`（仅扫描 model / DTO / controller / mapper / migration / config）|
| 安全约束 | 只读、不执行代码、不读 `.env`/secret/key 文件 |
| 预算上限 | 单次评估 ≤ 50 个文件 read + ≤ 200 次 grep；超出请聚焦范围 |
| 命中机制 | 关键词驱动（PRD 抽出业务术语 → grep 代码命名变体），优先 schema/model/proto |
| 调级规则 | PRD 缺失但代码已实现 → 降一级；PRD 与代码冲突 → 强制升 P0 |

---

## 与其他 Skill 的边界

| 用户意图 | 应触发的 skill |
|---|---|
| 评估/评审现有 PRD 质量 | **prd-quality-review（本 Skill）** |
| 从 PRD 生成系分文档 | `prd-to-design` |
| 从 PRD 直接生成代码 | `prd-to-code` |
| 从已有系分文档生成代码 | `design-to-code` |

---

## 自定义与扩展

### 新增问题
在对应 `checklists/0X-*.md` 末尾追加 `Qn` 编号，按表格格式填入：问题、判定标准、命中证据示例。

### 调整权重
修改 `SKILL.md`「评估维度」表中的「权重」列，确保合计 = 100%。

### 沉淀新 pattern
从评估过程中发现的优秀写法，按 P-13、P-14 ... 顺序追加到 `examples/good-prd-patterns.md`。

### 领域调权（建议）
- **金融/账务类**：D6（校验）、D7（历史兼容）、D9（非功能/资金安全）权重 +2-4%
- **纯前端 / SaaS 类**：D4（流程）、D5（字段）、D9（性能/UX）权重 +2-4%
- **内部工具类**：D3（价值）、D9（合规）权重 -2%

---

## 变更记录

| 版本 | 日期 | 变更 |
|---|---|---|
| v1.0 | 2026-04-22 | 初版：10 维度 + 123 问题 + 12 pattern + 报告模板 |
| v1.1 | 2026-04-22 | 新增代码上下文扫描：`code_root` 参数（默认 cwd）+ CODE_SCAN 阶段 + 代码佐证叠加规则 + PRD-代码冲突检出；扩充自动触发关键词 |

---

## License

Internal use only. Patterns 与案例均来自团队真实 PRD，请勿外传。
