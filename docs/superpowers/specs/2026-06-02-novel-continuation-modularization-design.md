# novel-continuation 技能模块化重构设计

**日期**: 2026-06-02
**状态**: 待用户审阅
**原技能**: `novel-continuation/SKILL.md`（1430 行）

## 1. 目标

将单体 `novel-continuation` 技能拆分为 4 个独立技能，降低单个 SKILL.md 的复杂度（当前 1430 行超出 writing-skills 建议的 500 字），使每个技能可独立触发、独立测试、独立维护。

## 2. 拆分粒度决策

**采用方案 A：4 个独立技能 + 1 个轻量总入口**

```
novel-continuation/
  SKILL.md                    # 轻量总入口（流程图 + 导航 + 共享原则）
  chapter-splitting/SKILL.md  # 阶段 1
  outline/SKILL.md            # 阶段 2
  continuation/SKILL.md       # 阶段 3
  review/SKILL.md             # 阶段 4
```

4 个子技能**互不感知**（不互相引用），但可被总入口路由。

## 3. 模块边界

| 模块 | 来源（原 SKILL.md 行号） | 触发条件 | 关键产物 |
|------|------------------------|---------|---------|
| **chapter-splitting** | 第0步 0-A（133-366）+ 0-B（382-422）| 用户提供小说文件/文本；或 novel-projects/ 中有未完成项目 | `chapters/*.md` + `design/*` + `truth/*.json` + `meta/_project-meta.json` |
| **outline** | 第1步（426-512）+ 第2步（516-605）+ 第3步（609-642）+ 第3-A步（646-825）+ 文风模仿（1214-1265）| 已完成拆分，需要分析+大纲+强化约束 | `design/01-大纲.md` + `meta/02-写作计划.json`（含章节列表）|
| **continuation** | 第4步（829-1063），**不含**章节评审门和 33 维审计 | 大纲已批准，需逐章写作 | `chapters/第XX章-*.md` + 增量更新约束文档/真相文件 |
| **review** | 第5步（1067-1173）+ 章节评审门（974-1012）+ 33 维度（1269-1367）+ 附录约束文档体系（1183-1210）| 每章完成 / 全部章节完成 | `chapters/_review-第XX章.md` + 质量评分 + 修订动作 |

**评审相关全部归 review 技能**（用户决策）：章节评审门、33 维度完整清单、质量循环、整体架构评估。

## 4. 4 个子技能的 description（按 writing-skills "Use when..." 规范）

```yaml
# chapter-splitting
description: >
  Use when the user provides an existing novel file or text for
  continuation, or when resuming an unfinished project in
  novel-projects/. Triggers on: existing text, "import this novel",
  "continue from here", resumption after interruption.

# outline
description: >
  Use after chapter-splitting when chapter files exist and the user
  needs to analyze the novel, generate a continuation outline, and
  strengthen constraint documents. Triggers on: "write the outline",
  "plan the continuation", "analyze and plan".

# continuation
description: >
  Use after outline is approved when chapters need to be written
  serially. Triggers on: "start writing", "write chapter N",
  "continue the story", or when the project state currentStep is
  "writing".

# review
description: >
  Use after a chapter is written to run the chapter review gate
  (5-item checklist: character consistency, worldview, context
  alignment, hook quality, quality baseline), or when all chapters
  are complete to run the 33-dimension global audit, auto-revision
  (max 3 rounds), and overall architecture assessment.
```

## 5. 状态契约（`meta/_project-meta.json` 的 `currentStep` 流转）

```
[无]                → chapter-splitting
import-done         → outline
answers-ready       → outline
outline-ready       → outline
constraint-docs     → continuation
writing             → continuation / review（章节评审门）
quality-loop        → review
report-ready        → [终态]
```

| 技能 | 入场校验 | 退出写入 |
|------|---------|---------|
| chapter-splitting | 任意（首次或恢复） | `import-done` |
| outline | `import-done` | `answers-ready` → `outline-ready` → `constraint-docs` |
| continuation | `constraint-docs` | `writing`（每章完成时更新对应章节 status） |
| review | `writing`（章节门） 或 全部 completed（质量循环） | `quality-loop` → `report-ready` |

## 6. 中断恢复设计（两层机制）

### 6.1 粗粒度导航（currentStep）

总入口检测 `currentStep` 路由到对应子技能。

### 6.2 细粒度恢复（关键文件存在性 + 章节级 status）

#### chapter-splitting 恢复点表

新增 `importStage` 字段记录 0-A 内部进度：`"step1" | "step2" | "step3" | "step4" | "step5" | "step6" | "done"`。

