# SpecCaliber × pi 迁移规划（ARCH）

> 状态：规划定稿，待开工 P1（2026-08-30）
> 目标：把 SpecCaliber 流程从**纯手工推进**改为**程序（spec-caliber host CLI）自驱推进**；平台从 opencode **彻底迁移到 pi**，弃用 opencode。
> 前置事实：opencode 版 SpecCaliber 现状（5 命令 + explorer 子 agent + AGENTS.md 八节共识）已在 README/命令文件中文档化，本规划以该行为为准。

---

## 1. 背景与目标

- 现状：流程靠人手工逐条发 `/plan` → `/specify` → `/execute` → `/wrap-up`；守卫（预算、振荡、三相位、文件域）是提示词级自律；T/C 对抗在一场对话内同一 agent 先后扮演（对抗模型退化风险）。
- 目标：
  1. **spec-caliber host**（TypeScript CLI）作为只读驱动器：解析状态 → 查路由表定下一步 → 派生 pi 角色会话 → 机械守卫 → 干预点交还用户 → 证据日志。
  2. **守卫机械化**：attempt≤4、测试重写≤2、TDD 循环≤4、振荡检测、三相位门、文件域（T 不写 src / C 不写测试）、轮内禁 git commit——全部代码强制。
  3. **T/C 拆独立会话**：T 会话与 C 会话是两个独立 pi 进程，验收跨边界走文件，对抗模型结构性成立。
  4. **会话可接管**：任何角色会话都是 pi 会话文件，用户可随时 `pi --session <file>` 进 TUI 手动接管。
  5. Markdown 仍是唯一事实源：host 每步从文档重推状态，无独立可变计数器 → 无漂移。

---

## 2. 决策锁定（D1–D9）

| # | 决策 | 结论 |
| --- | --- | --- |
| D0 | 项目改名 | SpecForge → **SpecCaliber**（原名与既有项目撞名；候选名逐一核查 npm/GitHub 占名后于 2026-08-30 定稿：裸名空、"caliber"=口径/品质水准，覆盖精密·准确·可靠三意象）；品牌展示名 `SpecCaliber`，代码级 token 一律 `spec-caliber`（npm 包 / CLI 二进制 / 配置目录 / 路径） |
| D1 | 迁移节奏 | 先 P1（纯 pi 手工移植等价检查点）再上 host |
| D2 | host 驱动形态 | RPC 子进程：spec-caliber 派生 `pi --mode rpc`，JSONL 双向协议 |
| D3 | spec-caliber 交互形态 | 常驻 TUI；step / auto 同进程两档 |
| D4 | 分发模型 | 单 npm 包发布；pi 资源（prompts/extensions）经 `pi install` 由 pi 管理；host 自带资源不依赖安装状态 |
| D5 | pi 会话存放 | 与 pi 统一会话存储放在一起：`.pi/sessions/`（`.pi/settings.json` 设 `sessionDir: ".pi/sessions"`） |
| D6 | 项目文档布局 | 当前轮次文档入 `.spec-caliber/`；归档统一入 `.archive/`（替代 `planning/` + `planning/archive/`） |
| D7 | 信任模型 | host 只读：不写 plan.md/specs.md，一切状态写入由角色会话执行；host 拥有"决定下一步 + 拒绝越界 + 交还用户" |
| D8 | 守卫双层 | spec-caliber 步前门（超预算拒派/强制 L3）+ 会话内角色 extension（`tool_call` 拦截、appendEntry 记录） |
| D9 | T/C 分离时点 | P1 手工模式保持单会话双视角（与现状等价）；P3 随 host 落地拆独立会话 |

---

## 3. pi 能力盘点（与本迁移相关的接口）

**运行模式**（二进制 `pi`，npm 包 `@earendil-works/pi-coding-agent`，Node ≥ 22.19，当前基线 v0.84.4）：

