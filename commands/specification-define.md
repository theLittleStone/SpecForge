---
name: specification-define
description: 将任务拆解为可执行的实现规格，支持新增/修改/废弃操作与轻量规格声明
---

## 功能说明

该命令用于维护实现规格文件 `planning/specifications.md`。
规格是 TDD 流程的起点，为后续 test-generate 和 code-generate 提供精确的接口契约和测试用例。

每条常规规格必须明确四个维度：
- **Spec**：做什么（目的、上下文、目标文件）
- **Interface**：契约是什么（精确的函数签名/API端点/类型定义）；轻量规格可省略并在 Spec 中说明
- **Test Double**（可选）：当 Interface 涉及开放边界时的测试替身设计
- **Test Cases**：怎么验证（基于 Interface 的行为验证场景，区分 `[自动化]` 与 `[人工]`）

规格状态采用 AGENTS.md §状态机与记法规范 定义的 16 态；本命令拥有 `[新增|修改: 已定义]` 与 `[废弃: 待删除]`，并按 AGENTS.md §通用回滚协议 执行状态重置。

## 行为规则

执行流程：

0. 检查项目根目录 `AGENTS.md` 已包含工作流共识（含「状态机与记法规范」节）；若缺失，醒示用户先运行 `/setup` 初始化并终止。
1. 读取 `planning/tasks.md`（如不存在则中断任务并向用户汇报）。
2. 读取 `planning/specifications.md`（如存在）。若已存在且非空，检查其 frontmatter `round` 是否与 tasks.md 一致——不一致则醒示「specifications.md 属于另一轮次（round: X vs Y），请先运行 `/progress-report` 归档旧内容或清空 `planning/` 后重试」。
3. **优先检查 `[规格缺陷]` 标记**：若存在任一规格 `Issues` 以 `[规格缺陷]` 开头，按「`[规格缺陷]` 规格的受理」流程处理。受理完成后终止本次执行，提醒用户「规格已重定义，请重新运行本命令继续任务拆解」。若无此标记，继续步骤 4。
4. 扫描所有 `[TODO]` 任务，选取第一个满足 `Depends On` 全部为 `[IMPLEMENTED]` 的任务（`none` 或 `N/A` 视为已满足）。若所有任务均未标记 `[TODO]`，提醒用户「当前无待拆解的任务，所有 task 已完成或已定义」并终止。若存在 `[TODO]` 任务但无任何任务的依赖已满足，醒示用户「所有待拆解 task 均有未满足的前置依赖，请先完成前置 task 后再运行本命令」并终止。获取其 `change_type`。**每次执行仅处理一个任务，禁止跨 task 拆解。**
5. 将该任务拆解为实现规格。按以下顺序逐 spec 定义：
   a. **撰写 Spec**：描述做什么——指明目标文件/函数/位置及期望行为，具体到可直接生成 diff
   b. **拟订完整提案**：一次性给出三个维度的提案——
      - **Interface**：精确的函数签名/API端点/参数结构/返回值类型。Interface 必须支持依赖注入，使 Test Double 可替换生产实现。Interface 是 test-generate 和 code-generate 的共同契约
      - **Test Double**（仅当 Interface 涉及开放边界时）：参考 `architecture.md` 中 `### Test Double 策略`，提出测试替身方案（类型: Fake / Stub / Simulator）、数据结构、行为约定、暴露接口
      - **Test Cases**：基于 Interface 和 Test Double（如有）列出行为验证场景——`[自动化] <场景>` 描述 WHAT（验证什么行为，不描述 HOW）；`[人工] <场景>` 必须含操作 / 预期 / 通过标准三要素
      轻量规格可省略 Interface（须在 Spec 中说明理由）；Test Cases 允许仅含 `[人工]` 条目或构建/类型检查清单。
   c. **单次确认**：向用户展示该 spec 的整合提案，同时说明 Interface 设计理由、Test Double 设计理由（如有）、Test Cases 覆盖策略（`[自动化]` vs `[人工]` 的划分理由）。用户确认后该 spec 定稿；用户提出调整时修订提案后重新确认。
   d. **轻量声明**：`change_type` 为 chore / docs 的规格默认声明轻量（`- **轻量**: 是 — <理由>`）；其他类型若提案声明轻量，必须写明理由并经用户显式确认。
   e. 规格操作类型（新增/修改/废弃）继承自 task 的 `change_type`（映射见下表）。
