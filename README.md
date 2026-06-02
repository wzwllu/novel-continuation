# novel-continuation — 小说续写技能

用于续写当年"太监"掉的小说。技能族由 1 个轻量入口 + 4 个独立子技能组成，覆盖导入 → 大纲 → 续写 → 评审的全流程。

> ⚠️ 不建议用于出版。如用于商业用途，请注明"来自于饭票.md"

---

## 核心能力

| 能力 | 说明 |
|------|------|
| **小说导入** | 自动分割章节、三遍读取法逆向工程、生成约束文档体系 |
| **中断续写检测** | 自动检测未完成项目，精确恢复到中断位置（含 `writing-paused` 状态恢复）|
| **文风模仿** | 分析参考文本词汇/句式/节奏/语调特征，保持风格一致 |
| **约束文档体系** | 12 个约束文档（7 个 JSON 真相文件 + 5 个 Markdown 设计文档）确保持续一致性 |
| **33 维度质量审计** | 按题材自动激活对应维度，评分 0-100，自动修订最多 3 轮 |
| **无中断写作** | 从写作到质量循环结束，零用户交互，自动逐章流转 |

---

## 技能族结构

```
novel-continuation/
  SKILL.md                    # 轻量入口（路由，~90 行）
  chapter-splitting/SKILL.md  # 阶段 1：导入 + 三遍读取 + 约束文档
  outline/SKILL.md            # 阶段 2：分析 + 大纲 + 强化约束
  continuation/SKILL.md       # 阶段 3：逐章写作（串行，不含评审）
  review/SKILL.md             # 阶段 4：章节评审 + 33 维度 + 质量循环
```

入口 `SKILL.md` 根据 `meta/_project-meta.json.currentStep` 路由到对应子技能；子技能之间互不感知，独立可触发。

### 4 个子技能的触发条件

| 技能 | 何时调用 |
|------|---------|
| `chapter-splitting` | 提供小说文件 / 恢复未完成项目 |
| `outline` | 已拆分，需要分析 + 生成大纲 |
| `continuation` | 大纲已批准，开始逐章写作 |
| `review` | 章节评审门 / 全局质量循环 |

### 状态契约

`meta/_project-meta.json.currentStep` 流转：`import-done` → `answers-ready` → `outline-ready` → `constraint-docs` → `writing` → `quality-loop` → `report-ready`（`writing-paused` 用于用户主动暂停）。各子技能入场前校验当前状态。

---

## 项目目录结构

```
novel-projects/
  [项目名称]/
    meta/                       # 项目管理文件
      _project-meta.json        项目元数据
      02-写作计划.json          写作计划
    design/                     # 设计约束文档
      00-人物档案.md            人物档案
      01-大纲.md                大纲
      03-世界设定书.md          世界设定
      04-时间线.md              时间线
      05-术语表.md              术语表
      06-核心驱动.md            核心驱动
      98-写作决策日志.md        写作决策日志
      99-冲突日志.md            冲突日志
      style-guide.md            文风指南
    chapters/                   # 章节正文
      第XX章-标题.md            各章节
      _markers.md               速读标记索引
      _review-第XX章.md         章节评审报告
    truth/                      # JSON 真相文件
      world-state.json          世界状态
      character-matrix.json     角色关系矩阵
      resource-ledger.json      资源账本
      chapter-summaries.json    章节摘要
      subplot-board.json        支线进度板
      emotional-arcs.json       情感弧线
      pending-hooks.json        待处理钩子
```

---

## 使用流程

```
入口：novel-continuation/SKILL.md（检测 currentStep，路由到对应子技能）
  │
  ├─[无/未完成]→ chapter-splitting → import-done
  │
  ├─[import-done]→ outline
  │   1. 分析文本（人物/情节/结构/文风/设定/专有名词/时间线）
  │   2. 建议续写章数 + 询问 2 个问题（章数？新增角色？）
  │   3. 生成大纲 → 用户批准
  │   3-A. 强化约束文档
  │
  ├─[constraint-docs]→ continuation
  │   4. 逐章写作（串行模式，零中断）
  │      每章完成后调用 review 技能做章节评审门
  │
  └─[writing/quality-loop]→ review
      5. 章节评审（5 项检查）+ 33 维度审计
         修订循环（最多 3 轮）+ 整体架构评估
         → 完成报告
```

---

## 关键原则

- **chapter-splitting 是强制入口** - 提供文件时必须先执行 0-A 导入流程
- **展示而非讲述**：用动作和对话表现
- **冲突驱动剧情**：每章必须有冲突或转折
- **悬念承上启下**：每章结尾留下钩子
- **逐章写作，全流程零中断**：一旦开始，连续完成所有章节
- **只使用串行模式**：不并行、不 Teams
- **质量循环**：写作完成后审计修订，直到达标
- **约束文档体系**：12 个文档（5 design + 7 truth）保证一致性

---

## 参考

完整技能族：

- [入口](novel-continuation/SKILL.md) - 轻量路由
- [chapter-splitting](novel-continuation/chapter-splitting/SKILL.md) - 阶段 1：导入
- [outline](novel-continuation/outline/SKILL.md) - 阶段 2：分析 + 大纲
- [continuation](novel-continuation/continuation/SKILL.md) - 阶段 3：续写
- [review](novel-continuation/review/SKILL.md) - 阶段 4：评审
