---
description: "企业微信机器人通知：通过 Webhook 发送 Markdown 消息到群聊或私聊。当需要发送通知、告警、汇报时调用。"
---

当需要发送通知（如任务完成、构建结果、告警、定时汇报等）时，使用企业微信机器人 Webhook 发送消息。

---

## 配置读取

从项目 `.env` 文件或环境变量中读取：
- `WECOM_WEBHOOK_KEY` — 必需，机器人 Webhook Key
- `WECOM_CHAT_ID` — 可选，默认接收对象

## 发送命令

```bash
curl -s 'https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key='"$WECOM_WEBHOOK_KEY" \
  -H 'Content-Type: application/json' \
  -d '{
    "chatid": "'"$WECOM_CHAT_ID"'",
    "msgtype": "markdown",
    "markdown": {
      "content": "消息正文"
    }
  }'
```

## chatid 规则

| chatid 值 | 行为 |
|-----------|------|
| 个人英文名（如 `zhangsan`） | 私聊推送给该用户 |
| 群聊 ID | 推送到指定群聊 |
| 空字符串 `""` | 广播到机器人所在的所有群聊 |

发送时可以临时覆盖 chatid，不影响 `.env` 中的默认值。

## 消息格式

- 使用 Markdown 格式，支持标题、加粗、链接
- 开头用简短标题说明通知类型（如 `**🚀 部署完成**`）
- 关键信息加粗突出
- 保持简洁，避免过长
