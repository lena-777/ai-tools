# Commands

存放通用的 AI Commands（可执行的指令）。

## 什么是 Command

Command 是一个可直接执行的 AI 指令，通常以 `/command-name` 的形式调用，完成一个具体的任务。

## 索引

| 名称 | 简介 | 适用场景 |
|------|------|----------|
| [do-tasks](do-tasks/) | 自动消费数据库需求任务（支持串行/并行） | 页面提需求 → AI 自动实现 |
| [git-sync](git-sync/) | 一键 git 双向同步 | 提交 + rebase 拉取 + 自动解决冲突 + 推送 |

## 新建 Command

1. 新建一个文件夹，以 command 名称命名（kebab-case）
2. 填写 `README.md` 和 `command.md`
3. 更新本文件和根目录 `README.md` 的索引表
