# Channels & Routines - push-based & scheduled agents

> **Channels** - דרכים לדחוף הודעות לsession רץ (webhooks, chat, CI). **Routines** - Claude שרץ ב-schedule על תשתית Anthropic.

**מקורות:** [channels](https://code.claude.com/docs/en/channels) · [channels-reference](https://code.claude.com/docs/en/channels-reference) · [routines](https://code.claude.com/docs/en/routines) · [scheduled-tasks](https://code.claude.com/docs/en/scheduled-tasks) · [desktop-scheduled-tasks](https://code.claude.com/docs/en/desktop-scheduled-tasks)

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>בדרך כלל - <em>אתה</em> פונה לClaude. הקובץ הזה על שני דברים הפוכים:</p>

<p>🔔 <strong>Channels</strong>: Claude מקשיב לאירועים מבחוץ. למשל: ה-CI נכשל → הודעה נשלחת אוטומטית ל-Claude → הוא מתחיל לחקור למה. כמו שיש לך שרת שמקבל התראות - רק ש-Claude בצד השני.</p>

<p>⏰ <strong>Routines</strong>: Claude עובד <strong>לפי לוח זמנים</strong>, בלי שתפעיל אותו. "כל בוקר ב-9 תסקור את ה-PRs הפתוחים ותכתוב סיכום ב-Slack" - וזה קורה כל יום. המחשב שלך יכול להיות כבוי - זה רץ בענן של Anthropic.</p>

<p>בונוס: <code>/loop</code> - Claude מריץ את עצמו שוב ושוב בתוך session (למשל "בדוק כל 30 שניות אם ה-build הסתיים").</p>
</div>

---

## Channels - הזרם הפוך

Normal flow: אתה → Claude → tools. **Channels:** ___event חיצוני___ → Claude.

Use cases:
- CI נכשל → push לClaude ל-session הרץ
- Sentry alert → Claude מתחיל לדבג
- Slack mention → Claude מקבל את ההודעה
- Webhook → Claude מגיב ל-real-world event

### איך זה עובד

Channel = **MCP server עם capability מיוחדת** שיכולה לשלוח notifications ל-session.

```bash
claude --channels telegram,webhooks
```

ה-channels נמצאים בקונפיג כ-MCP servers עם `channel: true`.

---

## Channel contract

### Server declares capability

```json
{
  "capabilities": {
    "experimental": {
      "claude-code-channels": {
        "version": "1.0"
      }
    }
  }
}
```

### Notification events

השרת שולח:
```json
{
  "method": "notifications/message",
  "params": {
    "content": "Build failed: TypeError in auth.ts:42",
    "senderId": "ci-system",
    "metadata": {"url": "https://ci/build/1234"}
  }
}
```

Claude רואה את ההודעה בcontext, יכול להגיב.

### Reply tools

השרת חושף tools לClaude להגיב:

```
mcp__telegram__reply
mcp__slack__send_message
```

Claude יכול לשלוח הודעה בחזרה דרך ה-channel.

---

## Built-in channel plugins

- **Telegram** - bot מדבר עם Claude
- **Discord** - server channels
- **iMessage** - native macOS
- **Webhooks** - custom HTTP

התקנה:
```bash
claude plugin install claude-code-channels:telegram
```

### Configuration

כל channel plugin עם setup משלו. Telegram:

```bash
export TELEGRAM_BOT_TOKEN=...
export TELEGRAM_CHAT_ID=...
claude --channels telegram
```

---

## `allowedChannelPlugins` - managed

```json
// managed settings
{
  "channelsEnabled": true,
  "allowedChannelPlugins": ["telegram", "slack"]
}
```

**Team/Enterprise only.** Default: allowlist של Anthropic בלבד.

---

## Permission relay - security

כש-channel משולב, יש risk: הודעה ב-Telegram יכולה להכיל prompt injection. Claude רואה כ-user message.

הגנות:
- **Sender gating** - channel מציין מי שלח, Claude יודע מה לאמת.
- **Permission relay** - Claude מבקש אישור דרך אותו channel. user מאשר/דוחה בTelegram.
- **Rate limiting** ב-channel plugin level.
- **Scope restriction** - אסור ל-channel לתת לClaude tools חדשים.

---

## Routines - Claude בענן על schedule

Routines = **scheduled agents** שרצים על תשתית מנוהלת של Anthropic. לא על המחשב שלך.

### יצירה

**מה-CLI:**
```bash
/schedule
```
Dialog אינטראקטיבי - prompt, schedule (cron), triggers.

**מה-web:**
[claude.ai/code](https://claude.ai/code) → Routines section.

### Triggers

| Trigger | מתי רץ |
|---------|--------|
| `cron` | time-based - `0 9 * * MON` |
| `api` | HTTP POST לURL ייעודי |
| `github` | event ב-GitHub repo (PR, issue, push) |
| `manual` | trigger ידני |

### דוגמה: Morning PR review

```yaml
name: morning-pr-review
schedule:
  type: cron
  expression: "0 9 * * 1-5"   # Mon-Fri 9am
prompt: |
  Review open PRs in myorg/myrepo.
  Flag any that:
  - Haven't been updated in 3+ days
  - Have test failures
  - Have unresolved review comments
  Post summary to #engineering Slack.
tools:
  - mcp__github__*
  - mcp__slack__*
```

### Use cases

- **Morning triage** - PRs, issues, alerts
- **Overnight CI analysis** - summarize failures
- **Weekly dependency audit** - npm outdated, security advisories
- **Docs sync after merges** - update README on main branch updates
- **Customer support triage** - auto-categorize tickets

### Cost model

Routines = **metered** - משלמים על runs. Unlike CLI sessions (שמתעסקים ברסיוס שלך), routines ב-Anthropic infrastructure.

---

## `/loop` - scheduled בתוך session

Routines ב-Anthropic infrastructure, `/loop` ב-terminal שלך, בתוך session:

```
/loop 5m /check-build-status
/loop 30s /poll-ci
/loop /foo     # dynamic - Claude decides pacing
```

- `5m` - כל 5 דקות
- בלי interval - Claude בוחר תזמון עצמאי עם `ScheduleWakeup` tool

### Dynamic pacing

Claude משתמש ב-`ScheduleWakeup` tool:
- `delaySeconds: 60-3600`
- מחליט בכל iteration כמה לחכות
- prompt cache (5 min) - החלטות של <270s או >1200s יעילות יותר

### Use cases

- Polling for long build
- Monitoring status page
- Babysitting CI pipeline

---

## Desktop scheduled tasks

Claude Code Desktop מאפשר tasks scheduled על המחשב שלך:

- רץ לוקלית - גישה לקבצים מקומיים
- Automation בלי Anthropic infrastructure
- Task Scheduler/cron under the hood

Setup: Desktop app → Settings → Scheduled Tasks.

---

## מתי מה

| תרחיש | כלי |
|-------|-----|
| Build status בתוך session | `/loop` |
| Daily PR review | Routine (cloud) |
| Automation שדורשת קבצים מקומיים | Desktop scheduled task |
| Trigger על GitHub event | Routine |
| Poll every 30s for a minute | `/loop 30s` |
| Monthly security audit | Routine |
| React to Slack mention | Channel (Slack plugin) |

---

## המשך → [advanced_features.md](#/deep/advanced)
