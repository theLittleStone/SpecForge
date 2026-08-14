---
name: setup
description: 初始化项目 AGENTS.md，写入流水线全局行为约束与工作流共识规则
---

## 功能说明

该命令用于在当前 OpenCode 工作目录下创建 `AGENTS.md` 文件，写入流水线全局行为约束与工作流共识规则。
OpenCode 启动时自动读取工作目录下的该文件并注入每个会话上下文，作为全部流水线命令的公共共识。

共识共 8 节：交互协议、安全基线、执行约束、状态机与记法规范、失败路由协议、通用回滚协议、人工验证落地规则、轻量 spec 规则。各命令文件不再重复这些规则，仅引用本节名称。

## 行为规则

执行流程：

1. 检查当前 OpenCode 工作目录下的 `AGENTS.md` 是否存在
2. 若不存在 → 直接写入
3. 若存在 → 询问用户意图：
   - 覆盖 → 用新内容替换
   - 跳过 → 不操作
   - 追加 → 在当前文件末尾追加规则内容
4. 写入后告知用户「AGENTS.md 已就绪。所有流水线命令依赖其中的共识内容」

## 输出

写入当前 OpenCode 工作目录下的 `AGENTS.md`，内容如下：

```
## 交互协议
- 任何文件变更前先输出 diff 预览，等待用户确认后写入。
- 用户拒绝时回到生成步骤修订——拒绝不是终点。
- 确认点只针对可执行制品（提案、diff、重构方案）；不设「确认理解」类门控。

## 安全基线
- 不编造需求、规格或 API 用法。不确定时搜代码库或问用户。
- 知识盲区用 grep/glob/websearch，必要时调 knowledge-augment 或 style-resolver。

## 执行约束
- 一次只处理一个 task。活跃 task = tasks.md 首个 [DEFINED]。
- 变更代码后同步维护模块引用索引（architecture.md 的 `# Dependency Index` 与规格的 `Depended By`；废弃规格的 `Depended By` 以 grep 实际引用扫描为准）。废弃模块先解引用再移入 `_deprecated/`，禁止直接删除。
- 轮内禁止 git 提交。git 提交仅由 progress-report 在归档收尾后执行。

## 状态机与记法规范
- 规格标题标记采用 `[操作类型: 状态]` 完整格式，共 16 态。状态机是跨 agent 的交接契约语言：每个命令读取规格状态决定行为，状态只由拥有它的命令推进。

| 状态 | 设置者 | 含义 |
|------|--------|------|
| `[新增: 已定义]` | specification-define | 规格已定义，待生成测试（轻量规格待直接实现） |
| `[新增: 测试已生成]` | test-generate | 测试已生成并确认 RED，待实现 |
| `[新增: 已实现]` | code-generate | 代码已实现，待验证 |
| `[新增: 已测试]` | test-verify | 已通过测试，待审查 |
| `[新增: 待修复]` | test-verify / code-review | 验证未通过，按失败路由协议返工 |
| `[新增: 已验证]` | code-review | 已通过审查（终态） |
| `[修改: 已定义]` | specification-define | 修改规格已定义，待生成测试（轻量规格待直接实现） |
| `[修改: 测试已生成]` | test-generate | 修改测试已生成并确认 RED，待实施 |
| `[修改: 已实现]` | code-generate | 修改已实施，待验证 |
| `[修改: 已测试]` | test-verify | 修改已通过测试，待审查 |
| `[修改: 待修复]` | test-verify / code-review | 修改验证未通过，按失败路由协议返工 |
| `[修改: 已验证]` | code-review | 修改已通过审查（终态） |
| `[废弃: 待删除]` | specification-define | 已定义（含待删目标与 Depended By），待解引用 |
| `[废弃: 已解引用]` | code-generate | 引用方已全部移除，代码已移入 `_deprecated/`，验证清单通过 |
| `[废弃: 待修复]` | test-verify / code-review | 解引用验证未通过，按失败路由协议返工 |
| `[废弃: 已删除]` | progress-report | `_deprecated/` 代码已物理删除（收尾），终态 |

