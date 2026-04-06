# upsert-tools

一键安装或更新 ai-tools 仓库中的 command / skill。

## 核心语义

借鉴 MySQL 的 UPSERT 语义：

- **有 URL** → 不存在则安装，存在则更新
- **只给名字** → 查找本地已安装记录，有则更新，无则提示提供完整 URL
- **无参数** → 批量更新所有已安装的 tool

> **注意**：批量更新只会处理通过本命令安装的 tool（有 `.tool.source.json`）。手动复制的存量 tool 会被忽略，如需纳入管理请用 URL 方式重新安装。

## 用法

```bash
# 安装或更新指定 tool（完整 URL）
/upsert-tools github:lena-777/core-ai-tools/commands/git-sync
/upsert-tools github:lena-777/core-ai-tools/skills/simplify ref:v1.0

# 按名字更新已安装的 tool
/upsert-tools do-tasks
/upsert-tools git-sync

# 批量更新所有已安装的 tool
/upsert-tools

# 只检查更新，不执行
/upsert-tools --check
```

## URL 格式

```
github:{owner}/{repo}/{type}/{name}
```

- `type`：`commands` 或 `skills`（从路径自动识别）
- `name`：tool 名称（路径最后一段）

可选参数：
- `ref:{branch_or_tag}`：指定分支或 tag，默认 `main`

## .tool.source.json

每个通过本命令安装的 tool 都会在本地保存一个 `.tool.source.json` 文件，记录来源信息：

```json
{
  "name": "git-sync",
  "type": "command",
  "repo": "github:lena-777/core-ai-tools",
  "path": "commands/git-sync",
  "files": ["git-sync.md"],
  "commit": "a3f7b2e",
  "updated_at": "2026-04-06T10:00:00Z",
  "ref": "main"
}
```

该文件用于批量更新时的来源追溯和版本对比。没有此文件的 tool（手动安装的）不会被批量更新流程处理。

## 执行流程

### 单个 upsert（有 URL）

1. 解析 URL，提取 owner / repo / path / type / name / ref
2. 检查本地 `.tool.source.json` 判断是否已安装
3. 通过 `gh api` 获取远端目录和最新 commit
4. 对比 commit，相同则跳过
5. **读取远端 README.md / how-to-use.md**，检查依赖、配置要求、破坏性变更
6. 下载远端 tool 文件写入本地
7. 更新本地 `.tool.source.json`（本地元信息，不从远端同步）
8. 展示安装/更新结果 + 醒目提示文档中的注意事项

### 按名字更新（只给名字）

1. 在本地 `.claude/commands/` 和 `.claude/skills/` 中查找 `.tool.source.json`
2. 找到 → 从中读取来源信息，转入「单个 upsert」流程（从步骤 3 开始）
3. 未找到 → 提示用户提供完整 URL 来安装

### 批量 upsert（无参数）

1. 扫描 `.claude/commands/` 和 `.claude/skills/` 下的 `.tool.source.json`
2. 忽略没有 `.tool.source.json` 的存量 tool
3. 逐个检查远端最新 commit
4. 展示更新清单（含 commit diff 和更新说明）
5. `--check` 模式到此结束；否则询问用户确认
6. 逐个执行：读文档 → 下载写入 → 更新元信息
7. 展示结果 + 统一展示所有注意事项

## 文件存放规则

- **单文件 command**：平铺为 `.claude/commands/{name}.md`，元信息存 `.claude/commands/.{name}.tool.source.json`
- **多文件 tool**：存入 `.claude/{type}s/{name}/` 目录，元信息存目录内 `.tool.source.json`
