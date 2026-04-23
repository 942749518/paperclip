# Paperclip 项目架构文档

> 本文档基于项目官方文档和代码结构分析生成  
> 生成时间: 2026-04-23

---

## 一、项目概述

### 1.1 项目定位

**Paperclip** 是面向 AI 自主公司的控制平面（Control Plane）系统，用于管理、协调和治理由 AI 智能体组成的公司组织结构。一个 Paperclip 实例可以运行多个公司（Company），每个公司都是由 AI 智能体构成的完整组织。

### 1.2 核心目标

- 管理智能体作为员工（hire, organize, track）
- 定义组织结构（org charts）
- 实时跟踪工作（实时监控每个智能体的工作状态）
- 控制成本（预算、支出跟踪、燃尽率）
- 目标对齐（工作可追溯至公司顶层目标）
- 公司知识存储（共享大脑）

### 1.3 设计理念

1. **不强制智能体运行方式**：支持多种适配器（OpenClaw、Python脚本、Claude Code、Codex等）
2. **Company 是组织单元**：所有业务实体都隶属于一个 Company
3. **控制平面而非执行平面**：Paperclip 负责编排，智能体在任意位置运行并通过 API "回家"
4. **所有工作可追溯至目标**：任务必须能够解释为什么对公司目标重要

---

## 二、架构设计

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        Paperclip Control Plane                   │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │    Server    │  │     UI       │  │       CLI            │   │
│  │  (Express)   │  │  (React+Vite)│  │  (Node/TS)           │   │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘   │
│         │                 │                      │               │
│  ┌──────┴─────────────────┴──────────────────────┴──────────┐   │
│  │                      Packages                             │   │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ │   │
│  │  │   db   │ │ shared │ │adapters│ │plugins │ │adapter-│ │   │
│  │  │Drizzle │ │types/  │ │claude/ │ │system  │ │ utils  │ │   │
│  │  │PG/SQL  │ │consts  │ │codex/  │ │        │ │        │ │   │
│  │  │        │ │        │ │cursor  │ │        │ │        │ │   │
│  │  └────────┘ └────────┘ └────────┘ └────────┘ └────────┘ │   │
│  └────────────────────────────────────────────────────────────┘   │
│                              │                                   │
│  ┌───────────────────────────┴───────────────────────────────┐   │
│  │                     PostgreSQL (Data Store)                │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │   │
│  │  │  Embedded    │  │  Local       │  │  Hosted          │ │   │
│  │  │  (PGlite)    │  │  (Docker)    │  │  (Supabase)      │ │   │
│  │  └──────────────┘  └──────────────┘  └──────────────────┘ │   │
│  └────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘

                    ▲
                    │ API Calls
                    │
    ┌───────────────┼───────────────┐
    │               │               │
┌───┴───┐    ┌──────┴──────┐  ┌────┴────┐
│ Agent │    │   Agent     │  │  Agent  │
│ (HTTP)│    │  (Process)  │  │ (Custom)│
└───────┘    └─────────────┘  └─────────┘
```

### 2.2 目录结构

```
paperclip/
├── cli/                    # CLI 命令行工具
├── doc/                    # 项目文档
│   ├── GOAL.md            # 项目愿景与目标
│   ├── PRODUCT.md         # 产品定义
│   ├── SPEC-implementation.md  # V1 实现规范
│   ├── DEVELOPING.md      # 开发指南
│   ├── DATABASE.md        # 数据库文档
│   └── ...
├── packages/              # 共享包（Monorepo）
│   ├── db/               # Drizzle ORM + PostgreSQL
│   ├── shared/           # 共享类型、常量、验证器
│   ├── adapters/         # 智能体适配器实现
│   │   ├── claude/      # Claude 适配器
│   │   ├── codex/       # Codex 适配器
│   │   └── cursor/      # Cursor 适配器
│   ├── adapter-utils/    # 适配器共享工具
│   └── plugins/          # 插件系统
├── server/               # Express REST API 服务器
│   ├── src/
│   │   ├── routes/      # API 路由
│   │   ├── services/    # 业务逻辑服务
│   │   └── ...
├── ui/                   # React + Vite 前端
│   ├── src/
│   │   ├── pages/       # 页面组件
│   │   ├── components/  # 共享组件
│   │   └── ...
├── docker/               # Docker 配置
├── scripts/              # 构建脚本
├── tests/                # 测试用例
└── skills/               # 智能体技能定义
```

### 2.3 运行模式

| 模式 | 描述 | 使用场景 |
|------|------|----------|
| `local_trusted` | 单用户本地可信部署，无登录摩擦 | 本地开发、个人使用 |
| `authenticated` | 需登录模式，支持私有/公共网络部署 | 团队协作、生产环境 |

---

## 三、核心功能模块

### 3.1 公司管理（Company）

**核心实体**：`companies` 表

**功能**：
- 创建/列出/获取/更新/归档公司
- 公司状态管理：`active | paused | archived`
- 每个业务记录必须隶属于一个公司

**字段**：
```typescript
{
  id: uuid          // 主键
  name: string      // 公司名称
  description: text // 描述
  status: enum      // active | paused | archived
}
```

### 3.2 智能体管理（Agents）

**核心实体**：`agents` 表

**功能**：
- 智能体生命周期管理
- 组织结构树（`reports_to` 字段）
- 状态管理：`active | paused | idle | running | error | terminated`
- 适配器配置（`adapter_type`, `adapter_config`）
- 预算控制（`budget_monthly_cents`, `spent_monthly_cents`）

**状态流转**：
```
idle → running → idle
          ↓
        error → idle

