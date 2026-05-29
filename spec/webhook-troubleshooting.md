# 每日股票分析 - Webhook 推送问题排查记录

## 问题描述

用户反馈：「每日股票分析」工作流运行完成后，分析报告没有推送到预期的自定义 Webhook（钉钉机器人）。

---

## 调查过程

### 代码层分析

**通知渠道配置入口**（`src/config.py`）：

```python
custom_webhook_urls=[
    u.strip()
    for u in os.getenv('CUSTOM_WEBHOOK_URLS', '').split(',')
    if u.strip()
]
```

- 从环境变量 `CUSTOM_WEBHOOK_URLS` 读取，逗号分隔，支持多个 URL。
- 若环境变量为空或未配置，`custom_webhook_urls` 为空列表。

**渠道检测逻辑**（`src/notification.py`）：

```python
if self._custom_webhook_urls:
    channels.append(NotificationChannel.CUSTOM)
```

- 只有 `custom_webhook_urls` 非空才会注册 CUSTOM 渠道。
- 若无任何渠道注册，`_send_notifications()` 会直接打印 `通知渠道未配置，跳过推送` 并返回。

**发送逻辑**（`src/notification_sender/custom_webhook_sender.py`）：

- 钉钉 URL 识别规则：URL 包含 `dingtalk` 或 `oapi.dingtalk.com`。
- 钉钉消息体有约 20000 字节限制，超长时自动分批发送（`_send_dingtalk_chunked`）。
- HTTP 成功判断：**仅 status_code == 200**（Discord 图片接口例外，支持 200/204）。

---

### CI 日志对比

通过 GitHub Actions MCP 工具查看两次运行的日志：

| 运行编号 | 时间（UTC） | 结论 |
|----------|------------|------|
| Run #1（24768914944）| 2026-04-22 08:41 | ❌ **未推送** |
| Run #2（24769499946）| 2026-04-22 08:55 | ✅ 推送成功 |

**Run #1 关键日志**：
```
决策仪表盘日报已保存: .../reports/report_20260422.md
通知渠道未配置，跳过推送   ← 根本原因
```

**Run #2 关键日志**：
```
已配置 1 个通知渠道：自定义Webhook
自定义 Webhook 1（钉钉）推送成功
自定义 Webhook 推送完成：成功 1/1
通知发送完成：成功 1 个，失败 0 个
大盘复盘推送成功
```

---

## 根本原因

**第一次运行时 `CUSTOM_WEBHOOK_URLS` Secret 未配置**（或配置后尚未生效），导致系统判断「无通知渠道」，报告生成后直接跳过推送。

---

## 排查清单

按优先级排序：

1. **Secret 未配置或名称拼错**
   - 路径：GitHub Repo → Settings → Secrets and variables → Actions → Secrets
   - 变量名必须精确为：`CUSTOM_WEBHOOK_URLS`（区分大小写）
   - CI 日志中的「通知渠道」配置检查区段应显示：`自定义Webhook: ✅ 已配置`

2. **URL 格式问题**
   - 多个 URL 用英文逗号分隔，URL 首尾不能有空格或换行。
   - 错误示例：`https://oapi.dingtalk.com/robot/send?access_token=xxx ` （末尾有空格）

3. **Webhook 返回非 200 状态码**
   - 代码只认 `200` 为成功，若端点返回 `201`/`204` 会误报失败。
   - 可在日志中看到：`自定义 Webhook 推送失败: HTTP 201`

4. **非交易日跳过整个 pipeline**
   - 默认 `TRADING_DAY_CHECK_ENABLED=true`，非交易日不运行分析，也不推送。
   - 手动触发时可勾选「强制运行」（`force_run=true`）跳过此检查。

---

## 涉及的关键文件

| 文件 | 作用 |
|------|------|
| `.github/workflows/daily_analysis.yml` | 定时触发（周一至周五 UTC 10:00）、环境变量注入 |
| `src/config.py` | 从环境变量解析 `custom_webhook_urls` |
| `src/notification.py` | 渠道检测（`_detect_all_channels`）、统一发送入口（`send`、`_send_notifications`） |
| `src/notification_sender/custom_webhook_sender.py` | 实际 HTTP POST、钉钉分批发送、payload 格式适配 |
| `src/core/pipeline.py` | 个股分析完成后触发推送（`_send_notifications`） |
| `src/core/market_review.py` | 大盘复盘完成后触发推送 |

---

## 支持的通知渠道

系统支持以下渠道，可同时配置多个，全部推送：

| 渠道 | 环境变量 |
|------|---------|
| 企业微信 Webhook | `WECHAT_WEBHOOK_URL` |
| 飞书 Webhook | `FEISHU_WEBHOOK_URL` |
| Telegram | `TELEGRAM_BOT_TOKEN` + `TELEGRAM_CHAT_ID` |
| 邮件 SMTP | `EMAIL_SENDER` + `EMAIL_PASSWORD` + `EMAIL_RECEIVERS` |
| Pushover | `PUSHOVER_USER_KEY` + `PUSHOVER_API_TOKEN` |
| PushPlus | `PUSHPLUS_TOKEN` |
| Server酱3 | `SERVERCHAN3_SENDKEY` |
| 自定义 Webhook（钉钉/Bark 等）| `CUSTOM_WEBHOOK_URLS` |
| Discord | `DISCORD_WEBHOOK_URL` 或 `DISCORD_BOT_TOKEN` + `DISCORD_MAIN_CHANNEL_ID` |
| Slack | `SLACK_WEBHOOK_URL` 或 `SLACK_BOT_TOKEN` + `SLACK_CHANNEL_ID` |
| AstrBot | `ASTRBOT_URL` + `ASTRBOT_TOKEN` |
