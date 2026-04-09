# 安装配置指引

安装此 skill 后，按以下步骤引导用户完成配置：

## Step 1: 获取 Webhook Key

- 检查项目 `.env` 文件中是否已有 `WECOM_WEBHOOK_KEY`
- 如果没有，询问用户获取：
  > 请提供企业微信机器人的 Webhook Key。
  > 获取方式：在企业微信中点击机器人头像 → 查看详情 → 复制 Webhook 地址中 `?key=` 后面的部分。
  > 注意：只有机器人的创建者才能看到 Webhook 地址。
- 用户提供 key 后，追加写入项目根目录 `.env` 文件：`WECOM_WEBHOOK_KEY=用户提供的key`

## Step 2: 配置默认接收对象

- 询问用户："你希望通知默认发送到哪里？"，并给出选项：
  1. 填写个人英文名（如 `zhangsan`）→ 私聊推送给该用户
  2. 填写群聊 ID → 推送到指定群聊
  3. 不填，留空 → 广播到机器人所在的所有群聊
- 用户回答后，追加写入 `.env` 文件：`WECOM_CHAT_ID=用户提供的值`（留空则写 `WECOM_CHAT_ID=`）

## Step 3: 验证配置

- 用以下命令发送一条测试消息：
  ```bash
  source .env 2>/dev/null
  curl -s 'https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key='"$WECOM_WEBHOOK_KEY" \
    -H 'Content-Type: application/json' \
    -d '{"chatid":"'"$WECOM_CHAT_ID"'","msgtype":"markdown","markdown":{"content":"✅ 企业微信通知配置成功"}}'
  ```
- 如果返回 `errcode: 0`，告知用户配置完成
- 如果失败，展示错误信息并协助排查
