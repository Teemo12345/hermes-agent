---
sidebar_position: 12
title: "Cron 故障排除"
description: "诊断和修复常见的 Hermes cron 问题 — 作业未触发、交付失败、技能加载错误和性能问题"
---

# Cron 故障排除

当 cron 作业未按预期行为时，按顺序进行这些检查。大多数问题属于以下四类之一：定时、交付、权限或技能加载。

---

## 作业未触发

### 检查 1：验证作业存在且处于活动状态

```bash
hermes cron list
```

查找作业并确认其状态为 `[active]`（不是 `[paused]` 或 `[completed]`）。如果显示 `[completed]`，重复次数可能已耗尽 — 编辑作业以重置它。

### 检查 2：确认调度正确

格式错误的调度会静默默认为一次性或完全被拒绝。测试您的表达式：

| 您的表达式 | 应该评估为 |
|----------------|-------------------|
| `0 9 * * *` | 每天上午 9:00 |
| `0 9 * * 1` | 每周一上午 9:00 |
| `every 2h` | 从现在开始每 2 小时 |
| `30m` | 从现在开始 30 分钟后 |
| `2025-06-01T09:00:00` | 2025 年 6 月 1 日 UTC 时间 9:00 AM |

如果作业触发一次然后从列表中消失，这是一个一次性调度（`30m`、`1d` 或 ISO 时间戳）— 预期行为。

### 检查 3：网关是否在运行？

Cron 作业由网关的后台计时器线程触发，该线程每 60 秒计时一次。常规的 CLI 聊天会话**不会**自动触发 cron 作业。

如果您期望作业自动触发，您需要一个正在运行的网关（`hermes gateway` 或 `hermes serve`）。对于一次性调试，您可以使用 `hermes cron tick` 手动触发计时。

### 检查 4：检查系统时钟和时区

作业使用本地时区。如果您的机器时钟错误或处于与预期不同的时区，作业将在错误的时间触发。验证：

```bash
date
hermes cron list   # 比较 next_run 时间与本地时间
```

---

## Delivery Failures

### Check 1: Verify the deliver target is correct

Delivery targets are case-sensitive and require the correct platform to be configured. A misconfigured target silently drops the response.

| Target | Requires |
|--------|----------|
| `telegram` | `TELEGRAM_BOT_TOKEN` in `~/.hermes/.env` |
| `discord` | `DISCORD_BOT_TOKEN` in `~/.hermes/.env` |
| `slack` | `SLACK_BOT_TOKEN` in `~/.hermes/.env` |
| `whatsapp` | WhatsApp gateway configured |
| `signal` | Signal gateway configured |
| `matrix` | Matrix homeserver configured |
| `email` | SMTP configured in `config.yaml` |
| `sms` | SMS provider configured |
| `local` | Write access to `~/.hermes/cron/output/` |
| `origin` | Delivers to the chat where the job was created |

Other supported platforms include `mattermost`, `homeassistant`, `dingtalk`, `feishu`, `wecom`, `weixin`, `bluebubbles`, `qqbot`, and `webhook`. You can also target a specific chat with `platform:chat_id` syntax (e.g., `telegram:-1001234567890`).

If delivery fails, the job still runs — it just won't send anywhere. Check `hermes cron list` for updated `last_error` field (if available).

### Check 2: Check `[SILENT]` usage

If your cron job produces no output or the agent responds with `[SILENT]`, delivery is suppressed. This is intentional for monitoring jobs — but make sure your prompt isn't accidentally suppressing everything.

A prompt that says "respond with [SILENT] if nothing changed" will silently swallow non-empty responses too. Check your conditional logic.

### Check 3: Platform token permissions

Each messaging platform bot needs specific permissions to receive messages. If delivery silently fails:

- **Telegram**: Bot must be an admin in the target group/channel
- **Discord**: Bot must have permission to send in the target channel
- **Slack**: Bot must be added to the workspace and have `chat:write` scope

### Check 4: Response wrapping

By default, cron responses are wrapped with a header and footer (`cron.wrap_response: true` in `config.yaml`). Some platforms or integrations may not handle this well. To disable:

```yaml
cron:
  wrap_response: false
```

---

## Skill Loading Failures

### Check 1: Verify skills are installed

```bash
hermes skills list
```

Skills must be installed before they can be attached to cron jobs. If a skill is missing, install it first with `hermes skills install <skill-name>` or via `/skills` in the CLI.

