# Discord Channel Setup Fixes

**Date**: 2026-02-21
**Author**: Claude Code
**Status**: Resolved

## Summary

在配置 Discord 频道时发现了两个问题并已修复：

1. **add-discord skill 引入 Apple Container 强依赖** - 导致 Docker 用户无法启动服务
2. **第三方 Anthropic API 兼容问题** - 环境变量名不匹配导致 agent 无法认证

---

## Problem 1: Apple Container 强制依赖

### 现象

```
Error: Apple Container system is required but failed to start
```

服务启动失败，即使 Docker/Colima 正常运行。

### 根因分析

`add-discord` skill 的 `modify/src/index.ts` 文件包含了旧版本的 `ensureContainerSystemRunning()` 函数，直接调用 Apple Container 命令：

```typescript
// 旧代码（有问题）
function ensureContainerSystemRunning(): void {
  execSync('container system status', { stdio: 'pipe' });  // Apple Container
  execSync('container system start', { stdio: 'pipe' });
  // ...
}
```

而项目已经将容器运行时抽象到 `container-runtime.ts`，支持 Docker：

```typescript
// 正确的抽象（container-runtime.ts）
export const CONTAINER_RUNTIME_BIN = 'docker';

export function ensureContainerRuntimeRunning(): void {
  execSync(`${CONTAINER_RUNTIME_BIN} info`, { stdio: 'pipe' });  // Docker
}
```

**时间线**：
| 提交 | 说明 |
|------|------|
| 旧版本 | index.ts 直接包含 Apple Container 代码 |
| c6e1bfe | 提取 runtime 抽象到 container-runtime.ts |
| 607623a | 切换到 Docker |
| 51788de | add-discord skill 基于此版本开发（包含旧代码） |

### 修复方案

修改 `src/index.ts`：

1. 添加导入：
```typescript
import { cleanupOrphans, ensureContainerRuntimeRunning } from './container-runtime.js';
```

2. 删除旧的 `ensureContainerSystemRunning()` 函数（第 398-458 行）

3. 更新 `main()` 调用：
```typescript
// 旧
ensureContainerSystemRunning();

// 新
ensureContainerRuntimeRunning();
cleanupOrphans();
```

### 文件变更

- `src/index.ts` - 唯一修改的文件

---

## Problem 2: 第三方 Anthropic API 环境变量不兼容

### 现象

Discord bot 正常连接，但 agent 响应：

```
Not logged in · Please run /login
```

### 根因分析

`container-runner.ts` 的 `readSecrets()` 函数只读取特定变量名：

```typescript
// 修复前
function readSecrets(): Record<string, string> {
  return readEnvFile(['CLAUDE_CODE_OAUTH_TOKEN', 'ANTHROPIC_API_KEY']);
}
```

但使用第三方兼容 API（如智谱 AI）时，环境变量使用不同名称：

```env
ANTHROPIC_BASE_URL=https://open.bigmodel.cn/api/anthropic
ANTHROPIC_AUTH_TOKEN=xxx
```

变量名不匹配导致 secrets 无法传递给容器内的 agent。

### 修复方案

修改 `src/container-runner.ts`：

```typescript
// 修复后
function readSecrets(): Record<string, string> {
  return readEnvFile([
    'CLAUDE_CODE_OAUTH_TOKEN',
    'ANTHROPIC_API_KEY',
    'ANTHROPIC_AUTH_TOKEN',    // 新增
    'ANTHROPIC_BASE_URL',      // 新增
  ]);
}
```

### 文件变更

- `src/container-runner.ts` - 添加两个环境变量支持

---

## Verification

修复后的验证步骤：

1. `npm run build` - TypeScript 编译通过
2. `npm test` - 339 个测试全部通过
3. `launchctl kickstart -k gui/$(id -u)/com.nanoclaw` - 服务重启成功
4. 日志确认：`Discord bot connected` + `NanoClaw running`
5. Discord 中发送消息，agent 正常响应

---

## Lessons Learned

1. **Skill 维护问题**：add-discord skill 基于旧版本代码开发，没有同步更新。建议 skill 应该使用抽象层而非直接调用实现。

2. **环境变量命名**：Anthropic API 的环境变量名有多种变体（`ANTHROPIC_API_KEY` vs `ANTHROPIC_AUTH_TOKEN`），第三方兼容服务可能使用不同名称。代码应该支持常见的变体。

3. **错误信息改进**：`Not logged in` 的错误信息可以更明确地指出缺少哪些环境变量，便于排查。