idle ↔ paused
running → paused (需要取消流程)
* → terminated (仅董事会，不可逆)
```

**适配器类型**：
- `process`：本地进程执行（shell 命令、Python 脚本等）
- `http`：HTTP webhook 调用外部智能体

### 3.3 目标管理（Goals）

**核心实体**：`goals` 表

**功能**：
- 目标层级结构：`company | team | agent | task`
- 父子关系支持
- 目标状态：`planned | active | achieved | cancelled`
- 每个公司至少有一个根 `company` 级目标

### 3.4 任务管理（Issues）

**核心实体**：`issues` 表

**功能**：
- 任务层级（通过 `parent_id` 链接到父任务）
- 单责任人模型（`assignee_agent_id`）
- 原子性签出机制（Atomic Checkout）
- 状态管理：`backlog | todo | in_progress | in_review | done | blocked | cancelled`
- 优先级：`critical | high | medium | low`
- 评论系统（`issue_comments` 表）

**状态流转**：
```
backlog → todo | cancelled
todo → in_progress | blocked | cancelled
in_progress → in_review | blocked | done | cancelled
in_review → in_progress | done | cancelled
blocked → todo | in_progress | cancelled
```

**原子签出 API**：
```http
POST /issues/:issueId/checkout
{
  "agentId": "uuid",
  "expectedStatuses": ["todo", "backlog", "blocked", "in_review"]
}
```

### 3.5 项目管理（Projects）

**核心实体**：`projects` 表

**功能**：
- 项目与目标关联
- 状态管理：`backlog | planned | in_progress | completed | cancelled`
- 环境变量配置（`env` JSONB 字段）
- 项目负责人（`lead_agent_id`）

### 3.6 心跳与适配器（Heartbeat & Adapters）

**核心实体**：`heartbeat_runs` 表

**适配器接口**：
```typescript
interface AgentAdapter {
  invoke(agent: Agent, context: InvocationContext): Promise<InvokeResult>;
  status(run: HeartbeatRun): Promise<RunStatus>;
  cancel(run: HeartbeatRun): Promise<void>;
}
```

**Process 适配器配置**：
```json
{
  "command": "string",
  "args": ["string"],
  "cwd": "string",
  "env": {"KEY": "VALUE"},
  "timeoutSec": 900,
  "graceSec": 15
}
```

**HTTP 适配器配置**：
```json
{
  "url": "https://...",
  "method": "POST",
  "headers": {"Authorization": "Bearer ..."},
  "timeoutMs": 15000,
  "payloadTemplate": {"agentId": "{{agent.id}}", "runId": "{{run.id}}"}
}
```

**调度规则**：
- 最低 30 秒间隔
- 默认最大并发运行数：5
- 跳过条件：智能体暂停/终止、已有运行在进行中、预算硬限制已触发

### 3.7 成本与预算（Costs & Budgets）

**核心实体**：`cost_events` 表

**预算层级**：
1. 公司月度预算
2. 智能体月度预算
3. 项目预算（可选）

**执行规则**：
- 软告警阈值：默认 80%
- 硬限制：100% 时自动暂停智能体、阻止新签出/调用

**成本事件 API**：
```http
POST /companies/:companyId/cost-events
{
  "agentId": "uuid",
  "issueId": "uuid",
  "provider": "openai",
  "model": "gpt-5",
  "inputTokens": 1234,
  "outputTokens": 567,
  "costCents": 89,
  "occurredAt": "2026-02-17T20:25:00Z",
  "billingCode": "optional"
}
```

### 3.8 治理与审批（Approvals）

**核心实体**：`approvals` 表

**审批类型**：
- `hire_agent`：招聘智能体
- `approve_ceo_strategy`：批准 CEO 战略提案

**流程**：
1. 智能体或董事会创建审批请求
2. 董事会批准或拒绝
3. 批准后创建智能体（对于招聘）
4. 决策记录在 `activity_log` 中

### 3.9 活动日志（Activity Log）

**核心实体**：`activity_log` 表

**功能**：
- 记录所有变更操作
- 审计追踪
- 支持排查和合规

**字段**：
```typescript
{
  id: uuid
  company_id: uuid
  actor_type: enum  // agent | user | system
  actor_id: uuid/text
  action: string    // 操作类型
  entity_type: string
  entity_id: uuid/text
  details: jsonb    // 详细数据
  created_at: timestamptz
}
```

### 3.10 资产与附件（Assets & Attachments）

**核心实体**：`assets` 表

**存储提供商**：
- `local_disk`：本地磁盘存储（默认）
- `s3`：S3 兼容对象存储

**本地存储路径**：`~/.paperclip/instances/default/data/storage`

### 3.11 文档系统（Documents）

**核心实体**：`documents`, `document_revisions`

**功能**：
- 支持 Markdown 格式
- 版本历史追踪
- 与任务关联（`issue_documents`）

---

## 四、数据模型

### 4.1 核心表关系

```
companies
  ├── agents (company_id)
  │     ├── agent_api_keys (agent_id)
  │     └── reports_to → agents.id
  ├── goals (company_id)
  │     └── parent_id → goals.id
  ├── projects (company_id)
  │     ├── goal_id → goals.id
  │     └── lead_agent_id → agents.id
  ├── issues (company_id)
  │     ├── project_id → projects.id
  │     ├── goal_id → goals.id
  │     ├── parent_id → issues.id
  │     ├── assignee_agent_id → agents.id
  │     └── created_by_agent_id → agents.id
  ├── issue_comments (issue_id)
  ├── heartbeat_runs (agent_id)
  ├── cost_events (agent_id, issue_id, project_id, goal_id)
  ├── approvals (company_id)
  ├── activity_log (company_id)
  ├── assets (company_id)
  ├── documents (company_id)
  ├── company_secrets (company_id)
  └── company_secret_versions (secret_id)