| 文件存在性 | 推断进度 | 恢复动作 |
|----------|---------|---------|
| 无任何文件 | 全新 | 0-A 步骤1 开始 |
| `chapters/*.md` 存在但 `chapters/_markers.md` 缺失 | 步骤1 完成 | 跳到步骤2 第一遍 |
| `_markers.md` 存在但 `design/*.md` 部分缺失 | 步骤2 完成 | 补缺失的 design 文件 |
| `design/*.md` 齐全但 `truth/*.json` 部分缺失 | 步骤4 部分完成 | 补缺失的 truth 文件 |
| 全部存在 | 步骤4 完成 | 跳到步骤5（自检）|

#### outline 恢复点表

| 字段/文件 | 推断进度 | 恢复动作 |
|---------|---------|---------|
| `_project-meta.json.answers` 缺失 | 第1步 | 重做分析 + 询问 |
| `answers` 存在但 `design/01-大纲.md` 缺失 | 第2步完成 | 跳到第3步（生成大纲）|
| `01-大纲.md` 存在但 `chapters` 数组为空 | 第3步未填计划 | 填章节列表 |
| `01-大纲.md` 存在且 `chapters` 非空 | 第3步完成 | 跳到第3-A步（强化约束）|
| 5 个 `design/*.md` 齐全 | 第3-A步完成 | 退出 |

#### continuation 恢复点表

```
读 meta/02-写作计划.json.chapters 找第一个 status != "completed" 的章节 N
  ↓
检查 chapters/第N章-*.md 文件状态:
  ├─ 不存在 → 从头写第N章
  ├─ 存在但字数 < 2500 → 默认续写（保留已写内容并补全到 3000+）
  │   └─ 续写前对照 design/01-大纲.md 校验**大纲覆盖度**（核心事件/章节意图/钩子的覆盖率）：
  │       ├─ 覆盖度 ≥ 70% → 续写
  │       └─ 覆盖度 < 70% → 写入 design/98-写作决策日志.md 并重写
  └─ 存在且字数 ≥ 2500 → 检查 _review-第N章.md
      ├─ 不存在 → 调用 review 技能
      └─ 存在 → 找下一章
```

**章节状态字段**：`meta/02-写作计划.json.chapters[i].status ∈ { "pending", "in_progress", "completed", "failed" }`。中断时保持 `in_progress`，完成后才写 `completed`。

#### review 恢复点表

```
读 meta/02-写作计划.json.chapters 找 _review-第N章.md 缺失或评审不通过的章节 N
  ↓
├─ _review-第N章.md 不存在 → 章节评审门（重做）
├─ _review-第N章.md 存在但 status != "passed" → 修订循环
└─ 全部评审通过 → 检查是否需要质量循环
   ├─ 02-写作计划.json.status != "auditing" → 启动质量循环
   └─ status == "auditing" 且 retryCount < 3 → 继续当前轮
   └─ status == "auditing" 且 retryCount >= 3 → 退出修订，生成报告
```

### 6.3 异常中断（用户主动停止）

用户在任意时刻输入"停止"或"暂停"：
1. 主 agent 立即停止当前动作
2. 更新 `meta/_project-meta.json.currentStep` 为当前阶段最细粒度状态
3. 写入 `design/98-写作决策日志.md` 一行"中断记录"（时间戳 + 原因）
4. 输出"已暂停，恢复点：X"信息给用户

## 7. 各 SKILL.md 内部结构

### 7.1 chapter-splitting/SKILL.md

```
1. Frontmatter (name + description)
2. 概述
3. 何时使用
4. 核心工作流（dot 流程图）
5. 写前确认
6. 0-A 步骤1-7
7. 0-B 未完成项目检测 + 恢复点表
8. ✅ 完成确认门
9. 红旗
```

### 7.2 outline/SKILL.md

```
1. Frontmatter
2. 概述
3. 何时使用
4. 核心工作流
5. 写前确认
6. 恢复点表
7. 第1步：分析文本
8. 第2步：建议+询问2个问题
9. ⛔ 创建项目文件
10. 第3步：生成大纲
11. 第3-A步：强化约束文档
12. 文风模仿（高级功能）
13. 红旗
```

### 7.3 continuation/SKILL.md

```
1. Frontmatter
2. 概述
3. 何时使用
4. 核心工作流
5. 写前确认
6. 恢复点表
7. 无中断写作区段
8. 逐章创作流程
   - 步骤 1: 写前分析
   - 步骤 2: 撰写
   - 步骤 3: 撰写后优化
   - 步骤 4: 收尾
9. 自动流转（每章最后一步）→ 调用 review 技能
10. 红旗
```

### 7.4 review/SKILL.md

```
1. Frontmatter
2. 概述
3. 何时使用（两种模式：章节评审门 / 全局质量循环）
4. 核心工作流
5. 写前确认
6. 恢复点表
7. 章节评审门（5 项）
8. 33 维度质量审计（完整清单 + 评分校准 + 题材激活规则）
9. 质量循环（自动修订 3 轮）
10. 整体架构评估（6 项）
11. 完成报告
12. 红旗
```

