---
name: tg-matrix-lead-gen
description: Telegram 矩阵获客：登录/注册、绑定 TG 执行号、一句话启动完整获客、跨天查进度
metadata:
  layer: L1-capability
  requires: []
  tags: [telegram, lead-generation, tg-matrix, crm, outreach]
---

# TG Matrix Lead Generation

当用户表达“帮我在 TG 里面获客 / 做 Telegram 获客 / 找 TG 客户”等意图时，按 TG Matrix 的完整获客流程处理。

## 工具前提

优先使用已连接的 `tg-matrix` MCP 工具。不要让用户手动提供 `auth_token`。

推荐远程 MCP 配置：

```json
{
  "url": "https://tg-matrix-agent.vidau.info/mcp",
  "enabled": true,
  "connect_timeout": 60,
  "timeout": 300,
  "headers": {},
  "tools": {
    "prompts": true,
    "resources": true
  }
}
```

## 核心规则

- 不要向用户索要 `auth_token`，token 是系统内部凭据。
- 登录/注册优先调用 `web_login_credential` / `web_register_credential`，把用户输入的密码原文传给 `credential_text`。
- 不要使用容易被宿主脱敏成 `***` 的 `password` 参数，除非确认宿主不会脱敏。
- 不要让用户分步启动“搜群、清洗、私聊”。启动获客任务只调用 `start_lead_generation` 一次。
- Telegram 执行号只需绑定一次。后续任务先调用 `account_list` 检查，不要每次都要求重新绑定。
- 看到脱敏号码时，先调用 `account_list`，使用返回的 `phone_ref`，不要要求用户补完整手机号。
- `api_id` / `api_hash` 由 TG Matrix 服务端配置，不要向用户索要。
- 任务处于 `partial + auto_resume` 时，只查询进度，不重复创建任务。
- `can_end=false` 对已结束任务只表示无需结束，不代表旧任务占用账号。只有 `blocks_accounts=true`、`blocking_accounts` 非空，或 `account_list` 显示 `task_lock_id` 指向该任务时，才说明任务占资源。

## 标准流程

1. 用户提出 TG 获客需求后，先确认 TG Matrix 登录状态。
2. 未登录时，引导用户选择“登录已有账号”或“注册新账号”。
3. 登录/注册成功后，调用 `account_list` 检查 Telegram 执行号。
4. 没有执行号时，询问手机号，并在发送验证码前询问是否需要独立代理/IP 隔离。
5. 需要代理时收集 `proxy_url`，格式固定为 `socks5://账号:密码@IP:端口`，传给 `account_send_code`。
6. 用户提供验证码后调用 `account_verify_code(code=...)`；需要 2FA 时再调用 `account_verify_2fa(password=...)`。
7. 启动任务前收集产品/服务、目标客户或关键词、私聊话术、目标加群数量。
8. 启动前必须确认两个策略：
   - “关键词需要我帮你 AI 扩展吗？”
   - “首封私聊要不要先破冰？”
9. 信息齐全后调用 `start_lead_generation`，保存返回的 `task_id`。
10. 用户问进度时，优先用当前 `task_id` 调 `get_task_status`；没有本地记录时调用 `list_active_tasks` 找回最近任务。

## 参数约定

- 用户选择关键词 AI 扩展：`rewrite_keywords=true`。
- 用户选择不扩展关键词：`rewrite_keywords=false`。
- 用户选择先破冰：`use_icebreaker=true`，`rewrite_outreach=true`。
- 用户选择不破冰、直接发开发信：`use_icebreaker=false`，`rewrite_outreach=true`。
- 只有用户明确要求“原文直发、不改写”时，才设置 `rewrite_outreach=false`。

## 账号不可用处理

读取 `account_list` 和 `get_task_status` 返回的诊断字段，不要猜测原因。

常见原因：

- `cooling` / `init`：账号孵化或冷却中。用户确认风险后可调用 `account_skip_incubation`。
- `work_rest`：处于工作时段休息。若 payload 显示可唤醒，可询问用户是否调用 `account_wake_work_rest`。
- `auth_expired` / “会话未授权” / “授权已过期” / “已失效”：在当前会话询问是否重新授权，不要让用户去后台或账号绑定页。
- `official_warn_cooldown`：Telegram 风控保护冷却，告诉用户预计恢复时间。
- 每日加群/私聊额度已满：说明恢复时间，等待系统自动续跑。

## 授权过期处理

如果授权过期：

1. 询问用户是否在当前会话重新授权该 TG Matrix 执行号。
2. 用户确认后，调用 `account_list` 获取 `phone_ref`。
3. 调用 `account_send_code(phone=phone_ref)`。
4. 用户提供验证码后调用 `account_verify_code(code=...)`。
5. 需要 2FA 时调用 `account_verify_2fa(password=...)`。
6. 成功后调用 `account_list` 和 `get_task_status(task_id)` 确认任务继续。

## 回复风格

用户说“帮我在 TG 里面获客”时，优先回复：

“可以。我先帮你走完整 TG 获客流程：确认 TG Matrix 登录状态，检查是否已绑定 Telegram 执行号；任务启动前我会向你确认产品/关键词、私聊话术，以及关键词是否 AI 扩展、首封私聊是否先破冰。整个任务会一次性启动完整流程，不会让你分步操作。”

## 禁止

- 不要问“你有没有 token”。
- 不要把 MCP 工具名暴露成用户必须理解的步骤。
- 不要让用户分别说“先加群、再清洗、再私聊”。
- 不要把授权过期引导到“后台/账号绑定页面”。
- 不要把 `can_end=false` 解释为旧任务占用资源。
- 不要说“没有跳过孵化工具”，当前工具集中有 `account_skip_incubation`。
