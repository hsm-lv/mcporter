---
summary: '技术架构分析：MCPorter 的技术栈、模块设计、核心功能与实现细节。'
read_when:
  - 'Understanding the project structure and design decisions'
  - 'Onboarding new contributors or reviewing the codebase'
---

# MCPorter 技术架构分析

> 本文档分析 MCPorter 的技术栈、架构设计以及核心功能实现。

---

## 一、项目概述

**MCPorter**（v0.7.x）是一个面向 [Model Context Protocol（MCP）](https://github.com/modelcontextprotocol/specification) 的 TypeScript 运行时、CLI 工具集和代码生成工具包。它的核心目标是让开发者和 AI 代理能够：

- **零配置**地发现并调用系统中已配置的 MCP 服务器；
- 通过**命令行**直接调用 MCP 工具（`mcporter call`）；
- 通过**编程 API** 在 TypeScript 项目中组合多个 MCP 工具；
- 一键将任意 MCP 服务器**生成**为独立可分发的 CLI 工具；
- 为 MCP 服务器自动**生成 TypeScript 类型定义**，实现强类型访问。

---

## 二、技术栈

### 运行时与语言

| 层次 | 技术 |
|------|------|
| 语言 | TypeScript 5.x（ESM 模块，`"type": "module"`） |
| 运行时 | Node.js ≥ 20.11；可选用 Bun 进行打包/编译 |
| 包管理 | pnpm 10.x（workspace 模式） |

### 核心依赖

| 依赖 | 用途 |
|------|------|
| `@modelcontextprotocol/sdk` | MCP 协议客户端（stdio / SSE / Streamable-HTTP 传输） |
| `zod` | 配置模式定义与运行时验证 |
| `commander` | CLI 参数解析基础框架 |
| `acorn` | JavaScript 表达式解析器，用于函数调用语法（`tool(arg1, arg2)`） |
| `rolldown` | 为生成的 CLI 进行 Node.js Bundle 打包 |
| `jsonc-parser` | 支持注释和尾逗号的 JSONC 配置文件解析 |
| `@iarna/toml` | 备用 TOML 格式配置解析 |
| `ora` | CLI 进度动画 |
| `es-toolkit` | 工具函数库（替代 lodash） |

### 开发工具

| 工具 | 用途 |
|------|------|
| Vitest 4.x | 单元测试 + 集成测试 |
| Biome | 代码格式化 + 部分 Lint |
| Oxlint + tsgolint | TypeScript 感知 Lint，零错误容忍 |
| `@typescript/native-preview` | 原生 TypeScript 编译器（`tsgo`）用于类型检查 |

---

## 三、模块架构

项目源码位于 `src/`，按职责分为以下核心层次：

```
src/
├── index.ts                 # 公共 API 入口（供库使用者 import）
├── cli.ts                   # CLI 入口（bin/mcporter）
│
├── runtime.ts               # 连接池化运行时（核心引擎）
├── server-proxy.ts          # 服务器代理（ergonomic camelCase API）
├── result-utils.ts          # 调用结果封装（.text/.json/.images 等）
│
├── config.ts                # 配置加载入口
├── config-schema.ts         # Zod 配置模式定义
├── config-normalize.ts      # 配置规范化（多字段别名合并）
├── config-imports.ts        # 从第三方配置文件导入
│
├── oauth.ts                 # OAuth 浏览器握手（本地 HTTP 回调服务器）
├── oauth-persistence.ts     # OAuth Token 持久化缓存
├── oauth-vault.ts           # 安全令牌存储
│
├── generate-cli.ts          # CLI 代码生成协调器
├── lifecycle.ts             # 服务器连接生命周期（keep-alive / ephemeral）
│
├── cli/                     # CLI 子命令实现
│   ├── call-command.ts      # `mcporter call`
│   ├── list-command.ts      # `mcporter list`
│   ├── auth-command.ts      # `mcporter auth`
│   ├── config-command.ts    # `mcporter config`
│   ├── daemon-command.ts    # `mcporter daemon`
│   ├── emit-ts-command.ts   # `mcporter emit-ts`
│   ├── generate-cli-runner.ts # `mcporter generate-cli`
│   ├── call-expression-parser.ts # JS 函数调用语法解析
│   ├── command-inference.ts # 命令自动推断（`mcporter linear.list_issues`）
│   └── generate/            # CLI 生成器子模块
│       ├── artifacts.ts     # 打包 / 编译产物
│       ├── template.ts      # 代码模板渲染
│       └── tools.ts         # 工具元数据
│
├── config/                  # 配置读取子模块
│   ├── path-discovery.ts    # 配置文件路径发现
│   ├── read-config.ts       # JSONC/TOML 文件读取
│   └── imports/             # 各 IDE 配置导入（cursor / claude / vscode 等）
│
├── daemon/                  # 后台守护进程
│   ├── host.ts              # 守护进程服务端（Unix Socket）
│   ├── client.ts            # 守护进程客户端
│   ├── protocol.ts          # JSON-RPC 协议类型定义
│   └── runtime-wrapper.ts   # 将 keep-alive 调用路由到守护进程
│
└── runtime/                 # 运行时底层子模块
    ├── transport.ts         # 传输层（stdio/SSE/HTTP）连接与 OAuth 注入
    ├── oauth.ts             # OAuth 连接重试逻辑
    ├── errors.ts            # 错误分类与连接重置判断
    └── utils.ts             # 超时、命令参数解析工具
```

### 架构分层图

```
┌─────────────────────────────────────────────────────────────────┐
│  用户层：CLI (`mcporter`) / 库 API (import from "mcporter")      │
└────────────┬───────────────────────────┬────────────────────────┘
             │                           │
    ┌────────▼──────────┐     ┌──────────▼───────────┐
    │   CLI 命令层       │     │   编程 API 层          │
    │  call/list/auth/  │     │  createRuntime()       │
    │  config/daemon/   │     │  createServerProxy()   │
    │  generate-cli/    │     │  callOnce()            │
    │  emit-ts          │     │                        │
    └────────┬──────────┘     └──────────┬─────────────┘
             │                           │
             └──────────┬────────────────┘
                        │
          ┌─────────────▼──────────────────┐
          │        Runtime（连接池）         │
          │   listTools / callTool /        │
          │   connect / close               │
          └───────┬───────────┬─────────────┘
                  │           │
     ┌────────────▼──┐  ┌─────▼──────────────────────┐
     │  Daemon 层     │  │  Transport 层               │
     │ (keep-alive   │  │  stdio / SSE / HTTP         │
     │  Unix Socket) │  │  + OAuth 注入 / 重试         │
     └───────────────┘  └────────────────────────────┘
                                   │
                   ┌───────────────▼──────────────┐
                   │  配置层（Config）              │
                   │  mcporter.json + 多 IDE 导入   │
                   │  Cursor/Claude/Codex/VS Code  │
                   └──────────────────────────────┘
```

---

## 四、核心功能与实现

### 4.1 零配置服务器发现（Config Discovery）

**实现文件**：`src/config.ts`、`src/config-imports.ts`、`src/config/imports/`

MCPorter 从多个来源自动合并 MCP 服务器配置：

1. 显式传入的 `configPath` 参数；
2. 环境变量 `MCPORTER_CONFIG`；
3. `<cwd>/config/mcporter.json[c]`；
4. `~/.mcporter/mcporter.json[c]`；
5. **导入**：Cursor（`.cursor/mcp.json`）、Claude Code/Desktop、Codex、Windsurf、OpenCode、VS Code 的配置文件。

配置文件使用 JSONC 格式（支持注释和尾逗号），通过 `jsonc-parser` 解析，Zod schema（`RawEntrySchema`）做运行时校验与规范化。环境变量占位符（`${VAR}`、`${VAR:-default}`、`$env:VAR`）在加载时展开。

---

### 4.2 连接池化运行时（Runtime）

**实现文件**：`src/runtime.ts`、`src/runtime/transport.ts`

`createRuntime()` 返回实现 `Runtime` 接口的对象，内部维护一个按服务器名索引的 `ClientContext` 连接池：

```
Runtime
  ├── listServers()          → 返回已知服务器名列表
  ├── listTools(server)      → 列出工具（带缓存）
  ├── callTool(server, tool) → 调用工具
  ├── connect(server)        → 建立/复用连接
  └── close([server])        → 关闭传输
```

**传输层**（`src/runtime/transport.ts`）根据 `ServerDefinition.command` 类型自动选择：

- `stdio`：`StdioClientTransport`，子进程管理
- `http`（Streamable HTTP）：`StreamableHTTPClientTransport`
- `sse`：`SSEClientTransport`

---

### 4.3 OAuth 认证流程

**实现文件**：`src/oauth.ts`、`src/oauth-persistence.ts`、`src/runtime/oauth.ts`

对于 OAuth 保护的 MCP 服务器：

1. 运行时检测到 `auth: "oauth"` 或 401 响应；
2. 在 `127.0.0.1` 随机端口启动本地 HTTP 回调服务器；
3. 用系统浏览器（`open`/`xdg-open`/`start`）打开授权 URL；
4. 等待回调获取授权码后完成 token 交换；
5. Token 持久化缓存至 `~/.mcporter/<server>/`，下次请求跳过浏览器流程。

OAuth timeout 默认 60 秒，可通过 `--oauth-timeout` 或 `MCPORTER_OAUTH_TIMEOUT_MS` 调整。

---

### 4.4 守护进程（Daemon）

**实现文件**：`src/daemon/`

对于标记为 `"lifecycle": "keep-alive"` 的服务器（如 `chrome-devtools`），MCPorter 通过 Unix Domain Socket 运行后台守护进程：

- **`daemon host`**：在 socket 上监听 JSON-RPC 请求（`callTool`、`listTools`、`status` 等），复用同一个 `Runtime` 实例，保持 MCP 连接持久化；
- **`DaemonClient`**：CLI 进程通过 socket 将请求转发给守护进程；
- **`createKeepAliveRuntime`**：透明包装，keep-alive 服务器的调用路由到守护进程，其余服务器直接调用；
- 守护进程自动检测配置文件变更（mtime），在配置更新后重启连接。

---

### 4.5 函数调用语法解析器

**实现文件**：`src/cli/call-expression-parser.ts`、`src/cli/call-argument-expression.ts`

`mcporter call` 支持两种参数风格：

```bash
# CLI 风格（colon/equals 分隔）
mcporter call linear.create_comment issueId:ENG-123 body:'Hello'

# 函数调用风格（JS 语法）
mcporter call 'linear.create_comment(issueId: "ENG-123", body: "Hello")'
```

函数调用风格使用 **acorn** 解析 AST，提取具名参数或位置参数，并按工具 JSON Schema 的 `required` 字段顺序自动映射。参数值支持字符串、数字、布尔、null、嵌套对象/数组，以及不带引号的宽松语法。

---

### 4.6 工具名自动纠错

**实现文件**：`src/cli/command-inference.ts`

输入工具名时（如 `linear.listIssues`），MCPorter 会：

1. 精确匹配；
2. 大小写不敏感 + 去标点匹配；
3. 编辑距离（Levenshtein）模糊匹配；
4. 自动重试并打印 `Did you mean list_issues?` 提示。

---

### 4.7 CLI 代码生成（`generate-cli`）

**实现文件**：`src/generate-cli.ts`、`src/cli/generate/`

`mcporter generate-cli <server>` 流程：

1. 连接目标 MCP 服务器，拉取完整工具列表（含 JSON Schema）；
2. 用模板引擎生成 TypeScript 文件（内嵌工具 schema 和再生成元数据）；
3. 可选使用 **Rolldown**（Node.js）或 **Bun** 打包为独立 `.js`；
4. 可选用 Bun `--compile` 生成独立可执行文件。

生成的 CLI 继承 MCPorter 的彩色 help 布局，并内嵌再生成元数据（可通过 `mcporter inspect-cli` 查看，`mcporter generate-cli --from` 重新生成）。

---

### 4.8 TypeScript 类型生成（`emit-ts`）

**实现文件**：`src/cli/emit-ts-command.ts`、`src/cli/emit-ts-templates.ts`

`mcporter emit-ts <server>` 根据工具 JSON Schema 生成：

- **`--mode types`**（默认）：`.d.ts` 接口文件，与 `mcporter list` 输出的签名一致；
- **`--mode client`**：同时生成 `.ts` 包装文件，开箱即用封装 `createRuntime` 和 `createServerProxy`。

---

### 4.9 服务器代理 API（`createServerProxy`）

**实现文件**：`src/server-proxy.ts`

通过 ES Proxy 机制，将属性访问映射为工具调用：

```ts
const chrome = createServerProxy(runtime, "chrome-devtools");
const result = await chrome.takeSnapshot(); // 等价于 callTool("take_snapshot", ...)
```

- camelCase → kebab-case 工具名转换（`takeSnapshot` → `take-snapshot`）；
- 位置参数按 JSON Schema `required` 字段顺序映射；
- 自动填充 JSON Schema `default` 值；
- 返回 `CallResult` 封装，提供 `.text()`、`.json()`、`.images()`、`.content()`、`.markdown()` 访问器。

---

## 五、测试体系

| 类型 | 路径 | 框架 |
|------|------|------|
| 单元测试 | `tests/*.test.ts` | Vitest |
| 集成测试（本地 MCP 进程） | `tests/*.integration.test.ts` | Vitest + 内置 stdio fixture 服务器 |
| Live 测试（真实端点） | `tests/live/*.test.ts` | 需设置 `MCP_LIVE_TESTS=1` |

测试覆盖范围包括：配置加载与规范化、OAuth 流程、守护进程、CLI 参数解析、工具调用、错误分类、代码生成产物等。

---

## 六、发布与分发

- **npm**：`pnpm publish`（自动执行 lint + test + build）；
- **Homebrew**（steipete/tap）：通过 tap 分发；
- **Bun 独立二进制**：`scripts/build-bun.ts` 编译为 `dist-bun/mcporter-<platform>-<arch>-v<version>.tar.gz`；
- **CI/CD**：GitHub Actions（`.github/workflows/ci.yml`），执行 `pnpm check` + `pnpm build` + `pnpm test`。
