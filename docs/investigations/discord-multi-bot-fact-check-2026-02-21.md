# Investigation: NanoClaw Discord 多 Bot 架构事实核查

## Summary
结论：你给出的说法在“当前代码实现”下基本成立，但有两处需要加限定词。
- “单实例只能连一个 Discord bot”是当前实现事实（TRUE）。
- “多 bot 要多实例 + 独立目录”在工程实践上正确，但“独立目录”是由 `cwd` 驱动的运维约束，不是框架强制编排器（PARTIALLY TRUE）。
- “Docker 只用于 agent 容器，NanoClaw 是宿主机 Node 进程”在默认部署路径成立，但不是部署层面的绝对禁止（PARTIALLY TRUE）。

## Symptoms
- 存在说法：一个 NanoClaw 实例只能连接一个 Discord bot。
- 存在说法：多 bot 需要多实例与独立 `.env/store/groups`。
- 存在说法：Docker 不运行 NanoClaw 主进程，仅运行 agent 任务。

## Investigation Log

### 2026-02-21 / Phase 2 - Context Builder 初筛
**Hypothesis:** 需要先验证配置模型、实例化路径、路由语义、容器边界。
**Findings:** context_builder 指向：单 `DISCORD_BOT_TOKEN`、单 `DiscordChannel` 构造点、`dc:<channelId>` 多频道映射、Docker 经 `container-runner` 调用。
**Evidence:** `src/config.ts`, `src/index.ts`, `src/channels/discord.ts`, `src/container-runtime.ts`, `src/container-runner.ts`。
**Conclusion:** 初步支持原说法，进入反证复核。

### 2026-02-21 / Phase 3 - 反证式复核（Context Builder 二次）
**Hypothesis:** 可能存在隐藏多 token 注入面、或单进程多 bot 路由路径。
**Findings:**
- IPC/DB/类型均无 bot token 维度；
- `findChannel`/`routeOutbound` 使用首个 `ownsJid` 命中，若硬塞多个 `DiscordChannel` 会产生同前缀冲突；
- 默认 service 明确是 `node dist/index.js`。
**Evidence:** `src/ipc.ts:350-367`, `src/db.ts:69-76,541`, `src/types.ts:35-41`, `src/router.ts:34,43`, `launchd/com.nanoclaw.plist:7-13`, `.claude/skills/setup/scripts/08-setup-service.sh:75-80,143-144`。
**Conclusion:** “当前实现只支持单 Discord bot/实例”成立；“多 bot 需多实例”在不改代码前提下成立。

### 2026-02-21 / Phase 4 - 逐文件证据采集
**Hypothesis:** 需确认每条说法的代码锚点并排除误读。
**Findings:**
1. **单 token/单实例路径**：
   - `DISCORD_BOT_TOKEN` 为单字符串：`src/config.ts:73-74`
   - 仅一处生产构造：`src/index.ts:425-426`
   - `DiscordChannel` 内仅单 `Client`：`src/channels/discord.ts:21`
2. **同 bot 多频道**：
   - 入站按 `channelId` 生成 `dc:<channelId>`：`src/channels/discord.ts:45`
   - 未注册频道仅写 metadata：`src/channels/discord.ts:125,128`; `src/channels/discord.test.ts:273-302`
   - 出站去 `dc:` 前缀后 fetch 频道：`src/channels/discord.ts:183`; `src/channels/discord.test.ts:634-640`
3. **多 bot 的数据模型缺口**：
   - 注册组无 token/bot 列：`src/db.ts:69-76,541`
   - IPC 注册无 token 字段：`src/ipc.ts:359-366`
   - `RegisteredGroup` 无 token 字段：`src/types.ts:35-41`
4. **Docker 边界**：
   - 运行时 binary 为 docker：`src/container-runtime.ts:10`
   - 消息/任务通过 `spawn(CONTAINER_RUNTIME_BIN, ...)` 跑容器：`src/container-runner.ts:267`, `src/task-scheduler.ts:103-114`
   - 默认服务直接运行 Node 主进程：`launchd/com.nanoclaw.plist:7-13`, `.claude/skills/setup/scripts/08-setup-service.sh:143-144`