```

### 4.2 关键索引

- `agents(company_id, status)`
- `agents(company_id, reports_to)`
- `issues(company_id, status)`
- `issues(company_id, assignee_agent_id, status)`
- `issues(company_id, parent_id)`
- `cost_events(company_id, occurred_at)`
- `heartbeat_runs(company_id, agent_id, started_at desc)`

---

## 五、API 架构

### 5.1 REST API 设计

**基础路径**：`/api`

**公司相关**：
```
GET  /companies
POST /companies
GET  /companies/:companyId
PATCH /companies/:companyId
POST /companies/:companyId/archive
```

**智能体相关**：
```
GET  /companies/:companyId/agents
POST /companies/:companyId/agents
GET  /agents/:agentId
PATCH /agents/:agentId
POST /agents/:agentId/pause
POST /agents/:agentId/resume
POST /agents/:agentId/terminate
POST /agents/:agentId/keys
POST /agents/:agentId/heartbeat/invoke
```

**任务相关**：
```
GET  /companies/:companyId/issues
POST /companies/:companyId/issues
GET  /issues/:issueId
PATCH /issues/:issueId
POST /issues/:issueId/checkout
POST /issues/:issueId/release
POST /issues/:issueId/admin/force-release
POST /issues/:issueId/comments
```

**成本与预算**：
```
POST /companies/:companyId/cost-events
GET  /companies/:companyId/costs/summary
GET  /companies/:companyId/costs/by-agent
PATCH /companies/:companyId/budgets
PATCH /agents/:agentId/budgets
```

### 5.2 错误码语义

| 状态码 | 含义 |
|--------|------|
| 400 | 验证错误 |
| 401 | 未认证 |
| 403 | 未授权 |
| 404 | 未找到 |
| 409 | 状态冲突（签出冲突、无效状态流转）|
| 422 | 语义规则违反 |
| 500 | 服务器错误 |

### 5.3 认证机制

**董事会认证**：
- 基于 Session 的认证
- 董事会对部署中的所有公司拥有完全读写权限

**智能体认证**：
- Bearer API Key 认证
- 每个 Key 映射到一个智能体和一个公司
- Key 只能访问其所属公司的数据

---

## 六、前端架构

### 6.1 技术栈

- **框架**：React + TypeScript
- **构建工具**：Vite
- **UI 组件**：Storybook（独立目录）
- **样式**：标准 CSS/SCSS

### 6.2 页面路由

```
/                              # 仪表板
/companies                     # 公司列表/创建
/companies/:id/org            # 组织结构图和智能体状态
/companies/:id/tasks          # 任务列表/看板
/companies/:id/agents/:agentId  # 智能体详情
/companies/:id/costs          # 成本和预算仪表板
/companies/:id/approvals      # 待处理/历史审批
/companies/:id/activity       # 审计/事件流
```

### 6.3 Storybook

```bash
pnpm storybook      # 启动 Storybook（端口 6006）
pnpm build-storybook # 构建静态输出
```

Storybook 配置位于 `ui/storybook/`，组件审查文件与应用程序路由分离。

---

## 七、CLI 工具

### 7.1 基本命令

```bash
# 开发服务器
pnpm dev              # 启动开发服务器（监视模式）
pnpm dev:once         # 运行一次（不监视）
pnpm dev:list         # 列出当前管理的开发进程
pnpm dev:stop         # 停止开发进程

