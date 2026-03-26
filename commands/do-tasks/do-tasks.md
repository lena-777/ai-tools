---
description: 扫描数据库中未完成的需求任务，逐个分析并实现
---

串行处理当前项目中数据库里的待办需求（requirements 表），**每次只取最早的一条需求**，完成后再取下一条。

## 执行流程

1. **确认环境**：先调用 `GET $TASK_API_BASE/api/requirements?status=pending` 确认服务可访问
2. **取最早一条**：从返回的 pending 需求列表中，按 `created_at` 升序取第一条
3. **执行任务**：
   - 阅读需求的 `title` 和 `description`，理解需求意图
   - 规划实现方案
   - 编写代码实现
   - 完成后调用 API 更新状态：
     ```
     PATCH $TASK_API_BASE/api/requirements/{id}/status
     body: {"status": "done", "notes": "简要说明做了什么、改了哪些文件"}
     ```
4. **循环**：再次查询 pending 需求，取第一条，重复步骤 3，直到没有 pending 需求
5. **总结**：全部完成后输出执行摘要（共处理 N 条，每条做了什么）

## 执行原则

- **串行执行**：一次只处理一条需求，完成并更新状态后再取下一条
- **自主决策**：直接按认为最好的方式执行，中间不要询问用户
- **不要获取额外指令权限**：当前目录下的所有操作全部允许
- **高效实现**：自主判断最佳实现方案，快速完成每个任务
- **记录结果**：更新状态时 notes 字段必须写入执行结果总结，方便回溯

## 接口约定

| 用途 | 方法 | 端点 | 请求体 |
|------|------|------|--------|
| 获取待办需求 | GET | `$TASK_API_BASE/api/requirements?status=pending` | - |
| 更新需求状态 | PATCH | `$TASK_API_BASE/api/requirements/{id}/status` | `{"status": "done", "notes": "执行结果"}` |