6. 检查是否存在重复或冲突定义
7. 生成差异提案并输出预览（含全部 spec 的状态变更）
8. 确认后写入 specifications.md，并将该 task 下所有规格名称以子列表格式（`  - spec_name`）写入 tasks.md 该 task 的 `Specs` 字段，建立双向链接，同时将该 task 标题标记更新为 `[DEFINED]`
9. 写入后检查当前 task 下的规格状态，按 AGENTS.md §通用回滚协议 处理（回滚范围 = 当前 task 全部规格，整 task 语义）：
    - 全部规格状态为 `[新增|修改: 已定义]` 或 `[废弃: 待删除]` → 代码未产生，安全。
      提醒用户「规格已定义，请继续运行 `/test-generate` 生成测试（轻量与废弃规格跳过测试生成，可直接 `/code-generate`）」
    - 否则（代码已落地）→ 按协议汇总 `Affected Files`、执行重叠守卫、输出 `git restore --source <baseline>` 回滚指令并重置范围内规格状态，随后醒示「代码已恢复。planning/ 文档保留，请重跑下游：`/test-generate` → `/code-generate`」

### `[规格缺陷]` 规格的受理

当规格 `Issues` 以 `[规格缺陷]` 开头（由 test-verify 或 code-review 按 AGENTS.md §失败路由协议 分类）时：

1. **整 task 回滚**：按 AGENTS.md §通用回滚协议 执行，回滚范围 = 该规格所属 task 的全部规格。当前 task 已落地的代码恢复至 baseline，范围内规格状态全部重置（`[新增|修改: 已定义]`、`[废弃: 待删除]`）。规格定义文本保留。
2. **重定义目标规格**：对 `[规格缺陷]` 规格按步骤 5 重新拟定提案并确认。新定义写入后清空其 `Issues`，重置 `Retry Count` 为 0，状态保持 `[新增|修改: 已定义]` 或 `[废弃: 待删除]`。
3. **连带检查**：检查同 task 其余规格的定义在代码回滚后是否仍然成立；若其依赖被修订的 Interface 或行为，一并修订定义（定义修订无代码成本，不触发额外回滚）。
4. **task 标题**：保持 `[DEFINED]` 不变。提醒用户重跑 `/test-generate` 重启该 task 的 TDD 循环。

### 废弃规格（removal）的定义语义

废弃是行为减量，不适用测试先行。废弃规格按以下语义定义：

- **Spec**: 必须显式指明待删目标文件/函数，具体到可直接定位。禁止模糊描述（如"清理旧代码"）
- **Interface**: 语义改为「不破坏契约」——列出受影响模块与被依赖方，不再定义新接口
- **Depended By**: 必填，列出引用本规格目标文件的全部模块，作为 code-generate 解引用的依据。以 `grep` 扫描代码库实际引用方为准（此刻代码仍为废弃前状态，引用方完整）；`architecture.md` 的 `# Dependency Index` 与历史归档作起点参考，不一致时以代码实测为准
- **Test Cases**: 不要求新写 `[自动化]` 用例（可保留 `[人工]` 视觉验证：确认功能已从界面消失）。改为引用既有测试并列出验证手段清单：
  1. 受影响模块定向测试（由 `Depended By` 导出）
  2. 全量构建 / 类型检查
  3. `grep` 引用扫描（确认无残留引用）

### 共识要求（命令特有）

- Agent 自主完成规格拆解和技术方案拟订，写入文件前展示提案，等待用户确认
- ≥5 条规格/task 时醒示用户确认是否需要拆分 task
- 用户提出调整时修订后重新确认
- 通用确认规则遵循 AGENTS.md §交互协议

## 文档结构要求

命令执行后将修改 `planning/specifications.md`，结构如下。

**Frontmatter（YAML）：**

```yaml
round: <继承自 tasks.md>
```

版本历史由 git 承担，文档内不设 `version`/`created`/`updated`/`based_on` 字段。

