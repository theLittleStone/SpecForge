---
name: architecture-design
description: 维护架构设计文件，支持新建、修改、重构、删除等多种变更类型下的架构演进
---

## 功能说明

该命令用于维护架构设计文件 `planning/architecture.md`。  
架构允许逐步完善，也允许后续更新，而不是一次性完整设计。

执行前必须读取 `requirement.md` 作为基础。

## 行为规则

执行流程：

1. 读取 `planning/requirement.md`（必须存在，如不存在则向用户汇报，禁止自行添加），获取其 `round`、`change_type`、`status`。若 `status` 为 `draft`，提醒用户「requirement.md 尚未确认（status: draft），建议先运行 requirement-define 完成确认」。
2. 读取 `planning/architecture.md`（如不存在则新建）。若已存在，检查其 frontmatter `round` 是否与 requirement.md 一致——不一致则醒示「architecture.md 属于另一轮次（round: X vs Y），请先运行 `/progress-report` 归档旧内容或清空 `planning/` 后重试」
3. 分析需求与当前架构
4. 分析并填充 `# Impact Analysis`（变更对现有模块、数据流、API 的影响范围与副作用），必须包含：
   - **开放边界识别**：列出本轮变更涉及的外部系统边界（邮件、支付、存储、GUI 渲染、非确定性源等），标注是否可通过 Test Double 覆盖
5. 同步维护 `# Dependency Index`：比对本轮 `# Changes` 涉及的模块，更新其 `Depends On` / `Depended By`（新增/移除引用同步更新）。涉及废弃/删除的模块，在 `Depended By` 列出其全部引用方，作为后续解引用与物理删除的依据
6. 生成差异修改提案
7. 输出预览
8. 确认后写入，并将 frontmatter 中 `status` 更新为 `confirmed`
9. 写入后检查下游产物，扫描 `planning/specifications.md`（如存在）：
    - 全部规格状态为 `[新增: 已定义]`、`[修改: 已定义]` 或 `[废弃: 待删除]`（或文件不存在）→ 代码未产生，安全。
     提醒用户「架构已更新。代码尚未产生，请按顺序重跑：`/task-breakdown` → `/specification-define`」
    - 存在 `[测试已生成]`、`[已实现]`、`[已测试]`、`[待修复]`、`[已验证]`、`[已解引用]` 或 `[已删除]` 状态 → 代码已落地，需回滚：
      a. 汇总所有状态为 `[测试已生成]`、`[已实现]`、`[已测试]`、`[待修复]`、`[已验证]`、`[已解引用]` 或 `[已删除]` 的规格的 `Affected Files` 字段，去重
     b. 若 `Affected Files` 为空，用 `git diff --name-only HEAD` 获取实际变动
     c. 过滤掉 `planning/` 目录下的文件与 `_deprecated/` 内路径（后者由删除目录步骤处理），输出回滚指令：
         git restore <file1> <file2> ...
         （若规格含 `[已解引用]` / `[已删除]` 状态，另需删除项目根目录 `_deprecated/` 及其中的移动副本）
         （若存在 git 未跟踪的新文件，提醒手动删除）
      d. 醒示「代码已恢复。planning/ 文档保留，请重跑下游：`/task-breakdown` → `/specification-define`」

### 共识要求

- Agent 自主完成 Changes 切割和 Design Decisions 拟订
- 写入文件前展示提案，等待用户确认
- 涉及关键技术选型时主动说明利弊，由用户拍板
- 用户提出修改时回到分析步骤修订

## 文档结构要求