### Check 2: Check skill name vs. skill folder name

Skill names are case-sensitive and must match the installed skill's folder name. If your job specifies `ai-funding-daily-report` but the skill folder is `ai-funding-daily-report`, confirm the exact name from `hermes skills list`.

### Check 3: Skills that require interactive tools

Cron jobs run with the `cronjob`, `messaging`, and `clarify` toolsets disabled. This prevents recursive cron creation, direct message sending (delivery is handled by the scheduler), and interactive prompts. If a skill relies on these toolsets, it won't work in a cron context.

Check the skill's documentation to confirm it works in non-interactive (headless) mode.

### Check 4: Multi-skill ordering

When using multiple skills, they load in order. If Skill A depends on context from Skill B, make sure B loads first:

```bash
/cron add "0 9 * * *" "..." --skill context-skill --skill target-skill
```

In this example, `context-skill` loads before `target-skill`.

---

## Job Errors and Failures

### Check 1: Review recent job output

If a job ran and failed, you may see error context in:

1. The chat where the job delivers (if delivery succeeded)
2. `~/.hermes/logs/agent.log` for scheduler messages (or `errors.log` for warnings)
3. The job's `last_run` metadata via `hermes cron list`

### Check 2: Common error patterns

**"No such file or directory" for scripts**
The `script` path must be an absolute path (or relative to the Hermes config directory). Verify:
```bash
ls ~/.hermes/scripts/your-script.py   # Must exist
hermes cron edit <job_id> --script ~/.hermes/scripts/your-script.py
```

**"Skill not found" at job execution**
The skill must be installed on the machine running the scheduler. If you move between machines, skills don't automatically sync — reinstall them with `hermes skills install <skill-name>`.

**Job runs but delivers nothing**
Likely a delivery target issue (see Delivery Failures above) or a silently suppressed response (`[SILENT]`).

**Job hangs or times out**
The scheduler uses an inactivity-based timeout (default 600s, configurable via `HERMES_CRON_TIMEOUT` env var, `0` for unlimited). The agent can run as long as it's actively calling tools — the timer only fires after sustained inactivity. Long-running jobs should use scripts to handle data collection and deliver only the result.

### Check 3: Lock contention

The scheduler uses file-based locking to prevent overlapping ticks. If two gateway instances are running (or a CLI session conflicts with a gateway), jobs may be delayed or skipped.

Kill duplicate gateway processes:
```bash
ps aux | grep hermes
# Kill duplicate processes, keep only one
```

### Check 4: Permissions on jobs.json

Jobs are stored in `~/.hermes/cron/jobs.json`. If this file is not readable/writable by your user, the scheduler will fail silently:

```bash
ls -la ~/.hermes/cron/jobs.json
chmod 600 ~/.hermes/cron/jobs.json   # Your user should own it
```

---

## Performance Issues

### Slow job startup

Each cron job creates a fresh AIAgent session, which may involve provider authentication and model loading. For time-sensitive schedules, add buffer time (e.g., `0 8 * * *` instead of `0 9 * * *`).

### Too many overlapping jobs

The scheduler executes jobs sequentially within each tick. If multiple jobs are due at the same time, they run one after another. Consider staggering schedules (e.g., `0 9 * * *` and `5 9 * * *` instead of both at `0 9 * * *`) to avoid delays.

### Large script output

Scripts that dump megabytes of output will slow down the agent and may hit token limits. Filter/summarize at the script level — emit only what the agent needs to reason about.

---

## Diagnostic Commands

```bash
hermes cron list                    # Show all jobs, states, next_run times
hermes cron run <job_id>            # Schedule for next tick (for testing)
hermes cron edit <job_id>           # Fix configuration issues
hermes logs                         # View recent Hermes logs
hermes skills list                  # Verify installed skills
```

---

## Getting More Help

If you've worked through this guide and the issue persists:

1. Run the job with `hermes cron run <job_id>` (fires on next gateway tick) and watch for errors in the chat output
2. Check `~/.hermes/logs/agent.log` for scheduler messages and `~/.hermes/logs/errors.log` for warnings
3. Open an issue at [github.com/NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) with:
   - The job ID and schedule
   - The delivery target
   - What you expected vs. what happened
   - Relevant error messages from the logs

---

*For the complete cron reference, see [Automate Anything with Cron](/docs/guides/automate-with-cron) and [Scheduled Tasks (Cron)](/docs/user-guide/features/cron).*