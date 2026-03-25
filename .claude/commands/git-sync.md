---
description: 一键完成 git 双向同步：提交本地改动、rebase 拉取远端、自动解决冲突、推送
---

按以下步骤完成双向同步：

## 第一步：提交本地改动

1. 运行 `git status` 和 `git diff --stat` 查看当前改动
2. 如果有未提交的改动：
   - `git add -A`
   - 提交：若用户提供了 commit message 则使用，否则根据 `git diff --cached --stat` 自动生成简洁提交信息
3. 如果没有改动，跳过提交步骤

## 第二步：拉取远端代码（rebase 方式）

1. 运行 `git pull --rebase origin master`
2. 如果 rebase 过程中出现冲突：
   - 运行 `git diff --name-only --diff-filter=U` 列出冲突文件
   - 逐个读取冲突文件，分析冲突内容（`<<<<<<<` / `=======` / `>>>>>>>` 标记）
   - 自动解决冲突：优先保留双方改动（合并），若无法合并则保留本地版本（ours）
   - 解决后 `git add <conflicted-files>`
   - 运行 `git rebase --continue`
   - 若 rebase 仍有问题，运行 `git rebase --abort` 并告知用户手动处理

## 第三步：推送到远端

1. 运行 `git push origin master`
2. 输出推送结果及最近 5 条提交记录 `git log --oneline -5`，确认同步成功
