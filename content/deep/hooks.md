# Hooks - אוטומציה דטרמיניסטית

> Hooks הם shell commands (או HTTP endpoints, או prompts) שרצים אוטומטית ב-lifecycle events. שלא כמו CLAUDE.md שהוא "הוראות שקלוד מנסה לעקוב אחריהן", hooks הם **קוד שרץ**. זו הדרך היחידה לאכוף התנהגות.

**מקור:** [hooks](https://code.claude.com/docs/en/hooks)

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>תחשוב על <strong>שעון מעורר</strong>. כשמגיע הזמן (האירוע) - הוא מצלצל ומפעיל משהו. <strong>Hooks הם בדיוק זה</strong>: אתה אומר "כל פעם שClaude עורך קובץ - תפעיל את prettier כדי לסדר את הקוד". וזה קורה אוטומטית, בלי שאתה צריך להזכיר לClaude.</p>

<p>בניגוד להוראות שאתה כותב לClaude (שהוא <em>מנסה</em> לעקוב אחריהן אבל לא מובטח), <strong>hooks הם קוד שמופעל</strong> - מובטח שיירוץ. זו הדרך היחידה <strong>לאכוף</strong> התנהגות.</p>

<p>שימושים נפוצים: סידור קוד אחרי כל עריכה, רישום פעולות בlog, חסימת פקודות מסוכנות (כמו <code>rm -rf</code>), שליחת הודעה כשClaude מסיים משימה.</p>

<p>הקובץ הזה מסביר את כל סוגי האירועים שאתה יכול "להתחבר" אליהם, איך לכתוב את הסקריפט, ואיך לבדוק שהכול עובד.</p>
</div>

---

## מודל מנטלי

```
Event happens in Claude Code
        ↓
Hooks configured for this event run
        ↓
Hook output may: modify behavior / block / add context / notify
        ↓
Claude continues (or stops, if blocked)
```

**Hook ≠ tool.** Tool = Claude בוחר מה להריץ. Hook = רץ אוטומטית ב-event מסוים.

---

## טבלת Events - רשימה מלאה

| Event | מתי רץ | יכול לחסום? |
|-------|---------|-------------|
| `SessionStart` | session חדש או resumed | ❌ |
| `UserPromptSubmit` | אחרי שהקלדת, לפני שClaude רואה | ✅ |
| `PreToolUse` | לפני שClaude מריץ tool | ✅ |
| `PermissionRequest` | לפני שמוצג דיאלוג הרשאה | ✅ |
| `PermissionDenied` | Auto mode דחה tool | ❌ |
| `PostToolUse` | אחרי שtool הצליח | ❌ |
| `PostToolUseFailure` | אחרי שtool נכשל | ❌ |
| `Notification` | התראה נשלחה | ❌ |
| `SubagentStart` | subagent נוצר | ❌ |
| `SubagentStop` | subagent סיים | ✅ |
| `TaskCreated` / `TaskCompleted` | agent team tasks | ✅ |
| `Stop` | Claude סיים תשובה | ✅ |
| `StopFailure` | turn נגמר ב-API error | ❌ |
| `TeammateIdle` | agent teammate idle | ✅ |
| `InstructionsLoaded` | CLAUDE.md / rule נטען | ❌ |
| `ConfigChange` | settings שונו | ✅ |
| `CwdChanged` | directory שונה | ❌ |
| `FileChanged` | קובץ שמעקב אחריו שונה | ❌ |
| `WorktreeCreate` / `WorktreeRemove` | worktree operations | ✅ / ❌ |
| `PreCompact` / `PostCompact` | לפני/אחרי compaction | ✅ / ❌ |
| `Elicitation` / `ElicitationResult` | MCP server מבקש input | ✅ |
| `SessionEnd` | session מסתיים | ❌ |

"יכול לחסום" = exit code 2 או `decision: block` מונע את הפעולה.

---

## סוגי handlers - ארבעה

### 1. Command hook (הכי נפוץ)

```json
{
  "type": "command",
  "command": "/path/to/script.sh",
  "shell": "bash",
  "timeout": 600,
  "async": false,
  "asyncRewake": false,
  "if": "Bash(git *)",
  "once": false,
  "statusMessage": "Checking..."
}
```

- **async: true** - רץ ברקע, לא חוסם.
- **asyncRewake: true** - רץ ברקע; אם exit 2 → Claude מתעורר עם stderr כ-system reminder.
- **`if`** - מתנה, מופיע רק ב-tool events: `Bash(git *)`, `Edit(*.ts)`.

### 2. HTTP hook

```json
{
  "type": "http",
  "url": "http://localhost:8080/hook",
  "headers": {"Authorization": "Bearer $MY_TOKEN"},
  "allowedEnvVars": ["MY_TOKEN"],
  "timeout": 30
}
```

- POST body = hook input JSON.
- 2xx + empty → success (exit 0).
- 2xx + JSON → הסכמה המלאה של output.
- 2xx + plain text → מתווסף כ-context.
- Non-2xx → non-blocking error.

### 3. Prompt hook

```json
{
  "type": "prompt",
  "prompt": "Is this safe?\n$ARGUMENTS",
  "model": "claude-opus-4-8",
  "timeout": 30
}
```

Claude עצמו מחליט yes/no. שימושי לבדיקות סמנטיות.

### 4. Agent hook

```json
{
  "type": "agent",
  "prompt": "Validate: $ARGUMENTS",
  "model": "claude-opus-4-8",
  "timeout": 60
}
```

Spawns subagent עם כלי read (Read, Grep, Glob). הכי חזק, הכי יקר.

---

## Exit codes

| Code | משמעות | מה קורה |
|------|---------|---------|
| `0` | הצלחה | JSON output מנותח אם יש |
| `2` | **חוסם** | stderr מוצג ל-Claude כ-error |
| `1` or other | non-blocking | שורה ראשונה של stderr ב-transcript |

**Exit code 2 הוא החשוב ביותר.** רק הוא חוסם.

---

## JSON I/O

### Input (common, כל event)

```json
{
  "session_id": "abc123",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/dir",
  "permission_mode": "default",
  "hook_event_name": "PreToolUse",
  "agent_id": "xyz",
  "agent_type": "Explore"
}
```

### Output (common)

```json
{
  "continue": true,
  "stopReason": "message",
  "suppressOutput": false,
  "systemMessage": "warning",
  "hookSpecificOutput": {
    "additionalContext": "text to add to Claude's context"
  }
}
```

**`additionalContext` = הדרך להזין מידע ל-Claude מה-hook.** exit 0 עם plain stdout לא נכנס ל-context, רק ל-debug log.

---

## PreToolUse - הדוגמה החשובה ביותר

```json
{
  "hookSpecificOutput": {
    "permissionDecision": "allow|deny|ask|defer",
    "permissionDecisionReason": "why",
    "updatedInput": { /* שינוי ה-tool input! */ },
    "additionalContext": "more info"
  }
}
```

אפשר **לשנות את ה-input לפני שtool רץ**. למשל: hook רואה `rm -rf /` → מחליף ל-`rm -rf ./temp`.

---

## Matcher syntax

```json
{
  "matcher": "*",                    // הכל (או השמט)
  "matcher": "Bash",                 // exact
  "matcher": "Edit|Write",           // OR
  "matcher": "mcp__.*",              // regex (יש תו לא-אלפאנומרי)
}
```

**Event-specific matchers:**

| Event | matcher matches |
|-------|----------------|
| Tool events | tool name |
| `SessionStart` | `startup`, `resume`, `clear`, `compact` |
| `SessionEnd` | `clear`, `resume`, `logout`, `prompt_input_exit` |
| `Notification` | `permission_prompt`, `idle_prompt`, `auth_success` |
| `SubagentStart/Stop` | agent type - `Explore`, `Plan`, או custom |
| `FileChanged` | filename: `.envrc\|.env` |
| `StopFailure` | `rate_limit`, `authentication_failed`, `billing_error` |
| `InstructionsLoaded` | `session_start`, `nested_traversal`, `path_glob_match` |

---

## מיקומים ו-scope

| מיקום | Scope | משותף |
|-------|-------|--------|
| `~/.claude/settings.json` | כל הפרויקטים | לא |
| `.claude/settings.json` | פרויקט | כן (commit) |
| `.claude/settings.local.json` | פרויקט | לא (.gitignore) |
| Plugin `hooks/hooks.json` | כש-plugin פעיל | כן |
| Skill/Agent frontmatter | lifetime של ה-skill/agent | כן |
| Managed policy | ארגון | כן (admin) |

### בתוך `settings.json`

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "scripts/validate-bash.sh",
            "timeout": 5
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "prettier --write $CLAUDE_TOOL_INPUT_FILE_PATH"
          }
        ]
      }
    ]
  },
  "disableAllHooks": false
}
```

---

## Env vars זמינים ב-script

| Var | משמעות |
|-----|---------|
| `$CLAUDE_PROJECT_DIR` | project root |
| `${CLAUDE_PLUGIN_ROOT}` | plugin install dir |
| `${CLAUDE_PLUGIN_DATA}` | plugin persistent data |
| `$CLAUDE_ENV_FILE` | קובץ להתמדת env vars (SessionStart, CwdChanged, FileChanged) |
| `$CLAUDE_CODE_REMOTE` | `"true"` ב-web environments |

---

## Hooks ב-skills ו-agents

ב-frontmatter של skill או subagent:

```yaml
---
name: deploy
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./validate-deploy.sh"
  Stop:
    - hooks:
        - type: command
          command: "./notify-team.sh"
