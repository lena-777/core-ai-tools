# git-sync

> 一键完成 git 提交、拉取、推送的双向同步，自动处理冲突。

## 设计目的

日常开发中，git 同步是最频繁的操作之一，但手动执行 add → commit → pull → push 既繁琐又容易出错：

- **忘记拉取就推送** — 导致 push 被拒绝，还得重新 pull rebase
- **rebase 冲突不会处理** — 手忙脚乱，容易丢代码
- **commit message 懒得写** — 随手写个 "update"，后续无法追溯

git-sync 将整个流程自动化：**提交本地改动 → rebase 拉取远端 → 自动解决冲突 → 推送到远端**，一个命令搞定。

## 使用场景

### 场景一：日常开发同步

写完一段代码，想快速提交并同步到远端，不想手动敲一堆 git 命令。

```bash
/git-sync
```

### 场景二：带自定义 commit message

有明确的提交说明，想指定 commit message。

```bash
/git-sync 修复登录页面样式问题
```

### 场景三：配合 /loop 定时自动同步

希望代码定时自动同步，比如每 10 分钟检查一次有没有新改动。

```bash
/loop 10m /git-sync
```

## 执行流程

```
git status / git diff --stat — 检查本地改动
         ↓
有改动？ → git add -A → git commit（自动生成或使用指定 message）
         ↓
git pull --rebase origin master — 拉取远端代码
         ↓
有冲突？ → 逐个分析并自动解决 → git rebase --continue
         ↓
git push origin master — 推送到远端
         ↓
git log --oneline -5 — 输出最近提交，确认同步成功
```

## 冲突处理策略

当 rebase 出现冲突时，Claude 会：

1. 列出所有冲突文件
2. 逐个读取并分析冲突内容（`<<<<<<<` / `=======` / `>>>>>>>` 标记）
3. **优先合并双方改动**，若无法合并则保留本地版本（ours）
4. 如果 rebase 仍然失败，执行 `git rebase --abort` 回退并告知用户手动处理

## 如何使用

将本仓库的 `commands/git-sync/git-sync.md` 复制到你项目的 `.claude/commands/git-sync.md` 即可。

```bash
cp commands/git-sync/git-sync.md your-project/.claude/commands/git-sync.md
```

## 目录结构

```
commands/git-sync/
├── README.md          ← 本文件（设计说明 & 总览）
└── git-sync.md        ← command 模板（复制到 .claude/commands/）
```
