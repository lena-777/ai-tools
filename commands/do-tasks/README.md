# do-tasks

> 让 Claude Code 自动消费数据库中的需求任务，逐条分析并实现。你只需写下要做什么，Claude 会自动一条条执行完。

## 设计目的

在日常使用 Claude Code 时，我们经常遇到这些痛点：

- **单次对话难以管理多个任务** — 需求多了容易遗漏
- **人工逐条下发指令效率低** — 每完成一个还得手动安排下一个
- **长时间任务无法持续运行** — Claude 空闲后不会主动干活
- **缺乏可视化的任务进度追踪** — 不知道做到哪了

do-tasks 通过 **任务队列 + REST API + Command 调度** 的组合，一站式解决了这些问题。

核心理念：**你只需写下要做什么，Claude 会自动一条条执行完。**

## 使用场景

### 场景一：已有前端项目，想加入需求管理

你有一个正在开发的 Web 项目（比如个人博客、管理后台），希望在页面上随手提需求，让 Claude 自动实现。

→ 完整接入：数据库 + API + 前端 UI + do-tasks command

### 场景二：新建系统，从零搭建

你正在启动一个新项目，还没有任何代码。让 Claude 先帮你搭建整套任务管理基础设施，然后通过任务队列驱动后续所有开发。

→ 完整接入：Claude 会根据你选的技术栈从零搭建

### 场景三：纯后端项目，没有前端页面

你的项目是 CLI 工具、后端服务等，没有前端页面。你只需要任务队列能力，通过 API 或直接操作数据库来添加任务。

→ 精简接入：只需数据库 + API，跳过前端 UI，通过 curl / 脚本 / 数据库客户端直接写入任务

```bash
# 通过 curl 添加需求
curl -X POST http://127.0.0.1:5001/api/requirements \
  -H "Content-Type: application/json" \
  -d '{"title": "添加用户登录功能"}'
```

