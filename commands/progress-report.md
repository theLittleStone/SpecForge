---
name: progress-report
description: 汇总当前任务进度，按变更类型分组统计，并在完成时执行归档与总结
---

## 功能说明

该命令用于读取 `planning/` 目录中的所有规划文件，分析当前项目进度，并根据状态执行报告或收尾操作。

## 行为规则

执行流程：

1. 检查 `planning/` 是否存在
2. 读取以下文件（如果存在）：
   - requirement.md
   - architecture.md
   - tasks.md
   - specifications.md
   - style_guide.md
3. 聚合状态信息，按 `change_type` 分组统计
4. 判断当前进度状态：
   - 无任务
   - 未完成
   - 已完成
5. 根据状态执行对应操作

## 状态判断

### 无任务

- planning 目录不存在或为空

### 未完成

存在以下任一情况：

- spec 状态为 `[新增: 已定义]` / `[新增: 测试已生成]` / `[新增: 已实现]` / `[新增: 已测试]` / `[新增: 待修复]`
- spec 状态为 `[修改: 已定义]` / `[修改: 测试已生成]` / `[修改: 已实现]` / `[修改: 已测试]` / `[修改: 待修复]`
- spec 状态为 `[废弃: 待删除]` / `[废弃: 测试已生成]`
- task 状态为 TODO 或 DEFINED

### 已完成

满足以下全部条件：

- 所有 `[新增: xxx]` 规格为 `[新增: 已验证]`
- 所有 `[修改: xxx]` 规格为 `[修改: 已验证]`
- 所有 `[废弃: xxx]` 规格为 `[废弃: 已删除]`
- 所有 task 为 IMPLEMENTED

## 输出

根据 `planning/` 目录中各文件的状态信息，输出当前进度报告；全部完成时执行归档。

### 未完成状态

#### 行为

默认不执行归档，提醒用户继续运行流水线。

若用户明确要求强制归档：
1. 生成 `planning/summary.md`，标注「未完成」
2. 将 `planning/` 下所有文件（除 `archive/` 目录外）**复制**（非移动）到 `planning/archive/planning_{round}/`
3. 追加 `./changelist.md` 条目，标注 `(incomplete)`
4. 清空 `planning/` 目录（除 `archive/` 外），允许新轮次开始

#### Markdown（用户可读）

**注意：不执行归档操作。** 检测到未完成文档，提醒用户继续运行流水线（如 `/specification-define`、`/test-generate`、`/code-generate`、`/test-verify`、`/code-review`）。

```md
## 当前进度

- 规格完成率：XX%
- 任务完成率：XX%

## 按变更类型统计

| 变更类型 | 总数 | 已完成 | 进度 |
|----------|------|--------|------|
| 新增     | X    | X      | XX%  |
| 修改     | X    | X      | XX%  |
| 废弃     | X    | X      | XX%  |

## 未完成规格
- [新增: 已定义] spec_a
- [新增: 测试已生成] spec_b
- [修改: 已实现] spec_c
- [新增: 待修复] spec_d
- [废弃: 待删除] spec_e

## 未完成任务
- Task 1
- Task 2

## 下一步建议
...
```

### 已完成状态

#### 执行操作

1. 生成总结文件 `planning/summary.md`
2. 创建 `planning/archive/planning_{round}/` 子目录（`round` 取自 `planning/requirement.md` frontmatter）
3. 将 `planning/` 下所有文件（除 `archive/` 目录外）移入该子目录
4. 在项目根目录 `./changelist.md` 中追加当前轮次条目（格式详见「Changelist 格式」章节）
   - 若 `./changelist.md` 不存在则新建
   - 条目包含：`round`（标题 `## Round <round>`）、`change_type`、本轮主要变动摘要（2-5 条，提取自 `planning/requirement.md` 的 `# Goal` 章节）、Archive 路径指针
5. 可选执行 git 提交

#### Markdown

```md
## 项目已完成

所有任务与规格均已实现并验证。

## 变更类型汇总

| 变更类型 | 完成项数 |
|----------|---------|
| 新增     | X       |
| 修改     | X       |
| 废弃     | X       |

## 总结
...

## 归档位置
planning/archive/planning_XXXX/
```

### Changelist 格式

`./changelist.md` 是跨轮次持久文件，每次归档后追写一轮。格式如下：

```md
---
round: 20260507-0138
version: 1
created: 2026-05-07
updated: 2026-05-07
---

# Changelist

## Round 20260507-0138 [feature]
- **Changes**:
  - 新增模板渲染引擎
  - 新增请求参数校验模块
- **Archive**: planning/archive/planning_20260507-0138/

## Round 20260507-1423 [enhancement]
- **Changes**:
  - 支持批量删除用户
  - 修改日志格式为 JSON
- **Archive**: planning/archive/planning_20260507-1423/
```

### 无任务状态

#### Markdown

```md
当前无进行中的任务。
```

## 约束

- 状态判断必须基于 tasks.md 和 specifications.md
- 不允许仅根据文件存在判断完成
- 归档操作必须保留历史数据
- 不允许删除未归档的 planning 内容
- 不允许删除 `./changelist.md`
- 完成标准必须同时满足所有 change_type 的终态条件
