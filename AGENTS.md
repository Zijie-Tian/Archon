# AGENTS.md — Archon 项目架构指南

> 本文件记录 Archon 代码库的核心架构理解，供 AI 开发助手参考。
> 基于代码库分析生成，涵盖设计哲学、模块关系、关键模式。

## 项目定位

**Archon** 是一个开源的 AI 编码工作流引擎。它将软件开发流程（规划、实现、验证、代码审查、PR 创建）编码为可重复的 YAML 工作流，让 AI 编码助手（Claude Code、Codex）能够按照确定性流程执行开发任务。

核心类比：
- Dockerfile → 基础设施标准化
- GitHub Actions → CI/CD 标准化  
- **Archon → AI 编码工作流标准化**

## 架构概览

### 技术栈
- **运行时**: Bun + TypeScript（严格模式）
- **数据库**: SQLite（默认）/ PostgreSQL（可选）
- **Web 框架**: Hono + OpenAPI (Zod)
- **前端**: React + Vite + Tailwind v4 + shadcn/ui + Zustand
- **AI SDK**: Claude Agent SDK、Codex SDK、Pi (社区)

### Monorepo 结构（11 个包）

```
packages/
├── paths/       # @archon/paths — 基础层：路径解析、日志、遥测（零依赖）
├── git/         # @archon/git — Git 操作：worktree、分支、仓库（依赖 paths）
├── providers/   # @archon/providers — AI 提供商：Claude、Codex、Pi（依赖 paths）
├── isolation/   # @archon/isolation — 隔离环境：worktree 管理（依赖 git + paths）
├── workflows/   # @archon/workflows — 工作流引擎：DAG 执行（依赖 git + paths + providers/types）
├── core/        # @archon/core — 业务逻辑：数据库、编排器、命令处理（依赖所有上层）
├── adapters/    # @archon/adapters — 平台适配器：Slack、Telegram、GitHub、Discord（依赖 core）
├── server/      # @archon/server — HTTP 服务器 + Web 适配器（SSE）（依赖 adapters）
├── cli/         # @archon/cli — 命令行接口（依赖 server + core）
├── web/         # @archon/web — React 前端（零 @archon/* 依赖，使用 OpenAPI 生成类型）
└── docs-web/    # @archon/docs-web — 文档站点（Astro Starlight）
```

### 依赖方向（严格分层）

```
paths → git → providers/isolation → workflows → core → adapters/server/cli
                                      ↑
                                web（独立，通过 API 通信）
```

**关键规则**:
- `@archon/paths` 零 `@archon/*` 依赖
- `@archon/providers/types` 是契约子路径，零 SDK 导入
- `@archon/workflows` 不依赖 `@archon/core`（避免循环依赖）
- `@archon/web` 零 `@archon/*` 依赖，类型来自 OpenAPI 代码生成

## 核心抽象层

### 1. 平台适配器接口 (IPlatformAdapter)

所有聊天平台（Slack、Telegram、GitHub、Discord、Web）实现统一接口：

```typescript
interface IPlatformAdapter {
  sendMessage(conversationId, message, metadata?): Promise<void>;
  ensureThread(conversationId, context?): Promise<string>;
  getStreamingMode(): 'stream' | 'batch';
  getPlatformType(): string;
  start(): Promise<void>;
  stop(): void;
  sendStructuredEvent?(conversationId, event: MessageChunk): Promise<void>;
  emitRetract?(conversationId): Promise<void>;
}
```

**设计原则**: 
- 认证检查封装在适配器内部（如 Slack 的 `SLACK_ALLOWED_USER_IDS`）
- 未授权用户静默拒绝，记录脱敏日志
- Web 适配器通过 SSE（Server-Sent Events）实现实时流式传输

### 2. AI 提供商接口 (IAgentProvider)

```typescript
interface IAgentProvider {
  sendQuery(prompt, cwd, resumeSessionId?, options?): AsyncGenerator<MessageChunk>;
  getType(): string;
  getCapabilities(): ProviderCapabilities;
}
```

**MessageChunk** 是区分联合类型（9 种变体）：
- `assistant`（流式文本）、`thinking`、`system`
- `tool` / `tool_result`（工具调用）
- `result`（最终结果，含 token 用量）
- `rate_limit`、`workflow_dispatch`

**设计原则**:
- 契约层 (`@archon/providers/types`) 零 SDK 依赖
- 每个提供商注册时声明静态能力标志（MCP、hooks、agents 等）
- 原始 `nodeConfig` 由提供商内部翻译为 SDK 特定选项

