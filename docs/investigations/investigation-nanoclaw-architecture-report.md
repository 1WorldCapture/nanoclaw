# Investigation: NanoClaw 全量技术架构分层与模块化梳理

## Summary
本次调查确认：NanoClaw 是一个“宿主单进程编排（`src/`）+ 容器隔离执行（`container/`）+ skills-engine 变更系统（`skills-engine/`）”的双面架构（运行时面 + 变更交付面）。
项目具备清晰的层次化边界，但文档与实现在“容器运行时默认值/迁移路径”上存在潜在认知漂移风险。

## Symptoms
- 缺少一份统一、可执行的“分层 + 分模块”技术架构图谱。
- 运行时、容器执行、skills-engine、CI/脚本、技能资产分散在多个目录，依赖关系不易一次看清。

## Investigation Log

### 2026-02-21 / Phase 1 - Initial Assessment
**Hypothesis:** 项目可拆为稳定的分层结构（接入、编排、执行、存储、安全、扩展、交付）。
**Findings:** 基于仓库结构与 docs 入口，初步判断存在“热路径运行时”与“离线变更系统”两条主轴。
**Evidence:**
- `docs/SPEC.md:20-73`（Host + Container 总览图）
- `docs/REQUIREMENTS.md:19-62`（单进程、skills over features、runtime 路线）
- 目录事实：`src/`, `container/`, `skills-engine/`, `.claude/skills/`, `scripts/`, `.github/workflows/`
**Conclusion:** 假设成立，进入系统化探索。

### 2026-02-21 / Phase 2 - Context Builder Systematic Discovery
**Hypothesis:** `src` 是运行时核心，`skills-engine` 是代码变更引擎，`container/agent-runner` 是隔离执行与 IPC 桥。
**Findings:** context_builder 选取 60 个关键文件，覆盖 docs、runtime、container、skills-engine、scripts、CI，关系链完整。
**Evidence:**
- `src/index.ts`（主启动与主循环）
- `src/group-queue.ts`（每组串行 + 全局并发）
- `src/container-runner.ts`（挂载、容器执行、流式输出）
- `src/ipc.ts`（IPC watcher + 授权）
- `container/agent-runner/src/index.ts`（query loop）
- `skills-engine/apply.ts`, `update.ts`, `replay.ts`, `rebase.ts`, `uninstall.ts`
**Conclusion:** 假设成立，进入深入验证与分层固化。

### 2026-02-21 / Phase 3 - Follow-up Deep Dives (Oracle)
**Hypothesis:** 可稳定抽象为 L1-L7 分层，且每层边界可被代码函数锚定。
**Findings:** 得到可落地的分层模型（见“Architecture Layers”），并补全三条端到端流程。
**Evidence:**
- L1 编排入口：`src/index.ts:121 (processGroupMessages)`, `215 (runAgent)`, `294 (startMessageLoop)`, `406 (main)`
- L2 执行层：`src/container-runner.ts:61 (buildVolumeMounts)`, `219 (runContainerAgent)`
- L3 存储层：`src/db.ts:10 (createSchema)`, `281 (getNewMessages)`, `330 (createTask)`
- L4 通道层：`src/channels/whatsapp.ts:46 (connect)`, `200 (sendMessage)`, `252 (syncGroupMetadata)`
- L6 容器内 runner：`container/agent-runner/src/index.ts:356 (runQuery)`, `492 (main)`
- L7 变更引擎：`skills-engine/apply.ts:38 (applySkill)`, `update.ts:124 (applyUpdate)`
**Conclusion:** 分层假设成立，关键流可端到端追踪。

### 2026-02-21 / Phase 4 - Evidence Gathering
**Hypothesis:** 近期演进可能导致“文档叙述与当前实现”局部不一致。
**Findings:**
1. **运行时默认值已切换到 Docker**：`src/container-runtime.ts:10` 明确 `CONTAINER_RUNTIME_BIN = 'docker'`。
2. **Apple Container 被定位为技能迁移路径**：`/.claude/skills/convert-to-apple-container/manifest.yaml` 修改 runtime 相关文件。
3. **skills-engine 与多通道架构为近期大变更主线**：
   - `51788de`（skills engine v0.1 + multi-channel）
   - `c6e1bfe`（抽离 container-runtime）
   - `607623a`（Apple -> Docker 默认）
   - `7181c49`（新增 convert-to-apple-container skill）
4. **CI 已将技能组合测试制度化**：
   - `scripts/generate-ci-matrix.ts:39-75`
   - `.github/workflows/skill-tests.yml:1-84`
**Evidence:**
- `src/container-runtime.ts:10-26,58-63`
- `.claude/skills/convert-to-apple-container/manifest.yaml:1-13`
- `scripts/run-ci-tests.ts:53-99`
- `.github/workflows/skills-only.yml:15-31`（限制“新增 skill + 修改 core source”的混合 PR）
**Conclusion:** “实现漂移风险”假设成立（尤其运行时叙述层面），但核心边界并未失控。

