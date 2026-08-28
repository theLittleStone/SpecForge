---
name: requirement-define
description: 维护项目需求文档，支持 feature/enhancement/refactor/bugfix/removal/chore/docs 等多种变更类型
---

## 功能说明

该命令用于维护项目的需求定义文件 `planning/requirement.md`。  
需求不是一次性生成，而是可以反复修改和确认的长期资产。

执行时会先读取已有内容（如果存在），在此基础上进行分析与更新，而不是直接覆盖。

## 行为规则

### 苏格拉底对话

requirement-define 是流水线入口，需求质量决定后续所有环节。
Agent 不直接接受用户初始描述，必须通过追问与用户达成深度共识：

- **追问"为什么"** — 不只问"做什么"，追问底层目标与动机
- **挑战隐含假设** — 指出用户描述中未言明的假设，请用户确认
- **展开替代方案** — 对非平凡选择主动提出备选路径，分析利弊
- **穷举边界条件** — 引导用户思考异常路径、失败场景和回退策略
- **保留未知** — 无法确认的记入 Open Questions，不做假设填充
- **使用结构化交互** — 遇到选项明确的问题时，优先使用 question 工具让用户选择，而非开放式提问

### 执行流程

0. 检查项目根目录 `AGENTS.md` 已包含工作流共识（含「状态机与记法规范」节）；若缺失，醒示用户先运行 `/setup` 初始化并终止。
1. 读取 `planning/requirement.md`：
   - 存在 → 当前轮次可能进行中。询问用户意图：
     - 编辑本轮 → 保留已有 `round` 不变，继续分析
     - 开启新轮次 → 提示用户先运行 `/progress-report` 归档当前内容，再重新执行本命令
   - 不存在 → 读取 `./changelist.md`
      - changelist.md 存在 → 此为后续轮次，从最近一轮 Archive 路径获取历史 `architecture.md`，辅助填写 `# Scope`。首次创建时填写 `round` 为当前系统时间戳（格式 `YYYYMMDD-HHMM`，24小时制，精确到分钟），并记录 `baseline` 为当前 git HEAD（`git rev-parse HEAD`；非 git 仓库为 `none`）
      - changelist.md 也不存在 → 首次项目对话，新建 `planning/requirement.md`，`round` 为当前系统时间戳，并记录 `baseline` 为当前 git HEAD（同上）
2. 识别 `change_type`（feature | enhancement | refactor | bugfix | removal | chore | docs），分析当前内容与用户输入

3. **澄清探询**：
   不直接起草文档。围绕目标/范围/约束/替代方案/风险逐层追问，
   每轮聚焦 1-2 个问题。Agent 判断各维度已清晰后，
   向用户提议进入起草阶段，用户确认后进入步骤 4。
   用户也可随时要求停止探询、直接进入起草。

4. 基于共识起草 `# Goal`、`# Scope`、`# Constraints`、`# Open Questions`
5. 生成 diff 预览
6. 等待用户确认。用户提出修改意见 → 回到步骤 4 修订，直到确认
7. 确认后写入新版本，并将 frontmatter 中 `status` 更新为 `confirmed`
8. 写入后检查下游产物：扫描 `planning/specifications.md`（如存在），按 AGENTS.md §通用回滚协议 处理（回滚范围 = 本轮全部规格）：
     - 全部规格状态为 `[新增|修改: 已定义]`、`[废弃: 待删除]`（或文件不存在）→ 代码未产生，安全。
     提醒用户「需求已更新。代码尚未产生，请按顺序重跑下游命令：`/architecture-design` → `/task-breakdown` → `/specification-define` → `/test-generate`」
     - 否则（代码已落地）→ 按协议汇总 `Affected Files`、执行重叠守卫、输出 `git restore --source <baseline>` 回滚指令（含 `_deprecated/` 与 git 历史特殊处理），重置范围内规格状态，并将 tasks.md 全部 task 标记重置为 `[TODO]`（协议要求），随后醒示「代码已恢复至本轮开始前。planning/ 文档保留，请重跑下游全链：`/architecture-design` → `/task-breakdown` → `/specification-define`」

