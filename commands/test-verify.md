---
name: test-verify
description: 独立测试运行与验证器——执行测试并根据调用上下文判断 PASS/FAIL，更新规格状态
---

## 功能说明

该命令是 TDD 流程中的独立验证节点，被 test-generate、code-generate、code-review 多阶段复用。
它不生成任何代码，只执行测试并基于调用上下文判断结果。

## 行为规则

执行流程：

1. 读取 `planning/specifications.md`，定位当前活跃 task（`planning/tasks.md` 中首个 `[DEFINED]` 的 task）
2. 识别需要验证的规格——根据调用上下文和规格当前状态确定：

   | 调用上下文 | 目标规格状态 | 期望结果 | 通过后新状态 | 失败后新状态 |
   |-----------|-------------|---------|------------|------------|
   | test-generate（RED 确认） | `[新增: 已定义]`、`[修改: 已定义]` | **FAIL** | —（仅确认 RED，不更新状态） | —（醒示 agent） |
   | code-generate（GREEN 验证 — 新增/修改） | `[新增: 已实现]`、`[修改: 已实现]` | **PASS** | `[已测试]` | `[待修复]` |
   | code-generate（GREEN 验证 — 废弃） | `[废弃: 已解引用]` | **PASS** | —（保持已解引用） | `[废弃: 待修复]` |
   | code-review（影响范围回归） | 当前 task `[已测试]`/`[废弃: 已解引用]` + 文件重叠的 `[已验证]` 规格 | **PASS** | —（审查结果由 code-review 裁决） | —（醒示 code-review） |

   **RED 确认规则**：RED 上下文（test-generate 调用）仅接受「断言失败」作为预期 FAIL。若失败结果全部为 error（测试无法编译/加载/运行），不确认 RED，醒示 agent「测试代码可能无法编译或运行（可能缺少 stub 占位），请检查后重试」，不更新规格状态。

3. 检测测试框架（同 test-generate 逻辑）
4. 执行目标 spec 的测试（从 `Affected Files` 字段获取测试文件路径）。废弃规格改为执行验证清单（详见「废弃规格的验证」）
5. 分析测试结果：
   - 统计通过/失败/跳过的用例数
   - 将失败结果分类为「失败类型」：
     - **断言失败**：测试运行成功、断言不满足（runner 的 failure 类别）
     - **error**：编译/加载/运行异常，断言未执行（runner 的 error、套件加载失败、编译错误等）
   - 提取失败用例的具体断言信息（预期值 vs 实际值、错误堆栈）
6. 根据上表映射更新规格状态，失败时将详情（含失败类型）写入 `Issues` 字段，并递增 `Retry Count`

### 废弃规格的验证

对 `[废弃: 已解引用]` 规格（code-generate GREEN 验证上下文），不执行单元测试，而是执行验证清单：

1. 受影响模块定向测试（由 spec 的 `Depended By` 导出，从 `Affected Files` 获取测试路径）
2. 全量构建 / 类型检查（确认无悬空引用）
3. `grep` 引用扫描（确认待删符号无残留引用）

全部通过 → PASS；任一失败 → FAIL，将失败详情与受影响模块写入 `Issues`。

### 失败处理细则

#### 当 code-generate（GREEN 验证）FAIL 时：

1. 将失败信息写入 spec 的 `Issues` 字段：
   - 失败类型（断言失败 / error）
   - 错误信息（断言失败：预期值 vs 实际值；error：编译/加载/运行错误详情）
   - 失败用例名称
2. 递增 `Retry Count`
3. 状态更新为 `[新增: 待修复]` 或 `[修改: 待修复]`
4. 醒示用户「测试未通过，请运行 `/code-generate` 根据 Issues 修复代码」
5. Retry Count ≥ 3 时额外醒示：「该规格已连续失败 3 次，建议检查测试用例是否正确或重新评估实现方案」

#### 当 code-review（影响范围回归）FAIL 时：

1. 不自动更新 spec 状态
2. 醒示用户「回归测试失败——重构可能引入了意外行为变更，请检查最近修改」

### 废弃规格的特殊处理