---
```

- רצים **רק כש-skill/agent פעיל**.
- ב-subagent frontmatter, `Stop` → `SubagentStop` אוטומטית.

---

## Security - plugin hooks

**Plugin subagents לא יכולים להשתמש ב-hooks, mcpServers, או permissionMode** ב-frontmatter. הסיבה: plugin לא אמור להריץ shell שרירותי בלי אישור מפורש.

רוצה את זה? העתק את ה-agent ל-`.claude/agents/` או `~/.claude/agents/`.

---

## `/hooks` - תצוגה

פקודה בתוך Claude Code. מראה:
- כל event שיש לו hook
- matcher
- handler מלא
- מקור: `[User]`, `[Project]`, `[Local]`, `[Plugin]`, `[Session]`, `[Built-in]`

קריאה בלבד. לעריכה - settings.json.

---

## Patterns ברמת מומחה

### 1. Auto-format אחרי edit

```json
{
  "PostToolUse": [{
    "matcher": "Edit|Write",
    "hooks": [{
      "type": "command",
      "command": "prettier --write \"$CLAUDE_TOOL_INPUT_FILE_PATH\" 2>/dev/null; true"
    }]
  }]
}
```

### 2. חסימת rm destructive

```json
{
  "PreToolUse": [{
    "matcher": "Bash",
    "hooks": [{
      "type": "command",
      "command": "scripts/block-destructive.sh",
      "if": "Bash(rm *)"
    }]
  }]
}
```

`scripts/block-destructive.sh`:
```bash
#!/bin/bash
INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command')
if echo "$CMD" | grep -qE '(rm -rf|:(){:\|:&};:)'; then
  echo "Blocked: dangerous command" >&2
  exit 2
