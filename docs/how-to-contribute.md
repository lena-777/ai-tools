# 贡献指南

感谢你为 ai-tools 做出贡献！

## 贡献流程

1. Fork 本仓库
2. 创建你的分支：`git checkout -b feat/your-skill-name`
3. 按照模板新建 skill 或 command
4. 更新相关索引表
5. 提交 PR

## 目录规范

- 每个 skill/command 是一个独立文件夹
- 文件夹名使用 `kebab-case`（如 `code-review`、`smart-commit`）
- 必须包含 `README.md`（给人看的说明）和 `skill.md`/`command.md`（实际 prompt）

## 文件说明

| 文件 | 用途 |
|------|------|
| `README.md` | 使用说明、场景、参数，面向使用者 |
| `skill.md` / `command.md` | 实际的 prompt 内容，面向 AI |
| `examples/` | （可选）使用示例 |

## 命名规范

- 名称应简洁明确，体现核心功能
- 使用英文 kebab-case 命名
- 避免过于通用的名称（如 `helper`、`tool`）

## 质量要求

- README 中必须包含：一句话简介、使用场景、使用方式
- prompt 内容应清晰、可复现
- 建议提供至少一个使用示例
