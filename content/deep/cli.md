# CLI Reference - `claude` command

> Reference מלא לכל פקודות ו-flags של CLI.

**מקור:** [cli-reference](https://code.claude.com/docs/en/cli-reference)

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>כשאתה מריץ <code>claude</code> בטרמינל, אפשר להוסיף <strong>flags</strong> - אופציות שמשנות איך Claude יתנהג. למשל <code>claude --model opus</code> לבקש את המודל החזק.</p>

<p>הקובץ הזה הוא <strong>רשימה מלאה של כל ה-flags</strong>: איזה מודל, איזה מצב הרשאה, האם להגביל את הכלים, האם להמשיך שיחה קודמת, ועוד עשרות אופציות.</p>

<p>בנוסף: <strong>פקודות sub-command</strong> (כמו <code>claude mcp add</code>, <code>claude plugin install</code>), ו-<strong>slash commands</strong> שאפשר להקליד בתוך session (<code>/help</code>, <code>/compact</code>, <code>/memory</code>).</p>

<p>זה לא קובץ לקריאה מהתחלה לסוף - זה רפרנס לחפש בו כשאתה צריך "איך מגדירים X".</p>
</div>

---

## Primary commands

| Command | תיאור |
|---------|-------|
| `claude` | session אינטראקטיבי |
| `claude "query"` | session עם prompt התחלתי |
| `claude -p "query"` | print mode - שאילתה, pipe, exit |
| `cat file \| claude -p "query"` | process piped input |
| `claude -c` / `--continue` | המשך לsession אחרון ב-cwd |
| `claude -r <id\|name>` / `--resume` | resume ספציפי |
| `claude update` | עדכון לגרסה אחרונה |
| `claude auth login` | sign in (`--email`, `--sso`, `--console`) |
| `claude auth logout` | sign out |
| `claude auth status` | authentication status |
| `claude agents` | list subagents |
| `claude setup-token` | OAuth token ל-CI |

---

## Subcommand groups

| Group | שימוש |
|-------|--------|
| `claude mcp add/list/remove/login/logout/add-json/add-from-claude-desktop/reset-project-choices` | MCP servers |
| `claude plugin install/update/uninstall/marketplace add` | Plugins |
| `claude auto-mode defaults/config/critique` | Auto-mode classifier |
| `claude remote-control` | start Remote Control server |

---

## Core flags

| Flag | משמעות |
|------|---------|
| `--model <name>` | bottled name (`sonnet`, `opus`) או full ID |
| `--effort <level>` | `low`, `medium`, `high`, `max` (Opus only) |
| `--permission-mode <mode>` | `default`, `acceptEdits`, `plan`, `auto`, `dontAsk`, `bypassPermissions` |
| `--enable-auto-mode` | **הוסר ב-v2.1.111** - השתמש ב-`--permission-mode auto` |
| `--verbose` | verbose logging |
| `-v` / `--version` | version number |
| `--name <n>` / `-n` | display name (resumable) |
| `--session-id <uuid>` | use specific session ID |

---

## Working directory & files

| Flag | משמעות |
|------|---------|
| `--add-dir <path>...` | additional working dirs |
| `--worktree <name>` / `-w` | git worktree isolation |
| `--plugin-dir <path>` | load local plugin (ניתן לחזרה) |

---

## Tools & permissions

| Flag | משמעות |
|------|---------|
| `--tools <list>` | **restrict** which built-ins available (`""` = none, `"default"` = all) |
| `--allowedTools <patterns>` | pre-approve without prompt |
| `--disallowedTools <patterns>` | remove from context |
| `--dangerously-skip-permissions` | === `--permission-mode bypassPermissions` |
| `--allow-dangerously-skip-permissions` | add bypass to Shift+Tab cycle without starting there |

---

## System prompt

| Flag | משמעות |
|------|---------|
| `--system-prompt <text>` | **replace** default |
| `--system-prompt-file <path>` | replace with file |
| `--append-system-prompt <text>` | **append** to default |
| `--append-system-prompt-file <path>` | append from file |

**Best practice:** תעדיף append - שומר על היכולות המובנות.

---

## Session management

| Flag | משמעות |
|------|---------|
| `--continue` / `-c` | most recent in cwd |
| `--resume <id\|name>` / `-r` | specific session (interactive picker אם בלי arg) |
| `--fork-session` | create new ID (עם resume/continue) |
| `--from-pr <number\|url>` | resume sessions linked ל-PR |
| `--teleport` | resume web session in terminal |
| `--cloud <task>` | create web session (הכינוי הישן `--remote` עדיין עובד) |
| `--remote-control` / `--rc` | interactive + Remote Control |

---

## Agents

| Flag | משמעות |
|------|---------|
| `--agent <name>` | main session uses agent's prompt + tools |
| `--agents <json>` | define subagents dynamically |
| `--teammate-mode <mode>` | `auto`, `in-process`, `tmux` |

---

## Init / maintenance

| Flag | משמעות |
|------|---------|
| `--init` | run init hooks + interactive |
| `--init-only` | init hooks, exit |
| `--maintenance` | run maintenance hooks |
| `--ide` | auto-connect to IDE |

---

## MCP

| Flag | משמעות |
|------|---------|
| `--mcp-config <file\|json>` | load MCP config |
| `--strict-mcp-config` | only use `--mcp-config`, ignore others |
| `--channels <list>` | listen for channel notifications |

> **חדש (v2.1.186, 22.6.2026):** `claude mcp login <name>` ו-`claude mcp logout <name>` מבצעים OAuth או מנקים credentials לשרת MCP מוגדר - ישירות מהשורת פקודה, בלי לפתוח את תפריט `/mcp` האינטראקטיבי. שימושי כששרת MCP מאבד את ה-auth שלו.

---

## Print mode (`-p` / `--print`) flags

רק במצב non-interactive.

| Flag | משמעות |
|------|---------|
| `--output-format <fmt>` | `text`, `json`, `stream-json` |
| `--input-format <fmt>` | `text`, `stream-json` |
| `--json-schema <schema>` | structured output validation |
| `--max-turns <N>` | limit agentic turns |
| `--max-budget-usd <N>` | cost cap |
| `--no-session-persistence` | don't save session |
| `--include-hook-events` | stream hook events (עם stream-json) |
| `--include-partial-messages` | partial streaming events |
| `--replay-user-messages` | re-emit user messages on stdout |
| `--fallback-model <name>` | auto-fallback when overloaded |
| `--permission-prompt-tool <tool>` | MCP tool handles permissions |
| `--exclude-dynamic-system-prompt-sections` | better cache reuse across machines |

---

## Chrome / browser

| Flag | משמעות |
|------|---------|
| `--chrome` | enable Chrome integration |
| `--no-chrome` | disable for this session |

---

## Debug

| Flag | משמעות |
|------|---------|
| `--debug [categories]` | debug mode - `"api,hooks"`, `"!statsig"` |
| `--debug-file <path>` | debug logs to specific file |

---

## Settings control

| Flag | משמעות |
|------|---------|
| `--settings <path\|json>` | load additional settings |
| `--setting-sources <list>` | `user,project,local` - which to load |

---

## Bare mode - `--bare`

Minimal startup: skips hooks, skills, plugins, MCP, auto memory, CLAUDE.md. Tools: Bash, Read, Edit.

```bash
claude --bare -p "Quick question"
```

Sets `CLAUDE_CODE_SIMPLE=1`. שימושי ל-scripts מהירים.

---

## Safe mode - `--safe-mode`

מצב פתרון תקלות (v2.1.169, 8.6.2026): מפעיל את Claude Code עם **כל ההתאמות האישיות מנוטרלות** - hooks, plugins, skills, MCP servers, וגם CLAUDE.md מותאם.

```bash
claude --safe-mode
```

השימוש: לבדוק אם באג מגיע מההגדרות שלך או מ-Claude עצמו. בניגוד ל-`--bare` (שמכוון ל-scripts מהירים ומגביל גם את הכלים ל-Bash/Read/Edit), ל-`--safe-mode` יש מטרה אחת - לבודד את הבעיה על ידי כיבוי כל מה שאתה הוספת.

---

## In-session slash commands

### Built-in

| Command | פעולה |
|---------|-------|
| `/help` | עזרה |
| `/clear` | session נקי |
| `/compact [focus]` | compact context |
| `/context` | usage דיאגרמה |
| `/cost` | עלות session |
| `/memory` | ניהול memory files |
| `/init` | generate CLAUDE.md |
| `/doctor` | diagnostics |
| `/model` | החלף מודל |
| `/permissions` | browse rules |
| `/hooks` | browse hooks |
| `/mcp` | MCP servers status |
| `/skills` | browse skills |
| `/agents` | manage subagents |
| `/plugin` | manage plugins |
| `/schedule` | create routine |
| `/team` | **לא קיים כפקודה** - צוותי סוכנים נוצרים בבקשה חופשית ("צור צוות סוכנים ל...") |
| `/rename <n>` | rename session |
| `/status` | session status |
| `/config key=value` | set any setting from the prompt (v2.1.181) |
| `/add-dir <path>` | add directory mid-session |
| `/cd <path>` | move session to NEW working dir, no cache rebuild (v2.1.169) |
| `/tui fullscreen` | flicker-free mode (במקום `/fullscreen` הישן) |
| `/voice` | toggle voice dictation |
| `/fast` | toggle fast mode |
| `/exit` | quit |

> **חדש (יוני 2026):**
>
> `/cd <path>` (v2.1.169, 8.6.2026) - מעביר את ה-session הנוכחי לתיקיית עבודה חדשה באמצע השיחה, **בלי לבנות מחדש את ה-prompt cache**. ה-CLAUDE.md של התיקייה החדשה מתווסף כהודעה (message) במקום להחליף את ה-system prompt, ואחסון ה-session עובר למיקום החדש. נוח לעבודה חוצת-פרויקטים בתוך session אחד.
>
> `/config key=value` (v2.1.181, 17.6.2026) - מגדיר כל setting ישירות מתוך ה-prompt (למשל `/config thinking=false`). עובד גם ב-print mode (`-p`) וגם ב-Remote Control.

### Bundled skills (slash commands)

| Command | פעולה |
|---------|-------|
| `/simplify` | refactor |
| `/batch` | bulk operations |
| `/debug` | debugging guide |
| `/loop [interval] [prompt]` | repeat prompt |
| `/claude-api` | Claude API dev help |
| `/commit` | git commit workflow |
| `/review-pr [num]` | PR review |
| `/autofix-pr [num]` | auto-fix PR |
| `/team-onboarding` | מדריך קליטה לצוות, שנוצר מהיסטוריית השימוש שלך ב-30 הימים האחרונים |
| `/ultraplan` | **הוסר** - השתמש במצב תכנון (`/plan`) |

### Bash prefix

```
!git status       # אתה מריץ, Claude רואה את התוצאה
```

### File mentions

```
@src/auth.ts     # attach file
@github:issue://123   # MCP resource
```

---

## Environment variables (מרכזי)

| Var | משמעות |
|-----|---------|
| `ANTHROPIC_API_KEY` | API key |
| `CLAUDE_CODE_USE_BEDROCK=1` | AWS Bedrock provider |
| `CLAUDE_CODE_USE_VERTEX=1` | Google Vertex |
| `CLAUDE_CODE_USE_FOUNDRY=1` | Azure |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` | disable auto memory |
| `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1` | load CLAUDE.md מ-additional dirs |
| `CLAUDE_CODE_SUBAGENT_MODEL` | override subagent model |
| `CLAUDE_CODE_SIMPLE=1` | bare mode |
| `CLAUDE_CODE_NEW_INIT=1` | interactive /init |
| `ENABLE_TOOL_SEARCH` | MCP tool search mode |
| `MAX_MCP_OUTPUT_TOKENS` | MCP output limit |
| `MCP_TIMEOUT` | server connect timeout (ms) |
| `MCP_CLIENT_SECRET` | OAuth client secret |
| `SLASH_COMMAND_TOOL_CHAR_BUDGET` | skill descriptions budget |
| `CLAUDE_CODE_DEBUG_LOGS_DIR` | debug logs dir |
| `CLAUDE_CODE_USE_POWERSHELL_TOOL=1` | PowerShell on Windows |
| `CLAUDE_REMOTE_CONTROL_SESSION_NAME_PREFIX` | RC naming prefix |
| `CLAUDE_PROJECT_DIR` | project root (in hooks) |
| `CLAUDE_PLUGIN_ROOT` | plugin install dir (in plugin scripts) |
| `CLAUDE_PLUGIN_DATA` | plugin persistent data |
| `CLAUDE_ENV_FILE` | hook env persistence |

רשימה מלאה: [env-vars](https://code.claude.com/docs/en/env-vars)

---

## Useful patterns

### One-shot scripted task

```bash
claude -p --output-format json --max-turns 5 "Analyze bug in auth.py" | jq .
```

### CI integration

```bash
export ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY
claude -p --no-session-persistence --max-budget-usd 2.00 "Review this PR" < pr.diff
```

### Parallel worktrees

```bash
claude -w feature-auth    # session 1
claude -w feature-api     # session 2, separate directory + worktree
```

### Headless review

```bash
git diff main | claude -p --system-prompt "You are a strict code reviewer" --output-format json
```

### Export session for migration

```bash
cp ~/.claude/projects/<project-hash>/<session-id>.jsonl ./backup/
```

---

## המשך → [tools_reference.md](#/deep/tools) · [settings.md](#/deep/settings)
