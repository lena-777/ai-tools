# ai-tools

AI 通用工具集 — 存放通用的 AI Skills 和 Commands。

## 目录结构

```
ai-tools/
├── skills/          # AI Skills（可复用的能力模块）
├── commands/        # AI Commands（可执行的指令）
└── docs/            # 通用文档（命名规范等）
```

## Skills 索引

| 名称 | 简介 | 适用场景 |
|------|------|----------|
| [frontend-design](skills/frontend-design/) | 极简黑白苹果玻璃风设计规范 | 前端页面生成、UI 样式修改 |

## Commands 索引

| 名称 | 简介 | 适用场景 |
|------|------|----------|
| [do-tasks](commands/do-tasks/) | 自动消费数据库需求任务（含串行/并行/团队三种模式） | 页面提需求 → AI 自动实现 |
| [git-sync](commands/git-sync/) | 一键 git 双向同步 | 提交 + rebase 拉取 + 自动解决冲突 + 推送 |
| [upsert-tools](commands/upsert-tools/) | 一键安装/更新 tool | 给 URL 安装、给名字更新、无参数批量更新 |

## 快速开始

1. 浏览上方索引表，找到你需要的 skill 或 command
2. 进入对应文件夹，查看 `README.md` 了解使用说明
3. 用 `/upsert-tools` 一键安装到你的项目中