默认不直接覆盖文件，必须基于差异更新。

## 文档结构要求

需求文档必须包含以下结构。

**Frontmatter（YAML）：**

```yaml
round: <YYYYMMDD-HHMM>（首次创建时由系统时间生成，修改时保留原值不变）
baseline: <HEAD sha>（首次创建时记录当前 git HEAD，`git rev-parse HEAD`；非 git 仓库为 none，修改时保留原值不变）
status: draft | confirmed
change_type: feature | enhancement | refactor | bugfix | removal | chore | docs
```

版本历史由 git 承担，文档内不设 `version`/`created`/`updated` 字段。

**正文章节：**

```md
# Goal              — 本轮变更目标与期望效果
# Scope             — 涉及的范围：模块/文件/接口区域
# Constraints       — 技术约束、边界条件、注意事项
# Open Questions    — 未决问题（必须保留，可为空；已关闭条目用 `[resolved]` / `[won't-fix]` 标记）
```

其中 `Open Questions` 必须保留，用于记录未确定问题。

每条未决问题有两种关闭方式：

- `[resolved] <结论>`：已与用户确认，问题关闭（如「- 折扣与满减叠加还是取最大？[resolved] 取最大」）
- `[won't-fix] <理由>`：明确不做，问题关闭（如「- 超时订单自动取消？[won't-fix] 本期不支持」）

未标记的条目为「仍开放」。progress-report 判定「已完成」前，`Open Questions` 必须为空或全部条目已关闭。

## 输入来源

本命令是流水线入口，输入来自用户的自然语言描述，无需额外 JSON 输入。

### 读取自身文档

执行时 agent 应：

1. 读取 `planning/requirement.md`
   - 存在 → 当前轮次进行中，直接使用
   - 不存在 → 读取 `./changelist.md`
      - changelist.md 存在 → 此为后续轮次，从最近一轮 Archive 路径（形如 `planning/archive/planning_xxx/`）获取历史 `architecture.md`，提取 `# Changes` 辅助填写 `# Scope`
      - changelist.md 也不存在 → 首次创建
2. 从现有文档的 frontmatter（`---` 包围的 YAML 块）中提取 `round`、`change_type`、`status`
3. 从正文提取 `# Goal`、`# Scope`、`# Constraints`、`# Open Questions` 等章节内容
4. 结合用户输入与当前文档状态，判断操作动作（create / update / modify / refactor / removal / chore / docs / review）

### 上游文档解析提示

- `change_type` 位于文件顶部的 YAML frontmatter 块中
- `# Scope` 为正文二级标题，列出本次变更涉及的现有模块/文件路径
- `./changelist.md` 每轮以 `## Round <timestamp> [type]` 为标题，以 `- **Archive**: planning/archive/planning_<timestamp>/` 指向归档位置
- `round` 位于 frontmatter，格式为 `YYYYMMDD-HHMM`
- 打开 Archive 路径下的 `architecture.md`，其 `# Changes` 表格列出本轮变更条目，用于辅助填写 `# Scope`

## 输出

命令执行后将修改 `planning/requirement.md`，结构见「文档结构要求」。

### 交互预览

执行时以 diff 格式展示修改提案，包含当前状态分析、修改内容与说明。

## 下一步

需求已确认后，提醒用户运行 `/architecture-design` 进行架构设计。

## 约束

- 不允许删除结构字段
- 不允许一次性重写全部内容（除非 create）
- 必须填充 `# Scope` 描述变更涉及的范围
- 如果信息不足，应保留在 Open Questions 中
- 已关闭的 Open Questions 条目（`[resolved]` / `[won't-fix]`）不再阻塞收尾；仍开放的条目阻塞 progress-report 判定已完成
- 若 `planning/specifications.md` 中存在超出 `[新增|修改: 已定义]`、`[废弃: 待删除]` 状态的规格（即代码已落地），需求回滚前必须先按 AGENTS.md §通用回滚协议 恢复受影响的代码文件；agent 不自行 undo 代码变更