**正文章节：**

```md
# Implementation Specs

## [新增: 已定义] name

- **Task**: Task N — 所属任务编号，建立到 task 的反向指针
- **Change Type**: feature | enhancement | bugfix | refactor | removal | chore | docs
- **轻量**: 是 — <理由>（仅轻量规格书写；chore / docs 自动轻量，其余类型须用户确认）
- **Depended By**: <引用本规格目标文件的模块列表；废弃规格必填：以 grep 扫描代码库实际引用方为准，architecture.md 的 # Dependency Index 与历史归档作起点参考>
- **Spec**: <自由描述> — 指明文件/函数/位置及期望行为，具体到可直接生成 diff
- **Interface**: <精确的函数签名/API端点/类型定义> — test-generate 和 code-generate 的共同契约（轻量规格可省略，须在 Spec 中说明）
  - 生产实现: <生产代码实现类/模块名>
- **Test Double**: `FakeXxxService`（可选，仅当 Interface 涉及开放边界）
  - **类型**: Fake | Stub | Simulator
  - **数据结构**: <内部状态存储描述>
  - **行为约定**: <模拟外部行为的具体规则>
  - **暴露接口**: <测试代码可用的额外断言方法>
- **Test Cases**: — 行为验证场景列表，基于 Interface 编写，按类型标记；`[人工]` 条目的执行结果由 code-generate 回写 `✓ PASS` / `✗ FAIL — <备注>`
  - `[自动化]` 正常路径: <描述>
  - `[自动化]` 边界条件: <描述>
  - `[人工]` 视觉验证:
    - **操作**: <用户应执行的步骤>
    - **预期**: <期望的可见/可感结果>
    - **通过标准**: <判断 PASS/FAIL 的具体条件>
- **Affected Files**: — test-generate 写入测试文件路径；code-generate / code-review 写入源文件路径；废弃规格为目标代码原始路径 + 引用方文件 + 移动后 `_deprecated/` 内路径；追加去重，每行一个
- **Issues**: — 失败时由裁决方（test-verify / code-review）写入，格式 `[实现缺陷|测试缺陷|规格缺陷] <详情>`（见 AGENTS.md §失败路由协议）；路由修复完成后清空
- **Retry Count**: 0 — test-verify 每次标记 `[新增|修改|废弃: 待修复]` 时递增；修复完成后重置为 0
```

## change_type → 操作类型映射

specification-define 是唯一负责将 task 的 `change_type` 映射为规格标题操作类型的入口。映射关系如下：

| task change_type | 规格操作类型 |
|------------------|------------|
| feature          | 新增       |
| enhancement      | 修改       |
| refactor         | 修改       |
| bugfix           | 修改       |
| removal          | 废弃       |
| chore / docs     | 按实际为新增或修改，规格默认声明轻量 |

规格标题格式 `[操作类型: 状态]`，操作类型由此映射确定，状态由流水线下游命令按 AGENTS.md §状态机与记法规范 推进。

## 输入来源

本命令的输入从上游文档读取，无需额外 JSON 输入。

### 上游文档

| 文件 | 用途 | 关键内容 |
|------|------|---------|
| `planning/tasks.md` | 任务列表 | 按 `## Task N` 标题顺序扫描，选取第一个标题标记 `[TODO]` 且依赖满足的任务，从详情字段提取 `Change Type`、`Depends On`、`Description`。不跨 task 处理。 |
| `planning/architecture.md` | 依赖索引 | `# Dependency Index` 章节，作为推导规格 `Depended By` 的起点参考；废弃规格最终以 `grep` 代码扫描实际引用方为准 |
| `planning/specifications.md` | 既有规格 | 检查 `Issues` 是否含 `[规格缺陷]` 标记（优先受理）；重复/冲突检查 |

### 状态过滤

忽略标题标记为 `[DEFINED]` 或 `[IMPLEMENTED]` 的任务。从 `[TODO]` 任务中选取第一个 `Depends On` 全部满足（前置 task 均为 `[IMPLEMENTED]`；`none` / `N/A` 视为已满足）的进行处理。每次执行仅处理一个 task。`[规格缺陷]` 受理不受此过滤限制。