对于 `[废弃: 已解引用]` 状态的规格：
- 调用来源为 code-generate（GREEN 验证）：
  - PASS → 保持 `[废弃: 已解引用]`，确认解引用与移动安全
  - FAIL → 状态更新为 `[废弃: 待修复]`，写入 Issues（验证清单失败详情 + 受影响模块），递增 Retry Count。
    醒示用户「解引用验证未通过，请运行 `/code-generate` 根据 Issues 修复受影响模块」
- 调用来源为 code-review（影响范围回归）：
  - PASS → 保持 `[废弃: 已解引用]`
  - FAIL → 醒示用户「回归测试失败——解引用可能存在遗漏，请检查最近修改」

## 输入来源

| 文件 | 用途 | 关键内容 |
|------|------|---------|
| `planning/specifications.md` | 实现规格 | 目标 spec 的 `Affected Files`（测试文件路径；废弃规格为引用方、目标代码原始路径与 `_deprecated/` 内文件路径）、`Test Cases`、`Issues` |
| `planning/tasks.md` | 任务列表 | 定位当前活跃 task（首个 `[DEFINED]`） |

### 执行模式与处理顺序

按调用上下文确定执行范围：

| 调用上下文 | 执行范围 |
|-----------|---------|
| test-generate（RED 确认） | 当前活跃 task 的目标 spec |
| code-generate（GREEN 验证） | 当前活跃 task 的目标 spec |
| code-review（影响范围回归） | 当前活跃 task 全部规格 + `Affected Files` 与当前 task 重叠的规格（含 `[已验证]` 终态） |
| progress-report（收尾全量回归） | `planning/specifications.md` 全部规格（含 `[已验证]` 终态） |

执行范围内按 `planning/specifications.md` 中规格从上到下顺序依次验证。

#### 影响范围回归算法

用于 code-review（影响范围回归）上下文：

1. 汇总当前活跃 task 全部规格的 `Affected Files`（含 code-review 重构新增的路径），去重
2. 扫描 `planning/specifications.md` 中所有规格（含 `[已验证]` 终态规格）的 `Affected Files`
3. 筛选与步骤 1 集合存在任一路径重叠的规格，去重
4. 依次执行这些规格的测试（废弃规格执行验证清单）
5. 任一规格 FAIL → 整体回归 FAIL，报告失败详情与受影响规格清单

#### 全量回归

`planning/specifications.md` 中全部规格的测试，由 `progress-report` 收尾硬门触发（见 progress-report「已完成状态」）。

## 输出

执行测试并更新规格状态。输出格式：

```md
## 测试执行结果

| 规格 | 测试文件 | 通过 | 失败 | 失败类型 | 跳过 | 结果 |
|------|---------|------|------|---------|------|------|
| spec_a | src/__tests__/a.test.ts | 3 | 0 | - | 0 | ✓ PASS |
| spec_b | src/__tests__/b.test.ts | 1 | 2 | 断言失败 | 0 | ✗ FAIL |
| spec_c | src/__tests__/c.test.ts | 0 | 0 | error（编译失败） | 0 | ✗ error |

（废弃规格为验证清单结果：定向测试 / 全量构建 / grep 引用扫描，非单元测试统计）

（code-review 影响范围回归时，表格列出当前 task 全部规格 + 文件重叠的已验证规格）

## 失败详情

### [新增: 待修复] spec_b（失败类型：断言失败）
- **Issues**:
  - 测试「密码错误返回 401」失败：预期 401，实际 500
  - 测试「账户锁定返回 423」失败：预期 423，实际 200
- **Retry Count**: 2

## 下一步

- PASS → 提醒用户继续当前流程的下一步
- FAIL → 提醒用户运行 `/code-generate` 修复后重新运行本命令
```

## 约束

- 不生成任何代码或测试文件
- 仅执行验证，不修改测试代码
- 不允许跳过失败分析直接标记状态
- RED 确认上下文仅接受「断言失败」为预期 FAIL；error 不得确认 RED，须先解决编译/加载/运行问题
- 废弃规格验证失败时必须详细列出失败详情与受影响模块
- 新增/修改规格仅执行 `[自动化]` 测试（Test Double 对 test-verify 透明）；废弃规格执行验证清单（定向测试 + 全量构建 + grep 引用扫描）；均不涉及 `[人工]` checklist