# 数据库
pnpm db:generate      # 生成 Drizzle 迁移
pnpm db:migrate       # 运行迁移
pnpm db:backup        # 备份数据库

# 测试
pnpm test             # 运行 Vitest 测试
pnpm test:watch       # 监视模式运行测试
pnpm test:e2e         # 运行 E2E 测试（Playwright）

# 构建
pnpm build            # 构建所有包
pnpm typecheck        # 类型检查
```

### 7.2 Paperclip CLI

```bash
# 运行完整流程
pnpm paperclipai run

# 配置
pnpm paperclipai configure --section storage
pnpm paperclipai configure --section database
pnpm paperclipai configure --section secrets

# 上下文设置
pnpm paperclipai context set --api-base http://localhost:3100 --company-id <id>

# 工作区操作
pnpm paperclipai issue list --company-id <id>
pnpm paperclipai issue create --company-id <id> --title "标题"
pnpm paperclipai dashboard get

# Git worktree 支持
pnpm paperclipai worktree init              # 初始化工作区
pnpm paperclipai worktree:make <name>       # 创建新工作区
pnpm paperclipai worktree repair            # 修复工作区
pnpm paperclipai worktree reseed            # 重新种子工作区
```

---

## 八、开发指南

### 8.1 环境要求

- **Node.js**: 20+
- **包管理器**: pnpm 9+

### 8.2 快速开始

```bash
# 1. 安装依赖
pnpm install

# 2. 启动开发服务器
pnpm dev

# 3. 访问应用
# API: http://localhost:3100
# UI: http://localhost:3100（由 API 服务器在开发中间件模式下提供）
```

### 8.3 数据库配置

**选项 1：嵌入式 PostgreSQL（默认，零配置）**
```bash
# 不设置 DATABASE_URL，自动使用嵌入式 PostgreSQL
# 数据存储在：~/.paperclip/instances/default/db/
pnpm dev
```

**选项 2：本地 Docker PostgreSQL**
```bash
# 启动 PostgreSQL
docker compose up -d

# 配置连接
# .env:
# DATABASE_URL=postgres://paperclip:paperclip@localhost:5432/paperclip
```

**选项 3：托管 PostgreSQL（Supabase）**
```bash
# 直接连接（端口 5432）用于迁移
DATABASE_MIGRATION_URL=postgres://...