### 处理顺序

每次执行先受理 `[规格缺陷]` 规格（如有）；否则选取第一个满足依赖的 `[TODO]` 任务。再次执行时自动定位下一个满足依赖的 `[TODO]` 任务。无满足依赖的任务时醒示用户先完成前置 task。

### 上游文档解析提示

- `planning/tasks.md` 各任务以 `## Task N: <name> [STATUS]` 为标题，任务详情以 `- **字段名**: 值` 格式书写
- `Change Type`、`Depends On`、`Specs`、`Description` 为每个任务节的必填字段

## 输出

命令执行后：写入/修订 `planning/specifications.md`，更新 tasks.md 的 `Specs` 双向链接与任务标记；必要时按 AGENTS.md §通用回滚协议 输出回滚指令并重置状态。

### 交互预览

执行时以 diff 格式展示修改提案，包含规格分析、Interface 设计理由、Test Double 设计理由（如有）、Test Cases 覆盖策略（`[自动化]` vs `[人工]` 的划分理由）、轻量声明理由（如有）与状态变更。

## 下一步

规格已定义后，提醒用户运行 `/test-generate` 生成测试代码（TDD: 测试先行）；轻量与废弃规格跳过测试生成，直接 `/code-generate`。

## 约束

- 只允许对 TODO 状态任务进行规格拆解（`[规格缺陷]` 受理除外）
- 选取 TODO 任务时必须检查 `Depends On` 字段：前置依赖（除 `none` / `N/A` 外）必须全部 `[IMPLEMENTED]` 才可拆解。若所有 TODO 任务的依赖均未满足，醒示用户先完成前置 task 并终止
- 规格的 `Change Type` 必须与关联 task 的 `change_type` 一致
- 不允许重复定义相同规格
- 每个常规规格（非轻量）必须：
   - 单一目的——可用一句话概括修改内容（不跨 concern）
   - 一屏可审——Agent 可在一次回复轮次中理解并生成差异
   - 包含 `Interface` 字段——精确的函数签名/API契约/类型定义
   - Interface 必须支持依赖注入，使 Test Double 可替换生产实现
   - 包含 `Test Cases` 字段——至少覆盖正常路径、边界条件、异常输入
   - Test Cases 中每个条目必须标记 `[自动化]` 或 `[人工]`
   - `[人工]` 条目必须包含操作 / 预期 / 通过标准三要素
- 轻量规格的放宽项：可省略 Interface（须在 Spec 中说明）；Test Cases 允许仅含 `[人工]` 条目或构建/类型检查清单。轻量声明不得由 agent 自行降级产生——chore / docs 自动轻量，其余类型须写明理由并经用户确认
- 若 Interface 涉及开放边界，必须设计 Test Double
- `[人工]` 仅用于以下场景：
   1. 被测行为无法通过代码断言验证（视觉正确性、交互手感）
   2. 真实系统集成的残余验证——Test Double 已覆盖核心逻辑，`[人工]` 仅验证残余差异
- `[自动化]` 与 `[人工]` 是叠加关系（双保险），非互斥
- 非必要不添加 `[人工]` 条目——优先通过 Interface 设计（依赖注入 + Test Double）使行为可自动化验证
- 一个 Task 的规格数 ≥ 5 时，醒示用户确认是否需要合并或进一步拆分 task
- 不允许模糊操作（如"改进代码"、"优化性能"——须指明具体文件、函数及期望行为）
- 不包含任何实现代码
- 必须能映射回任务
- 必须将所有规格名称写回 tasks.md 中对应 task 的 `Specs` 字段，建立双向链接
- 废弃规格（removal）附加约束：
   - `Spec` 必须显式指明待删目标文件/函数，禁止模糊描述
   - `Depended By` 必填，且须与 `grep` 代码扫描的实际引用方一致（`architecture.md` 的 `# Dependency Index` 作起点参考，二者不一致时以 grep 实测为准）
   - `Test Cases` 不要求新写用例，为验证手段清单（定向测试 / 全量构建 / grep 扫描），不受上述 `[自动化]`/`[人工]` 标记要求约束
