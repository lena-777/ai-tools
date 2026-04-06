# frontend-design

> 前端设计规范：黑白主题、苹果玻璃风、无 AI 味、排版考究。

## 设计目的

AI 生成的前端页面普遍存在"AI 味"：多色渐变、彩虹配色、过度圆角、满屏 emoji、花哨动画。这些页面看起来华丽但缺乏设计感。

frontend-design 定义了一套极简设计系统，让 AI 生成的页面风格统一、克制、优雅：

- **黑白灰为主** — 最多 1 种强调色
- **Apple 玻璃风** — 毛玻璃效果、轻阴影、适度圆角
- **排版考究** — 清晰的字号层级、舒适的行高、充足的留白
- **拒绝花哨** — 无多色渐变、无弹跳动画、无装饰性 emoji

## 使用场景

### 场景一：作为 Skill 自动生效

安装到 `.claude/skills/` 后，涉及前端页面生成或样式修改时，Claude 会自动参考这套设计规范。

### 场景二：在其他项目中使用

```bash
/upsert-tools github:lena-777/ai-tools/skills/frontend-design
```

## 规范要点

| 类别 | 规则 |
|------|------|
| 色彩 | 黑白灰 + 最多 1 种强调色，禁止多色渐变 |
| 玻璃效果 | backdrop-filter blur(20px)，透明度 0.6~0.8 |
| 圆角 | 12px ~ 20px，不超过 24px |
| 排版 | 系统字体，字号 13~48px 分 7 级，行高 1.6 |
| 间距 | 8px 网格，大量留白 |
| 动画 | 150~300ms 微交互，禁止弹跳旋转 |
| Emoji | 最多 1-2 个功能性，禁止纯装饰 |

## 如何安装

将 `skills/frontend-design/frontend-design.md` 复制到项目的 `.claude/skills/frontend-design/` 下：

```bash
mkdir -p your-project/.claude/skills/frontend-design
cp skills/frontend-design/frontend-design.md your-project/.claude/skills/frontend-design/
```

## 目录结构

```
skills/frontend-design/
├── README.md              ← 本文件（设计说明 & 总览）
└── frontend-design.md     ← skill 模板（复制到 .claude/skills/frontend-design/）
```