- 交互式 TUI（默认）；`-p` print 单发；`--mode rpc`（JSONL 命令+事件流+**extension UI 子协议**）；`--mode json` 单向事件流。
- 关键 CLI：`--tools` / `--exclude-tools`（严格工具白名单）、`--system-prompt` / `--append-system-prompt`、`--session` / `--no-session` / `--session-dir` / `--name`、`-e` 扩展、`--model` / `--thinking`、`--approve`（project trust）。
- 非交互模式（rpc/json/-p）无 trust 弹窗，由 `defaultProjectTrust` + `--approve` 控制——headless 运行必须显式配置。

**Extension 机制（本迁移的核心）**：

- `pi.on("tool_call")`：可拦截/改写工具调用（`{block: true, reason, terminate}`；`event.input` 可变）→ 文件域与命令策略的机械闸门。
- `pi.appendEntry(customType, data)`：会话级持久状态（不进 LLM 上下文、重启可恢复）→ PTS 相位/事件日志落点。
- `pi.registerTool` / `pi.registerCommand` / `pi.setActiveTools` / `pi.exec` / `ctx.ui`（select/confirm/input/notify）。
- **无内置 subagent**（官方设计哲学）；官方模式 = 派生 `pi --mode json -p` 子进程 + Markdown agent 定义（frontmatter: name/description/tools/model），官方示例 `examples/extensions/subagent/`。

**资源与约定**：

- Prompt templates：`~/.pi/agent/prompts/`、`.pi/prompts/`、包内 `prompts/` 或 `pi.prompts`——文件名即命令名（opencode commands 的直接映射）。
- 包（pi packages）：`pi install npm:...` 安装 extensions/prompts/skills；`pi config` 显隐、`pi update` 升级、`pi list` 列包；包声明 `keywords: ["pi-package"]` + package.json `pi` 清单。
- AGENTS.md 原生加载（cwd 向上查找）——八节共识无需改动。
- 会话即文件（JSONL 树）：`pi --session <file>` 恢复/接管/分支。
- Windows：默认 Git Bash；`sessionDir` 支持相对路径（`.pi/sessions`）；优先级 `--session-dir` > 环境变量 > settings.json。

---

## 4. 目标架构

```
            ┌──────────────────────────────────────────────┐
            │   spec-caliber（host CLI，TypeScript，只读驱动器）      │
            │  解析状态 → 路由表定下一步 → 派生角色会话 →        │
            │  机械守卫 → 干预点交还用户 → 证据日志             │
            └───┬────────┬────────┬────────┬────────┬───────┘
        RPC/JSONL 子进程（pi --mode rpc，自带 -e 角色 extension）
   ┌────────┬───┴────┬───┴────┬───┴────┬───┴─────┬─────────┐
   ▼        ▼        ▼        ▼        ▼         ▼
 planner  specify    T 视角    C 视角   explorer  wrap-up
 写plan.md 写specs.md 只写测试  只写实现  只读      回归+证据
```

- **spec-caliber 只读**：不写 plan/specs；角色会话写文档（经用户确认的预览）。
- **T/C 分离**：独立会话、独立上下文；C 审 T 的测试、T 审 C 的实现，均为跨进程边界动作。
- **守卫双层**：spec-caliber 步前门（预算/振荡/三相位）+ 会话内 extension（tool_call 文件域拦截、禁 git commit、appendEntry 记录）。

---

## 5. 文件布局

### 5.1 SpecCaliber 仓库（= npm 包 `spec-caliber`）

```
SpecCaliber/
├── package.json          # bin: spec-caliber；pi: { prompts, extensions }；keywords: ["pi-package"]；engines: node>=22.19
├── prompts/              # setup / plan / specify / execute / wrap-up
│                         #   pi install 后成为 /setup /plan /specify /execute /wrap-up（手工路径）
│                         #   同时也是 spec-caliber 派生角色会话时的角色提示词源（单一事实源）
├── extensions/           # 子 agent 派生器（P1）；角色守卫 extension（P3，tool_call 拦截 + 事件记录）
├── agents/               # explorer.md（pi agent 格式）
├── src/                  # spec-caliber host 源码（P2 起）：core / driver / tui / cli
├── deprecated/           # 原 commands/_deprecated（不再加载，留档）
├── docs/                 # 原 archive/（AFTERREAD、AFTERREAD2 研究文档）
├── ARCH.md               # 本文档
└── README.md / README-zh.md
```