### 3. 工作流存储接口 (IWorkflowStore)

```typescript
interface IWorkflowStore {
  createWorkflowRun(...): Promise<WorkflowRun>;
  getWorkflowRun(id): Promise<WorkflowRun | null>;
  resumeWorkflowRun(id): Promise<WorkflowRun>;
  updateWorkflowRun(id, updates): Promise<void>;
  createWorkflowEvent(...): Promise<void>;  // 永不抛出
  getCompletedDagNodeOutputs(runId): Promise<Map<string, string>>;
}
```

**设计原则**:
- 窄接口（16 个方法），实现由 `@archon/core` 提供
- `createWorkflowEvent` 是观察性的——工作流执行继续，无论事件持久化是否成功
- 路径独占锁防止并发工作流冲突

### 4. 隔离存储接口 (IIsolationStore)

```typescript
interface IIsolationStore {
  getById(id): Promise<IsolationEnvironmentRow | null>;
  create(env): Promise<IsolationEnvironmentRow>;
  updateStatus(id, status): Promise<void>;
}
```

**设计原则**:
- 仅 5 个方法，极简契约
- 隔离解析器返回区分联合结果（`existing | new | none | blocked`）

## 关键设计模式

### 模式 1: 依赖注入（结构子类型）

无运行时 DI 容器，利用 TypeScript 结构子类型实现：

```typescript
// @archon/workflows 定义
interface WorkflowDeps {
  store: IWorkflowStore;
  getAgentProvider: (id: string) => IAgentProvider;
  loadConfig: (cwd: string) => Promise<WorkflowConfig>;
}

// @archon/core 提供
export function createWorkflowDeps(): WorkflowDeps {
  return {
    store: createWorkflowStore(),      // 桥接核心 DB
    getAgentProvider,                  // 来自注册表
    loadConfig: loadMergedConfig,      // 配置加载器
  };
}
```

### 模式 2: 桥接模式 (Store Adapter)

`@archon/core` 将核心 DB 模块桥接到工作流引擎的 `IWorkflowStore`：

```typescript
// packages/core/src/workflows/store-adapter.ts
export function createWorkflowStore(): IWorkflowStore {
  return {
    createWorkflowRun: workflowDb.createWorkflowRun,
    getWorkflowRun: workflowDb.getWorkflowRun,
    createWorkflowEvent: async (data) => {
      try { await workflowEventDb.createWorkflowEvent(data); }
      catch (err) { /* 保证不抛出契约 */ }
    },
  };
}
```

### 模式 3: 懒加载日志器（测试兼容）

每个模块使用延迟初始化模式，允许测试 mock 拦截：

```typescript
let cachedLog: ReturnType<typeof createLogger> | undefined;
function getLog(): ReturnType<typeof createLogger> {
  if (!cachedLog) cachedLog = createLogger('module.name');
  return cachedLog;
}
```

### 模式 4: 区分联合 + 穷尽检查

```typescript
// 隔离解析结果
type IsolationResolution =
  | { status: 'existing'; cwd: string; env: IsolationEnvironmentRow }
  | { status: 'new'; cwd: string; env: IsolationEnvironmentRow }
  | { status: 'none'; cwd: string; env: null };

// Git 结果
type GitResult<T> = 
  | { ok: true; value: T }
  | { ok: false; error: GitError };

// 会话转换触发器
type TransitionTrigger =
  | 'first-message'
  | 'plan-to-execute'
  | 'isolation-changed'
  | 'reset-requested'
  | 'worktree-removed'
  | 'conversation-closed';
```

### 模式 5: 错误分类（Fail Fast）

```typescript
// 工作流执行器共享
function classifyError(error): 'TRANSIENT' | 'FATAL' | 'UNKNOWN' {
  // FATAL 模式优先（防止带崩溃后缀的认证错误被重试）
  if (FATAL_PATTERNS.some(p => message.includes(p))) return 'FATAL';
  if (TRANSIENT_PATTERNS.some(p => message.includes(p))) return 'TRANSIENT';
  return 'UNKNOWN';
}

// 隔离错误
function classifyIsolationError(err): { message: string; known: boolean } {
  // 11 种已知模式映射到用户友好消息
}
```

### 模式 6: Zod 优先验证

