# do-tasks 接入指南

> 本文档是 do-tasks command 的详细接入指南。总体介绍请参考 [README.md](README.md)。

## 工作原理

```
用户在页面提交需求 → tasks 表写入 pending 记录
                         ↓
         Claude Code 执行 /do-tasks（或 /loop 5m /do-tasks）
                         ↓
         读取 pending 任务 → 分析需求 → 编码实现 → 标记 done
```

---

## 快速接入

### 前提条件

- 项目有可用的数据库（SQLite、PostgreSQL 等均可）
- 项目有后端服务，能提供 REST API
- 已安装 [Claude Code](https://docs.anthropic.com/en/docs/claude-code)

### 步骤

把下面这段话直接发给 Claude Code，它会自动帮你搭建整套功能：

> 请帮我的项目接入「需求任务」功能，具体要求：
>
> **1. 数据库**
> 在现有数据库中创建 tasks 表，字段：id、title、description、priority(low/medium/high)、status(pending/in_progress/done)、notes、created_at、updated_at。给 status 加索引。
>
> **2. REST API（5 个端点）**
> - GET /api/tasks?status=xxx — 获取任务列表，按 created_at 升序，支持 status 筛选
> - POST /api/tasks — 创建任务（title 必填，priority 默认 medium）
> - PUT /api/tasks/:id — 更新任务全部字段
> - PATCH /api/tasks/:id/status — 快速更新状态，可选 notes 字段
> - DELETE /api/tasks/:id — 删除任务
>
> **3. 前端 UI**
> 在页面右下角添加浮动按钮「需求」，点击弹出侧边面板，功能包括：
> - 快速输入框（⌘+Enter 提交）
> - 任务列表（点击切换状态 pending→in_progress→done）
> - 未完成数量红色角标
> - 已完成任务默认折叠，点击可展开
> - hover 显示删除按钮
>
> 请参考项目现有的技术栈和代码风格来实现。

搭建完成后，将本仓库的 `commands/do-tasks/do-tasks.md` 复制到你项目的 `.claude/commands/do-tasks.md`，并把其中的 `$TASK_API_BASE` 替换为实际的 API 地址（如 `http://127.0.0.1:5001`）。

---

## 接入完成后的使用方式

```bash
# 手动执行一次：消费所有 pending 任务
/do-tasks

# 定时轮询：每 5 分钟检查一次新需求并自动实现
/loop 5m /do-tasks
```

---

## 详细规格（供参考）

### 数据库 — tasks 表

```sql
CREATE TABLE IF NOT EXISTS tasks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT NOT NULL,                          -- 需求标题（必填）
    description TEXT DEFAULT '',                  -- 需求详细描述
    priority TEXT DEFAULT 'medium',               -- 优先级：low / medium / high
    status TEXT DEFAULT 'pending',                -- 状态：pending / in_progress / done
    notes TEXT DEFAULT '',                        -- AI 执行结果备注
    created_at TEXT DEFAULT (datetime('now')),     -- 创建时间
    updated_at TEXT DEFAULT (datetime('now'))      -- 更新时间
);
CREATE INDEX IF NOT EXISTS idx_tasks_status ON tasks(status);
```

### REST API 接口详情

**获取待办任务：**
```
GET /api/tasks?status=pending

Response:
{
  "tasks": [
    {
      "id": 1,
      "title": "添加用户登录功能",
      "description": "支持邮箱+密码登录，需要做表单校验",
      "priority": "high",
      "status": "pending",
      "notes": "",
      "created_at": "2025-03-25 10:00:00",
      "updated_at": "2025-03-25 10:00:00"
    }
  ]
}
```

**更新任务状态：**
```
PATCH /api/tasks/<id>/status
Content-Type: application/json

Request:  {"status": "done", "notes": "已实现登录功能，修改了 auth.py 和 login.html"}
Response: {"ok": true, "message": "状态已更新"}
```

**注意：**
- GET 返回的任务列表需按 `created_at` 升序排序（最早的排前面）
- PATCH 的 `notes` 字段为可选参数
- `status` 只接受 `pending` / `in_progress` / `done` 三个值

### 前端 UI — 需求浮动面板

```
┌─────────────────────┐
│  页面主内容           │
│                     │
│            ┌──────────────┐
│            │ 需求          ✕│  ← 面板标题 + 关闭
│            │───────────────│
│            │ [输入需求...] +│  ← 快速添加
│            │───────────────│
│            │ ○ 添加用户登录  ✕│  ← pending 任务
│            │ ○ 优化首页性能  ✕│
│            │───────────────│
│            │ 已完成 (2) ▸   │  ← 可折叠
│            └──────────────┘
│                    [需求]   │  ← FAB 按钮（带角标）
└─────────────────────┘
```

#### 参考 HTML

```html
<!-- FAB 按钮 -->
<div id="task-fab" onclick="toggleTaskPanel()">需求</div>

<!-- 需求面板 -->
<div id="task-panel" class="task-panel-hidden">
    <div class="task-panel-header">
        <span>需求 <span id="task-count" class="task-count-badge" style="display:none">0</span></span>
        <button class="task-panel-close" onclick="toggleTaskPanel()">&times;</button>
    </div>
    <div class="task-panel-input">
        <input type="text" id="task-quick-input"
               placeholder="输入需求，⌘+回车提交..."
               onkeydown="if(event.key==='Enter'&&(event.metaKey||event.ctrlKey))quickAddTask()">
        <button onclick="quickAddTask()">+</button>
    </div>
    <div id="task-panel-list" class="task-panel-list"></div>
</div>
```

---

## 目录结构

```
commands/do-tasks/
├── README.md          ← 本文件（接入指南）
└── do-tasks.md        ← command 模板（复制到 .claude/commands/）
```