### 5.2 目标项目（使用 SpecCaliber 的项目）

```
<target project>/
├── AGENTS.md             # 八节共识（pi 原生加载；路径不变）
├── style_guide.md        # 跨轮次持久（根目录，不变）
├── changelist.md         # 项目级变更日志（根目录，不变）
├── .pi/                  # pi 平台自身的配置与会话
│   ├── settings.json     # sessionDir: ".pi/sessions"（+ 其他平台配置）
│   └── sessions/         # 统一会话存储：全部角色会话 JSONL（由 pi 管理/恢复/接管）
├── .spec-caliber/               # 当前轮次工作区（本轮文档 + 机器状态）
│   ├── plan.md           # 计划图（本轮）        ← 提交版本控制
│   ├── specs.md          # 规格契约（本轮）      ← 提交版本控制
│   ├── summary.md        # 收尾总结（wrap-up 生成）← 提交版本控制
│   ├── events/           # 事件日志 *.jsonl      ← gitignore
│   ├── registry.json     # 角色 → 会话文件映射    ← gitignore
│   └── config.json       # 轮次配置：测试命令、每角色模型等 ← 提交版本控制
└── .archive/             # 归档统一存放
    └── planning_{round}/ # 每轮：plan.md + specs.md + summary.md + 事件证据
```

要点：

- `.pi/sessions/` 与 pi 的其他会话文件统一存放（D5）；`.spec-caliber/` 只放本轮文档与机器状态（D6）；归档全部进 `.archive/`（D6）。
- 机器生成物（会话、事件日志、注册表）gitignore；文档（plan/specs/summary/config）提交。
- style_guide.md / changelist.md 是跨轮次/项目级产物，保留根目录（假设 A2）。

---

## 6. 自驱流程纲要

```
【轮次启动】
 spec-caliber: AGENTS.md 八节检查（缺 → 提示运行 setup，终止）
 spec-caliber: .spec-caliber/plan.md 不存在 → 派生 planner 会话（模式 A 角色提示词）
         苏格拉底四问（spec-caliber 中继用户交互）→ 探索（自做 / 派生 explorer 子进程）
         → 复杂度评估 → 计划图预览 → 用户确认 → 写 .spec-caliber/plan.md

【工作项循环】（每次取活跃工作项：Depends On 全 DONE 的首个 TODO/ACTIVE）
 spec-caliber: 解析 .spec-caliber/plan.md + .spec-caliber/specs.md → 状态路由表 → 下一步

  工作项 TODO ──► specify 会话：规格展开预览 → 用户确认 → 写 .spec-caliber/specs.md（TODO→ACTIVE）
                  过大 → 拆分子工作项回写 .spec-caliber/plan.md（运行时分解）

  按规格状态路由：
   [已定义]    ─► T 会话：写测试 + Test Double，自证 RED（F2P 断言级失败）
                  spec-caliber 门：RED 证据有效？（非编译/加载错误）
   [测试中]    ─► C 会话：复现 RED + test-hacking 红名单审查
                  ✓ → [测试已验收]   ✗ → [测试缺陷]（T 重写 ≤2，第 3 次 → specify）
   [测试已验收] ─► C 会话：假设驱动实现 + Attempt Log → [实现中]
   [实现中]    ─► T 会话：验证 GREEN + 实现反模式审查
                  ✓ → [已验收]   ✗ → [待修复]（三分类路由）
   [待修复]    ─► spec-caliber 守卫裁决：
                  [实现缺陷] → C 预算内重试（L1 Debug 报告 → L2 候选采样 → L3）
                  [测试缺陷] → T 重写    [规格缺陷] → specify（失效该规格 + Interface 依赖者）
                  L3 / 振荡 / 停滞 → 用户终审（specify 或 plan 模式 B）
   [已验收]    ─► 分层验证 L2/L3（spec-caliber 以测试 exit code 为准）→ [人工] 项 spec-caliber 逐项引导回写
                  → 用户确认 → [已验证]（终态）

  工作项规格全终态 ─► C 自检 + T 复核（跨 spec）→ 工作项总结预览
                  → 用户确认 → DONE（三相位守卫：无验证记录不放行）

【全部 DONE → 收尾】
 wrap-up：L4 全量回归 → Goal 可追溯清单（spec-caliber 引导逐项确认）
        → 证据聚合（spec-caliber 计算：attempts / 验证轮 / 争议 / PTS / F2P·P2P 通过率）
        → summary.md（wrap-up 会话写 .spec-caliber/summary.md）
        → 归档（spec-caliber 文件操作：.spec-caliber/{plan,specs,summary,events} → .archive/planning_{round}/，
           changelist.md 追加，.spec-caliber 清空保留 config.json，可选 git commit）
```

