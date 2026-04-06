---
description: "安装或更新 ai-tools 中的 command/skill：给 URL 则单个 upsert，不给则批量更新所有已安装的 tool"
---

按以下流程执行 upsert 操作。参数来自 `$ARGUMENTS`。

---

## 第零步：解析参数

从 `$ARGUMENTS` 中提取：

- **URL 参数**：格式 `github:{owner}/{repo}/{type}/{name}`（如 `github:lena-777/core-ai-tools/commands/git-sync`）
- **名字参数**：不含 `github:` 前缀的普通名字（如 `do-tasks`、`git-sync`）
- **ref 参数**：可选，格式 `ref:{branch_or_tag}`，默认 `main`
- **--check 标志**：如果存在，表示只检查不更新

判断走哪个分支：

- 参数以 `github:` 开头 → **单个 upsert 流程**（第一步）
- 参数是普通名字（不含 `github:`） → **按名字查找流程**（第三步）
- 无参数 → **批量 upsert 流程**（第二步）

---

## 第一步：单个 upsert 流程

### 1.1 解析 URL

从 URL `github:{owner}/{repo}/{path...}` 中提取：

- `owner`：仓库拥有者
- `repo`：仓库名
- `path`：去掉 `github:{owner}/{repo}/` 后的剩余部分（如 `commands/git-sync`）
- `type`：从 path 第一段判断 — `commands/*` → `command`，`skills/*` → `skill`
- `name`：path 的最后一段（如 `git-sync`）
- `ref`：从 `ref:xxx` 参数获取，默认 `main`

### 1.2 检查本地是否已安装

依次检查以下路径是否存在 `.tool.source.json`：

- 多文件模式：`.claude/{type}s/{name}/.tool.source.json`
- 单文件模式：`.claude/commands/.{name}.tool.source.json`

结果：
- **文件存在** → 已安装，记录当前 `commit` 值（`old_commit`）
- **文件不存在** → 新安装

### 1.3 获取远端信息

执行以下两个 gh api 调用：

```bash
# 列出目录内容
gh api "repos/{owner}/{repo}/contents/{path}?ref={ref}"

# 获取最新 commit
gh api "repos/{owner}/{repo}/commits?path={path}&sha={ref}&per_page=1"
```

从目录列表中：
- 识别所有文件（`.md` 及其他）
- 区分两类：
  - **tool 文件**：排除 `README.md` 和 `how-to-use.md` 后的 `.md` 文件
  - **文档文件**：`README.md`、`how-to-use.md`（如果存在）

从 commits 中提取：
- `new_commit`：最新 commit 的 SHA（取前 7 位）
- `commit_messages`：commit message 摘要

### 1.4 对比 commit

如果已安装且 `old_commit` == `new_commit`（前 7 位）→ 输出 `✅ {name} 已是最新 ({commit})` 并结束。

### 1.5 读取远端文档（README.md / how-to-use.md）

**每次安装或更新前必须执行此步骤。**

如果远端目录中存在 `README.md` 或 `how-to-use.md`：

```bash
# 下载并读取文档内容
gh api "repos/{owner}/{repo}/contents/{path}/README.md?ref={ref}"
gh api "repos/{owner}/{repo}/contents/{path}/how-to-use.md?ref={ref}"
```

将 base64 content 解码后**仔细阅读**，重点关注：
- ⚠️ 是否有**前置依赖**需要安装（如 npm 包、系统工具、MCP server 等）
- ⚠️ 是否有**配置要求**（如 settings.json 权限、环境变量等）
- ⚠️ 是否有**破坏性变更**（breaking changes）或迁移说明

如果发现有需要用户关注的内容，在安装/更新完成后**醒目提示用户**（见 1.8）。

### 1.6 下载并写入 tool 文件

对每个 tool 文件，从远端下载：

```bash
# 获取文件内容（base64 编码）
gh api "repos/{owner}/{repo}/contents/{path}/{filename}?ref={ref}"
```

将返回的 `content` 字段（base64）解码后写入本地 `.claude/{type}s/{name}/{filename}`。

> **注意**：如果 type 是 command 且只有单个 .md 文件，则直接写为 `.claude/commands/{name}.md`（平铺，不建子目录）。
> 如果有多个 .md 文件，则写入 `.claude/{type}s/{name}/` 目录下。

### 1.7 更新 .tool.source.json

在本地对应位置创建或更新 `.tool.source.json`（这是本地元信息，不从远端下载）：