- 记法：写入 planning/ 文档的状态必须写完整格式，不得省略操作类型。竖线写法 `[新增|修改: x]` 表示操作类型并列枚举，仅用于本文件与命令文件内部的行文，不得写入 planning/ 文档。
- 轻量规格与常规规格共用本状态表，仅跳过 test-generate 阶段。

## 失败路由协议（Issues 三分类）
- 失败信息统一写入规格的 `Issues` 字段，格式 `[类型] <失败详情>`。类型三选一，决定返工路由：
  - `[实现缺陷]` → 运行 `/code-generate` 修复实现
  - `[测试缺陷]` → 运行 `/test-generate` 修复测试代码
  - `[规格缺陷]` → 运行 `/specification-define` 重定义规格
- 裁决方（test-verify / code-review）写入 Issues 时必须同时分类并给出路由建议；实现方（code-generate）仅在举证且用户确认后调整分类。
- 实现方永不修改测试代码。疑似测试错误只能分类路由。
- Retry Count ≥ 3 时必须三选一路由，禁止盲目重试。

## 通用回滚协议
上游命令修改规划文档/规格且下游已产生代码时，按本协议回滚：
- 回滚范围：requirement / architecture / task 级修改 = 本轮全部规格；specification 级重定义（含 `[规格缺陷]` 路由）= 当前 task 全部规格（整 task 回滚）。
1. 汇总回滚范围内全部规格的 `Affected Files`，去重。
2. 重叠守卫：与回滚范围**之外**所有 `[新增|修改: 已验证]`、`[废弃: 已解引用]` 规格的 `Affected Files` 求交集。
   - 无交集 → 输出 `git restore --source <baseline> <file1> <file2> ...`（baseline 取自 requirement.md frontmatter；若为 none/空，省略 `--source` 退化为默认 HEAD）
   - 有交集 → 交集文件禁止自动回滚，列出并醒示「该文件含已验证规格的改动，请人工精确回退本次变更的片段；重新实现后由 /code-review 影响范围回归重新验证」
3. 过滤 `planning/` 目录下的文件与 `_deprecated/` 内路径。
4. 特殊处理：范围含 `[废弃: 已解引用]` → 另需删除项目根目录 `_deprecated/` 及其中的移动副本；含 `[废弃: 已删除]` → 代码已物理删除，须用 `git log --diff-filter=D` 从 git 历史定位找回；git 未跟踪的新文件 → 提醒手动删除。
5. 状态重置：回滚范围内的规格全部重置为初始状态（`[新增|修改: 已定义]`、`[废弃: 待删除]`）；同时按级别重置 tasks.md 的 task 标记——requirement / architecture / task 级回滚将全部 task 标记重置为 `[TODO]`（任务描述保留，重跑 `/task-breakdown` 与 `/specification-define` 时复核修订，既有规格以 `[已定义]` 状态在原地修订，不得重复新建）；specification 级回滚（整 task）task 保持 `[DEFINED]`。planning/ 文档保留，提醒重跑下游。

## 人工验证落地规则
- `[人工]` 测试条目由用户在 code-generate 阶段逐项执行，code-generate 必须把结果回写 specs.md 对应 Test Cases 条目：通过追加 `✓ PASS`，失败追加 `✗ FAIL — <失败备注>`（含用户反馈）。
- 结果未落地，规格状态不得推进；code-review 以落地标记为人工验收的唯一依据。

## 轻量 spec 规则
- 声明：规格含 `- **轻量**: 是 — <理由>`。change_type 为 chore / docs 时自动轻量；其他类型声明轻量必须说明理由并经用户确认；agent 不得自行降级。
- 流转：跳过 test-generate，`[新增|修改: 已定义]` → /code-generate 实现 → /test-verify 轻量验收 → /code-review 审查。
- 轻量规格可省略 Interface（须在 Spec 中说明）；Test Cases 允许仅含 `[人工]` 条目或构建/类型检查清单。
- 轻量验收 = 全量构建 + 类型检查（若适用）+ `[人工]` 条目全部落地为 `✓ PASS`。
```

## 约束

- 不允许写入除以上 8 节外的额外规则
- 写入前必须展示内容预览，等待用户确认
