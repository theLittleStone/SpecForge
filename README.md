# SpecForge

**A specification-driven TDD workflow framework for AI coding agents**

[License: Apache-2.0] [Platform: OpenCode]

> ⚠ **This project is a work in progress** — under active development. Command behavior, directory structure, and documentation are subject to change.

---

## What is SpecForge?

SpecForge is a **specification-driven TDD pipeline** built for AI coding agents (specifically [opencode](https://opencode.ai)). It transforms natural language requirements into a **plan graph** of work items, expands each work item into a precise implementation spec just before execution, then drives separate agent conversations through a test-first engineering workflow with strict state gating.

Instead of letting a single AI agent design, code, and grade itself, SpecForge splits responsibilities across multiple agents that distrust each other: define the `Interface` contract → generate tests → confirm the tests fail (RED) → write code to pass them (GREEN) → review and refactor (REFACTOR). Every line of code is traceable back to a spec, verified by a neutral test runner, and recoverable through **local invalidation** instead of whole-round rollback.

---

## Design Philosophy

SpecForge guarantees quality through **separation, not trust**, built on three structural principles:

1. **Files as agent context isolation** — Each planning artifact is the primary working context of a **separate agent conversation**, self-contained enough that a conversation only needs to read the documents it needs and write its own output. No agent carries the entire pipeline context.

2. **Evidence-conditioned plan graph** — Planning is not a fixed chain but a **graph**: nodes are work items (with dependency edges and explicit failure routes), and structure is decided by evidence and complexity, not by a preset sequence. `complexity: minimal | task | full` determines how many planning layers the round needs; specs are expanded **lazily** (per work item, right before execution) instead of all upfront. Simple tasks naturally collapse to a short chain — that is by design, not degradation.

3. **Adversarial TDD trust model** — In the programming phase, test / code / review are designed to be executed by **different agents that mutually distrust each other**:
   - **Test agent** (`/test-generate`): trusts only the spec. Writes tests and Test Doubles, and forces RED (tests must fail before implementation). It is the **sole owner of test code** — `[test-defect]` repairs are routed back to it as well.
   - **Code agent** (`/code-generate`): trusts only the tests. Reads them to understand expected behavior, then implements; **never modifies test code** — suspected test errors are only classified and routed.
   - **Review agent** (`/code-review`): audits both code and test quality, refactors, and propagates completion upward.
   - **Neutral verifier** (`/test-verify`): generates no code; judges only test results and updates spec states. Neither side can grade itself. On FAIL it classifies Issues into three types per the failure routing protocol.
   - **The user** is the final judge, verifying `[manual]` items that automation cannot reach; results are written back to specs.md by code-generate.

---

## Pipeline Overview

```mermaid
flowchart TD
    PLAN["/plan<br/>Socratic dialogue → explore → assess complexity → plan graph"]
    PLAN --> SPEC["/specify<br/>expand active work item into spec contract"]

    SPEC --> ENTRY(( ))

    subgraph TDD["TDD Cycle (per work-item specs)"]
        ENTRY --> RED["/test-generate<br/>RED: Write tests, confirm FAIL"]
        RED --> GREEN["/code-generate<br/>GREEN: Implement, make tests pass"]
        GREEN --> REFACTOR["/code-review<br/>REFACTOR: Review & refactor"]
        REFACTOR -->|next spec| ENTRY
    end

    TEST["/test-verify"]
    TEST -.-> RED
    TEST -.-> GREEN
    TEST -.-> REFACTOR

    REFACTOR -->|all work items DONE| REPORT["/progress-report"]
```

**Three phases**:

1. **Planning (adaptive)** — `/plan` runs the Socratic dialogue (why / assumptions / alternatives / boundaries), explores the codebase, assesses complexity (`minimal | task | full`), and produces a **plan graph** (`planning/plan.md`: work items with dependencies and failure routes). `/specify` expands the active work item into a spec contract (`planning/specs.md`) lazily — right before it enters the TDD cycle. When a work item turns out too large, it is split and written back into the graph (runtime decomposition). Plan revisions propagate through **change classification + impact rules** and invalidate only affected work items.

2. **Programming (TDD cycle)** — `test-generate → code-generate → code-review` forms the RED → GREEN → REFACTOR loop. Each spec of the current work item runs through the whole loop before the next spec. When all specs of a work item reach terminal state, code-review marks the work item `DONE` in plan.md, and `/specify` can expand the next one.

3. **Closeout** — `progress-report` enforces hard acceptance gates (full regression, Goal traceability, closed unresolved questions), then archives the round.

**test-verify** (dotted lines) is a shared **neutral test runner** reused across all three TDD phases, interpreting PASS/FAIL according to the calling context. It generates no code and advances spec states solely from test results — neither the test agent nor the code agent can grade itself. On FAIL it classifies Issues into three types per the failure routing protocol. It also powers `code-review`'s impact-scope regression (including the pre-refactor baseline run) and `progress-report`'s full regression.

Non-behavioral changes (chore / docs) take the **lightweight spec** path: RED is skipped, and after implementation the spec goes through `/test-verify`'s lightweight acceptance and `/code-review` audit, sharing the same terminal states. Rework is driven by the **failure routing protocol** — the three-class Issues (`[实现缺陷]` / `[测试缺陷]` / `[规格缺陷]`) route back to the implementation / test / spec stage respectively, giving both test errors and spec errors a defined exit. Failures are routed to the **right recovery target** rather than re-running the whole chain.

---

## Key Features

### Full TDD Pipeline

RED (write tests, confirm FAIL) → GREEN (implement, make tests pass) → REFACTOR (review & refactor). Each phase has strict state guards — a FAIL at any stage blocks progression and prompts for repair.

### Adversarial TDD Roles

test / code / review are intended to be executed by **different agents that distrust each other**: the test agent writes tests that must fail (RED), the code agent implements only to satisfy them (GREEN), the review agent audits both (REFACTOR), and `/test-verify` grades solely by test results. `[manual]` items are judged by the user as the final authority.

### Adaptive Planning (Plan Graph)

The four fixed stages (requirement → architecture → task → spec) are fused into one **plan graph** (`planning/plan.md`):

- **Work items as nodes**: type (from a small set: internal implementation / API signature / data structure / pure addition / pure removal / docs-config), one-line description referencing evidence, `Depends On` (ON-SUCCESS prerequisite topology), `Affects`, optional design reference, status (`TODO` / `ACTIVE` / `DONE`)
- **Failure routes as ON-FAILURE edges**: default three-class routing; per-work-item overrides allowed (e.g. "on `[实现缺陷]`, first check whether upstream W2 changed an API signature — the surface failure point may not be the root cause")
- **Evidence-conditioned parsimony**: complexity assessment (signals: module count, API/signature change, data-structure change, open boundaries, cross-module deps) decides whether design nodes are needed; `minimal` rounds skip the design layer entirely
- **Deferred work**: new work items discovered during execution are registered in the graph and expanded by `/specify` (runtime decomposition)
- **Negative evidence**: failed attempts are recorded; the same solution must not be retried verbatim

### Spec-as-Contract (Lazy Expansion)

Specs are **not** defined upfront for the whole round. `/specify` expands exactly one work item at a time — the active one (first `TODO` whose dependencies are all `DONE`) — right before it enters the TDD cycle. Each spec carries: `Spec` (diff-generatable behavior), `Interface` (precise signature, must support dependency injection), `Test Double` (only at open boundaries), `Test Cases` (`[automation]` for WHAT verification + `[manual]` with operation/expected/acceptance-criteria). When a work item is too large for one diff, it is split back into the graph.

### Test Double Dual-Layer Model

Specs involving external boundaries (APIs, databases, filesystem, email, time/randomness) automatically require a Test Double design:

- **Layer 1**: Test Double → automated tests (fast, deterministic core logic coverage)
- **Layer 2**: Manual testing → visual/interactive/real integration checks (residual gaps Test Doubles can't reach)

The two layers are additive, not mutually exclusive.

### 16-State Spec Machine

Each spec flows through a strict state machine using the `[Operation: State]` format. The state machine is the **inter-agent contract language**: each agent reads a spec's state to decide what to do and how to verify, then hands off by advancing it. States are advanced only by the command that owns them.

| State Label | Set By | Meaning |
| ------------- | -------- | --------- |
| `[新增: 已定义]` | specify | Spec defined, ready for test generation |
| `[新增: 测试已生成]` | test-generate | Tests generated and confirmed RED |
| `[新增: 已实现]` | code-generate | Code implemented, awaiting verification |
| `[新增: 已测试]` | test-verify | Tests passed, awaiting review |
| `[新增: 待修复]` | test-verify / code-review | Verification failed, rework via failure routing protocol |
| `[新增: 已验证]` | code-review | Reviewed and verified (terminal) |
| `[修改: 已定义]` | specify | Modification spec defined |
| `[修改: 测试已生成]` | test-generate | Modification tests generated and confirmed RED |
| `[修改: 已实现]` | code-generate | Modification implemented |
| `[修改: 已测试]` | test-verify | Modification tests passed |
| `[修改: 待修复]` | test-verify / code-review | Modification verification failed, rework via failure routing protocol |
| `[修改: 已验证]` | code-review | Modification reviewed and verified (terminal) |
| `[废弃: 待删除]` | specify | Spec defined (with removal target and `Depended By`), awaiting de-reference |
| `[废弃: 已解引用]` | code-generate | Referrers all removed, code moved to `_deprecated/`, verification checklist (targeted tests + full build + grep reference scan) pass |
| `[废弃: 待修复]` | test-verify / code-review | De-reference verification failed, rework via failure routing protocol |
| `[废弃: 已删除]` | progress-report | Code physically deleted from `_deprecated/` (progress-report closeout), terminal |

Work items carry a coarse three-state marker in plan.md: `TODO` (not yet expanded) → `ACTIVE` (spec defined or executing) → `DONE` (all specs terminal); details live in the spec 16-state machine.

Lightweight specs share this state machine with regular specs, skipping only the test-generate phase (see "Lightweight Specs"). The canonical 16-state table lives in the `AGENTS.md` written by `/setup` (§ State Machine & Notation); each command's state-filter rules live in its own command file. Command files use the pipe shorthand `[新增|修改: x]` for parallel enumeration, but the full `[OperationType: State]` form is mandatory when writing into planning/ documents.

### Failure Routing Protocol (Issues Three-Class)

Failures no longer have a single exit. When test-verify / code-review judge FAIL, they must write the failure into the spec's `Issues` field as `[type] <detail>` with one of three classifications, deciding the rework route:

| Issues Type | Meaning | Routed To |
| ------------- | --------- | ----------- |
| `[实现缺陷]` | Implementation violates the contract (assertion fails and the assertion matches the spec) | `/code-generate` fixes the implementation |
| `[测试缺陷]` | The test code itself is wrong (references fields not in the spec, assertion contradicts spec text, etc.) | `/test-generate` fixes the test |
| `[规格缺陷]` | The spec is incomplete/self-contradictory and needs redefinition | `/specify` redefines the spec (single spec + its Interface dependents invalidated) |

Accompanying discipline: the code agent and review agent **never modify test code** (`/test-generate` is the sole owner); when `Retry Count ≥ 3`, blind retries are forbidden — one of the three routes must be chosen. Before classifying, check whether an upstream dependency changed (e.g. an API signature change breaking an assertion) — the surface failure point may not be the root cause.

### Local Invalidation (replaces whole-round rollback)

When `/plan` revises the plan or `/specify` handles a `[规格缺陷]`, affected work items are located via **change classification + impact rules** (e.g. API signature change → callers + override hierarchy; internal implementation → localized; removal → grep-verified referrers), then handled by tier:

- **DONE with no file overlap** → untouched (verified state is never broken by unrelated revisions)
- **TODO (not yet expanded)** → plan revised directly, zero cost
- **ACTIVE (code landed)** → spec state reset to initial, `Retry Count` zeroed, definition text preserved, work item marked "redo" — code is **kept** and overwritten by the redo (git restore only when the user explicitly asks or the working tree is badly polluted)

An **overlap guard** protects verified specs outside the invalidation scope: files shared with their `Affected Files` are never auto-invalidated and are listed for precise manual handling. Failed causes are recorded in the plan graph's negative-evidence section; the same solution must not be retried verbatim.

### Lightweight Specs

Not every change suits test-first. Specs with `change_type` of `chore` / `docs` are declared lightweight automatically; other types may be declared lightweight after user confirmation (spec carries `- **轻量**: 是 — <reason>`; the agent must not self-downgrade). Lightweight specs:

- May omit Interface (must be explained in Spec); Test Cases may contain only `[manual]` items or a build/typecheck checklist
- Skip test-generate; `/code-generate` implements directly
- GREEN acceptance = full build + typecheck + all `[manual]` items landed as `✓ PASS` (run by the `/test-verify` lightweight checklist)
- Then go through `/code-review` to the `[已验证]` terminal state, same as regular specs

Note: lightweight (document-level, saves Interface fields) and `minimal` complexity (process-level, saves the design layer in `/plan`) are orthogonal concepts that can stack.

### Manual Verification Persistence

`[manual]` tests are not verbal: after code-generate walks the user through each item, it must write the result back to the Test Cases entry in specs.md (`✓ PASS` / `✗ FAIL — <note>`). State must not advance until results are landed; code-review uses these landed marks as the sole cross-session evidence of manual acceptance.

### Deprecated Code Lifecycle

Removal is a behavior reduction, not test-first work. Deprecated specs flow through a dedicated lifecycle:

1. **Define**: `/specify` records the removal target and its `Depended By` (derived from a real `grep` scan of the codebase), and writes the verification checklist (targeted tests / full build / grep scan) into `Test Cases`
2. **Skip test generation**: `test-generate` skips deprecated specs entirely — no tests or RED confirmation
3. **De-reference & move**: `code-generate` removes all references, moves the code into `_deprecated/` (excluded from build/test config), syncs `Depended By` / work-item `Affects`, then `test-verify` runs the verification checklist
4. **Closeout**: `progress-report` physically deletes `_deprecated/` and sets `[废弃: 已删除]`

### Change Classification & Impact Rules

Instead of a maintained module index, work items carry a change type from a small set, each with deterministic impact rules: internal implementation (localized: file + direct callers), API signature change (callers + override hierarchy + dependent files), data structure change (readers/writers + persistence + serialization), pure addition (new files + integration points), pure removal (referrers, grep-verified), docs/config (respective files). `Affects` is derived from these rules rather than from free-form impact analysis; references are always verified by actual grep scans.

### Socratic Requirements Gathering

`/plan` does not blindly accept user descriptions. Instead, it asks "why", challenges implicit assumptions, explores alternatives, and exhausts edge cases before drafting. Only after reaching deep consensus with the user does it produce the plan graph. Unresolved questions are recorded in the requirement node of plan.md and must be closed (`[resolved]` / `[won't-fix]`) before the round can be closed.

### Recursive Completion Propagation

When `code-review` completes a spec, it walks upward: checks whether all specs under the current work item have reached terminal state. If yes, it marks the work item `DONE` in plan.md. When all work items are done, it prompts the user to run `progress-report` for archival. Removal specs reach their terminal state at `[废弃: 已解引用]` — physical deletion is deferred to the `progress-report` closeout.

### Cross-Round Archival

When a round completes, `progress-report` first enforces hard acceptance gates — full regression PASS, `# Goal` traceability confirmed, and all unresolved questions closed. It then physically deletes `_deprecated/`, auto-generates a summary, archives all planning files to `planning/archive/`, appends an entry to `changelist.md`, and preserves historical context for the next round. (`style_guide.md` stays in the project root for reuse across rounds; it is not archived.)

---

## Installation

```bash
# Clone into opencode config directory
git clone https://github.com/<your-org>/specforge.git ~/.config/opencode/

# Or just copy the commands and skills
cp -r commands/ skills/ ~/.config/opencode/
```

After installation, run `/setup` in the target project to initialize `AGENTS.md` (every pipeline command checks for it first), then start the pipeline with `/plan`.

---

## Core Pipeline Commands

| Command | Stage | Description |
| --------- | ------- | ------------- |
| `/plan` | Planning | Socratic dialogue (why / assumptions / alternatives / boundaries) → codebase exploration → complexity assessment (`minimal`/`task`/`full`) → plan graph with work items, dependencies, failure routes, design nodes (full complexity), deferred work and negative evidence. Also revises existing plans via change classification + impact rules + local invalidation. Produces `planning/plan.md` |
| `/specify` | Planning | Expands the active work item (first `TODO` whose dependencies are all `DONE`) into a spec contract: Spec / Interface / Test Double / Test Cases; splits oversized work items back into the graph (runtime decomposition); handles `[规格缺陷]` redefinition (single spec + Interface dependents invalidated). Produces `planning/specs.md` |
| `/test-generate` | **RED** (test agent) | Generate test code from specs, run to confirm FAIL. Generate Test Double code. Assembles `[manual]` items into a checklist. The **sole owner of test code** — repairs `[测试缺陷]`-routed tests. Skips deprecated and lightweight specs |
| `/code-generate` | **GREEN** (code agent) | Read test code to understand expected behavior, implement to pass tests; lightweight specs implemented directly. Guides `[manual]` verification and writes results back to specs.md. Never modifies test code; failures routed by the three-class Issues protocol |
| `/code-review` | **REFACTOR** (review agent) | Code review + refactoring + regression testing. Runs a **baseline regression before any refactor** (even with zero refactor items); runs impact-scope regression after each refactor, rolls back on failure. Reviews test code quality (classify-and-route only, never edits tests). Verifies `[manual]` completion via landed marks in specs.md. Propagates completion upward (work item → `DONE`) on success |
| `/test-verify` | Verify (neutral) | Neutral test runner reused by test-generate / code-generate / code-review / progress-report. Interprets PASS/FAIL by calling context; classifies failures as assertion vs error; on FAIL writes three-class Issues with routing advice; executes impact-scope regression (code-review) and full regression (progress-report); runs the deprecated verification checklist (targeted tests + full build + grep scan); runs the lightweight acceptance checklist (build + typecheck + manual-landing check) |
| `/progress-report` | Close | Gate on full regression + Goal traceability + closed unresolved questions, then physically delete `_deprecated/` (setting `[废弃: 已删除]`), archive to `planning/archive/`, append to `changelist.md` |

---

## Auxiliary Commands

| Command | Description |
| --------- | ------------- |
| `/deep-debug` | System-level bug investigation using hypothesis-driven analysis of code, interfaces, and tests |
| `/explain-to-me` | Explain code, architecture, or technical concepts by synthesizing local code with web information |
| `/setup` | Initialize project `AGENTS.md` with pipeline-wide behavior constraints and the 8-section workflow consensus (interaction protocol / safety baseline / execution constraints / state machine & notation / failure routing protocol / local invalidation protocol / manual-landing rule / lightweight spec rule). OpenCode auto-injects it into every session; commands reference it instead of duplicating |

---

## Built-in Skills

SpecForge ships with two opencode skills that the pipeline invokes on demand:

### knowledge-augment

Fills knowledge gaps about programming languages, frameworks, and libraries with concise examples and common pitfalls. Invoked on demand — explicitly referenced by deep-debug / explain-to-me, and the AGENTS.md safety baseline lets any command call it.

### style-resolver

Auto-detects the project's language and framework, then generates `style_guide.md` based on existing code conventions or community standards. Invoked by test-generate / code-generate when `style_guide.md` is missing (code-review only reads the existing `style_guide.md` as audit basis; it never generates one) to ensure consistent code style.

---

## Project Structure

```text
commands/               # Pipeline command definitions (Markdown + YAML frontmatter)
  plan.md               # Planning: dialogue → explore → assess → plan graph / revise
  specify.md            # Planning: lazy spec expansion + [规格缺陷] redefinition
  test-generate.md      # TDD RED (test agent)
  code-generate.md      # TDD GREEN (code agent)
  code-review.md        # TDD REFACTOR (review agent)
  test-verify.md        # Neutral verifier
  progress-report.md    # Closeout & archival
  setup.md              # AGENTS.md consensus initializer
  deep-debug.md         # Auxiliary: hypothesis-driven debugging
  explain-to-me.md      # Auxiliary: explain code/architecture
  _deprecated/          # Retired planning commands (requirement-define / architecture-design /
                        # task-breakdown / specification-define), kept for reference
skills/                 # Agent auxiliary skills
  knowledge-augment/
    SKILL.md
  style-resolver/
    SKILL.md
    style-guide-example.md
```

---

## License

Apache License 2.0
