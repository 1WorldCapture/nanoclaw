# Investigation: WhatsApp → Discord Channel Migration Feasibility

Date: 2026-02-21

## Summary
项目具备“核心流程与渠道适配器分离”的基础，切换到 Discord 在工程上可行；但并非只做概念映射即可。必须补齐两类关键改造：
1) 启动/运维路径里对 WhatsApp 的硬依赖；
2) Discord 适配器与数据模型（`channel`/`is_group`）的一致性。

## Symptoms
- 当前运行时代码仍只实例化 WhatsApp 通道。
- setup/verify 流程将 WhatsApp 认证视为健康必需项。
- 仓库存在 `add-discord` 技能模板，但与当前主线存在漂移。

## Investigation Log

### Phase 1 - Initial assessment
**Hypothesis:** 代码可能已经有通道抽象，只差新增 Discord adapter。  
**Findings:** 存在 `Channel` 抽象与 `findChannel` 路由，但主启动流程仍是 WhatsApp-only。  
**Evidence:**
- `src/types.ts:81` `interface Channel`
- `src/router.ts:40` `findChannel(...)`
- `src/index.ts:431-433` 仅创建并连接 `WhatsAppChannel`
**Conclusion:** 部分成立（Partially confirmed）。

### Phase 2 - Systematic exploration (`context_builder`)
**Hypothesis:** 系统核心链路大多渠道无关。  
**Findings:** DB、队列、调度、容器执行以 `chat_jid` 字符串驱动，天然可复用。  
**Evidence:**
- `src/db.ts:141` `storeChatMetadata(...)`
- `src/group-queue.ts`（按 `groupJid` 串行 + 全局并发）
- `src/task-scheduler.ts:120` `sendMessage(task.chat_jid, ...)`
- `src/container-runner.ts:42` `chatJid`
**Conclusion:** 成立（Confirmed）。

### Phase 3 - Oracle follow-up deep dive
**Hypothesis:** add-discord 模板可直接无脑应用。  
**Findings:** 模板可作为强参考，但存在关键漂移/缺口。  
**Evidence:**
- 模板 Discord metadata 未传 `channel/isGroup`：
  - `.claude/skills/add-discord/add/src/channels/discord.ts:125` `onChatMetadata(chatJid, timestamp, chatName)`
- 主线群组发现依赖 `is_group`：
  - `src/index.ts:103` `.filter((c) => ... && c.is_group)`
- 主线 `OnChatMetadata` 已含 `channel/isGroup`：
  - `src/types.ts:98-103`
- 主线容器运行时代码已演进，而模板 index 仍是旧实现：
  - 模板：`.claude/skills/add-discord/modify/src/index.ts:398-433`（`execSync('container system ...')`）
  - 主线：`src/index.ts:19,401-403`（`ensureContainerRuntimeRunning`/`cleanupOrphans`）
  - 运行时实现：`src/container-runtime.ts:10,23,58`
**Conclusion:** 否定（Rejected），“不能无脑应用”。

### Phase 4 - Evidence gathering: coupling & impact
**Hypothesis:** Discord-only 会被 WhatsApp 强耦合阻断。  
**Findings:** 存在代码层 + 运维层双重耦合。  
**Evidence:**
- 代码层：
  - `src/index.ts:431-433` 无条件连接 WhatsApp
  - `src/index.ts:459` IPC 的 `syncGroupMetadata` 绑定 `whatsapp?.syncGroupMetadata(...)`
  - `src/ipc.ts:326-333` `refresh_groups` 调用 `deps.syncGroupMetadata(true)`
- 运维层：
  - `.claude/skills/setup/SKILL.md:101` 专章“WhatsApp Authentication”
  - `.claude/skills/setup/SKILL.md:140` “Sync and Select Group”
  - `.claude/skills/setup/scripts/05b-list-groups.sh:18` 仅查 `jid LIKE '%@g.us'`
  - `.claude/skills/setup/scripts/09-verify.sh:66-68,88` 缺 WhatsApp auth 判失败
  - `.claude/skills/setup/scripts/06-register-channel.sh:54` 假设 DB 来自 sync-groups 步骤
**Conclusion:** 成立（Confirmed）。

### Phase 4 - Evidence gathering: docs/tests/deps drift
**Hypothesis:** 文档/测试/依赖需要同步改造。  
**Findings:** 目前仍明显以 WhatsApp 为默认事实。  
**Evidence:**
- 文档：
  - `README.md:51` WhatsApp I/O
  - `README.md:138` 架构图是 WhatsApp pipeline
  - `docs/SPEC.md:317` "User sends WhatsApp message"
  - `docs/SPEC.md:476` `send_message` 描述为发 WhatsApp