---

## 7. spec-caliber 功能清单

1. **spec-caliber-core（纯函数状态机核心，可单测）**
   - `.spec-caliber/plan.md` / `.spec-caliber/specs.md` 解析器（frontmatter、工作项表、规格块、状态记法、Issues、Attempt Log）
   - next-action 解析（execute.md §状态路由表代码移植）
   - 守卫评估器：预算计数（从 Attempt Log 重推，无独立计数器）、振荡检测（连续相同诊断）、三相位门（无验证不放行 DONE）、L0–L3 升级判定、停滞诊断
   - 文件域规则：T 可写 = 测试文件 + specs.md 状态字段；C 可写 = Affected Files + 状态字段
2. **会话驱动器**：按角色组装 pi 启动参数（model、tools、`--session-dir`、`-e` 角色 extension、`--name`）；RPC 事件流解析（`agent_settled` = 步完成）；会话注册表（角色 → 会话文件，registry.json）；重入时 `--session <file>` 恢复；Windows 兼容（Git Bash、.cmd shim）
3. **双层守卫**：spec-caliber 步前门（超预算拒派、强制 L3）+ 会话内角色 extension（tool_call 拦截文件域 / 禁 git commit；appendEntry 记 PTS 相位与事件）
4. **人机接口**：TUI（status / next / run）；干预点暂停（规格展开、候选选择、L3、[人工] 项、DONE、争议终审）；step / auto 双档；extension UI 子协议中继（会话内提问 → spec-caliber 界面）
5. **证据与收尾**：`.spec-caliber/events/<round>.jsonl` 事件日志；`spec-caliber status`（状态 + 下一步 + 预算余量 + 风险）；wrap-up 统计计算（attempts / 验证轮 / 争议 / PTS / F2P·P2P 通过率）；归档文件操作（.archive/、changelist、.spec-caliber 清理）
6. **配置**：`.spec-caliber/config.json`（测试命令、每角色模型、会话目录）；轮次生命周期（init / archive）

---

## 8. 迁移阶段（每阶段独立可交付）

### P1 — 纯 pi 手工移植（等价检查点）

任务：

1. 仓库改造为 npm 包骨架：package.json（bin: spec-caliber；pi: { prompts, extensions }；keywords: ["pi-package"]；engines）+ 目录 prompts/ extensions/ agents/ deprecated/ docs/
2. `commands/*.md` → `prompts/*.md`：
   - frontmatter 适配（pi：`description` + 可选 `argument-hint`，文件名即命令名；去掉 opencode 的 name/其他字段）
   - 路径改写：`planning/plan.md` → `.spec-caliber/plan.md`；`planning/specs.md` → `.spec-caliber/specs.md`；`planning/archive/planning_{round}/` → `.archive/planning_{round}/`；wrap-up 清空目标改为 `.spec-caliber/`（保留 config.json）
   - opencode 措辞清除：explorer 调用从"opencode task 工具 + subagent_type"改为"pi 子 agent 派生器"；兜底描述同步
