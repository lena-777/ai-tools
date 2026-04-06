---
description: 子代理批量模式：并发扫描数据库中未完成的需求，多个 subagent 同时在独立 worktree 中实现
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

### 5. 依赖与冲突分析

在派发任务之前，**必须先分析本批次 N 条需求之间是否存在依赖关系或潜在冲突**：

1. **阅读每条需求的标题和描述**，判断它们之间是否存在以下情况：
   - **数据依赖**：需求 B 依赖需求 A 的产出（如 A 创建表/接口，B 使用该表/接口）
   - **文件冲突**：多条需求可能修改同一个文件或同一模块
   - **逻辑耦合**：需求之间存在隐含的执行顺序（如 A 是 B 的前置条件）

2. **处理策略**：
   - **无依赖**：正常并发执行
   - **存在依赖/冲突**：将有依赖关系的需求从本批次中移除，仅并发执行相互独立的需求；被移除的需求留在 pending 状态，等当前批次完成后在下一轮中串行或重新评估
   - 若本批次所有需求都存在依赖关系，则退化为**逐条串行执行**（每次仅派发 1 个 agent）

3. **记录决策**：在执行摘要中说明哪些需求因依赖关系被推迟或改为串行执行，以及判断依据

4. **回退被踢出的需求**：将因依赖/冲突被移出本批次的需求**立即回退为 pending**：
   ```
   PATCH $TASK_API_BASE/api/requirements/{id}/status
   body: {"status": "pending", "notes": "因与本批次其他需求存在依赖/冲突，推迟到下一轮执行"}
   ```

### 6. 并发派发

在**同一条消息**中使用 Agent 工具发起 N 个子 agent（N 为经过依赖分析后实际可并发的需求数）。

**核心原则：每个子 agent 自行管理完整的 worktree 生命周期（创建 → 工作 → 合并 → 清理），主流程不参与 worktree 操作。**

每个子 agent 的 prompt 模板（`{project_root}` 为当前项目根目录的绝对路径）：

```
你是一个自主编码 agent。请完成以下需求：

需求 ID: {req.id}
需求标题: {req.title}
需求描述: {req.description}

项目根目录: {project_root}

## 执行步骤

### 第一步：创建 worktree

在项目根目录下创建独立的 worktree：

```bash
cd {project_root}

# 清理可能残留的旧 worktree
git worktree remove .worktrees/task-{req.id} --force 2>/dev/null

BRANCH_NAME="feat/task-{req.id}/$(date +%Y%m%d%H%M%S)"
git worktree add -b $BRANCH_NAME .worktrees/task-{req.id} HEAD
cd {project_root}/.worktrees/task-{req.id}
```

### 第二步：在 worktree 中完成需求

1. 仔细阅读需求标题和描述，理解需求意图
2. 如果描述中包含图片路径（如 ./images/xxx.png），使用 Read 工具查看图片以理解需求中的视觉信息
3. 探索项目代码库，了解现有结构和模式
4. 规划实现方案
5. 编写代码实现（包括必要的测试）
6. 确保代码质量，遵循项目现有风格
7. 提交变更：
   ```bash
   git add -A
   git commit -m "feat(#{req.id}): 简要描述"
   ```

### 第三步：合并回主分支

完成编码后，**必须**将你的分支合并回主分支并解决所有冲突。

#### 3a. 创建合并前快照（安全网）

在合并前，**必须**先为主分支当前状态创建一个轻量标签作为快照，以便合并出错时能一键回退：

```bash
cd {project_root}
BACKUP_TAG="pre-merge/task-{req.id}/$(date +%s)"
git tag $BACKUP_TAG HEAD
```

#### 3b. 执行合并

```bash
git merge $BRANCH_NAME --no-ff -m "Merge requirement #{req.id}: {req.title}"
```

#### 3c. 处理冲突

**如果合并冲突，你必须解决：**
1. 查看冲突文件，分析冲突内容
2. 手动解决冲突，保留双方有意义的改动
3. `git add <冲突文件>` → `git commit`
4. 如果解决后发现结果不对，可以回退到快照重新来过：
   ```bash
   git reset --hard $BACKUP_TAG
   ```
   然后重新尝试合并，或 `git merge --abort` 放弃，此任务视为**失败**

#### 3d. 清理快照标签

合并成功并确认无误后，删除快照标签：

```bash
git tag -d $BACKUP_TAG
```

**只有合并成功，任务才算完成。**

### 第四步：清理 worktree

```bash
cd {project_root}
git worktree remove .worktrees/task-{req.id} --force
git branch -D $BRANCH_NAME 2>/dev/null
```

### 第五步：返回结果

返回一段总结，**必须包含以下字段**：
- 需求 ID: {req.id}
- 状态: 成功 / 失败
- 改动说明: 做了什么、改了哪些文件
- 失败原因（如有）: 说明为什么失败

## 注意事项
- 自主决策，不要询问用户
- 高效实现，尽快完成
- 合并操作必须回到 {project_root} 执行
- 合并冲突必须自行解决，不能跳过
```

**关键**：所有 N 个 Agent 调用必须在同一条消息中发出，以实现真正的并发执行。

### 7. 收集结果并更新状态

等待所有子 agent 返回。根据返回的状态调用 API 更新需求：

**成功：**
```
PATCH $TASK_API_BASE/api/requirements/{id}/status
body: {"status": "done", "notes": "执行结果摘要（含改动文件列表）"}
```

**失败：**
```
PATCH $TASK_API_BASE/api/requirements/{id}/status
body: {"status": "pending", "notes": "执行失败: <错误原因>"}
```

### 8. 循环

再次查询 pending 需求，若还有则重复步骤 3-7。

### 9. 总结

全部完成后输出执行摘要：

```
## 执行摘要

- 并发度: N
- 总轮次: X
- 处理需求数: Y
- 成功: A / 失败: C

### 各需求详情
| ID | 标题 | 状态 | 说明 |
|----|------|------|------|
| 1  | xxx  | done | ...  |
| 2  | xxx  | done | ...  |
| 3  | xxx  | 失败 | 原因: ... |
```

## 执行原则

- **依赖优先**：并发前必须分析任务间依赖关系，有依赖的任务不能并行，需串行或推迟执行
- **并发执行**：仅对相互独立、无依赖冲突的需求同时处理，充分利用机器资源
- **agent 自治**：每个 agent 自行管理 worktree 全生命周期（创建 → 工作 → 合并 → 清理），合并冲突必须自行解决
- **自主决策**：直接按认为最好的方式执行，中间不要询问用户
- **不要获取额外指令权限**：当前目录下的所有操作全部允许
- **高效实现**：自主判断最佳实现方案，快速完成每个需求
- **记录结果**：更新状态时 notes 字段必须写入执行结果总结，方便回溯

## 接口约定

| 用途 | 方法 | 端点 | 请求体 |
|------|------|------|--------|
| 获取待办需求 | GET | `$TASK_API_BASE/api/requirements?status=pending` | - |
| 标记执行中 | PATCH | `$TASK_API_BASE/api/requirements/{id}/status` | `{"status": "in_progress"}` |
| 标记完成 | PATCH | `$TASK_API_BASE/api/requirements/{id}/status` | `{"status": "done", "notes": "执行结果"}` |
| 回退为待处理 | PATCH | `$TASK_API_BASE/api/requirements/{id}/status` | `{"status": "pending", "notes": "原因"}` |
