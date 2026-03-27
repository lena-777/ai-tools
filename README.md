# ai-tools

AI 通用工具集 — 存放通用的 AI Skills 和 Commands。

## 目录结构

```
ai-tools/
├── skills/          # AI Skills（可复用的能力模块）
├── commands/        # AI Commands（可执行的指令）
└── docs/            # 通用文档
```

## Skills 索引

| 名称 | 简介 | 适用场景 |
|------|------|----------|
| | | |

## Commands 索引

| 名称 | 简介 | 适用场景 |
|------|------|----------|
| [do-tasks](commands/do-tasks/) | 自动消费数据库需求任务 | 页面提需求 → AI 自动实现 |
| [do-tasks-parallel](commands/do-tasks-parallel/) | 并发消费数据库需求任务 | 多 agent 独立 worktree 并行实现 |
| [git-sync](commands/git-sync/) | 一键 git 双向同步 | 提交 + rebase 拉取 + 自动解决冲突 + 推送 |

## 快速开始

1. 浏览上方索引表，找到你需要的 skill 或 command
2. 进入对应文件夹，查看 `README.md` 了解使用说明
3. 实际的 prompt 内容在 `skill.md` 或 `command.md` 中
