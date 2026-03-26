---
description: 并发扫描数据库中未完成的需求，多个 agent 同时在独立 worktree 中实现
---

并发处理当前项目中数据库里的待办需求（requirements 表），**同时派发多条需求到独立 worktree 中并行执行**。

可选参数 `$ARGUMENTS` 指定并发度（数字，默认 3，上限 5）。

## 执行流程

### 1. 解析并发度

从 `$ARGUMENTS` 中读取数字 N：
- 若为空或非数字，N = 3
- 若超过 5，N = 5
- 若小于 1，N = 1

### 2. 确认环境

调用 `GET $TASK_API_BASE/api/requirements?status=pending` 确认服务可访问。若失败则终止并提示用户检查 `$TASK_API_BASE` 环境变量。

### 3. 批量取需求

从返回的 pending 需求列表中，按 `created_at` 升序取前 N 条需求。若不足 N 条则取全部。若无 pending 需求，输出"无待处理需求"并结束。

### 4. 批量标记 in_progress

对取到的每条需求**立即**调用 API 标记为 `in_progress`，防止其他实例重复拾取：

```
PATCH $TASK_API_BASE/api/requirements/{id}/status
body: {"status": "in_progress"}
```

所有 PATCH 调用尽量在同一条消息中并发发出。

### 5. 并发派发

在**同一条消息**中使用 Agent 工具发起 N 个子 agent，每个 agent 必须设置 `isolation: "worktree"`。

每个子 agent 的 prompt 模板：

```
你是一个自主编码 agent。请完成以下需求：

需求 ID: {req.id}
需求标题: {req.title}
需求描述: {req.description}

执行要求：
1. 仔细阅读需求标题和描述，理解需求意图
2. 探索项目代码库，了解现有结构和模式
3. 规划实现方案
4. 编写代码实现（包括必要的测试）
5. 确保代码质量，遵循项目现有风格
6. 完成后将所有变更 commit 到当前分支，commit message 需清晰描述改动内容
7. 最后返回一段简要总结：做了什么、改了哪些文件、关键决策说明

注意：
- 你在一个独立的 git worktree 中工作，可以自由修改文件
- 不需要调用任何 API 更新需求状态，主流程会统一处理
- 自主决策，不要询问用户
- 高效实现，尽快完成
```

**关键**：所有 N 个 Agent 调用必须在同一条消息中发出，以实现真正的并发执行。

### 6. 收集结果

等待所有子 agent 返回。记录每个 agent 的：
- 执行结果摘要
- worktree 分支名（从 Agent 返回的结果中获取）
- 成功/失败状态

### 7. 合并 worktree 变更

对每个**成功完成**的 agent 的 worktree 分支，在主分支上执行合并：

```bash
git merge <worktree-branch> --no-ff -m "Merge requirement #<id>: <title>"
```

#### 冲突处理

- 合并时若出现冲突，先尝试查看冲突文件并自行解决（选择合理的合并结果）
- 若冲突过于复杂无法自动解决，执行 `git merge --abort`，并记录该任务需要人工介入
- 第一个 agent 的分支合并不会冲突；后续分支的冲突概率随任务相关度上升
- 建议用户按任务粒度合理拆分，减少并发冲突

### 8. 更新需求状态

对每条需求调用 API 更新状态：

**成功完成且合并成功的需求：**
```
PATCH $TASK_API_BASE/api/requirements/{id}/status
body: {"status": "done", "notes": "执行结果摘要（含改动文件列表）"}
```

**合并冲突需人工介入的需求：**
```
PATCH $TASK_API_BASE/api/requirements/{id}/status
body: {"status": "pending", "notes": "并发执行完成但合并冲突，需人工介入。冲突文件: ..."}
```

**执行失败的需求：**
```
PATCH $TASK_API_BASE/api/requirements/{id}/status
body: {"status": "pending", "notes": "执行失败: <错误原因>"}
```

### 9. 循环

再次查询 pending 需求，若还有则重复步骤 3-8。

### 10. 总结

全部完成后输出执行摘要：

```
## 执行摘要

- 并发度: N
- 总轮次: X
- 处理需求数: Y
- 成功: A / 冲突需介入: B / 失败: C

### 各需求详情
| ID | 标题 | 状态 | 说明 |
|----|------|------|------|
| 1  | xxx  | done | ...  |
| 2  | xxx  | done | ...  |
| 3  | xxx  | 需介入 | 冲突文件: ... |
```

## 执行原则

- **并发执行**：同时处理多条需求，充分利用机器资源
- **worktree 隔离**：每个 agent 在独立 worktree 中工作，互不干扰
- **自主决策**：直接按认为最好的方式执行，中间不要询问用户
- **不要获取额外指令权限**：当前目录下的所有操作全部允许
- **高效实现**：自主判断最佳实现方案，快速完成每个需求
- **记录结果**：更新状态时 notes 字段必须写入执行结果总结，方便回溯
- **安全合并**：合并冲突时优先保守处理，无把握则回退

## 接口约定

| 用途 | 方法 | 端点 | 请求体 |
|------|------|------|--------|
| 获取待办需求 | GET | `$TASK_API_BASE/api/requirements?status=pending` | - |
| 标记执行中 | PATCH | `$TASK_API_BASE/api/requirements/{id}/status` | `{"status": "in_progress"}` |
| 标记完成 | PATCH | `$TASK_API_BASE/api/requirements/{id}/status` | `{"status": "done", "notes": "执行结果"}` |
| 回退为待处理 | PATCH | `$TASK_API_BASE/api/requirements/{id}/status` | `{"status": "pending", "notes": "原因"}` |