```typescript
// DAG 节点使用 superRefine 而非 z.union()
const dagNodeSchema = z.object({
  id: z.string().min(1),
  // ...所有可能的字段
}).superRefine((raw, ctx) => {
  // 互斥性：恰好一个 command/prompt/bash/loop/approval/script/cancel
  // 非 AI 节点的 AI 字段限制
}).transform(raw => /* 生成 6 种变体之一 */);
```

### 模式 7: 单例工厂 + 配置注入

```typescript
// 隔离提供商工厂
let provider: IIsolationProvider | null = null;
let configuredLoader: RepoConfigLoader = () => Promise.resolve(null);

export function configureIsolation(loader: RepoConfigLoader): void {
  configuredLoader = loader;
  provider = null;  // 重置以使用新加载器
}

export function getIsolationProvider(): IIsolationProvider {
  provider ??= new WorktreeProvider(configuredLoader);
  return provider;
}
```

## 数据流：端到端示例

用户从 Slack 发送 "fix issue #42"：

```
1. Slack 适配器接收消息
   └─→ handleMessage(telegramAdapter, conversationId, message)

2. 编排器 (orchestrator-agent.ts)
   ├─ 获取/创建会话
   ├─ 检查确定性命令（非 "/" 开头，跳过）
   ├─ 加载所有代码库 + 发现工作流
   ├─ 构建系统提示（项目/工作流上下文）
   └─ 批量模式发送到 AI

3. AI 响应：
   "我将使用 archon-fix-github-issue 工作流..."
   /invoke-workflow archon-fix-github-issue --project ... --prompt "fix issue #42"

4. 编排器检测 /invoke-workflow
   └─→ 解析工作流名称、项目、合成提示

5. dispatchOrchestratorWorkflow()
   ├─ 更新会话 codebase_id
   ├─ validateAndResolveIsolation()
   │   └─ IsolationResolver.resolve()
   │       ├─ 检查现有环境 → 无
   │       ├─ 查找可重用 → 无
   │       └─ 创建新 worktree
   ├─ 链接隔离环境到会话
   └─ 调度工作流（前台/后台）

6. executeWorkflow()
   ├─ 解析配置、提供商、模型
   ├─ 路径锁保护
   ├─ 创建产物目录
   └─ 委托给 executeDagWorkflow()

7. DAG 执行器按拓扑顺序运行节点：
   ├─ plan → AI 生成计划
   ├─ implement → AI 编写代码
   ├─ run-tests → bash: "bun run validate"
   ├─ review → AI 审查变更
   ├─ approve → 暂停，等待人工审批
   └─ create-pr → AI 创建 PR

8. 状态更新回流：
   ├─ 事件发射器 → 结构化事件 → SSE 桥接 → Web UI
   ├─ platform.sendMessage() → 用户看到进度
   └─ DB 更新运行状态、事件、元数据
```

## 关键文件速查

| 组件 | 入口文件 | 关键接口 |
|------|---------|---------|
| **服务器** | `packages/server/src/index.ts` | `startServer()` |
| **CLI** | `packages/cli/src/cli.ts` | 命令路由 |
| **编排器** | `packages/core/src/orchestrator/orchestrator-agent.ts` | `handleMessage()` |
| **命令处理** | `packages/core/src/handlers/command-handler.ts` | `handleCommand()` |
| **工作流执行器** | `packages/workflows/src/executor.ts` | `executeWorkflow()` |
| **DAG 执行器** | `packages/workflows/src/dag-executor.ts` | `executeDagWorkflow()` |
| **工作流加载器** | `packages/workflows/src/loader.ts` | `parseWorkflow()` |
| **工作流路由** | `packages/workflows/src/router.ts` | `resolveWorkflowName()` |
| **配置加载** | `packages/core/src/config/config-loader.ts` | `loadConfig()` |
| **数据库连接** | `packages/core/src/db/connection.ts` | `getDatabase()` |
| **隔离解析器** | `packages/isolation/src/resolver.ts` | `IsolationResolver` |
| **提供商注册表** | `packages/providers/src/registry.ts` | `registerBuiltinProviders()` |
| **Web 适配器** | `packages/server/src/adapters/web.ts` | `WebAdapter` |

## 设计哲学（来自 CLAUDE.md）

### KISS — 保持简单
- 优先直接控制流，而非巧妙的元编程
- 优先显式分支和类型接口，而非隐藏动态行为
- 错误路径应明显且局部化

### YAGNI — 你不会需要它
- 没有具体用例不添加配置键、接口方法、功能标志
- 没有当前调用者不引入推测性抽象
- 不支持的路径显式报错，而非添加部分假支持

