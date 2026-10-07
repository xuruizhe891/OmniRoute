# OmniRoute 对 OpenCode Go 公告的完整应对分析

## 背景上下文

### OpenCode Go 官方公告（2026年9月）

OpenCode 发布了编程 Agent 流量规范，要求客户端：

> 你的客户端应当：
> - 发送典型的编程 Agent 流量
> - 使用自身专属的 user agent 标识（例如 my-coding-agent/1.0），而不是通用的 SDK 或 HTTP 库名称
> - 为每段对话在 x-opencode-session 中发送稳定的会话 ID，以便我们优化路由和提示词缓存

### 观察现象：构建失败（Job 111237838907）

- CI 构建在 5 分钟 26 秒时被 runner 强制关闭
- 日志显示："The runner has received a shutdown signal"
- PR #15322 涉及 OpenCode free-tier 工具指纹问题

---

## 第一性原理分析

### 问题根源：客户端身份识别升级

反问式分解：

1. **OpenCode 在做什么？**
   - 不是简单速率限制
   - 而是**客户端行为归因**（client attribution）
   - 识别：真实代码 agent vs. 代理/冒充客户端

2. **在哪些场景？**
   - 场景 A：OpenCode Go（authenticated）- 需要保留 third-party agent 身份
   - 场景 B：OpenCode Zen free-tier（keyless）- 需要强制特定工具指纹 + 契约

3. **为什么是两个问题，但对应一个公告？**
   - 同一原因的两个不同表现
   - free-tier：通过工具指纹 + request contract 识别真实 agent
   - Go：通过 User-Agent + session 识别真实 agent

---

## OmniRoute 的两条并行应对链路

### 链路 1：OpenCode Go User-Agent 与 Session 保留

#### 问题触发
- **Issue #15311**：2026-10-01 报告
  - 标题："fix(providers): OpenCode Go rewrites third-party User-Agent as OpenCode CLI"
  - 描述：OmniRoute 把第三方 agent 的 UA（如 `my-coding-agent/1.0`）改写成 `opencode/1.18.31`
  - 根本原因：`applyCliDefaults()` 无差别替换所有非 OpenCode CLI UA
  
- **为什么错误？**
  - Zen free-tier 需要 UA rewrite（Cloudflare 拒绝 datacenter IPs 发来的 generic SDK UA）
  - 但 Go authenticated path 需要保留 agent 身份
  - OmniRoute 之前没有区分这两个场景

#### 修复方案
- **PR #15453**：2026-10-03 merged（9 小时前）
  - 标题："fix(providers): keep a third-party agent's own User-Agent on OpenCode Go"
  - 主要改动：`open-sse/utils/opencodeHeaders.ts`

#### 代码实现细节

**文件：`open-sse/utils/opencodeHeaders.ts`**

```typescript
// 新增字段（line 167-169）
@param options.keepAgentUserAgent - OpenCode Go (#15311): keep a client User-Agent that
  names the agent itself, as Go's client requirements ask. A generic SDK / HTTP-library
  UA is still replaced, and a missing one is still filled.
```

**核心逻辑（line 255-258）**

```typescript
function keepsClientUserAgent(userAgent: string | undefined, keepAgentUserAgent: boolean) {
  if (satisfiesOpencodeUserAgentContract(userAgent)) return true;
  return keepAgentUserAgent && !isGenericClientUserAgent(userAgent);
}
```

**关键设计**
- `satisfiesOpencodeUserAgentContract()`：检查 `opencode/<version >= 1.17>`
- `isGenericClientUserAgent()`：识别 curl/python-requests/axios/node-fetch 等库
- 当 `keepAgentUserAgent=true` 时：
  - ✅ 保留真实 agent UA（如 `my-coding-agent/1.0`）
  - ✅ 保留 OpenCode 契约 UA
  - ❌ 替换 generic SDK/HTTP 库 UA
  - ✅ 填补缺失 UA