# 连接池（端口 6543）用于应用
DATABASE_URL=postgres://...
```

### 8.4 重置开发数据库

```bash
rm -rf ~/.paperclip/instances/default/db
pnpm dev
```

### 8.5 工作区（Worktree）开发

```bash
# 创建新工作区
pnpm paperclipai worktree:make paperclip-pr-432

# 进入工作区
cd /path/to/worktree

# 正常开发（自动加载工作区配置）
pnpm dev
```

### 8.6 Tailscale/私有网络开发

```bash
# LAN 模式
pnpm dev --bind lan

# Tailnet 模式
pnpm dev --bind tailnet

# 添加允许的主机名
pnpm paperclipai allowed-hostname my-host
```

---

## 九、打包与发布

### 9.1 构建流程

```bash
# 1. 预检工作区链接
pnpm run preflight:workspace-links

# 2. 构建所有包
pnpm -r build

# 3. 类型检查
pnpm -r typecheck

# 4. 运行测试
pnpm test:run
```

### 9.2 发布命令

```bash
# Canary 发布
pnpm release:canary

# 稳定版发布
pnpm release:stable

# GitHub Release
pnpm release:github

# 回滚最新版本
pnpm release:rollback
```

### 9.3 发布前置检查

在发布前必须运行：
```bash
pnpm -r typecheck
pnpm test:run
pnpm build
```

### 9.4 依赖锁定策略

- GitHub Actions 拥有 `pnpm-lock.yaml`
- **不要在 PR 中提交 `pnpm-lock.yaml`**
- PR CI 在清单更改时验证依赖解析
- 推送到 `master` 时重新生成锁定文件

---

## 十、部署指南

### 10.1 Docker 部署

**快速开始（无需本地 Node）**：
```bash
# 构建并运行
docker build -t paperclip-local .
docker run --name paperclip \
  -p 3100:3100 \
  -e HOST=0.0.0.0 \
  -e PAPERCLIP_HOME=/paperclip \
  -v "$(pwd)/data/docker-paperclip:/paperclip" \
  paperclip-local

# 或使用 Docker Compose
docker compose -f docker/docker-compose.quickstart.yml up --build
```

**数据持久化**：
- 挂载 `/paperclip` 到持久化卷
- 参考 `doc/DOCKER.md` 了解 API Key 配置

### 10.2 生产环境配置

**环境变量**：
```bash
# 数据库
DATABASE_URL=postgres://...
DATABASE_MIGRATION_URL=postgres://...  # 用于迁移的直接连接

# 部署模式
PAPERCLIP_MODE=authenticated  # 或 local_trusted

# 存储
PAPERCLIP_STORAGE_PROVIDER=s3  # 或 local_disk
PAPERCLIP_STORAGE_S3_BUCKET=...
PAPERCLIP_STORAGE_S3_REGION=...

# 密钥管理
PAPERCLIP_SECRETS_STRICT_MODE=true
PAPERCLIP_SECRETS_MASTER_KEY_FILE=/path/to/key

# 可选：禁用公司删除
PAPERCLIP_ENABLE_COMPANY_DELETION=false
```

### 10.3 Supabase 托管

1. 在 [database.new](https://database.new) 创建项目
2. 获取连接字符串（直接连接端口 5432，池化连接端口 6543）
3. 设置 `DATABASE_URL` 为池化连接
4. 设置 `DATABASE_MIGRATION_URL` 为直接连接
5. 如果使用池化连接，禁用 prepared statements

### 10.4 部署检查清单

- [ ] 设置 `DATABASE_URL` 和 `DATABASE_MIGRATION_URL`
- [ ] 配置存储提供商（local_disk 或 s3）
- [ ] 设置密钥管理（生产环境启用 strict mode）
- [ ] 配置自动备份
- [ ] 启用监控和日志
- [ ] 运行数据库迁移
- [ ] 验证健康检查端点：`GET /api/health`

---

## 十一、测试策略

### 11.1 测试类型

**单元测试**：
- 状态转换守卫（智能体、任务、审批）
- 预算执行规则
- 适配器调用/取消语义

**集成测试**：
- 原子签出冲突行为
- 审批到智能体创建流程
- 成本摄取和汇总正确性
- 暂停时运行中的任务（优雅取消后强制终止）

**端到端测试**：
- 董事会创建公司 → 雇佣 CEO → 批准战略 → CEO 接收工作
- 智能体报告成本 → 预算阈值达到 → 自动暂停发生
- 跨团队任务委托

### 11.2 回归测试最小集

发布候选版必须通过的测试：
1. 认证边界测试
2. 签出竞态测试
3. 硬预算停止测试
4. 智能体暂停/恢复测试
5. 仪表板汇总一致性测试

### 11.3 运行测试

```bash
# 单元测试
pnpm test