fi
exit 0
```

### 3. Inject context ב-SessionStart

```json
{
  "SessionStart": [{
    "hooks": [{
      "type": "command",
      "command": "scripts/gather-context.sh"
    }]
  }]
}
```

`gather-context.sh`:
```bash
#!/bin/bash
cat <<EOF
{
  "hookSpecificOutput": {
    "additionalContext": "Today's date: $(date +%F)\\nRecent commits: $(git log -5 --oneline)"
  }
}
EOF
```

### 4. Notification ב-Stop

```json
{
  "Stop": [{
    "hooks": [{
      "type": "command",
      "command": "terminal-notifier -title 'Claude' -message 'Done' 2>/dev/null; true"
    }]
  }]
}
```

### 5. Async background check

```json
{
  "PostToolUse": [{
    "matcher": "Edit",
    "hooks": [{
      "type": "command",
      "command": "npm run lint -- --fix",
      "async": true
    }]
  }]
}
```

לא חוסם. רץ ברקע, לא משפיע על Claude.

---

## Debugging hooks

1. **`/hooks`** - וודא ש-hook רשום.
2. **הפעל ידנית:** `echo '{"hook_event_name":"PreToolUse",...}' | bash script.sh`
3. **בדוק exit code:** `echo $?` אחרי ריצה.
4. **logs:** exit 0 stdout הולך ל-debug log, לא ל-Claude.
5. **Timeout:** default בד"כ 60s. long-running → `"timeout": 600` או `"async": true`.

---

## המשך → [skills.md](#/deep/skills)