> 💡 不想用 curl？我们还提供了一个 **[独立任务管理页](task-manager.html)**（零依赖单文件 HTML），浏览器打开即可管理任务。详见 [how-to-use.md](how-to-use.md#没有前端用独立任务管理页)。

### 场景四：用 do-tasks-parallel 并发执行

当任务之间互不冲突且机器资源充足时，可以使用并行模式同时执行多条任务，大幅提升效率。

→ 每个 agent 在独立 git worktree 中工作，完成后自动合并回主分支

```bash
# 默认 3 个 agent 并发
/do-tasks-parallel

# 指定 5 个 agent 并发
/do-tasks-parallel 5

# 持久化轮询
/loop 5m /do-tasks-parallel 3
```

## 串行 vs 并行对比

| 特性 | do-tasks（串行） | do-tasks-parallel（并行） |
|------|------------------|---------------------------|
| 执行方式 | 串行，一次一条 | 并发，同时多条 |
| 工作目录 | 当前目录 | 每个任务独立 worktree |
| 适用场景 | 任务间有依赖 | 任务间互不冲突 |
| 资源占用 | 低 | 较高（多个 agent 同时运行） |
| 合并策略 | 无需合并 | 自动合并 + 冲突处理 |

## 系统架构

```
┌──────────────────────────────────────────────────────────────┐
│                        用户浏览器                              │
│                    (Web UI — 任务面板)                         │
│         ┌───────────────────────────────────┐                │
│         │  📋 任务列表    [输入框] [提交]     │                │
│         │  🟢 完成: 重构登录页面              │                │
│         │  🟠 进行中: 添加暗色模式            │                │
│         │  ⚪ 待处理: 优化首页性能            │                │
│         └───────────────────────────────────┘                │
└──────────────────┬───────────────────────────────────────────┘
                   │ HTTP API                    ▲
                   ▼                             │ curl / 脚本（无前端时）
┌──────────────────────────────────────────────────────────────┐
│                     后端服务 (REST API)                        │
│                                                              │
│  GET    /api/requirements?status=xxx  → 获取需求列表                 │
│  POST   /api/requirements             → 创建新需求                   │
│  PATCH  /api/requirements/:id/status  → 更新状态 & 备注              │
└──────────────────┬───────────────────────────────────────────┘
                   │ 数据库
                   ▼
┌──────────────────────────────────────────────────────────────┐
│                       requirements 表                            │
│                                                              │
│  id | title        | status      | notes | created_at        │
│  1  | 重构登录页面  | done        | ...   | 2026-03-23 10:00  │
│  2  | 添加暗色模式  | in_progress | ...   | 2026-03-23 10:05  │
│  3  | 优化首页性能  | pending     |       | 2026-03-23 10:10  │
└──────────────────────────────────────────────────────────────┘
                   ▲
                   │ API 调用
┌──────────────────┴───────────────────────────────────────────┐
│              Claude Code (/do-tasks Command)                  │
│                                                              │
│  串行: 取任务 → 分析需求 → 编码实现 → 标记完成 → 取下一个      │
│  并行: 批量取任务 → N 个 agent 同时执行 → 合并分支 → 取下一批  │
│                                                              │
│  由 /loop 5m /do-tasks 或 /loop 5m /do-tasks-parallel 驱动   │
└──────────────────────────────────────────────────────────────┘
```

## 核心用法

### 串行模式 — `/do-tasks`

#### 一次性执行

```bash
/do-tasks
```

Claude 从任务队列中逐条取出 pending 任务，分析需求、编码实现、标记完成，直到全部处理完毕。

#### 持久化轮询

```bash
/loop 5m /do-tasks
```

每 5 分钟自动检查一次新需求并执行。你可以把它想象成一个**永远在线的 Claude 助手**：你往任务队列里扔需求，它定时自动拾取并实现。

### 并行模式 — `/do-tasks-parallel`

#### 默认并发度（3）

```bash
/do-tasks-parallel
```

同时启动 3 个 agent，各自在独立 worktree 中执行一条任务。

#### 指定并发度

```bash
/do-tasks-parallel 5
```

同时启动 5 个 agent（上限）。

#### 持久化轮询

```bash
/loop 5m /do-tasks-parallel 3
```

每 5 分钟自动检查新需求，每次最多 3 个 agent 并发执行。

#### 并行执行流程

```
GET /api/requirements?status=pending — 获取待处理需求
         ↓
按 created_at 升序取前 N 条（N = 并发度）
         ↓
批量 PATCH 标记为 in_progress（防止重复拾取）
         ↓
同时发起 N 个 Agent（各自在独立 worktree 中）
   ├── Agent 1: worktree-a → 需求 #1
   ├── Agent 2: worktree-b → 需求 #2
   └── Agent 3: worktree-c → 需求 #3
         ↓
等待所有 agent 完成
         ↓
依次 git merge 各 worktree 分支到主分支
   ├── 成功 → 标记 done
   ├── 冲突 → 尝试解决 / 标记需介入
   └── 失败 → 回退为 pending
         ↓
循环，直到没有 pending 需求
         ↓
输出执行摘要
```

#### 冲突处理策略

并发执行时，多个 agent 可能修改相同文件，合并时可能产生冲突：

1. **预防**：合理拆分任务粒度，让每个任务修改不同的文件/模块
2. **自动解决**：合并时若冲突较简单（如不同区域的修改），尝试自动解决
3. **保守回退**：无法自动解决的冲突，执行 `git merge --abort` 并将任务回退为 pending
4. **顺序合并**：第一个 agent 的分支一定不会冲突，冲突只可能发生在后续分支

#### 并行模式注意事项

- **资源消耗**：并发度越高，CPU / 内存 / API 调用消耗越大，请根据机器配置合理设置
- **任务粒度**：建议每个任务修改独立的模块/文件，减少合并冲突概率
- **git 状态**：执行前确保工作目录干净（无未提交的修改），避免 worktree 创建失败
- **项目需为 git 仓库**：worktree 功能依赖 git

### 串行执行流程

```
GET /api/requirements?status=pending — 获取待处理需求
         ↓
按 created_at 升序取第一条
         ↓
阅读需求 → 规划方案 → 编码实现
         ↓
PATCH /api/requirements/:id/status — 标记为 done，附上完成说明
         ↓
回到第一步，直到没有 pending 需求
         ↓
输出执行摘要
```

## 任务状态流转

```
pending  ──→  in_progress  ──→  done
(待处理)       (执行中)         (已完成)
    ↑
    └──── 冲突/失败时回退（仅并行模式）
```

- **pending** — 新创建的任务，等待 Claude 拾取
- **in_progress** — Claude 正在执行中
- **done** — 已完成，附带执行说明（notes 字段）

## 如何接入

👉 详见 **[how-to-use.md](how-to-use.md)** — 包含前提条件、分步骤接入指南、详细 API 规格、数据库建表语句和前端 UI 参考。

## 目录结构

```
commands/do-tasks/
├── README.md              ← 本文件（设计说明 & 总览）
├── how-to-use.md          ← 详细接入指南（前提条件、步骤、API 规格）
├── do-tasks.md            ← 串行模式 command 模板（复制到 .claude/commands/）
├── do-tasks-parallel.md   ← 并行模式 command 模板（复制到 .claude/commands/）
└── task-manager.html      ← 独立任务管理页（无前端时使用）
```

## API 参考

| 方法 | 端点 | 说明 |
|------|------|------|
| GET | `/api/requirements?status=xxx` | 获取需求列表，支持 status 筛选，按 created_at 升序 |
| POST | `/api/requirements` | 创建新需求（title 必填） |
| PUT | `/api/requirements/:id` | 更新需求全部字段 |
| PATCH | `/api/requirements/:id/status` | 快速更新状态，可选 notes 字段 |
| DELETE | `/api/requirements/:id` | 删除需求 |