# 交互式监视模式
pnpm test:watch

# E2E 测试
pnpm test:e2e
pnpm test:e2e:headed  # 有头模式

# 发布冒烟测试
pnpm test:release-smoke
```

---

## 十二、安全考虑

### 12.1 数据安全

- 仅存储哈希后的智能体 API Key
- 日志中脱敏敏感信息（`adapter_config`、认证头、环境变量）
- 使用 CSRF 保护董事会 Session 端点
- 速率限制认证和密钥管理端点
- 每个实体获取/变更时严格检查公司边界

### 12.2 密钥管理

- 密钥值不内联存储在 `agents.adapter_config.env` 中
- 智能体环境条目应使用密钥引用
- 默认本地提供商：`local_encrypted`
- 默认密钥文件：`~/.paperclip/instances/default/secrets/master.key`

**严格模式**：
```bash
PAPERCLIP_SECRETS_STRICT_MODE=true
```

启用后，敏感环境键（如 `*_API_KEY`, `*_TOKEN`, `*_SECRET`）必须使用密钥引用而非内联明文。

### 12.3 密钥迁移

```bash
# 迁移现有内联环境密钥
pnpm secrets:migrate-inline-env         # 干运行
pnpm secrets:migrate-inline-env --apply # 应用迁移
```

---

## 十三、性能目标

| 指标 | 目标 |
|------|------|
| API p95 延迟 | < 250ms（标准 CRUD，1k 任务/公司）|
| 心跳调用确认 | < 2s（process 适配器）|
| 审批决策 | 无丢失（事务写入）|

---

## 十四、附录

### 14.1 关键文档索引

| 文档 | 内容 |
|------|------|
| `doc/GOAL.md` | 项目愿景与目标 |
| `doc/PRODUCT.md` | 产品定义与核心概念 |
| `doc/SPEC-implementation.md` | V1 实现规范（主要参考）|
| `doc/DEVELOPING.md` | 开发指南与 CLI 参考 |
| `doc/DATABASE.md` | 数据库配置与操作 |
| `doc/SPEC.md` | 长期产品规范 |
| `doc/TASKS.md` | 任务管理数据模型 |
| `doc/DOCKER.md` | Docker 部署详情 |
| `doc/DEPLOYMENT-MODES.md` | 部署模式定义 |
| `doc/CLI.md` | CLI 完整参考 |
| `doc/execution-semantics.md` | 执行语义详细文档 |

### 14.2 目录别名

```bash
# 主要目录
server/        → @paperclipai/server
ui/           → @paperclipai/ui
packages/db/  → @paperclipai/db
packages/shared/ → @paperclipai/shared
packages/adapters/claude/ → @paperclipai/claude-adapter
packages/adapters/codex/ → @paperclipai/codex-adapter
packages/adapters/cursor/ → @paperclipai/cursor-adapter
```

### 14.3 贡献指南

- 所有变更必须是公司范围的
- 保持契约同步（schema → shared → server → ui）
- 保留控制平面不变量（单责任人、原子签出、审批门控、预算硬停止）
- 变更后更新相关文档
- PR 必须遵循 `.github/PULL_REQUEST_TEMPLATE.md` 模板

### 14.4 相关脚本

```bash
# OpenClaw 集成测试
pnpm smoke:openclaw-join
pnpm smoke:openclaw-docker-ui

# 指标收集
pnpm metrics:paperclip-commits

# 评估测试
cd evals/promptfoo && npx promptfoo@0.103.3 eval

# 健康检查
curl http://localhost:3100/api/health
curl http://localhost:3100/api/companies
```

---

*文档结束*
