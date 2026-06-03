# novel-continuation — 小说续写技能

用于续写当年"太监"掉的小说。技能族由 1 个工程化入口 + 4 个独立子技能组成，覆盖导入 → 大纲 → 续写 → 评审的全流程。

> ⚠️ 不建议用于出版。如用于商业用途，请注明"来自于饭票.md"

---

## 核心能力

| 能力 | 说明 |
|------|------|
| **小说导入** | 自动分割章节、三遍读取法逆向工程、题材识别、生成约束文档体系 |
| **中断续写检测** | 按 `currentStep` 全局路由 + 子技能内部精细恢复，支持 `writing-paused` 状态恢复 |
| **文风模仿** | 分析参考文本词汇/句式/节奏/语调特征，保持风格一致（chapter-splitting 0-A 步骤4）|
| **约束文档双轨** | 必选 12 个（5 design + 7 truth）+ 按题材激活 + 运行时追加，三层分类 |
| **题材识别驱动 33 维审计** | 10 种 genre → 26 维度机械激活表，避免 AI 主观判断 |
| **5 项门 + 33 维拆分** | 每章跑 5 项门（轻量、必跑），全部完成后才跑 33 维（全局、按题材激活）|
| **Token 预算管理** | 长篇 20+ 章自动降级：完整 → 摘要 → 强制摘要三档阈值（60% / 85%）|
| **无中断写作** | 从写作到质量循环结束，零用户交互，自动逐章流转（4 个固定交互点）|
| **运行日志** | `meta/_run-log.jsonl` 记录 27 类事件，便于事后审计和断点恢复 |
| **可调参数** | 5 个核心阈值（minWordCount / reviewPassThreshold / outlineCoverageThreshold / maxRevisionRounds / maxTotalRevisionPerChapter）通过 `config` 字段集中管理 |

---

## 技能族结构

```
novel-continuation/
  SKILL.md                       # 工程化入口（路由 + 状态规范 + 全局交互点 + 通用红旗）
  chapter-splitting/SKILL.md     # 阶段 0：导入 + 三遍读取 + 题材识别 + 文风 + 约束文档
  outline/SKILL.md               # 阶段 1-3：分析 + 大纲 + 强化约束
  continuation/SKILL.md          # 阶段 4：逐章写作（串行，不含评审）
  review/SKILL.md                # 阶段 5：5 项门（每章）+ 33 维审计（全局）+ 质量循环
```

入口 `SKILL.md` 是工程化枢纽，包含：
- **全局恢复决策树**（按 `currentStep` 路由到 4 个子技能）
- **3 层状态字段规范**（currentStep / project.status / chapters[i].status）
- **全局交互点清单**（4 个固定节点，禁止其他位置 AskUserQuestion）
- **约束文档分类**（必选 / 按题材 / 运行时）
- **可调参数 config 字段**（5 个核心阈值 + token 预算）
- **通用红旗**（子技能只列本阶段特有）
- **异常决策树**（覆盖 7 种典型异常）

子技能之间互不感知，独立可触发。

### 4 个子技能的触发条件

| 技能 | 何时调用 |
|------|---------|
| `chapter-splitting` | 提供小说文件 / 0-A 阶段中断恢复 |
| `outline` | `currentStep` ∈ {`import-done`, `answers-ready`, `outline-ready`} |
| `continuation` | `currentStep == "writing"`，大纲已批准 |
| `review` | `currentStep` ∈ {`writing`（章节门）, `quality-loop`（33 维）} |

### 状态流转

`meta/_project-meta.json.currentStep`：`import-done` → `answers-ready` → `outline-ready` → `constraint-docs` → `writing` → `quality-loop` → `report-ready`

**3 层状态字段，层级严格分离：**

| 层级 | 字段 | 用途 |
|------|------|------|
| 项目阶段 | `_project-meta.json.currentStep` | 路由用，决定跳到哪个子技能 |
| 项目状态 | `02-写作计划.json.status` | 项目整体状态（章节列表之上） |
| 章节状态 | `02-写作计划.json.chapters[i].status` | 单章状态 |

