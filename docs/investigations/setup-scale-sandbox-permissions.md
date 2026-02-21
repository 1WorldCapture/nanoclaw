# Investigation: setup-scale / setup 容器沙箱与目录权限关系

## Summary
在这个项目中，Docker（或可选 Apple Container）确实是 Agent 的运行沙箱；但它只提供“容器边界”。
目录权限配置（allowlist + ro/rw + nonMainReadOnly）是另一层“挂载能力边界”，用于约束容器可见/可改的宿主机目录，二者并不重复。

## Symptoms
- 运行 setup 流程时会安装依赖并检查/使用 Docker。
- setup 流程会出现目录权限配置（只读/可写）。
- 用户疑问：既然在沙箱容器里运行，为何还要配置目录权限？

## Investigation Log

### Phase 1 - 初始假设
**Hypothesis:** Docker 可能只是 setup 期间使用；或目录权限配置是冗余。
**Findings:** 这两个假设都不成立。
**Evidence:** `src/container-runtime.ts:10`, `src/index.ts:407`, `src/container-runner.ts:262`
**Conclusion:** Docker 是运行时边界；权限配置有运行时实际作用。

### Phase 2 - context_builder 系统探索
**Hypothesis:** setup 与运行时之间可能存在断链，导致“权限配置只是安装时选项”。
**Findings:** 已定位完整链路：setup 生成 allowlist -> 运行时读取 allowlist + group containerConfig -> 构造 docker run 挂载参数。
**Evidence:** `.claude/skills/setup/scripts/07-configure-mounts.sh`, `src/mount-security.ts`, `src/db.ts`, `src/container-runner.ts`
**Conclusion:** 权限配置会在每次容器启动前被消费并强制执行。

### Phase 3 - Oracle 反证深挖
**Hypothesis:** “有容器就不需要 mount 权限”。
**Findings:** 若不做 mount 权限控制，挂载进容器的宿主目录会直接暴露给 Agent，存在机密读取、完整性破坏、路径逃逸风险。
**Evidence:** `src/container-runner.ts:207`, `src/container-runtime.ts:14`, `src/mount-security.ts:203`, `src/mount-security.ts:208`, `src/mount-security.ts:296`, `src/mount-security.ts:305`
**Conclusion:** 容器隔离与挂载权限是互补关系，不是二选一。

### Phase 4 - 代码/历史证据补强
**Hypothesis:** 近期可能有运行时迁移，导致文档或用户心智不一致。
**Findings:** 2026-02-20 发生了从 Apple Container 到 Docker 默认运行时的迁移，随后文档/skills 更新；存在局部文档漂移。
**Evidence:**
- `607623a` feat: convert container runtime from Apple Container to Docker
- `6b9b3a1` docs: update skills to use Docker commands after runtime migration
- `docs/SPEC.md:226` 仍描述 readonly 需 `--mount ... readonly`，但代码为 `:ro`（`src/container-runtime.ts:14`）
**Conclusion:** 用户困惑部分来自“安全宣传语 + 迁移后的局部文档漂移”。

## Root Cause
根因不是实现错误，而是**安全边界概念混淆**：
1. **容器边界（Sandbox）**：隔离进程与未挂载文件系统（`docs/SECURITY.md:22`）。
2. **挂载能力边界（Mount policy）**：决定“哪些宿主路径可进入容器、是否可写”。
   - allowlist 外置且不挂入容器（`src/config.ts:25`, `docs/SECURITY.md:26-28`, `src/types.ts:10`）
   - 无 allowlist 时 additional mounts 全阻断（`src/mount-security.ts:64`, `src/mount-security.ts:68`, `src/mount-security.ts:242`）
   - non-main 可被强制只读（`src/mount-security.ts:296`）
   - root 未允许写则强制只读（`src/mount-security.ts:305`）
3. setup 的“权限配置”本质是在配置第 2 层边界，不是在替代第 1 层。

## Key Evidence (A-E 对应)

### A. setup 是否启用 Docker 作为 Sandbox
- Docker 为默认：`src/container-runtime.ts:10`
- 启动即检查容器运行时：`src/index.ts:407`
- 每次调用通过 runtime spawn 容器：`src/container-runner.ts:262`
- setup 会检测 Docker 并执行 docker build/run：
  - `.claude/skills/setup/scripts/01-check-environment.sh:50`
  - `.claude/skills/setup/scripts/03-setup-container.sh:63`, `:78`, `:79`