```md
# Overview          — 本轮变更目标，一句话概括
# Changes            — 本轮变更切分，每条对应一个独立 concern（新建 / 修改 / 删除）
                        
                       | Change | Description | Dependencies |
                       |--------|-------------|--------------|
                       | 名（操作） | 此变更要做什么 | N/A / 其他 Change |

                       涉及开放边界的 Change 必须排在纯逻辑 Change 之后（Dependencies 列体现依赖关系）。
# Impact Analysis    — 变更对现有模块、数据流、API 的影响分析，副作用评估
# Dependency Index   — 模块级双向引用索引，随本轮变更保持最新

                     | Module | Depends On | Depended By |
                     |--------|------------|-------------|
                     | 模块名  | 依赖的模块（无则 N/A） | 引用本模块的模块（无则 N/A） |

                     新增/删除引用时同步更新。涉及废弃/删除的模块，必须在 Depended By 中列出全部引用方，作为后续解引用与物理删除的依据。
# Design Decisions   — 关键设计决策、技术选型、取舍理由
### Test Double 策略

**Test Double** 是用于替代真实外部依赖的测试替身，遵循与生产代码相同的接口契约。
当规格涉及系统边界（外部 API、数据库、文件系统、时间/随机数、GUI 渲染、消息队列等）时，
必须在 specification-define 阶段为相关 Interface 设计 Test Double。

#### 替身类型速查

| 类型 | 适用场景 | 示例 |
|------|---------|------|
| **Fake** | 需要行为等价但无需真实基础设施 | FakeEmailService 将邮件存入内存数组 |
| **Stub** | 仅需固定返回值 | 固定返回 "支付成功" 的支付网关 |
| **Simulator** | 需要模拟完整子系统行为 | LocalStack 模拟 AWS S3 |
| **Seeded Generator** | 非确定性输出（随机、时间） | `random.seed(42)` 固定随机序列 |

#### 双保险模型

Test Double 不替代最终人工验证，而是缩小人工测试范围到真正不可自动化的残余场景：
- **Layer 1**：Test Double → 自动化测试（核心逻辑路径验证，快速、确定性）
- **Layer 2**：人工测试 → 视觉/交互/真实集成验证（Test Double 不可达的残余差异）

二者叠加，非互斥。

#### 本轮策略

Agent 根据当前项目的技术栈和本轮变更，在此填写具体的 Test Double 选型与隔离方案。
```

允许局部更新，不要求每次完整覆盖。

## 输入来源

本命令的输入从上游文档读取，无需额外 JSON 输入。

### 上游文档

| 文件 | 用途 | 关键内容 |
|------|------|---------|
| `planning/requirement.md` | 需求定义 | frontmatter 中的 `change_type`；正文中的 `# Goal`、`# Scope`、`# Constraints` 章节 |

### 上游文档解析提示

- `change_type` 位于 `planning/requirement.md` 文件顶部的 YAML frontmatter 块（`---` 包围）中
- `# Scope` 为正文二级标题，其下列出本次变更涉及的现有模块/文件路径

## 输出

命令执行后将修改 `planning/architecture.md`。

### 目标文件结构（planning/architecture.md）

**Frontmatter（YAML）：**

```yaml
round: <继承自 requirement.md>
version: <int>
created: <ISO date>
updated: <ISO date>
based_on: planning/requirement.md
status: draft | confirmed
change_type: <继承自 requirement.md>
```

**正文章节：**

```md
# Overview          — 本轮变更目标，一句话概括
# Changes            — 本轮变更切分，每条对应一个独立 concern（新建 / 修改 / 删除）
                       
                       | Change | Description | Dependencies |
                       |--------|-------------|--------------|
                       | 名（操作） | 此变更要做什么 | N/A / 其他 Change |
# Impact Analysis    — 变更对现有模块、数据流、API 的影响分析，副作用评估
# Dependency Index   — 模块级双向引用索引（Module / Depends On / Depended By 表格），随变更保持最新
# Design Decisions   — 关键设计决策、技术选型、取舍理由
```

### 交互预览

执行时以 diff 格式展示修改提案，包含架构分析、修改内容与影响说明。

## 下一步

架构已确认后，提醒用户运行 `/task-breakdown` 进行任务拆分。

## 约束

- 必须基于 requirement.md
- 不允许删除核心代码（除非明确说明）
- 必须指出变更对系统的影响
- `# Changes` 必须使用表格格式，包含 Change / Description / Dependencies 三列
- 每条 Change 对应一个独立 concern，避免过粗（覆盖多个无关改动）或过细（每文件一个 Change）
- 必须以 # Impact Analysis 描述变更影响
- 涉及开放边界的 Change 必须延迟到纯逻辑 Change 之后（通过 Dependencies 列确保排序）
- 涉及删除/废弃的模块必须在 `# Dependency Index` 中列出其全部引用方（Depended By）