5. **目录隔离语义**：
   - `.env` 与 store/groups/data 都由 `process.cwd()` 决定：`src/env.ts:10-11`, `src/config.ts:23,33-35`
   - DB 固定在 `store/messages.db`：`src/db.ts:121`
**Conclusion:** 证据闭环完成。

### 2026-02-21 / Phase 4b - Git 历史核验
**Hypothesis:** 近期变更可能已引入多 bot 支持。
**Findings:** Discord 支持来自 `b2f1c16 feat: add Discord channel support`；`src/channels/discord.ts` 仅 1 次引入提交，未见后续多 bot 架构补丁。
**Evidence:** `git log` on `src/channels/discord.ts`（1 commit: `b2f1c16`）。
**Conclusion:** 当前分支下“单 bot 实现”仍是最新状态。

## Claim-by-claim Verdict

1. **“一个 NanoClaw 实例只能连接一个 Discord bot。”**
   **Verdict: TRUE（当前实现）**
   依据：单 token 配置 + 单构造点 + 单 client 模型（`src/config.ts:73-74`, `src/index.ts:425-426`, `src/channels/discord.ts:21`）。

2. **“当前架构：单 DISCORD_BOT_TOKEN、单 DiscordChannel、可监听多个频道（同 bot）。”**
   **Verdict: TRUE**
   依据：`dc:<channelId>` 映射与注册门控（`src/channels/discord.ts:45,125,128`；测试 `src/channels/discord.test.ts:273-302`）。

3. **“如果要多个 bot，需要运行多个 NanoClaw 实例；每个实例独立 .env/store/groups。”**
   **Verdict: PARTIALLY TRUE**
   - “多 bot 需要多实例（不改代码）”= **TRUE**；
   - “每实例独立目录”= **强运维约束**（由 `cwd` 语义驱动），不是内建实例编排器硬限制（`src/env.ts:10-11`, `src/config.ts:23,33-35`, `src/db.ts:121`）。

4. **“Docker 用于 agent 容器，不是 NanoClaw 本体；NanoClaw 在宿主机调用 Docker 处理消息任务。”**
   **Verdict: PARTIALLY TRUE（默认部署下等价于 TRUE）**
   - 默认脚本/服务确实是宿主机 `node dist/index.js`（`launchd/com.nanoclaw.plist:7-13`, `08-setup-service.sh:143-144`）；
   - Docker 由主进程调用运行 agent（`src/container-runtime.ts:10,25`, `src/container-runner.ts:267`）；
   - 但从部署学上，主进程仍可被外层容器化（代码未硬性禁止）。

## Root Cause
当前架构把 Discord 认证与路由定义为“进程级单实例适配器”：
- 配置层仅提供一个 Discord token；
- 启动层仅创建一个 DiscordChannel；
- 路由层对 Discord 仅用 `dc:` 前缀匹配；
- 持久化模型不含 bot 维度。
因此“单实例单 Discord bot”是实现结果，而非 Discord 协议本身限制。

## Eliminated Hypotheses
- **可通过 IPC/DB 动态注入多 bot token**：已排除（无字段/无处理路径）。
- **当前代码已支持多 DiscordChannel 共存且可正确路由**：已排除（`findChannel` 首个命中 + `dc:` 统一前缀会冲突）。
- **Docker 用来直接托管主进程**：默认路径已排除（服务文件为 Node 主进程）。

## Recommendations
1. 对外表述建议改为：**“当前实现下”** 一个实例对应一个 Discord bot。
2. 如果要多 bot：短期用多实例（不同 working directory）；长期需要改造 JID/路由/注册模型（引入 bot 维度）。
3. 在文档补充 caveat：同目录多进程会共享 `.env/store/groups/data`，可能引发冲突。
4. 补充测试：验证“若未来支持多 DiscordChannel，路由不会因 `dc:` 前缀冲突误投”。

## Preventive Measures
- 在 `src/index.ts` 附近增加注释/断言，明确 Discord 当前为单实例单 token 语义。
- 增加架构测试：禁止出现第二个生产 `new DiscordChannel(...)` 构造点而无对应路由升级。
- 文档中区分“默认部署事实”与“可自定义部署拓扑”，避免把运维习惯写成绝对架构定律。