### 7.5 novel-continuation/SKILL.md（总入口，轻量导航）

```
1. Frontmatter
2. 概述（1 段：包含 4 阶段的小说续写技能族）
3. 何时使用（含中断恢复引导）
4. 核心工作流（缩略总流程图）
5. 4 个子技能导航（description 引用）
6. 项目目录结构（共享）
7. 状态契约（共享）
8. 关键原则（共享）
9. 通用红旗（不可跳过 0-A、章节间零输出等）
```

## 8. 内容提取映射（原 SKILL.md → 目标文件）

| 原 SKILL.md 行号 | 内容 | 目标文件 |
|----------------|------|---------|
| 82-118 | 项目目录结构 | novel-continuation/SKILL.md |
| 69-79 | 关键原则 | novel-continuation/SKILL.md |
| 133-366 | 第0步 0-A | chapter-splitting/SKILL.md |
| 382-422 | 第0步 0-B（含精确恢复表） | chapter-splitting/SKILL.md |
| 426-512 | 第1步：分析文本 | outline/SKILL.md |
| 516-605 | 第2步：建议+询问 | outline/SKILL.md |
| 609-642 | 第3步：生成大纲 | outline/SKILL.md |
| 646-825 | 第3-A步：强化约束 | outline/SKILL.md |
| 829-1063 | 第4步：逐章写作 | continuation/SKILL.md |
| 974-1012 | 章节评审门 | review/SKILL.md |
| 1067-1173 | 第5步：质量循环 | review/SKILL.md |
| 1177-1210 | 附录：约束文档体系 | review/SKILL.md |
| 1214-1265 | 高级功能：文风模仿 | outline/SKILL.md |
| 1269-1367 | 33 维度质量审计 | review/SKILL.md |
| 1376-1408 | 红旗 | 总入口（通用部分）+ 各子技能（专属部分） |
| 1412-1429 | 示例 | novel-continuation/SKILL.md |

## 9. 验证清单

- [ ] 原 SKILL.md 1430 行内容全部迁移（不丢失）
- [ ] 4 个子技能可独立触发（description 包含具体触发词）
- [ ] 每个子技能入场前有 currentStep 校验
- [ ] 状态契约文档化在总入口
- [ ] 红旗按归属拆分到子技能
- [ ] 原 SKILL.md 文件改写为轻量入口
- [ ] 每个子技能入场时执行"恢复点探测"
- [ ] `_project-meta.json` 增加 `importStage` 字段
- [ ] 章节 partial 状态机制有明确定义（`in_progress` vs `completed`）
- [ ] 异常中断（用户主动停止）的状态保存流程
- [ ] 模拟测试：5 种中断点（步骤1中、章节中、评审中、修订中、报告前）

## 10. 实施步骤

```
1. 创建目录骨架
   novel-continuation/
     chapter-splitting/
     outline/
     continuation/
     review/

2. 内容提取（按第 8 节映射表）
   - 从原 SKILL.md 中提取对应行号范围
   - 写入目标 SKILL.md
   - 添加 frontmatter（name + description）

3. 重写总入口
   - 备份原 SKILL.md
   - 改写为轻量导航版本（第 7.5 节结构）

4. 验证
   - 内容完整性（原 1430 行全部覆盖）
   - 触发测试（4 个子技能可被正确触发）
   - 状态机测试（手动修改 currentStep 验证各技能拒绝入场）
   - 恢复点测试（5 种中断场景模拟）

5. 提交 git
   git add novel-continuation/
   git commit -m "refactor: split novel-continuation into 4 modular skills"
```

## 11. 不做什么

- **不**改变原 SKILL.md 的核心流程逻辑（仅拆分，不重写）
- **不**改变项目目录结构（`novel-projects/...` 不变）
- **不**改变约束文档体系（12 个文档不变）
- **不**做向后兼容（旧 `novel-continuation` 描述不保留兼容性）
- **不**增加新功能（如并行写作、可视化编辑器等）

## 12. 风险与缓解

| 风险 | 影响 | 缓解 |
|------|------|------|
| 用户不知道该调用哪个技能 | 中断恢复失效 | 总入口提供 currentStep 路由表 |
| 子技能入场状态校验失败 | 用户误操作 | 每个技能开头硬编码前置条件清单 |
| 4 个 SKILL.md 各自仍超 1000 字 | writing-skills 建议 < 500 字 | 接受超长（流程文档必须读完才能执行）|
| 章节 partial 状态判断错误 | 续写时风格不一致 | 大纲覆盖度 70% 阈值 + 决策日志记录 |
| 评审门与续写技能间循环依赖 | 状态机死锁 | 评审结果只写文件，不修改章节 status；续写读完评审结果后决定是否重写 |