- runtime 选择交互（Docker 默认，可选 Apple Container）：`.claude/skills/setup/SKILL.md:48`, `:50`

### B. 目录权限如何表示/持久化/执行
- setup 询问是否访问项目外目录及 rw/ro：`.claude/skills/setup/SKILL.md:165`, `:169`
- setup 脚本写 allowlist（默认 `nonMainReadOnly: true`）：`.claude/skills/setup/scripts/07-configure-mounts.sh:36`
- 非 empty 模式直接接收 JSON 并写文件：`07-configure-mounts.sh:44`, `:62`
- 持久化：`registered_groups.container_config`：`src/db.ts:75`
- 读写 JSON：`src/db.ts:530`, `:549`, `:574`
- 运行时注入 additional mounts：`src/container-runner.ts:174`
- ro/rw 转 docker 参数：`src/container-runner.ts:207`; ro 格式 `:ro`：`src/container-runtime.ts:14`

### C. 为什么在容器里仍要配置目录权限
- 容器是主边界，但攻击面由挂载决定：`docs/SECURITY.md:22`
- allowlist 外置且不可被容器内 agent 篡改：`docs/SECURITY.md:26-28`, `src/config.ts:25`
- 默认敏感模式阻断（`.ssh`, `.env` 等）：`src/mount-security.ts:24-44`
- 路径逃逸防护：`src/mount-security.ts:203`, `:208`

### D. 是否双重防护
- 第 1 层：容器隔离（runtime + non-root）
  - `src/container-runtime.ts:10`
  - `container/Dockerfile:62`
- 第 2 层：挂载策略（allowlist / blockedPatterns / ro-rw / nonMainReadOnly）
  - `src/mount-security.ts:64`, `:296`, `:305`, `:317`
- 第 3 层（补充）：main/non-main 默认挂载差异
  - main 可挂 project root：`src/container-runner.ts:70`, `:73`
  - non-main 不挂 project root，仅 group + global(ro)：`src/container-runner.ts:84`, `:97`

### E. 误解来源（文档/提示）
- 容器“安全”措辞较强，易被理解为“自动最小权限”
  - `README.md:6`, `:37`
  - `README_zh.md:6`, `:36`
  - `docs/SPEC.md:61`
- setup skill 仍提示改 `data/registered_groups.json`，但系统已迁移到 SQLite
  - `.claude/skills/setup/SKILL.md:173`
  - `src/db.ts:625`（该 json 仅迁移用途）

## Eliminated Hypotheses
- “项目不把 Docker 作为默认运行时” → 否（`src/container-runtime.ts:10`）
- “Docker 只用于 setup，不用于正式运行” → 否（`src/index.ts:407`, `src/container-runner.ts:262`）
- “有容器就不需要 mount 权限控制” → 否（`src/mount-security.ts:64`, `:296`, `:305`）
- “allowlist 缺失会让系统完全不可用” → 否（仅阻断 additional mounts；verify 不把它作为失败条件）
  - `src/mount-security.ts:242`
  - `.claude/skills/setup/scripts/09-verify.sh:80`, `:88-89`

## Recommendations
1. 在 README/README_zh “Secure by isolation”段后补一句：
   - 容器隔离 ≠ 自动最小权限；挂载目录仍需 allowlist + 只读优先。
2. 在 setup Step 9 增加明确提示：
   - “不配置 allowlist 不影响基础运行，但会禁用 additional mounts”。
3. 修正文档漂移：
   - `docs/SPEC.md:226` readonly 挂载说明与 `src/container-runtime.ts:14` 对齐。
4. 修正 setup 技能文案：
   - 不再引导编辑 `data/registered_groups.json`，改为 SQLite/当前真实入口。

## Preventive Measures
- 为 `mount-security.ts` 增加单元测试（当前 `container-runner.test.ts` 将其 mock，缺少独立覆盖）。
- 增加“安全边界术语”小节：Boundary 1（Container）+ Boundary 2（Mount Capability）。
- 在 CI 增加“文档与实现一致性”检查（关键命令/挂载语法/配置入口）。
