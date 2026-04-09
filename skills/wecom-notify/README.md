# wecom-notify

> 企业微信机器人通知：通过 Webhook 发送 Markdown 消息到群聊或私聊。

## 功能

通过企业微信机器人的 Webhook 接口，发送 Markdown 格式消息。支持：

- **私聊推送** — chatid 填个人英文名
- **群聊推送** — chatid 填群 ID
- **广播推送** — chatid 留空，推送到机器人所在的所有群聊

## 使用场景

- 任务完成后发送通知
- 构建/部署结果推送
- 定时汇报
- 告警通知
- 任何需要即时通知的场景

## 首次安装

1. 在企业微信群中找到你创建的机器人，**点击机器人头像**查看详情（需要是机器人创建者）
2. 复制 Webhook 地址中 `?key=` 后面的部分
3. 设置环境变量 `WECOM_WEBHOOK_KEY`，或在使用时手动提供

## 如何安装

```bash
/upsert-tools github:lena-777/ai-tools/skills/wecom-notify
```

或手动复制：

```bash
mkdir -p your-project/.claude/skills/wecom-notify
cp skills/wecom-notify/wecom-notify.md your-project/.claude/skills/wecom-notify/
```

## 目录结构

```
skills/wecom-notify/
├── README.md            ← 本文件（说明 & 总览）
└── wecom-notify.md      ← skill 模板（复制到 .claude/skills/wecom-notify/）
```