- 依赖/配置：
  - `package.json:19` 仅 `@whiskeysockets/baileys`，无 `discord.js`
  - `src/config.ts:8-10` 未读取 `DISCORD_*`
  - `.env.example` 为空
- 测试：
  - `src/routing.test.ts:16` 仅 WhatsApp JID 模式
  - 模板对照：`.claude/skills/add-discord/modify/src/routing.test.ts:21` 含 Discord JID
**Conclusion:** 成立（Confirmed）。

## Root Cause
不是“代码完全不能扩展 Discord”，而是存在两个根因：
1. **接入层根因**：主流程仍把 WhatsApp 作为唯一“实际连接通道”（即使底层已有多渠道抽象）。
2. **语义层根因**：群组发现逻辑依赖 `is_group`，而模板 Discord adapter 没有把 `channel/isGroup` 写全，导致可用群组列表等能力会退化。

## Hypotheses Result
- H1 渠道抽象是否足够：**Partially confirmed**
- H2 WhatsApp 强耦合是否存在：**Confirmed**
- H3 核心链路是否渠道无关：**Confirmed**
- H4 add-discord 是否可直接无脑应用：**Rejected**

## Engineering Work Required (if switching to Discord)

### P0 (must-have)
1. **新增 Discord 通道实现**（`src/channels/discord.ts`）：实现 `Channel`；`ownsJid('dc:')`；2k 分片发送。
2. **主流程多通道化**（`src/index.ts`）：引入 `DISCORD_BOT_TOKEN` / `DISCORD_ONLY`，按配置创建 Discord 与/或 WhatsApp。
3. **配置与依赖**：
   - `src/config.ts` 增加 `DISCORD_BOT_TOKEN`、`DISCORD_ONLY`
   - `package.json` 增加 `discord.js`
4. **metadata 一致性修复（关键）**：Discord 入站必须调用
   `onChatMetadata(chatJid, timestamp, name, 'discord', isGroup)`。

### P1 (strongly recommended)
5. **IPC 语义改造**：`refresh_groups` 从 WhatsApp-only 语义改为明确降级或抽象为跨渠道 `refresh_chats`。
6. **setup/verify 流程改造**：新增 Discord-only 路径，去除“必须 WhatsApp auth”的健康判断。
7. **文档更新**：README / README_zh / SPEC 改为“Channel-agnostic + 渠道附录”。

### P2 (prod quality)
8. Discord 出站失败重试/节流策略（当前模板是 best-effort 日志）。
9. DM 支持策略（是否支持、如何注册、是否需要 partials）。

## Functionality Impact Assessment

### Likely unaffected (core remains stable)
- 消息持久化与轮询处理（SQLite + polling）
- 按 chat_jid 的队列与并发控制
- 容器执行、会话续接、定时任务调度

### Affected / needs adaptation
- 通道认证与运维流程（目前是 WhatsApp QR/Pairing）
- 群组发现与注册辅助流程（当前脚本按 `@g.us` 查询）
- 文档与 runbook（当前话术/流程全是 WhatsApp）
- 触发体验（Discord 需把 mention 映射到现有 trigger 规则）

## Final Judgment on "full mapping only"
**结论：不够。**
仅做 WhatsApp↔Discord 概念映射还不够，至少还要完成：
- 启动/运维去 WhatsApp 硬依赖；
- Discord metadata 与 `is_group/channel` 数据一致性修复；
否则无法稳定支持 Discord-only。

## Recommendations
1. 以 `.claude/skills/add-discord` 为基础，但先做“模板校准”再应用（特别是 metadata 与 container-runtime 漂移）。
2. 先落地“并行双渠道（WA+Discord）”再切 Discord-only，降低迁移风险。
3. 追加测试：
   - Discord JID 路由
   - metadata 写入 `channel/is_group`
   - Discord-only 启动路径（不触发 WhatsApp 初始化）
   - `refresh_groups` 在 Discord-only 下的行为

## Preventive Measures
- 把 `onChatMetadata` 的 `channel/isGroup` 变成更强约束（至少在开发期加断言/测试）。
- 为技能模板建立“与主线漂移检测”CI（尤其 `index.ts` 的 runtime 管理逻辑）。
- 在 setup/verify 里引入“按启用渠道动态检查”的机制，避免单渠道硬编码。