**在 OpenCode 执行器中应用（`open-sse/executors/opencode.ts`）**

```typescript
// 仅在 Go surface 下启用
const isGoSurface = baseUrl.includes('opencode-go');
const keepAgent = isGoSurface && !gated;  // gated = free-tier contract check

forwardOpencodeClientHeaders(headers, clientHeaders, {
  keepAgentUserAgent: keepAgent,  // Go path: true, Zen free-tier: false
  sessionBody,
});
```

#### 会话稳定性

**文件：`open-sse/utils/opencodeSessionIdentity.ts`**

```typescript
// 支持多种会话标识源（line 2-13）
const HEADER_NAMES = [
  "x-opencode-session",
  "x-session-affinity",
  "x-session-id",
  "x-claude-code-session-id",
  "session_id",
  "thread_id",
];
```

**设计意义**
- 识别调用方发来的 native session headers
- 在多轮对话中保持稳定 ID
- 对应公告："为每段对话在 x-opencode-session 中发送稳定的会话 ID"

#### 测试覆盖

**新增测试文件**：`tests/unit/opencode-go-client-user-agent-15311.test.ts`

```typescript
// R1: Go keeps the agent's UA with synthesis unset and with synthesis on
// R2: the session is unchanged across turns
// R3: generic SDK / library UAs are replaced, and a missing UA is filled
// R4: a genuine opencode/1.18.31 UA is kept
// R5: the Zen free tier still rewrites a non-CLI UA
// R6: with synthesis off, the UA and session are forwarded as before
```

**关键测试对**
- ✅ R1 + R5：同一个函数，Go 和 Zen 行为不同（通过 `keepAgentUserAgent` 控制）
- ✅ R2 + R6：session 保留/转发逻辑独立于 UA 重写

---

### 链路 2：OpenCode Zen Free-Tier 工具指纹与契约

#### 问题触发
- **Issue #15220** / **PR #15322**：2026-10-02 报告并修复
  - 背景：keyless OpenCode free-tier models 返回 403 FreeTierError
  - 错误信息："OpenCode's free tier can only be used from within OpenCode"
  
#### 根本原因：工具指纹缺陷

OpenCode 上游在 2026 年 9 月对 free-tier 实施了 4 层检查：

```
必须同时满足（缺一不可）：
1. stream: true
2. tools: [非空数组]  ← 但要求特定工具集
3. x-opencode-session 头（格式：ses_ + 12 hex + 14 base62）
4. User-Agent: opencode/<version >= 1.17>
```

**关键发现**：上游对 `tools` 的检查**不只是"非空"**

测试发现（2026-09-17）：
- ❌ `[{name: "_noop"}]` → 403
- ❌ `[{name: "bash"}, {name: "grep"}]` → 403（缺少其他工具）
- ❌ `[{name: "Bash"}, ...]` → 403（大小写错误）
- ✅ `[{name: "bash"}, {name: "glob"}, {name: "grep"}, {name: "read"}, ...]` → 200

**这是一个"特征匹配"检查**
- OpenCode Zen 识别"真实编程 agent"的方式
- 通过特定的小写工具 quartet：bash / glob / grep / read

#### 修复方案
- **PR #15322**：2026-10-02 merged
  - 标题："fix(opencode): declare the free-tier tool fingerprint the upstream requires"
  - 涉及 11 个文件改动
  - 核心文件：`open-sse/executors/opencodeFreeTierContract.ts` 和 `open-sse/executors/opencode.ts`

#### 代码实现细节

**文件：`open-sse/executors/opencodeFreeTierContract.ts`**

```typescript
// 关键改动：不再替换，而是追加（ADDITIVE）
// 旧逻辑：tools = placeholder（可能被覆盖）
// 新逻辑：tools = [bash, glob, grep, read] + observedTools + clientTools

// 三层保护
1. 确保四元组总是存在（never replace）
2. 规范小写（canonicalise）
3. 追加而非替换客户端工具
```

