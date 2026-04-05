# do-tasks

> 让 Claude Code 自动消费数据库中的需求任务，逐条分析并实现。你只需写下要做什么，Claude 会自动一条条执行完。

## 设计目的

在日常使用 Claude Code 时，我们经常遇到这些痛点：

- **单次对话难以管理多个任务** — 需求多了容易遗漏
- **人工逐条下发指令效率低** — 每完成一个还得手动安排下一个
- **长时间任务无法持续运行** — Claude 空闲后不会主动干活
- **缺乏可视化的任务进度追踪** — 不知道做到哪了

do-tasks 通过 **任务队列 + REST API + Command 调度** 的组合，一站式解决了这些问题。

核心理念：**你只需写下要做什么，Claude 会自动一条条执行完。**

## 使用场景

### 场景一：已有前端项目，想加入需求管理

你有一个正在开发的 Web 项目（比如个人博客、管理后台），希望在页面上随手提需求，让 Claude 自动实现。

→ 完整接入：数据库 + API + 前端 UI + do-tasks command

### 场景二：新建系统，从零搭建

你正在启动一个新项目，还没有任何代码。让 Claude 先帮你搭建整套任务管理基础设施，然后通过任务队列驱动后续所有开发。

→ 完整接入：Claude 会根据你选的技术栈从零搭建

### 场景三：纯后端项目，没有前端页面

你的项目是 CLI 工具、后端服务等，没有前端页面。你只需要任务队列能力，通过 API 或直接操作数据库来添加任务。

→ 精简接入：只需数据库 + API，跳过前端 UI，通过 curl / 脚本 / 数据库客户端直接写入任务

```bash
# 通过 curl 添加需求
curl -X POST http://127.0.0.1:5001/api/requirements \
  -H "Content-Type: application/json" \
  -d '{"title": "添加用户登录功能"}'
```

