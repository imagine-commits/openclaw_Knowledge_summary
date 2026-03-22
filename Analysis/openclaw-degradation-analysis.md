# OpenClaw 降级处理机制分析

## 概述

OpenClaw 的降级处理（Model Fallback）机制是其高可用性的核心组件，当主模型失败时能够自动切换到备用模型，确保服务的连续性。该机制主要处理以下场景：

- API 速率限制（rate_limit）
- 服务过载（overloaded）
- 认证失败（auth/auth_permanent）
- 计费问题（billing）
- 超时（timeout）
- 模型不存在（model_not_found）
- 会话过期（session_expired）

---

## 核心文件结构

| 文件路径 | 功能描述 |
|---------|---------|
| `src/agents/model-fallback.ts` | 核心降级逻辑实现 |
| `src/agents/failover-error.ts` | 故障错误类型定义与分类 |
| `src/agents/model-fallback.types.ts` | 降级相关类型定义 |
| `src/agents/model-fallback-observation.ts` | 降级决策的日志观察 |
| `src/agents/pi-embedded-helpers/types.ts` | FailoverReason 类型定义 |
| `src/config/types.agents-shared.ts` | 模型配置类型定义 |

---

## 故障原因类型（FailoverReason）

定义于 `src/agents/pi-embedded-helpers/types.ts`：

```typescript
export type FailoverReason =
  | "auth"              // 认证失败（可恢复）
  | "auth_permanent"    // 永久性认证失败
  | "format"            // 响应格式错误
  | "rate_limit"        // 速率限制
  | "overloaded"        // 服务过载
  | "billing"           // 计费/配额问题
  | "timeout"           // 请求超时
  | "model_not_found"   // 模型不存在
  | "session_expired"   // 会话过期
  | "unknown";          // 未知错误
```

### HTTP 状态码映射

| FailoverReason | HTTP 状态码 |
|---------------|------------|
| billing | 402 |
| rate_limit | 429 |
| overloaded | 503 |
| auth | 401 |
| auth_permanent | 403 |
| timeout | 408 |
| format | 400 |
| model_not_found | 404 |
| session_expired | 410 |

---

## 降级处理流程

### 1. 主流程 (`runWithModelFallback`)

核心函数位于 `src/agents/model-fallback.ts`，执行以下步骤：

```
┌─────────────────────────────────────────────────────────────┐
│                    runWithModelFallback                      │
├─────────────────────────────────────────────────────────────┤
│  1. 解析降级候选模型                                          │
│     ├── 从配置获取主模型                                       │
│     ├── 从配置获取 fallback 列表                              │
│     └── 构建候选队列                                          │
│                                                              │
│  2. 遍历候选模型                                              │
│     ├── 检查认证profile是否在冷却期                           │
│     ├── 决定是否跳过或探测                                     │
│     └── 执行模型调用                                          │
│                                                              │
│  3. 成功 → 返回结果                                          │
│     失败 → 记录错误，继续下一个候选                            │
│                                                              │
│  4. 所有候选都失败 → 抛出汇总错误                              │
└─────────────────────────────────────────────────────────────┘
```

### 2. 冷却期管理（Cooldown）

当某个 provider 的所有认证 profile 都在冷却期时，系统会：

- **永久性问题**（auth/auth_permanent）：直接跳过该 provider 的所有模型
- **计费问题**（billing）：单 provider 模式下允许探测恢复
- **临时性问题**（rate_limit/overloaded/unknown）：允许在单轮降级中探测一次

### 3. 探测节流（Probe Throttling）

为防止频繁探测导致资源浪费，系统实现了探测节流机制：

```typescript
const MIN_PROBE_INTERVAL_MS = 30_000;  // 30秒最小间隔
const PROBE_MARGIN_MS = 2 * 60 * 1000; // 2分钟边际
const PROBE_STATE_TTL_MS = 24 * 60 * 60 * 1000; // 24小时TTL
const MAX_PROBE_KEYS = 256;            // 最大探测key数
```

---

## 关键实现细节

### 1. 错误分类 (`failover-error.ts`)

系统通过多种方式识别故障类型：

- **HTTP 状态码**：直接从响应中提取
- **错误码**：如 `RESOURCE_EXHAUSTED`、`RATE_LIMITED`
- **错误消息**：正则匹配常见错误模式
- **错误链**：递归检查 `cause` 属性