**关键设计文件**

1. **`open-sse/utils/opencodeFingerprint.ts`**（新文件）
   - 验证四元组拼写
   - 规范化大小写
   - 保留调用方原始名称

2. **`opencodeFreeTierContract.ts`**（修改）
   - `prepareFreeTierRequest()`：应用契约
   - `mergeClientToolsWithObserved()`：三元组合并
   - 保证 quartet 总是基础

3. **`opencode.ts` 的响应还原**
   ```typescript
   // 请求时发送规范名称（小写）
   // 响应时还原原始拼写
   confirmBorrowedToolNames(clientToolNames)  // 还原
   ```

#### 测试覆盖

**新增测试**
- `tests/unit/opencode-free-tier-fingerprint.test.ts`
- `tests/unit/opencode-free-tier-request-contract.test.ts`

**验证清单**
- ✅ 无工具数组 → 添加四元组
- ✅ 大小写错误 → 规范
- ✅ 部分工具 → 补齐
- ✅ 客户端工具 + 四元组 → 相加（不覆盖）
- ✅ 响应中 tool call 名称 → 还原为原始拼写

---

## 时间线与事件流

| 日期 | 事件 | 来源 | PR/Issue |
|------|------|------|---------|
| 2026-09 | OpenCode Go 公告发布 | 官方 | — |
| 2026-09-17 | 测试发现 free-tier 工具指纹要求 | 手工测量 | #15322 |
| 2026-10-01 | Issue #15311 报告：Go 的 UA 被改写 | 用户报告 | #15311 |
| 2026-10-02 | PR #15220 / #15322 修复 free-tier 工具指纹 | 自主发现 | #15322 |
| 2026-10-03 | PR #15453 修复 Go UA 保留 | 响应 issue | #15453 |
| 2026-10-06 | 两个 PR merged | — | #15453, #15322 |

---

## 审计视角：应对充分性评估

### 1. 及时性（Timeliness）

| 指标 | 评分 | 证据 |
|------|------|------|
| 问题发现速度 | ⭐⭐⭐⭐⭐ | 公告 9 月 → issue 10 月初，差不多 1 个月 |
| 修复速度 | ⭐⭐⭐⭐⭐ | issue 10-01 → PR merged 10-06，5 天内 |
| 预防性反应 | ⭐⭐⭐⭐⭐ | free-tier 问题是主动测量发现，不是等 user 报 403 |

### 2. 充分性（Sufficiency）

| 维度 | 覆盖 | 说明 |
|------|------|------|
| 场景隔离 | ✅ | Go surface 和 Zen free-tier 区分处理 |
| 协议层次 | ✅ | UA + session + tools 三层都覆盖 |
| 向后兼容 | ✅ | 旧逻辑还是有，只是通过 flag 分流 |
| 测试覆盖 | ✅ | 4 个新测试文件，6 个主要场景 |
| 文档更新 | ✅ | FREE_TIERS.md, ENVIRONMENT.md 更新 |

### 3. 可验证性（Verifiability）

| 控制点 | 实现 | 验证方式 |
|--------|------|---------|
| User-Agent 保留 | `keepsClientUserAgent()` | 单元测试 + 集成测试 |
| Session 稳定 | `resolveOpencodeSessionIdentity()` | 多轮对话测试 |
| 工具指纹 | `mergeClientToolsWithObserved()` | 手工测量（线上验证） |
| 响应还原 | `confirmBorrowedToolNames()` | 工具调用追踪测试 |

### 4. 规范性（Compliance）

| 规范要求 | OmniRoute 实现 |
|---------|--------------|
| "使用自身专属 user agent" | ✅ 保留 agent UA，不强制转换 |
| "不用通用 SDK/HTTP 库名称" | ✅ 显式过滤 generic UA list |
| "发送典型编程 Agent 流量" | ✅ 维持真实工具指纹 quartet |
| "稳定会话 ID" | ✅ 多源标识符读取 + deterministic hash |
| "用于路由和缓存优化" | ✅ 代码注释明确提到 prompt caching |