> 💡 不想用 curl？我们还提供了一个 **[独立任务管理页](task-manager.html)**（零依赖单文件 HTML），浏览器打开即可管理任务。详见 [how-to-use.md](how-to-use.md#没有前端用独立任务管理页)。

### 场景四：用 do-tasks-subagent 批量并发执行

当任务之间完全互不冲突且机器资源充足时，可以使用批量并行模式同时执行多条任务。

→ 每批取 N 条，每个 agent 在独立 git worktree 中工作，全部完成后合并回主分支

```bash
# 默认 3 个 agent 并发
/do-tasks-subagent

# 指定 5 个 agent 并发
/do-tasks-subagent 5

# 持久化轮询
/loop 5m /do-tasks-subagent 3
```

### 场景五：用 do-tasks-team 团队协作执行（推荐）

当任务较多且部分任务可能冲突时，团队模式是最佳选择。它会**智能分析任务间的关系，按角色分组调度**：同一角色内串行执行避免冲突，不同角色间并行加速。

→ 按角色分 worktree，角色内串行、角色间并行，动态扩展

```bash
# 默认最多 5 个角色 agent
/do-tasks-team

# 指定最大 agent 数
/do-tasks-team 8

# 持久化轮询
/loop 5m /do-tasks-team 5
```

## 三种模式对比

| 特性 | do-tasks（串行） | do-tasks-subagent（批量并行） | do-tasks-team（团队协作） |
|------|------------------|---------------------------|--------------------------|
| 执行方式 | 串行，一次一条 | 批量取 N 条 → 全部完成 → 下一批 | 按角色分组，角色内串行，角色间并行 |
| 工作目录 | 当前目录 | 每个任务独立 worktree | 每个角色独立 worktree |
| 适用场景 | 任务间有依赖 | 任务间基本独立 | 混合场景（部分冲突部分独立） |
| 冲突处理 | 无冲突 | 事前分析踢出冲突任务 + agent 自行合并解决 | 事前按角色隔离，预防冲突 |
| 任务认领 | 逐条 | 一批全完成才取下一批 | 各角色 agent 独立完成，不互相等待 |
| 动态任务 | 支持 | 批次间支持 | 每轮重新分析，可新建角色 |
| 资源占用 | 低 | 较高 | 中等（按角色数决定） |
| 推荐度 | 基础 | 任务完全独立时 | **通用推荐** |

## 系统架构

```
┌──────────────────────────────────────────────────────────────┐
│                        用户浏览器                              │
│                    (需求悬浮面板 / curl)                       │
└──────────────────┬───────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────┐
│              后端服务 (REST API) + requirements 表             │
└──────────────────┬───────────────────────────────────────────┘
                   ▲
                   │
┌──────────────────┴───────────────────────────────────────────┐
│              Claude Code (/do-tasks Command)                  │
│                                                              │
│  串行: 取任务 → 分析需求 → 编码实现 → 标记完成 → 取下一个      │
│  并行: 批量取任务 → N 个 worktree 同时执行 → 合并 → 取下一批   │
│  团队: 分析角色 → 按角色分 worktree → 角色内串行 → 合并        │
└──────────────────────────────────────────────────────────────┘
```

## 核心用法

### 串行模式 — `/do-tasks`

#### 一次性执行

```bash
/do-tasks
```

Claude 从任务队列中逐条取出 pending 任务，分析需求、编码实现、标记完成，直到全部处理完毕。

#### 持久化轮询

```bash
/loop 5m /do-tasks
```

每 5 分钟自动检查一次新需求并执行。你可以把它想象成一个**永远在线的 Claude 助手**：你往任务队列里扔需求，它定时自动拾取并实现。

### 并行模式 — `/do-tasks-subagent`

> **注意**：批量 subagent 模式，一批全部完成才取下一批。执行前会分析任务间依赖和冲突，将有冲突的任务推迟到下一轮。如果任务间关联较多，推荐使用团队模式。

#### 默认并发度（3）

```bash
/do-tasks-subagent
```

同时启动 3 个 agent，各自在独立 worktree 中执行一条任务。

#### 指定并发度

```bash
/do-tasks-subagent 5
```

同时启动 5 个 agent（上限）。

#### 持久化轮询

```bash
/loop 5m /do-tasks-subagent 3
```

每 5 分钟自动检查新需求，每次最多 3 个 agent 并发执行。

#### 并行执行流程

```
GET /api/requirements?status=pending — 获取待处理需求
         ↓
按 created_at 升序取前 N 条（N = 并发度）
         ↓
批量 PATCH 标记为 in_progress（防止重复拾取）
         ↓
同时发起 N 个 Agent（各自在独立 worktree 中）
   ├── Agent 1: worktree-a → 需求 #1
   ├── Agent 2: worktree-b → 需求 #2
   └── Agent 3: worktree-c → 需求 #3
         ↓
等待所有 agent 完成
         ↓
依次 git merge 各 worktree 分支到主分支
   ├── 成功 → 标记 done
   ├── 冲突 → agent 自行解决 / 无法解决则标记失败
   └── 失败 → 回退为 pending
         ↓
循环，直到没有 pending 需求
         ↓
输出执行摘要
```

#### 冲突处理策略

每个 agent 负责自行解决合并冲突：

1. **预防**：合理拆分任务粒度，让每个任务修改不同的文件/模块
2. **agent 自行解决**：合并时若有冲突，agent 必须自行分析并解决
3. **失败回退**：agent 无法解决冲突时，执行 `git merge --abort`，任务标记为失败并回退为 pending

#### 并行模式注意事项

- **资源消耗**：并发度越高，CPU / 内存 / API 调用消耗越大，请根据机器配置合理设置
- **任务粒度**：建议每个任务修改独立的模块/文件，减少合并冲突概率
- **git 状态**：执行前确保工作目录干净（无未提交的修改），避免 worktree 创建失败
- **项目需为 git 仓库**：worktree 功能依赖 git

### 团队模式 — `/do-tasks-team`（推荐）

> **最智能的模式**：自动分析任务间关系，按角色分组调度。同一角色串行避免冲突，不同角色并行提速。

#### 基本用法

```bash
# 默认最多 5 个角色 agent
/do-tasks-team

# 指定最大 agent 数
/do-tasks-team 8

# 持久化轮询
/loop 5m /do-tasks-team 5
```

#### 团队执行流程

```
GET /api/requirements?status=pending — 获取所有待处理需求
         ↓
分析需求 + 项目结构 → 按修改区域划分角色
         ↓
角色分配表（如 frontend-ui / backend-api / docs）
         ↓
全局影响的角色优先独占执行
         ↓
为每个角色启动 1 个 Agent（独立 worktree）
   ├── Agent(frontend-ui): #1 → #3 → #7（串行）
   ├── Agent(backend-api): #2 → #5（串行）
   └── Agent(docs): #6（串行）
         ↓                    ↑
         └── 各角色 agent 内部串行执行各自的任务
         ↓
所有角色完成 → 依次合并各 worktree 分支
         ↓
查询新增 pending 需求 → 匹配角色或新建角色 → 重复
         ↓
输出执行摘要（含角色分配详情）
```

#### 为什么推荐团队模式

1. **不怕冲突**：可能改同一文件的任务自动归入同一角色串行执行
2. **不浪费时间**：不像 subagent 模式要等最慢的 agent，各角色独立推进
3. **智能分组**：根据项目结构和需求内容自动分析，而非盲目并发
4. **动态适应**：新需求进来后可匹配已有角色或创建新角色

### 串行执行流程

```
GET /api/requirements?status=pending — 获取待处理需求
         ↓
按 created_at 升序取第一条
         ↓
阅读需求 → 规划方案 → 编码实现
         ↓
PATCH /api/requirements/:id/status — 标记为 done，附上完成说明
         ↓
回到第一步，直到没有 pending 需求
         ↓
输出执行摘要
```

## 任务状态流转

```
pending  ──→  in_progress  ──→  done
(待处理)       (执行中)         (已完成)
    ↑
    └──── 失败时回退（subagent / team 模式）
```

- **pending** — 新创建的任务，等待 Claude 拾取
- **in_progress** — Claude 正在执行中
- **done** — 已完成，附带执行说明（notes 字段）

## 如何接入

👉 详见 **[how-to-use.md](how-to-use.md)** — 包含前提条件、分步骤接入指南、详细 API 规格、数据库建表语句和前端 UI 参考。

## 目录结构

```
commands/do-tasks/
├── README.md              ← 本文件（设计说明 & 总览）
├── how-to-use.md          ← 详细接入指南（前提条件、步骤、API 规格）
├── do-tasks.md            ← 串行模式 command 模板（复制到 .claude/commands/）
├── do-tasks-subagent.md   ← 批量并行模式 command 模板（复制到 .claude/commands/）
├── do-tasks-team.md       ← 团队协作模式 command 模板（复制到 .claude/commands/）
└── task-manager.html      ← 独立任务管理页（无前端时使用）
```

## API 参考

| 方法 | 端点 | 说明 |
|------|------|------|
| GET | `/api/requirements?status=xxx` | 获取需求列表，支持 status 筛选，按 created_at 升序 |
| POST | `/api/requirements` | 创建新需求（title 必填） |
| PUT | `/api/requirements/:id` | 更新需求全部字段 |
| PATCH | `/api/requirements/:id/status` | 快速更新状态，可选 notes 字段 |
| DELETE | `/api/requirements/:id` | 删除需求 |
| POST | `/api/upload-image` | 上传图片（multipart/form-data），保存到 `./images/`，返回相对路径 |