### DRY + 三次法则
- 小范围局部逻辑允许重复以保持清晰
- 相同模式出现至少三次并稳定后才提取共享工具
- 提取时保持模块边界，避免隐藏耦合

### SRP + ISP — 单一职责 + 接口隔离
- 每个模块和包聚焦一个关注点
- 通过实现现有窄接口扩展行为
- 避免胖接口和"上帝模块"

### Fail Fast + 显式错误
- 对不支持或不安全状态尽早抛出清晰错误
- 永不静默吞没错误
- 有意回退需注释说明；否则抛出

### 跨进程边界无自主生命周期变更
- 无法可靠区分"在其他地方活跃运行"和"崩溃孤立"时
- 不得基于计时器或陈旧猜测自主标记工作为失败/取消/放弃
- 向用户展示模糊状态并提供一键操作

## 开发约定

### 导入模式
```typescript
// ✅ 类型导入
import type { IPlatformAdapter, Conversation } from '@archon/core';

// ✅ 值导入
import { handleMessage, ConversationLockManager } from '@archon/core';

// ✅ 命名空间导入（多导出子模块）
import * as conversationDb from '@archon/core/db/conversations';
import * as git from '@archon/git';

// ✅ 工作流引擎直接子路径
import type { WorkflowDeps } from '@archon/workflows/deps';
import { executeWorkflow } from '@archon/workflows/executor';

// ❌ 禁止：通用导入主包
import * as core from '@archon/core';

// ❌ 禁止：web 从 workflows 导入（服务器包）
import type { DagNode } from '@archon/workflows/schemas/dag-node';
// ✅ 正确：使用 api.ts 重导出
import type { DagNode } from '@/lib/api';
```

### Zod 模式约定
- 模式命名：camelCase，描述性后缀（`workflowRunSchema`、`errorSchema`）
- 类型推导：始终使用 `z.infer<typeof schema>`
- 从 `@hono/zod-openapi` 导入 `z`（非 `zod` 直接）
- 路由模式：`packages/server/src/routes/schemas/`
- 引擎模式：`packages/workflows/src/schemas/`

### 日志约定
```typescript
import { createLogger } from '@archon/paths';
const log = createLogger('domain.subsystem');

// 事件命名：{domain}.{action}_{state}
log.info({ conversationId }, 'session.create_started');
log.info({ conversationId, sessionId }, 'session.create_completed');
log.error({ err, conversationId }, 'session.create_failed');
```

### 测试隔离
- **不要从仓库根运行 `bun test`**——会发现所有包的测试文件并在单进程中运行，导致 ~135 个 mock 污染失败
- 始终使用 `bun run test`（使用 `bun --filter '*' test` 进行每包隔离）
- `mock.module()` 是进程全局且不可逆的——`mock.restore()` 不撤销它
- 使用 `spyOn()` 进行内部模块间谍，`spy.mockRestore()` 对间谍有效

## 扩展指南

### 添加新 AI 提供商
1. 在 `packages/providers/src/community/<id>/` 创建目录
2. 实现 `IAgentProvider` 接口
3. 在 `registry.ts:registerCommunityProviders()` 中注册
4. 声明能力标志（`ProviderCapabilities`）

### 添加新平台适配器
1. 在 `packages/adapters/src/chat/<platform>/` 创建目录
2. 实现 `IPlatformAdapter` 接口
3. 在适配器内部处理认证
4. 在 `packages/server/src/index.ts` 中接入

### 添加新工作流节点类型
1. 修改 `packages/workflows/src/schemas/dag-node.ts`（superRefine + transform）
2. 在 `packages/workflows/src/dag-executor.ts` 添加执行函数
3. 更新 `packages/workflows/src/loader.ts` 验证逻辑
4. 为 Web UI 重新生成 OpenAPI 类型：`bun generate:types`

### 添加新数据库表
1. 在 `packages/core/src/db/` 创建新文件（参考 `conversations.ts` 模式）
2. 导出命名函数和命名空间
3. 在 `packages/core/src/db/index.ts` 中注册
4. 添加迁移 SQL 到 `migrations/`

## 版本信息

- **生成时间**: 2026-05-14
- **基于版本**: v0.3.11
- **分支**: docs/agents-md-init
- **分析范围**: 完整代码库（packages/* 所有模块）

---

*本文件由 AI 助手基于代码库分析生成，应随代码演进定期更新。*
