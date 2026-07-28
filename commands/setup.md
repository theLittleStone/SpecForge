---
name: setup
description: 初始化项目 AGENTS.md，写入流水线行为约束规则
---

## 功能说明

该命令用于在当前 OpenCode 工作目录下创建 `AGENTS.md` 文件，写入流水线全局行为约束规则。
OpenCode 启动时自动读取工作目录下的该文件并插入上下文。

## 行为规则

执行流程：

1. 检查当前 OpenCode 工作目录下的 `AGENTS.md` 是否存在
2. 若不存在 → 直接写入
3. 若存在 → 询问用户意图：
   - 覆盖 → 用新内容替换
   - 跳过 → 不操作
   - 追加 → 在当前文件末尾追加规则内容
4. 写入后告知用户「AGENTS.md 已就绪」

## 输出

写入当前 OpenCode 工作目录下的 `AGENTS.md`，内容如下：

```
## 交互协议
- 任何文件变更前先输出 diff 预览，等待用户确认后写入。
- 用户拒绝时回到生成步骤修订——拒绝不是终点。

## 安全基线
- 不编造需求、规格或 API 用法。不确定时搜代码库或问用户。
- 知识盲区用 grep/glob/websearch，必要时调 knowledge-augment 或 style-resolver。

## 执行约束
- 一次只处理一个 task。活跃 task = tasks.md 首个 [DEFINED]。
```

## 约束

- 不允许写入除以上 3 节外的额外规则
- 写入前必须展示内容预览，等待用户确认
