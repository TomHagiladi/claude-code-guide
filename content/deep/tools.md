# Tools Reference - כל ה-tools המובנים

> Reference מלא של tools שClaude משתמש בהם. שמות, פרמטרים, דרישות permissions.

**מקור:** [tools-reference](https://code.claude.com/docs/en/tools-reference)

<div class="plain-language">
<h4>במילים פשוטות</h4>

<p>Claude לא באמת "עורך קבצים". הוא משתמש ב<strong>כלים</strong> - פונקציות מוכנות עם שמות. למשל: <code>Read</code> לקריאת קובץ, <code>Edit</code> לעריכה, <code>Bash</code> להרצת פקודה, <code>Grep</code> לחיפוש בקוד.</p>

<p>הקובץ הזה מפרט <strong>את כל הכלים המובנים</strong> - מה כל אחד עושה, אילו פרמטרים הוא מקבל, ומתי להשתמש בו.</p>

<p>למה זה חשוב? כי <strong>כל Rule של הרשאות בנוי על שמות הכלים</strong>. רוצה לחסום ריצת פקודות shell? אתה כותב Rule על הכלי <code>Bash</code>. רוצה לוודא שClaude לא קורא קבצי <code>.env</code>? Rule על <code>Read</code>.</p>

<p>גם שימושי להבין <strong>מה Claude בוחר מתי</strong>: למה הוא מריץ <code>Grep</code> במקום <code>Bash(grep ...)</code>? כי הכלי הייעודי יעיל יותר.</p>
</div>

---

## File operations

### Read

Read קובץ מה-filesystem.

**Input:**
- `file_path: string` - absolute path (חובה)
- `offset: number` - starting line
- `limit: number` - lines to read
- `pages: string` - PDF only (e.g., `"1-5"`)

**Permissions:** `Read(pattern)` לcontrol.

**Gotchas:**
- PDFs > 10 pages - חובה `pages`. max 20 per call.
- Jupyter notebooks - מחזיר cells + outputs.
- Images - מוצגים ויזואלית.
- Large text files - auto-truncated.

---

### Edit

עריכה precise של קובץ קיים.

**Input:**
- `file_path: string` - absolute
- `old_string: string` - בדיוק מה שיש בקובץ
- `new_string: string` - מחליף
- `replace_all: bool` - default false

**Permissions:** `Edit(pattern)` / `Write` (חלק מהמערכת).

**Gotchas:**
- חייב Read את הקובץ לפני Edit
- `old_string` **חייב להיות ייחודי** בקובץ, אלא אם `replace_all: true`
- preserves indentation - העתק עם tabs/spaces exact

---

### Write

יצירת קובץ חדש או rewrite מלא.

**Input:**
- `file_path: string`
- `content: string`

**Permissions:** `Write(pattern)`.

**Best practice:** תעדיף `Edit` לעריכת קובץ קיים - חוסך context (שולח רק diff).

---

### NotebookEdit

עריכת Jupyter notebook cell ספציפי.

---

## Search

### Glob

מציאת קבצים לפי pattern.

**Input:**
- `pattern: string` - glob (`"**/*.ts"`)
- `path: string` - cwd default

**Returns:** list of paths, sorted by mtime.

---

### Grep

חיפוש תוכן (ripgrep-based).

**Input:**
- `pattern: string` - regex
- `path: string` - optional
- `glob: string` - filter by pattern
- `type: string` - file type (`js`, `py`)
- `output_mode: "content" | "files_with_matches" | "count"`
- `-i`, `-n`, `-A <N>`, `-B <N>`, `-C <N>`, `head_limit`, `multiline`

**Best practices:**
- Default: `files_with_matches` - מהיר
- `content` עם context - רק כשצריך
- literal braces need escaping - `interface\{\}` ל-Go

---

## Execution

### Bash

הרצת shell command.

**Input:**
- `command: string`
- `description: string` - חובה, הסבר קצר
- `timeout: number` - ms, max 600000
- `run_in_background: bool`
- `dangerouslyDisableSandbox: bool`

**Permissions:** `Bash(pattern)` - ראה [permissions.md](#/deep/permissions).

**Avoid:** `find`, `grep`, `cat`, `head`, `tail`, `sed`, `awk`, `echo` - יש tools ייעודיים.

**Working directory persists** בין commands באותו session. Shell state (env vars) **לא**.

---

### Monitor

מעקב אחר תהליך background - כל שורת stdout = notification.

לשימוש עם `run_in_background: true` ל-streaming logs.

---

## Web

### WebSearch

חיפוש ברשת.

**Input:**
- `query: string`
- `allowed_domains: list[str]` - whitelist
- `blocked_domains: list[str]` - blocklist

**Permissions:** `WebSearch`.

---

### WebFetch

הבאת URL + AI summarization.

**Input:**
- `url: string`
- `prompt: string` - what to extract

**Permissions:** `WebFetch(domain:example.com)`.

**Behavior:**
- HTTP → auto-upgrade HTTPS
- Redirects - מציג URL החדש, צריך קריאה שנייה
- GitHub URLs - תעדיף `gh` CLI

---

## Interaction

### AskUserQuestion

Claude שואל user שאלה עם multiple choice.

**Input:**
- `question: string`
- `options: list[str]`

**Flow:** Claude ממתין לתשובה לפני המשך.

---

### TodoWrite

Claude מנהל todo list של המשימה.

**Input:**
- `todos: list[{content, status, activeForm}]`
- `status: "pending" | "in_progress" | "completed"`

**Only one in_progress ברגע נתון.** מומלץ לשימוש:
- משימות עם 3+ steps
- משימות עם multiple deliverables
- כשhuser מבקש רשימה

---

## Orchestration

### Agent (Task)

Spawn subagent.

**Input:**
- `description: string` - 3-5 word
- `prompt: string` - self-contained task
- `subagent_type: string` - `general-purpose`, `Explore`, `Plan`, custom
- `isolation: "worktree"` - optional
- `run_in_background: bool`
- `model: "opus" | "sonnet" | "haiku"`

**Built-in types:**
- `Explore` - read-only codebase research
- `Plan` - plan mode research
- `general-purpose` - flexible default

**גם custom agents** - שמות מ-`.claude/agents/`, `~/.claude/agents/`, plugins.

---

### ScheduleWakeup

ב-`/loop` dynamic mode. Claude מזמין את עצמו חזרה אחרי N שניות.

**Input:**
- `delaySeconds: number` - [60, 3600]
- `reason: string` - telemetry
- `prompt: string` - the /loop input

**עדיף 60-270s** (cache warm) או 1200-3600s (cost-effective). הימנע מ-300-1200 (worst of both).

---

## Meta

### ToolSearch

חיפוש deferred tools (MCP).

**Input:**
- `query: string` - "select:ToolName", או keywords
- `max_results: number`

Returns: full schemas של tools matching.

---

### Skill

הפעלת skill ב-main flow.

**Input:**
- `skill: string` - שם ה-skill
- `args: string` - optional arguments

---

## Plan mode controls

### ExitPlanMode

אחרי אישור המשתמש - יוצא מ-plan mode ומתחיל execution.

---

## Worktree

### EnterWorktree / ExitWorktree

הפעלת/יציאה מ-worktree isolation.

---

## Permissions per tool - default

| Tool | Approval required? |
|------|---------------------|
| Read, Grep, Glob | ❌ |
| WebSearch, WebFetch | ❌ (אבל rules של domain apply) |
| Bash | ✅ |
| Edit, Write | ✅ |
| Agent | תלוי ב-subagent |
| Skill | תלוי ב-skill |
| MCP tools | תלוי ברצף הרשאות |

---

## Tool selection - חוקים

1. **Dedicated tool > Bash** - יש Glob? לא `find`. יש Grep? לא `rg`. יש Read? לא `cat`.

2. **Parallel when independent** - קריאה של 3 קבצים שלא תלויים זה בזה → single message, 3 Read calls בו-זמנית.

3. **Sequential when dependent** - "קרא את X, ואז לפי מה שיש בו, חפש Y" → sequential.

4. **Subagent for scale** - קריאה של 10+ קבצים → Explore subagent.

5. **Web tool hierarchy** - search → fetch → analyze.

---

## המשך → [settings.md](#/deep/settings) · [env_vars.md](#/deep/settings) · [sandboxing_security.md](#/deep/sandbox)