---

## 代码流向图

### 请求路径
```
客户端请求
    ↓
OpencodeExecutor.execute()
    ↓
buildHeaders()
    ├─ 判断 surface（Go vs Zen）
    ├─ 判断是否 gated（free-tier check）
    ↓
forwardOpencodeClientHeaders()
    ├─ keepAgentUserAgent = isGo && !gated
    ├─ preserveUA() 
    ├─ preserveSession()
    ↓
applyCliDefaults()
    ├─ keepsClientUserAgent() ← 核心分流
    │  ├─ if Go: 保留 agent UA
    │  └─ if Zen: 替换 generic UA
    ├─ canonicalizeSession()
    ↓
prepareFreeTierRequest()  ← free-tier contract gate
    ├─ ensureToolQuartet()  ← 四元组强制
    ├─ mergeClientToolsWithObserved()
    ↓
上游 OpenCode
```

### 响应路径
```
上游响应（包含 tool calls）
    ↓
parseSSEToOpenAIResponse() / parseSSEToResponsesOutput()
    ↓
confirmBorrowedToolNames()
    ├─ 取出记录的客户端原始工具名
    └─ 还原响应中的 tool call 名称
    ↓
返回给客户端（工具名匹配）
```

---

## 与 CI 失败的关联分析

### Job 111237838907 失败原因

**表面症状**：
- 构建耗时 5 分钟 26 秒被中断
- 日志："Creating an optimized production build..." → runner shutdown

**根本原因推测**：
- 并非 PR #15322 本身的 code bug
- 而是 CI 并发负载问题：
  1. PR #15322 修改了 OpenCode 调用链（新增工具指纹校验）
  2. 如果 CI 在构建中运行 OpenCode 集成测试
  3. 新的工具指纹校验可能触发额外 I/O + 网络请求
  4. 导致总耗时超过预算

**证据**：
- 构建在 "Creating an optimized production build" 阶段失败
- 这是 Turbopack Next.js 构建，内存密集
- 没有配置 `timeout-minutes`
- 日志无异常，只是时间溢出

**修复方向**：
1. 在构建 job 中添加 `timeout-minutes: 20`
2. 或优化构建缓存策略（Turbopack 不用 webpack 旧缓存）

---

## 结论：OmniRoute 的应对模式

### 核心特点

1. **分场景策略，不是全局一刀切**
   - Go：保留 agent 身份
   - Zen free-tier：强制契约检查
   - 同一个函数，通过 flag 分流

2. **多层防御，不是单点修复**
   - User-Agent 识别与保留
   - Session 稳定性维持
   - 工具指纹 quartet 检查
   - 响应工具名还原

3. **测试驱动，不是事后补丁**
   - 4 个新测试文件
   - 手工测量线上验证
   - 回归测试 + 集成测试

4. **文档同步，不是代码更新**
   - FREE_TIERS.md probe matrix
   - ENVIRONMENT.md placeholder 说明
   - noauth.ts authHint/freeNote 更新

### 工程安全性

✅ **隔离** - 不同 surface 独立处理
✅ **渐进** - 旧逻辑保留，新逻辑通过 opt-in flag
✅ **可观测** - 测试用例完整，代码注释清晰
✅ **规范** - 严格按 OpenCode 公告要求实现

---

## 后续追查方向

1. **细化 user-agent 过滤列表**
   - `GENERIC_CLIENT_USER_AGENT_RE` 覆盖范围
   - 是否有新增 SDK 需要添加

2. **工具指纹 quartet 的稳定性**
   - OpenCode 是否会扩展或变更这个集合
   - 如何监控上游政策变化

3. **缓存策略与会话 ID**
   - prompt caching 实际命中率
   - session ID 分布是否均衡

4. **CI/CD 性能优化**
   - 构建 timeout 配置
   - 测试并发度调整