**取值空间：**
- `currentStep` ∈ {`import-done`, `answers-ready`, `outline-ready`, `constraint-docs`, `writing`, `writing-paused`, `quality-loop`, `report-ready`}
- 项目 `status` ∈ {`pending`, `in_progress`, `paused`, `auditing`, `completed`, `failed`}
- 章节 `status` ∈ {`pending`, `in_progress`, `completed`, `failed`}（**不得使用 `paused`**——暂停是项目级概念）

---

## 全局交互点（4 个固定节点）

整个工作流中，与用户对话的固定节点只有 4 个。其他位置一律禁止 AskUserQuestion：

| # | 阶段 | 交互内容 |
|---|------|---------|
| 1 | `chapter-splitting` 0-B | 询问"继续上次创作？" |
| 2 | `outline` 第 2 步 | 询问 2 个问题（章节数、是否增加人物）|
| 3 | `outline` 第 3 步 | 询问大纲批准 |
| 4 | `review` 完成报告 | 一次性汇报全部结果 |

---

## 题材识别 → 33 维激活

`chapter-splitting` 0-A 步骤3 自动识别题材，写入 `_project-meta.json.config.genre`。**review 技能按 genre 机械激活维度，禁止 AI 主观判断：**

| genre | 含义 |
|-------|------|
| `xianxia` | 仙侠/修真 |
| `xuanhuan` | 玄幻/西方玄幻 |
| `dushi` | 都市/现代 |
| `lishi` | 历史/穿越 |
| `kehuan` | 科幻/赛博 |
| `xuanyi` | 悬疑/推理 |
| `litrpg` | 系统/游戏 |
| `mori` | 末日/废土 |
| `qihuan` | 奇幻/西幻 |
| `general` | 通用/未识别（默认） |

**题材识别失败 → 默认 `general`，仅激活基础 8 维 + 全部题材通用维度。**

---

## 项目目录结构

```
novel-projects/
  [项目名称]/
    meta/                            # 项目管理文件
      _project-meta.json             # 元数据（含 config / genre / currentStep）
      02-写作计划.json               # 写作计划（含 chapters 数组）
      _run-log.jsonl                 # 运行日志（27 类事件，自动追加）
    design/                          # 设计约束文档
      00-人物档案.md                 # [必选] 人物档案
      01-大纲.md                     # [必选] 大纲
      02-风格指南.md                 # [按题材] 仅当用户提供参考文风时
      03-世界设定书.md               # [必选] 世界设定
      04-时间线.md                   # [必选] 时间线
      05-术语表.md                   # [必选] 术语表
      06-核心驱动.md                 # [必选] 主线/支线/伏笔追踪
      98-写作决策日志.md             # [运行时] 写作中不确定的决策记录
      99-冲突日志.md                 # [运行时] 跨章设定矛盾记录
    chapters/                        # 章节正文
      第XX章-标题.md                 # 各章节正文
      _markers.md                    # 速读标记索引
      _review-第XX章.md              # [每章] 5 项门评审报告
      _audit-第XX章.md               # [全局] 33 维审计报告（全部完成后才生成）
    truth/                           # JSON 真相文件
      world-state.json               # [必选] 世界状态
      character-matrix.json          # [必选] 角色关系矩阵
      resource-ledger.json           # [必选] 资源账本
      chapter-summaries.json         # [必选] 章节摘要（token 摘要模式核心）
      subplot-board.json             # [必选] 支线进度板
      emotional-arcs.json            # [必选] 情感弧线
      pending-hooks.json             # [必选] 待处理钩子
      数值系统.json                  # [按题材] 仅 xianxia/xuanhuan/litrpg/mori
      年代考据.json                  # [按题材] 仅 dushi/lishi
```

**约束文档三层分类：**
- **必选**（12 个）：5 design + 7 truth，任何题材都生成
- **按题材激活**（3 个）：02-风格指南、truth/数值系统、truth/年代考据
- **运行时追加**（3 个）：98-写作决策日志、99-冲突日志、meta/_run-log.jsonl

