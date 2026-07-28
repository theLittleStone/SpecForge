---
name: task-breakdown
description: 将架构设计拆解为可执行任务列表，支持新增/修改/重构/删除等多种变更类型
---

## 功能说明

该命令用于维护任务拆分文件 `planning/tasks.md`。  
任务必须来源于架构设计，并形成可执行顺序，同时具备明确的状态流转。

任务支持多种 `change_type`，对应不同类型的工作流：
- `feature`：新增功能
- `enhancement`：增强现有功能
- `refactor`：重构（不改行为）
- `bugfix`：修复缺陷
- `removal`：删除/废弃

任务是后续规格定义与代码实现的基础。

## 行为规则

执行流程：

1. 读取 `planning/architecture.md`（如不存在，向用户汇报，禁止自行添加），获取其 `round`、`change_type`、`status`。若 `status` 为 `draft`，提醒用户「architecture.md 尚未确认（status: draft），建议先运行 architecture-design 完成确认」。
2. 读取 `planning/tasks.md`（如不存在则允许新建）。若已存在，检查其 frontmatter `round` 是否与 architecture.md 一致——不一致则醒示「tasks.md 属于另一轮次（round: X vs Y），请先运行 `/progress-report` 归档旧内容或清空 `planning/` 后重试」
3. 分析任务与架构的一致性，确保 task 的 `change_type` 与架构一致（可更具体，不可冲突）
4. 生成差异修改提案
5. 输出预览
6. 确认后写入
7. 写入后检查下游产物，扫描 `planning/specifications.md`（如存在）：
    - 全部规格状态为 `[新增: 已定义]`、`[修改: 已定义]` 或 `[废弃: 待删除]`（或文件不存在）→ 代码未产生，安全。
     提醒用户「任务已更新。代码尚未产生，请继续运行 `/specification-define`」
    - 存在 `[测试已生成]`、`[已实现]`、`[已测试]`、`[待修复]`、`[已验证]` 或 `[已删除]` 状态 → 代码已落地，需回滚：
     a. 从当前处理 task 关联的规格中提取 `Affected Files` 字段，去重
     b. 若 `Affected Files` 为空，用 `git diff --name-only HEAD` 获取实际变动
     c. 过滤掉 `planning/` 目录下的文件，输出回滚指令：
        git restore <file1> <file2> ...
     d. 醒示「代码已恢复。planning/ 文档保留，请重跑 `/specification-define`」

默认不直接覆盖文件，必须基于差异更新。

### 共识要求

- Agent 自主完成依赖排序和任务粒度分析
- 写入文件前展示提案，等待用户确认
- 用户提出调整时修订后重新确认

## 文档结构要求

```md
# Task List

## Task 1: <name> [TODO]

- **Change Type**: feature | enhancement | refactor | bugfix | removal — 继承自架构，可进一步细化
- **Depends On**: <task # or none> — 前置依赖任务编号，none 表示无依赖
- **Specs**: （由 specification-define 填写，子列表格式，每行一条规格名称）
           - `spec_name_1`
           - `spec_name_2`
- **Description**: 任务目标、上下文与预期产出

## Task 2: <name> [DEFINED]
...
```

说明：

- 标题标记只能为：[TODO] / [DEFINED] / [IMPLEMENTED]
- `change_type` 继承自架构，可进一步细化：feature | enhancement | refactor | bugfix | removal
- 不同 `change_type` 的 IMPLEMENTED 含义不同：
  - feature → 代码已新增
  - enhancement → 修改已完成
  - refactor → 重构已完成
  - bugfix → 修复已完成
  - removal → 代码已删除
- Specs 为子列表格式（`  - spec_name`），每行一条规格名称，由 specification-define 填写，code-review 据此遍历关联规格

## 输入来源

本命令的输入从上游文档读取，无需额外 JSON 输入。

### 上游文档

| 文件 | 用途 | 关键内容 |
|------|------|---------|
| `planning/architecture.md` | 架构设计 | frontmatter 中的 `change_type`；正文中的 `# Changes` 表格（Change / Description / Dependencies 三列） |

### 状态过滤

忽略标题标记为 `[IMPLEMENTED]` 或 `[DEFINED]` 的任务。仅对处于 `[TODO]` 的任务生成/更新任务。

### 处理顺序

按 `# Changes` 表格的行序，从上到下处理。

### 上游文档解析提示

- `planning/architecture.md` 的 `change_type` 位于文件顶部的 YAML frontmatter 中
- `# Changes` 为表格格式，列 Change / Description / Dependencies。任务拆分按行序从上到下生成 task
- `planning/tasks.md` 各任务以 `## Task N: <name> [STATUS]` 为标题，内含 `Change Type`、`Depends On`、`Specs` 字段

## 输出

命令执行后将修改 `planning/tasks.md`。

### 目标文件结构（planning/tasks.md）

**Frontmatter（YAML）：**

```yaml
round: <继承自 architecture.md>
version: <int>
created: <ISO date>
updated: <ISO date>
based_on: planning/architecture.md
```

**正文章节：**

```md
# Task List

## Task 1: <name> [TODO]

- **Change Type**: feature | enhancement | refactor | bugfix | removal — 继承自架构，可进一步细化
- **Depends On**: <task # or none> — 前置依赖任务编号，none 表示无依赖
- **Specs**: （由 specification-define 填写，子列表格式，每行一条规格名称）
           - `spec_name_1`
           - `spec_name_2`
- **Description**: 任务目标、上下文与预期产出

## Task 2: <name> [DEFINED]
...
```

### 交互预览

执行时以 diff 格式展示修改提案，包含任务分析、修改内容与执行路径说明。

## 下一步

任务已拆分后，提醒用户运行 `/specification-define` 进行规格定义。

## 约束

- 每个任务必须可独立执行
- 每个任务必须明确 `change_type`
- 必须存在依赖关系（形成 DAG）
- 不允许循环依赖
- 必须映射到架构模块
- 不允许一次性重写全部任务（除非 create）
- 已为 DEFINED 或 IMPLEMENTED 的任务不得重复拆解