### 2026-02-21 / Phase 5 - Flow Tracing & Boundary Validation
**Hypothesis:** 三条主流程可完全由代码闭环验证。
**Findings:**
- **消息主链路闭环**：
  `whatsapp inbound -> db -> message loop -> group queue -> container runner -> agent runner -> outbound`
- **任务调度闭环**：
  `MCP tool(schedule_task) -> IPC 文件 -> host ipc 授权 -> scheduler loop -> container 执行`
- **skills 变更闭环**：
  `manifest/preflight -> backup -> file_ops -> merge/rerere -> structured ops -> test -> state`
**Evidence:**
- 消息链路：`src/channels/whatsapp.ts:145-199`, `src/index.ts:121-213,294-383`
- 调度链路：`container/agent-runner/src/ipc-mcp-stdio.ts:65-143`, `src/ipc.ts:154-275`, `src/task-scheduler.ts:33-216`
- 技能链路：`skills-engine/apply.ts:38-397`
**Conclusion:** 三条核心流程均可代码级验证，架构结构清晰。

## Architecture Layers (Final)

### L1 入口与编排层（Host Orchestration）
- **职责**：生命周期启动、消息轮询、流程编排、子系统挂接。
- **模块**：`src/index.ts`, `src/router.ts`。

### L2 执行协调层（Container Execution on Host）
- **职责**：构建挂载、拉起容器、流式解析、超时与回收。
- **模块**：`src/container-runner.ts`, `src/container-runtime.ts`。

### L3 数据与状态层（Persistence / State）
- **职责**：消息、任务、会话、路由游标、注册群组持久化。
- **模块**：`src/db.ts`（SQLite）。

### L4 通道接入层（Channel Adapters）
- **职责**：外部消息系统接入/回发。
- **模块**：`src/channels/whatsapp.ts`（当前内建）。

### L5 横切基础设施层（Config / Security / Env / Logging）
- **职责**：配置、密钥白名单读取、挂载安全校验、日志。
- **模块**：`src/config.ts`, `src/env.ts`, `src/mount-security.ts`, `src/logger.ts`, `src/types.ts`。

### L6 容器内执行层（Agent Runner）
- **职责**：Claude Agent SDK query 循环、MCP 工具暴露、IPC 输入轮询。
- **模块**：`container/agent-runner/src/index.ts`, `container/agent-runner/src/ipc-mcp-stdio.ts`。

### L7 变更与交付层（Skills Engine + Skills Assets + CI）
- **职责**：代码库可重放变更、冲突治理、技能分发与组合测试。
- **模块**：
  - 引擎：`skills-engine/*`
  - 资产：`.claude/skills/*`
  - 自动化：`scripts/*`, `.github/workflows/*`

## Root Cause
本调查不是故障排查；“根因”对应的是**架构认知分散**：
1. 项目同时拥有运行时面与变更交付面，两套逻辑都较完整；
2. 近期快速演进（runtime 抽离、Docker 默认化、Apple 转换技能化）使“文档默认叙述”和“当前实现状态”容易出现时间差；
3. 架构知识分布在代码、技能模板、CI、文档多个载体，缺少统一视图。

## Eliminated Hypotheses
1. **“当前默认 runtime 仍是 Apple Container”** → 已排除。`src/container-runtime.ts:10` 为 Docker。
2. **“skills 只是文档层提示，不影响工程主流程”** → 已排除。`scripts/apply-skill.ts` 直接调用 `skills-engine/apply.ts`，且 CI 存在技能组合测试。
3. **“调度与消息是独立旁路，不经过统一编排”** → 已排除。均汇入 `GroupQueue + runContainerAgent`。

## Recommendations
1. 在 `docs/SPEC.md` 增加“架构分层索引（L1-L7）”并附代码锚点。
2. 增加“Runtime 默认值与可选迁移矩阵”（Docker default + convert-to-apple-container skill）。
3. 将 IPC 协议（目录、消息 schema、授权矩阵）独立为 `docs/IPC_PROTOCOL.md`。
4. 在 CI 增加轻量“文档锚点存在性检查”（关键函数/文件改名时提醒更新文档）。
5. 为 `src/logger.ts` 建立统一 trace 字段规范（chatJid/taskId/sessionId/containerName）。

## Preventive Measures
- 建立 ADR（Architecture Decision Record）记录 runtime / security / skills-engine 关键决策。
- 每次涉及 `src/container-runtime.ts`, `skills-engine/*`, `docs/SPEC.md` 的 PR，触发“架构一致性检查”清单。
- 对高风险边界（IPC 授权、mount allowlist、skills replay）保持“代码锚点 + 文档索引”双维护。
