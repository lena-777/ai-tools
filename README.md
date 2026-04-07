# ai-tools

AI 通用工具集 — 存放通用的 AI Skills 和 Commands。

## 目录结构

```
ai-tools/
├── commands/        # AI Commands（可执行的指令）
│   ├── do-tasks/    # 自动消费需求任务（3 种模式）
│   ├── git-sync/    # 一键 git 双向同步
│   └── upsert-tools/# 一键安装/更新 tool
├── skills/          # AI Skills（可复用的能力模块）
│   └── frontend-design/
└── docs/            # 通用文档（命名规范等）
```

## Commands 索引

### do-tasks — 自动消费数据库需求任务

从任务队列中拉取需求，自动分析并实现。提供三种执行模式：

| 命令 | 模式 | 说明 | 适用场景 |
|------|------|------|----------|
| [`/do-tasks`](commands/do-tasks/) | 串行 | 逐条取出任务，一次执行一条 | 任务间有依赖 |
| [`/do-tasks-subagent [N]`](commands/do-tasks/) | 批量并行 | N 个 agent 在独立 worktree 中同时执行，全部完成后合并 | 任务间完全独立 |
| [`/do-tasks-team [N]`](commands/do-tasks/) | 团队协作 | 按角色智能分组，角色内串行、角色间并行 | 混合场景（推荐） |

> 三种模式均可搭配 `/loop` 实现持久化轮询，如 `/loop 5m /do-tasks-team 5`

### 其他 Commands

| 名称 | 简介 | 适用场景 |
|------|------|----------|
| [git-sync](commands/git-sync/) | 一键 git 双向同步 | 提交 + rebase 拉取 + 自动解决冲突 + 推送 |
| [upsert-tools](commands/upsert-tools/) | 一键安装/更新 tool | 给 URL 安装、给名字更新、无参数批量更新 |

## Skills 索引

| 名称 | 简介 | 适用场景 |
|------|------|----------|
| [frontend-design](skills/frontend-design/) | 极简黑白苹果玻璃风设计规范 | 前端页面生成、UI 样式修改 |

## 快速开始

1. 浏览上方索引表，找到你需要的 skill 或 command
2. 进入对应文件夹，查看 `README.md` 了解详细使用说明
3. 用 `/upsert-tools` 一键安装到你的项目中

```bash
# 示例：安装 do-tasks 到你的项目
/upsert-tools github:lena-777/ai-tools/commands/do-tasks
```