---

## 可调参数（`config` 字段）

```json
{
  "config": {
    "minWordCount": 3000,
    "reviewPassThreshold": 2500,
    "outlineCoverageThreshold": 0.7,
    "maxRevisionRounds": 3,
    "maxTotalRevisionPerChapter": 5,
    "genre": "xianxia",
    "tokenBudget": {
      "contextWindow": 200000,
      "switchToSummaryOnly": 0.6,
      "forceSummarize": 0.85
    }
  }
}
```

**单章修订轮次分配建议（默认 `maxTotalRevisionPerChapter=5`）：**
- 章节门：最多 3 轮
- 33 维审计：最多 2 轮
- 合计 ≤ 5，超限标记 `failed` 不阻塞流程

---

## 使用流程

```
入口：novel-continuation/SKILL.md
  │  读 currentStep → 全局恢复决策树
  │
  ├─[无/未完成]→ chapter-splitting → import-done
  │   0-A: 解析 → 三遍读取 → 隐式推断+题材识别 → 文风分析 → 约束文档 → 自检 → 校验
  │   0-B: 未完成项目检测（全局交互点 #1）
  │
  ├─[import-done]→ outline
  │   1. 分析文本（人物/情节/结构/文风/设定/专有名词/时间线）
  │   2. 建议续写章数 + 询问 2 个问题（全局交互点 #2）
  │   3. 生成大纲 → 强制检查点更新项目文件
  │   3-A. 强化约束文档
  │   → 全局交互点 #3 询问大纲批准（不通过时询问修改意见，2 轮后强制 failed）
  │
  ├─[constraint-docs]→ continuation
  │   4. 逐章写作（串行模式，零中断）
  │      步骤 1: 写前分析（按 token 预算动态降级）
  │      步骤 2: 撰写
  │      步骤 3: 撰写后优化
  │      步骤 4: 收尾 + 自动流转
  │      每章完成后调用 review 技能做章节门（5 项）
  │
  └─[writing/quality-loop]→ review
      5-A. 章节评审门（每章 5 项检查，写入 _review-第N章.md）
      5-B. 33 维度审计（全部完成后统一执行，写入 _audit-第N章.md）
      5-C. 整体架构评估（6 项）
      5-D. 完成报告（全局交互点 #4）
```

---

## 关键原则

- **chapter-splitting 是强制入口** - 提供文件时必须先执行 0-A 导入流程
- **展示而非讲述**：用动作和对话表现
- **冲突驱动剧情**：每章必须有冲突或转折
- **悬念承上启下**：每章结尾留下钩子
- **逐章写作，全流程零中断**：一旦开始，连续完成所有章节
- **只使用串行模式**：不并行、不 Teams
- **5 项门 + 33 维拆分**：每章轻量门，全局重审计
- **题材机械激活维度**：禁止 AI 主观判断
- **约束文档三层分类**：必选 / 按题材 / 运行时
- **可调参数集中管理**：5 个核心阈值 + token 预算

---

## 异常处理

入口 + 各子技能均提供异常决策树，覆盖：
- AI 输出超长（>8000 字）→ 自动分章
- 约束文档损坏 → 回到对应阶段重建
- JSON 不合法 → 备份后重建骨架
- 题材识别失败 → 默认 `general`
- Token 超限 → 切换摘要模式
- 修订轮次超限 → 标记 `failed` 不阻塞
- 用户主动暂停 → `currentStep = "writing-paused"` 保留恢复点

---

## 参考

完整技能族：

- [入口](novel-continuation/SKILL.md) - 路由 + 状态规范 + 全局交互点
- [chapter-splitting](novel-continuation/chapter-splitting/SKILL.md) - 阶段 0：导入
- [outline](novel-continuation/outline/SKILL.md) - 阶段 1-3：分析 + 大纲
- [continuation](novel-continuation/continuation/SKILL.md) - 阶段 4：续写
- [review](novel-continuation/review/SKILL.md) - 阶段 5：评审
