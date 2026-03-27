# do-tasks 接入指南

> 本文档是 do-tasks command 的详细接入指南。总体介绍请参考 [README.md](README.md)。

## 工作原理

```
用户在页面提交需求 → requirements 表写入 pending 记录
                         ↓
         Claude Code 执行 /do-tasks（或 /loop 5m /do-tasks）
                         ↓
         读取 pending 需求 → 分析需求 → 编码实现 → 标记 done
```

---

## 快速接入

### 前提条件

- 项目有可用的数据库（默认使用 SQLite，如果项目已接入其他数据库如 PostgreSQL、MySQL 则沿用现有数据库）
- 项目有后端服务，能提供 REST API
- 已安装 [Claude Code](https://docs.anthropic.com/en/docs/claude-code)

### 步骤

把下面这段话直接发给 Claude Code，它会自动帮你搭建整套功能：

> 请在我**现有项目**中接入「需求任务」功能。**不要创建新的服务或新项目**，直接在当前的后端服务中添加接口、在当前数据库中建表、在现有页面中添加 UI 组件。具体要求：
>
> **1. 数据库**
> 在现有数据库中创建 requirements 表（注意：不要使用 tasks 表名，避免与项目自身业务冲突。如果项目还没有数据库，请使用 SQLite），字段：id、title、description、status(pending/in_progress/done)、notes、created_at、updated_at。给 status 加索引。
>
> **2. REST API（5 个端点）**
> - GET /api/requirements?status=xxx — 获取需求列表，按 created_at 升序，支持 status 筛选
> - POST /api/requirements — 创建需求（title 必填）
> - PUT /api/requirements/:id — 更新需求全部字段
> - PATCH /api/requirements/:id/status — 快速更新状态，可选 notes 字段
> - DELETE /api/requirements/:id — 删除需求
>
> **3. 前端 UI**
> 在页面右下角添加浮动按钮「需求」，点击后在按钮上方弹出一个悬浮小面板（不是右侧整栏面板），功能包括：
> - 快速输入框（⌘+Enter 提交）
> - 任务列表（未完成任务支持编辑，点击可标记为 done；in_progress 状态由 AI 自动设置，无需手动切换）
> - 未完成数量红色角标
> - 已完成任务默认折叠，点击可展开
> - hover 显示删除按钮
> - 面板固定宽度约 360px，最大高度约 480px，内容超出时滚动
> - 点击面板外部区域自动关闭
>
> **重要：请先分析项目现有的技术栈、代码结构和风格，然后在此基础上添加功能，不要引入项目中没有的框架或创建独立的新服务。**

搭建完成后，将本仓库的 `commands/do-tasks/do-tasks.md` 复制到你项目的 `.claude/commands/do-tasks.md`，并把其中的 `$TASK_API_BASE` 替换为实际的 API 地址（如 `http://127.0.0.1:5001`）。

如果还需要并行模式，额外复制 `commands/do-tasks/do-tasks-parallel.md` 到 `.claude/commands/do-tasks-parallel.md`。

---

## 没有前端？用独立任务管理页

如果你的项目是纯后端、CLI 工具、或者你只是懒得在现有项目里加 UI，有两种方案：

### 方案一：独立 HTML 页面（推荐）

我们提供了一个 **零依赖的单文件 HTML 页面** — [`task-manager.html`](task-manager.html)，用浏览器打开就能管理任务。

**功能：**
- 配置 API 地址（页面顶部可修改，默认 `http://127.0.0.1:5001`）
- 需求列表：显示 pending / in_progress / done 状态
- 快速添加需求（标题 + 可选描述）
- 点击圆圈将需求标记为已完成（in_progress 状态由 Claude 自动设置，无需手动操作）
- 删除需求
- 按状态筛选
- 每 10 秒自动刷新

**使用方式：**

```bash
# 方式 1：直接从本仓库打开
open commands/do-tasks/task-manager.html    # macOS
xdg-open commands/do-tasks/task-manager.html  # Linux

# 方式 2：复制到你的项目中使用
cp commands/do-tasks/task-manager.html /path/to/your-project/task-manager.html
```

打开后在页面顶部修改 API 地址为你的后端地址即可。地址会保存在浏览器 localStorage 中，下次打开自动记住。

> **注意**：如果遇到跨域问题（CORS），需要在你的后端 API 中添加 CORS 头。大部分后端框架都有现成的 CORS 中间件。

### 方案二：curl 命令行（最精简）

不需要任何 UI，直接用命令行操作：

```bash
# 添加需求
curl -X POST http://127.0.0.1:5001/api/requirements \
  -H "Content-Type: application/json" \
  -d '{"title": "添加用户登录功能"}'

# 查看所有 pending 需求
curl http://127.0.0.1:5001/api/requirements?status=pending

# 查看所有需求
curl http://127.0.0.1:5001/api/requirements

# 删除需求
curl -X DELETE http://127.0.0.1:5001/api/requirements/1
```

两种方案可以混用——用 HTML 页面管理日常需求，偶尔用 curl 批量操作。

---

## 接入完成后的使用方式

```bash
# 手动执行一次：消费所有 pending 任务（串行）
/do-tasks

# 定时轮询：每 5 分钟检查一次新需求并自动实现
/loop 5m /do-tasks

# 并行执行：同时处理多条任务（默认 3 个 agent）
/do-tasks-parallel

# 指定并发度
/do-tasks-parallel 5

# 并行 + 定时轮询
/loop 5m /do-tasks-parallel 3
```

---

## 详细规格（供参考）

### 数据库 — requirements 表

```sql
CREATE TABLE IF NOT EXISTS requirements (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT NOT NULL,                          -- 需求标题（必填）
    description TEXT DEFAULT '',                  -- 需求详细描述
    status TEXT DEFAULT 'pending',                -- 状态：pending / in_progress / done
    notes TEXT DEFAULT '',                        -- AI 执行结果备注
    created_at TEXT DEFAULT (datetime('now')),     -- 创建时间
    updated_at TEXT DEFAULT (datetime('now'))      -- 更新时间
);
CREATE INDEX IF NOT EXISTS idx_requirements_status ON requirements(status);
```

### REST API 接口详情

**获取待办需求：**
```
GET /api/requirements?status=pending

Response:
{
  "requirements": [
    {
      "id": 1,
      "title": "添加用户登录功能",
      "description": "支持邮箱+密码登录，需要做表单校验",
      "status": "pending",
      "notes": "",
      "created_at": "2025-03-25 10:00:00",
      "updated_at": "2025-03-25 10:00:00"
    }
  ]
}
```

**更新需求状态：**
```
PATCH /api/requirements/<id>/status
Content-Type: application/json

Request:  {"status": "done", "notes": "已实现登录功能，修改了 auth.py 和 login.html"}
Response: {"ok": true, "message": "状态已更新"}
```

**注意：**
- GET 返回的需求列表需按 `created_at` 升序排序（最早的排前面）
- PATCH 的 `notes` 字段为可选参数
- `status` 只接受 `pending` / `in_progress` / `done` 三个值

### 前端 UI — 需求悬浮小面板

```
┌──────────────────────────────┐
│  页面主内容                     │
│                              │
│                              │
│                              │
│          ┌──────────────┐    │
│          │ 需求          ✕│    │  ← 悬浮小面板（~360px 宽）
│          │───────────────│    │
│          │ [输入需求...] +│    │  ← 快速添加
│          │───────────────│    │
│          │ ○ 添加用户登录  ✕│    │  ← pending 任务
│          │ ○ 优化首页性能  ✕│    │
│          │───────────────│    │
│          │ 已完成 (2) ▸   │    │  ← 可折叠
│          └──────────────┘    │
│                      [需求³] │  ← FAB 按钮（带角标）
└──────────────────────────────┘

面板从 FAB 按钮上方弹出，不占据侧边整栏
点击面板外部区域自动关闭
```

#### 参考 HTML

```html
<!-- FAB 按钮 -->
<div id="task-fab" onclick="toggleTaskPanel()">
  需求
  <span id="task-badge" class="task-badge" style="display:none">0</span>
</div>

<!-- 悬浮小面板（从 FAB 上方弹出） -->
<div id="task-panel" class="task-panel-hidden">
    <div class="task-panel-header">
        <span>需求</span>
        <button class="task-panel-close" onclick="toggleTaskPanel()">&times;</button>
    </div>
    <div class="task-panel-input">
        <input type="text" id="task-quick-input"
               placeholder="输入需求，⌘+回车提交..."
               onkeydown="if(event.key==='Enter'&&(event.metaKey||event.ctrlKey))quickAddTask()">
        <button onclick="quickAddTask()">+</button>
    </div>
    <div id="task-panel-list" class="task-panel-list"></div>
</div>
```

#### 参考 CSS 要点

```css
/* FAB 按钮 - 右下角固定 */
#task-fab {
  position: fixed; right: 24px; bottom: 24px;
  z-index: 1000; cursor: pointer;
}

/* 悬浮小面板 - 定位在 FAB 上方 */
#task-panel {
  position: fixed; right: 24px; bottom: 80px;   /* FAB 上方 */
  width: 360px; max-height: 480px;
  border-radius: 12px; box-shadow: 0 8px 32px rgba(0,0,0,0.15);
  overflow: hidden; display: flex; flex-direction: column;
  z-index: 1001;
}

/* 任务列表区域可滚动 */
.task-panel-list {
  flex: 1; overflow-y: auto;
}

/* 点击面板外部关闭 */
document.addEventListener('click', (e) => {
  if (!panel.contains(e.target) && !fab.contains(e.target)) closePanel();
});
```

---

## 目录结构

```
commands/do-tasks/
├── README.md              ← 设计说明 & 总览
├── how-to-use.md          ← 本文件（接入指南）
├── do-tasks.md            ← 串行模式 command 模板（复制到 .claude/commands/）
├── do-tasks-parallel.md   ← 并行模式 command 模板（复制到 .claude/commands/）
└── task-manager.html      ← 独立任务管理页（无前端时使用）
```