```typescript
// 错误分类优先级
1. 已是 FailoverError → 直接使用
2. HTTP 状态码 → 映射到 FailoverReason
3. 符号错误码（如 RESOURCE_EXHAUSTED）
4. 网络错误码（如 ETIMEDOUT）
5. 错误消息模式匹配
```

### 2. 降级候选解析

```typescript
// 解析降级候选模型
function resolveFallbackCandidates(params: {
  cfg: OpenClawConfig | undefined;
  provider: string;
  model: string;
  fallbacksOverride?: string[];
}): ModelCandidate[]
```

- 显式配置的 fallback 不会被模型白名单过滤
- 跨 provider 时，只有已在降级链中的模型才使用配置的 fallbacks
- 同 provider 内始终使用完整降级链

### 3. 图像模型降级

独立的图像模型降级处理：

```typescript
export async function runWithImageModelFallback<T>(params: {
  cfg: OpenClawConfig | undefined;
  modelOverride?: string;
  run: (provider: string, model: string) => Promise<T>;
}): Promise<ModelFallbackRunResult<T>>
```

---

## 配置方式

### 基本配置（agents.defaults.model）

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-sonnet-4-20250514",
        "fallbacks": [
          "anthropic/claude-3-5-sonnet-20241022",
          "openai/gpt-4o-mini"
        ]
      }
    }
  }
}
```

### 图像模型配置

```json
{
  "agents": {
    "defaults": {
      "imageModel": {
        "primary": "anthropic/claude-sonnet-4-20250514",
        "fallbacks": [
          "openai/dall-e-3"
        ]
      }
    }
  }
}
```

### 旧配置迁移

系统自动迁移旧配置格式：

| 旧配置 | 新配置 |
|-------|--------|
| `agent.model` | `agents.defaults.model.primary` |
| `agent.modelFallbacks` | `agents.defaults.model.fallbacks` |
| `agent.imageModel` | `agents.defaults.imageModel.primary` |
| `agent.imageModelFallbacks` | `agents.defaults.imageModel.fallbacks` |

---

## 观察与日志

### 降级决策日志

通过 `model-fallback-observation.ts` 记录详细决策：

```typescript
logModelFallbackDecision({
  decision: "candidate_failed",  // skip_candidate | probe_cooldown_candidate | candidate_failed | candidate_succeeded
  runId: "xxx",
  requestedProvider: "anthropic",
  requestedModel: "claude-sonnet-4",
  candidate: { provider: "openai", model: "gpt-4o-mini" },
  attempt: 2,
  total: 3,
  reason: "rate_limit",
  status: 429,
  nextCandidate: { provider: "anthropic", model: "claude-3-5-sonnet" }
})
```

### 日志标签

- `error_handling`
- `model_fallback`
- `skip_candidate` / `probe_cooldown_candidate` / `candidate_failed` / `candidate_succeeded`

---

## 特殊处理

### 1. 上下文溢出错误

上下文溢出错误由内部 runner 的压缩/重试逻辑处理，不会触发模型降级：

```typescript
// 上下文溢出错误立即重新抛出，不尝试降级
if (isLikelyContextOverflowError(errMessage)) {
  throw err;
}
```

### 2. 中止错误

只有显式名为 `AbortError` 的错误被视为用户中止，其他包装在 AbortError 内的错误（如 rate limit）会被正确分类：

```typescript
function isFallbackAbortError(err: unknown): boolean {
  // 只有显式 AbortError 才被视为用户中止
  // 包装的 rate limit 错误会被 coerceToFailoverError 转换
}
```

### 3. 认证 profile 轮换

支持多个认证 profile 的自动轮换和冷却期管理：

```typescript
// 解决认证 profile 顺序
const profileIds = resolveAuthProfileOrder({
  cfg: params.cfg,
  store: authStore,
  provider: candidate.provider,
});

// 检查是否有可用的 profile
const isAnyProfileAvailable = profileIds.some(
  (id) => !isProfileInCooldown(authStore, id)
);
```

---

## 总结

OpenClaw 的降级处理机制是一个复杂但可靠的系统，提供了：

1. **多层次错误分类**：从 HTTP 状态码到错误消息的全面识别
2. **智能冷却管理**：避免频繁探测，尊重 API 限制
3. **灵活的配置**：支持 per-agent 和全局的降级配置
4. **完整的观测**：详细的日志便于问题排查
5. **安全防护**：上下文溢出等错误不会被降级掩盖

这套机制确保了 OpenClaw 在各种 API 故障场景下都能保持服务连续性。