```json
{
  "name": "{name}",
  "type": "{type}",
  "repo": "github:{owner}/{repo}",
  "path": "{path}",
  "files": ["{file1}.md", "{file2}.md"],
  "commit": "{new_commit}",
  "updated_at": "{ISO 8601 当前时间}",
  "ref": "{ref}"
}
```

存放位置：
- 单文件平铺模式：`.claude/commands/.{name}.tool.source.json`
- 多文件目录模式：`.claude/{type}s/{name}/.tool.source.json`

### 1.8 展示结果

- 新安装：`✅ 已安装 {name} ({new_commit})`
- 已更新：`🆕 已更新 {name} ({old_commit} → {new_commit})`
  - 附带更新说明：从 commits 中提取 message

- 如果在步骤 1.5 中发现了依赖/配置/迁移等注意事项，**醒目展示**：

```
⚠️ 注意事项（来自 README.md / how-to-use.md）:
  - 需要安装依赖: xxx
  - 需要添加权限: xxx
  - 破坏性变更: xxx
```

---

## 第二步：批量 upsert 流程

### 2.1 扫描已安装的 tools

搜索以下路径的 `.tool.source.json` 文件：

```
.claude/commands/*/.tool.source.json
.claude/commands/.*.tool.source.json
.claude/skills/*/.tool.source.json
.claude/skills/.*.tool.source.json
```

读取每个文件，提取 `name`、`type`、`repo`、`path`、`commit`、`ref` 信息。

如果没有找到任何已安装的 tool → 输出提示信息并结束。

> **重要**：只处理有 `.tool.source.json` 的 tool（即通过本命令安装的）。手动复制安装的存量 tool 没有此文件，应直接忽略。如用户需要纳入管理，需先用 URL 方式重新安装。

### 2.2 逐个检查远端

对每个已安装的 tool：

```bash
# 解析 repo 字段: github:{owner}/{repo} → owner, repo
# 获取最新 commit
gh api "repos/{owner}/{repo}/commits?path={path}&sha={ref}&per_page=1"
```

对比本地 `commit` vs 远端最新 commit。

### 2.3 展示清单

格式化输出：

```
已安装的 tools:

  git-sync     a3f7b2e → d9c1f4a  🆕 有更新
  do-tasks     b2c3d4e → b2c3d4e  ✅ 已最新
  simplify     c3d4e5f → f8a2b1c  🆕 有更新

更新说明:
  git-sync: 支持自定义 remote 名称
  simplify: 优化提示词结构
```

### 2.4 判断是否继续

- 如果有 `--check` 标志 → 到此结束，不执行更新
- 如果所有 tools 都已最新 → 输出 `✅ 所有 tools 均为最新` 并结束
- 否则 → 使用 AskUserQuestion 询问用户是否确认更新

### 2.5 执行更新

对每个有更新的 tool，按照「单个 upsert 流程」的步骤 1.5 ~ 1.7 执行：
1. 读取远端文档（README.md / how-to-use.md），检查依赖和适配要求
2. 下载远端 tool 文件写入本地
3. 更新本地 `.tool.source.json`
4. 某个 tool 下载失败不影响其他 tool，跳过并报错

### 2.6 展示最终结果

```
更新完成:
  ✅ git-sync   a3f7b2e → d9c1f4a
  ✅ simplify   c3d4e5f → f8a2b1c
  ⏭️ do-tasks   b2c3d4e (已跳过，无需更新)
```

如果任何 tool 的文档中发现了依赖/配置/迁移等注意事项，**在最后统一醒目展示**：

```
⚠️ 注意事项:
  git-sync: 需要添加权限 Bash(git fetch:*)
  simplify: 新增依赖，请先运行 npm install xxx
```

---

## 第三步：按名字查找流程

当参数是普通名字（如 `do-tasks`）而非完整 URL 时执行此流程。

### 3.1 在本地查找 .tool.source.json

用参数作为 `{name}`，依次检查：

```
.claude/commands/{name}/.tool.source.json
.claude/commands/.{name}.tool.source.json
.claude/skills/{name}/.tool.source.json
.claude/skills/.{name}.tool.source.json
```

### 3.2 判断结果

- **找到 `.tool.source.json`** → 从中读取 `repo`、`path`、`ref`、`commit` 等信息，自动构造完整参数，转入**第一步：单个 upsert 流程**（从步骤 1.3 开始）
- **未找到 `.tool.source.json`** → 提示用户该 tool 尚未通过本命令安装，请提供完整 URL：

```
❓ 未找到 {name} 的安装记录。
   请提供完整路径来安装，例如：
   /upsert-tools github:{owner}/{repo}/commands/{name}
```