3. `agents/explorer.md` → pi agent 格式（frontmatter: name/description/tools/model；正文四节保留；工具集 read/grep/find/ls/bash 只读）
4. `extensions/` 最小子 agent 派生器（适配官方 `examples/extensions/subagent/`：发现 agents/ 与标准目录，单 agent 模式；license 署名保留）
5. README.md / README-zh.md 重写：安装（`npm i -g spec-caliber` + `pi install` 本地路径）、结构、手工路径说明、host 路线图预览
6. `.gitignore` 更新（node_modules/、dist/、.spec-caliber/events/、registry.json、.pi/sessions/）
7. 顺手清理：`archive/` 研究文档 → `docs/`；`.tmp/` 遗留调试文件删除
验收：本地 `pi install ./` 后手工跑通 /setup → /plan（含 explorer 派生）→ 最小轮次；/specify /execute /wrap-up 至少完成一次完整 TDD 循环（scratch 项目）

### P2 — spec-caliber-core（状态机内核）

- plan.md/specs.md 解析器；next-action 状态路由；守卫评估器；文件域规则
- 单元测试（fixture 规划树）
验收：单测全绿；状态路由表与 execute.md 语义一致性对照表通过

### P3 — spec-caliber CLI v1（RPC 驱动 + TUI）

- RPC 驱动（派生 `pi --mode rpc`、事件流、会话注册/恢复）
- TUI：status / next / run；干预点暂停；extension UI 中继；step/auto 双档
- 角色守卫 extension：T/C 文件域拦截、禁 git commit、appendEntry 事件记录
- **T/C 拆独立会话（对抗模型结构性成立）**
验收：auto 跑通一个完整工作项（含一次失败路由 + 一次干预点暂停）；step 逐动作可确认；kill 后 resume 无缝续跑

### P4 — explorer 子进程 + 验证机械化

- explorer：spec-caliber 直接派生 `pi --mode json -p` + `--append-system-prompt agents/explorer.md`
- 验证命令机械执行：角色提议测试命令（结构化输出）→ spec-caliber 执行、exit code 为证据
- 守卫加固：bash 命令策略（禁 git commit 等）、停滞诊断落地
验收：规划期 explorer 链路通；L2/L3 验证 exit code 进事件日志

### P5 — wrap-up 自动化 + 发布

- wrap-up 统计 spec-caliber 计算；归档文件操作自动化
- npm 发布（包名定稿）；README 终版（架构图/安装/分发）；全量回归；opencode 残留清零
验收：全新项目两条命令安装即用；完整轮次自驱（含 wrap-up）无人工文档操作

---

## 9. 假设与待定项

- A1：npm 包名 = `spec-caliber` 裸名（2026-08-30 已查空，P5 发布前复核）；CLI 二进制、项目配置目录同名 `spec-caliber` / `.spec-caliber/`
- A2：style_guide.md / changelist.md 保留根目录（跨轮次/项目级产物，不属于"本轮文档"）
- A3：`.spec-caliber/{plan,specs,summary,config}.json(md)` 纳入版本控制；`.spec-caliber/events/`、`.spec-caliber/registry.json`、`.pi/sessions/` gitignore
- A4：pi 版本基线 v0.84.4，Node ≥ 22.19；pi 升级策略：spec-caliber 不锁死，兼容性经 P3 回归验证
- A5：P1 手工模式 T/C 仍为单会话双视角（与现状等价）；结构性分离在 P3（D9）
- T1（待定）：测试命令来源——`.spec-caliber/config.json` 由 spec-caliber init 询问用户（P4 细化）
- T2（待定）：会话命名规范（round 号 → 角色 → 工作项/规格），P3 实现时定稿
